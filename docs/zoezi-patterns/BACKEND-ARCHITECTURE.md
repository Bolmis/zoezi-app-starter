# Backend Architecture Patterns

Production-proven patterns for building the backend of a Zoezi addon. Based on real addons running in production.

---

## Architecture Overview

```
Zoezi Platform (Vue 2 components)
    |
    | HTTPS POST with { hostname, user_id, session_hash, session_raw, ...data }
    v
Express.js Backend (Replit / Railway / Render)
    |
    ├── middleware/auth.js    → Verifies session with Zoezi, checks staff/user role
    ├── routes/*.js           → Business logic endpoints
    ├── services/zoezi.js     → Zoezi API calls (groups, statuses, TODOs, etc.)
    ├── services/supabase.js  → Database operations (multi-tenant, hostname-filtered)
    └── services/storage.js   → File uploads to Supabase Storage
```

Every request from a Zoezi component includes authentication context. The backend verifies it, then operates on behalf of the user.

---

## Directory Structure

```
your-addon/
├── index.js                    # Entry point (minimal — just starts server)
├── package.json
├── .replit                     # Replit deployment config
├── .env.example                # Required environment variables
├── src/
│   ├── server.js               # Express app config, middleware, route mounting
│   ├── cron.js                 # Scheduled jobs (optional)
│   ├── middleware/
│   │   └── auth.js             # Session verification + role checks
│   ├── routes/
│   │   ├── settings.js         # Admin configuration endpoints
│   │   ├── user.js             # Customer-facing endpoints
│   │   ├── webhooks.js         # Zoezi webhook handlers
│   │   └── jobs.js             # Manual job trigger endpoints
│   └── services/
│       ├── supabase.js         # Database client + query functions
│       ├── zoezi.js            # Zoezi API wrapper
│       └── storage.js          # Supabase Storage operations
├── uploads/                    # Temp directory for file uploads
└── docs/                       # Documentation
```

---

## Entry Point: index.js

Keep this minimal. Its only job is to start the server and validate environment.

```javascript
const app = require('./src/server');

const PORT = process.env.PORT || 5000;

// Fail fast if critical env vars are missing
const required = ['SUPABASE_URL', 'SUPABASE_KEY', 'ZOEZI_API_KEY'];
for (const key of required) {
  if (!process.env[key]) {
    console.error(`Missing required environment variable: ${key}`);
    process.exit(1);
  }
}

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## Express Server: src/server.js

```javascript
const express = require('express');
const cors = require('cors');
const rateLimit = require('express-rate-limit');

const app = express();

// Middleware
app.use(cors());  // Allow all origins (components come from various gym domains)
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100                    // limit each IP to 100 requests per window
});
app.use('/api/', limiter);

// Routes
app.use('/api/settings', require('./routes/settings'));
app.use('/api/user', require('./routes/user'));
app.use('/api/webhooks', require('./routes/webhooks'));
app.use('/api/jobs', require('./routes/jobs'));

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Error handling middleware
app.use((err, req, res, next) => {
  console.error('Error:', err);
  res.status(500).json({
    success: false,
    error: err.message || 'Internal server error'
  });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({ success: false, error: 'Endpoint not found' });
});

module.exports = app;
```

---

## Authentication Middleware: src/middleware/auth.js

This is the most critical piece. Every addon needs this exact pattern.

```javascript
const crypto = require('crypto');
const zoezi = require('../services/zoezi');

/**
 * Validate hostname is an allowed Zoezi domain (SSRF prevention)
 * All Zoezi gym domains end with .zoezi.se
 */
function validateHostname(hostname) {
  if (!hostname || typeof hostname !== 'string') return false;
  // Production: all gyms are on *.zoezi.se
  // localhost allowed for development only
  return hostname === 'localhost'
    || hostname === 'zoezi.se'
    || hostname.endsWith('.zoezi.se');
}

/**
 * Verify session hash with Zoezi
 */
async function verifySession(hostname, userId, sessionHash) {
  try {
    const url = `https://${hostname}/api/user/session/verify?hash=${encodeURIComponent(sessionHash)}&user_id=${userId}`;
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 15000);
    const response = await fetch(url, { signal: controller.signal });
    clearTimeout(timeout);
    if (!response.ok) return false;
    const data = await response.json();
    return data.valid === true;
  } catch (err) {
    console.error('Session verify error:', err.message);
    return false;
  }
}

/**
 * Check request origin matches hostname (prevent cross-gym attacks)
 */
function checkOrigin(req, hostname) {
  const origin = req.headers.origin || req.headers.referer || '';
  if (!origin) return false;
  try {
    const url = new URL(origin);
    return url.hostname === hostname;
  } catch {
    return false;
  }
}

/**
 * Authenticate any logged-in user
 * Use for customer-facing endpoints
 */
async function authenticateUser(req, res, next) {
  const { hostname, user_id, session_hash } = req.body;

  if (!hostname || !user_id || !session_hash) {
    return res.status(400).json({ success: false, error: 'Missing authentication parameters' });
  }

  if (!validateHostname(hostname)) {
    return res.status(400).json({ success: false, error: 'Invalid hostname' });
  }

  if (!checkOrigin(req, hostname)) {
    return res.status(403).json({ success: false, error: 'Origin mismatch' });
  }

  const sessionValid = await verifySession(hostname, user_id, session_hash);
  if (!sessionValid) {
    return res.status(401).json({ success: false, error: 'Invalid session' });
  }

  req.gymHostname = hostname;
  req.userId = user_id;
  next();
}

/**
 * Authenticate staff-only users
 * Use for admin/settings endpoints
 */
async function authenticateStaff(req, res, next) {
  const { hostname, user_id, session_hash } = req.body;

  if (!hostname || !user_id || !session_hash) {
    return res.status(400).json({ success: false, error: 'Missing authentication parameters' });
  }

  if (!validateHostname(hostname)) {
    return res.status(400).json({ success: false, error: 'Invalid hostname' });
  }

  if (!checkOrigin(req, hostname)) {
    return res.status(403).json({ success: false, error: 'Origin mismatch' });
  }

  const sessionValid = await verifySession(hostname, user_id, session_hash);
  if (!sessionValid) {
    return res.status(401).json({ success: false, error: 'Invalid session' });
  }

  // Check staff status via Zoezi API
  try {
    const user = await zoezi.getUser(hostname, user_id);
    if (!user.staff) {
      return res.status(403).json({ success: false, error: 'Staff access required' });
    }
  } catch (err) {
    return res.status(403).json({ success: false, error: 'Could not verify staff status' });
  }

  req.gymHostname = hostname;
  req.userId = user_id;
  next();
}

/**
 * Authenticate internal/webhook requests
 * Uses WEBHOOK_SECRET with timing-safe comparison
 */
function authenticateInternal(req, res, next) {
  const secret = process.env.WEBHOOK_SECRET;
  if (!secret) {
    return res.status(500).json({ success: false, error: 'Server misconfigured' });
  }

  const provided = req.headers['x-webhook-secret'] || req.query.secret;
  if (!provided || typeof provided !== 'string') {
    return res.status(401).json({ success: false, error: 'Unauthorized' });
  }

  const secretBuf = Buffer.from(secret);
  const providedBuf = Buffer.from(provided);
  if (secretBuf.length !== providedBuf.length || !crypto.timingSafeEqual(secretBuf, providedBuf)) {
    return res.status(401).json({ success: false, error: 'Unauthorized' });
  }

  next();
}

module.exports = { authenticateUser, authenticateStaff, authenticateInternal, validateHostname, checkOrigin };
```

---

## Route Pattern: src/routes/example.js

```javascript
const express = require('express');
const router = express.Router();
const { authenticateUser, authenticateStaff } = require('../middleware/auth');
const supabase = require('../services/supabase');
const zoezi = require('../services/zoezi');

// Staff-only endpoint
router.post('/admin-action', authenticateStaff, async (req, res, next) => {
  try {
    const hostname = req.gymHostname;
    const { param1, param2 } = req.body;

    // Always filter by hostname for multi-tenancy
    const { data, error } = await supabase
      .from('your_table')
      .select('*')
      .eq('hostname', hostname);

    if (error) throw error;

    res.json({ success: true, data });
  } catch (err) {
    next(err);
  }
});

// User endpoint
router.post('/user-action', authenticateUser, async (req, res, next) => {
  try {
    const hostname = req.gymHostname;
    const userId = req.userId;

    // Query with both hostname AND user_id
    const { data, error } = await supabase
      .from('your_table')
      .select('*')
      .eq('hostname', hostname)
      .eq('user_id', userId);

    if (error) throw error;

    res.json({ success: true, data });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

---

## Webhook Handler: src/routes/webhooks.js

```javascript
const express = require('express');
const router = express.Router();
const { authenticateInternal } = require('../middleware/auth');
const supabase = require('../services/supabase');
const zoezi = require('../services/zoezi');

/**
 * Handle Zoezi webhook events
 * Configure in webhooks.json in your zoezi-component repo
 */
router.post('/event-updated', authenticateInternal, async (req, res, next) => {
  try {
    const events = req.body.events || [];

    for (const event of events) {
      const { action, id, clubid } = event;

      if (action === 'update') {
        // Process the event
        // clubid maps to a hostname — resolve it
        console.log(`Webhook: ${action} on ${id} for club ${clubid}`);
      }
    }

    res.json({ success: true });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

---

## Cron Jobs: src/cron.js

```javascript
const cron = require('node-cron');
const supabase = require('./services/supabase');
const zoezi = require('./services/zoezi');

/**
 * Initialize scheduled jobs
 */
function initCronJobs() {
  // Daily check at 03:00 UTC
  cron.schedule('0 3 * * *', async () => {
    console.log('[CRON] Running daily maintenance...');
    try {
      // Your scheduled logic here
      // Example: check for expired records, send reminders, etc.
    } catch (err) {
      console.error('[CRON] Error:', err.message);
    }
  });

  console.log('Cron jobs initialized');
}

module.exports = { initCronJobs };
```

Start cron in index.js:
```javascript
const { initCronJobs } = require('./src/cron');
initCronJobs();
```

---

## File Upload Pattern

```javascript
const multer = require('multer');
const path = require('path');
const fs = require('fs');
const { v4: uuidv4 } = require('uuid');

const upload = multer({
  dest: path.join(__dirname, '../../uploads/temp'),
  limits: { fileSize: 5 * 1024 * 1024 },  // 5MB
  fileFilter: (req, file, cb) => {
    const allowed = ['application/pdf', 'image/jpeg', 'image/png', 'image/webp'];
    cb(null, allowed.includes(file.mimetype));
  }
});

router.post('/upload', upload.single('file'), async (req, res, next) => {
  try {
    const { hostname, user_id, session_hash } = req.body;

    // Verify session manually (multer runs before middleware)
    const isValid = await verifySession(hostname, user_id, session_hash);
    if (!isValid) {
      return res.status(401).json({ success: false, error: 'Invalid session' });
    }

    const file = req.file;
    const storedName = `${uuidv4()}${path.extname(file.originalname)}`;

    // Upload to Supabase Storage
    const fileBuffer = fs.readFileSync(file.path);
    const { error } = await supabase.storage
      .from('your-bucket')
      .upload(storedName, fileBuffer, { contentType: file.mimetype });

    // Clean up temp file
    fs.unlinkSync(file.path);

    if (error) throw error;

    res.json({ success: true, filename: storedName });
  } catch (err) {
    if (req.file) fs.unlinkSync(req.file.path);
    next(err);
  }
});
```

---

## Environment Variables

**.env.example:**
```bash
# Supabase (required)
SUPABASE_URL=https://xxxxxxxxxxxxx.supabase.co
SUPABASE_KEY=your-service-role-key

# Zoezi (required)
ZOEZI_API_KEY=your-addon-api-key

# Webhooks (required if using webhooks)
WEBHOOK_SECRET=your-webhook-secret

# Server
PORT=5000
NODE_ENV=production
```

---

## Replit Deployment Config

**.replit:**
```toml
entrypoint = "index.js"
modules = ["nodejs-22"]

[deployment]
run = ["node", "index.js"]
deploymentTarget = "autoscale"

[[ports]]
localPort = 5000
externalPort = 80
```

---

## Dependencies

```json
{
  "dependencies": {
    "@supabase/supabase-js": "^2.87.1",
    "cors": "^2.8.5",
    "express": "^5.2.1",
    "express-rate-limit": "^8.2.1",
    "multer": "^2.0.2",
    "node-cron": "^4.2.1",
    "uuid": "^13.0.0"
  },
  "devDependencies": {
    "jest": "^30.2.0",
    "supertest": "^7.2.2"
  }
}
```
