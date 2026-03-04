# Complete Zoezi Add-on Development Guide

**Version:** 1.0
**Last Updated:** 2026-01-13
**Purpose:** Comprehensive reference for building Zoezi platform add-ons/apps

---

## Table of Contents

1. [Overview](#overview)
2. [Project Structure](#project-structure)
3. [Zoezi API Integration](#zoezi-api-integration)
4. [Supabase Setup](#supabase-setup)
5. [Authentication & Security](#authentication--security)
6. [Frontend Components](#frontend-components)
7. [Backend Architecture](#backend-architecture)
8. [Common Patterns](#common-patterns)
9. [Deployment](#deployment)
10. [Checklist for New Projects](#checklist-for-new-projects)

---

## Overview

### What is a Zoezi Add-on?

A Zoezi add-on is an external application that extends the functionality of the Zoezi gym management platform. It typically consists of:

- **Backend API**: Node.js/Express server hosted separately (e.g., Replit)
- **Frontend Components**: Vue.js components embedded directly into Zoezi pages
- **Database**: Supabase for data persistence
- **Integration**: Communicates with Zoezi's API for user data, groups, and system features

### Key Characteristics

- **Multi-Tenant**: One backend serves multiple gym installations (filtered by hostname)
- **Embedded UI**: Frontend components run inside Zoezi's Vue.js application
- **Secure**: Session-based authentication verified through Zoezi's API
- **Stateless Backend**: Each request includes full authentication context

---

## Project Structure

### Recommended Directory Layout

```
your-addon-name/
├── index.js                    # Entry point
├── package.json                # Dependencies
├── .replit                     # Deployment config (if using Replit)
├── .env.example                # Environment variable template
├── README.md                   # Project documentation
├── uploads/                    # Temporary uploads (if needed)
├── docs/                       # Additional documentation
│   ├── ZOEZI_ADDON_GUIDE.md    # This guide
│   ├── ZOEZI_API_REFERENCE.md  # API endpoint reference
│   └── ZOEZI_QUICKSTART.md     # Quick start template
└── src/
    ├── server.js               # Express app configuration
    ├── middleware/
    │   └── auth.js             # Authentication middleware
    ├── routes/
    │   ├── settings.js         # Admin configuration endpoints
    │   ├── admin.js            # Management endpoints
    │   └── user.js             # Customer-facing endpoints
    └── services/
        ├── zoezi.js            # Zoezi API client
        ├── supabase.js         # Database operations
        └── [other-services].js # Additional services as needed
```

### Key Files Purpose

| File | Purpose |
|------|---------|
| `index.js` | Minimal entry point that starts the Express server |
| `src/server.js` | Main Express configuration, middleware, and route mounting |
| `src/middleware/auth.js` | Session verification and role-based access control |
| `src/services/zoezi.js` | All Zoezi API interactions |
| `src/services/supabase.js` | Database queries and operations |

---

## Zoezi API Integration

### Authentication Methods

Zoezi supports two authentication methods:

#### 1. API Key Authentication (Backend Operations)

Used for: Creating TODOs, managing statuses, creating resources

```javascript
const ZOEZI_API_KEY = process.env.ZOEZI_API_KEY;

async function zoeziApiPost(hostname, path, body) {
  const response = await fetch(`https://${hostname}${path}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Zoezi ${ZOEZI_API_KEY}`
    },
    body: JSON.stringify(body)
  });

  if (!response.ok) {
    throw new Error(`Zoezi API error: ${response.status}`);
  }

  return response.json();
}
```

#### 2. Session-Based Authentication (User Context)

Used for: Reading user data, managing groups, user-specific operations

```javascript
async function zoeziGet(hostname, sessionRaw, path) {
  const response = await fetch(`https://${hostname}${path}`, {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json',
      'Cookie': `session=${sessionRaw}`
    }
  });

  if (!response.ok) {
    throw new Error(`Zoezi API error: ${response.status}`);
  }

  return response.json();
}

async function zoeziPost(hostname, sessionRaw, path, body) {
  const response = await fetch(`https://${hostname}${path}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Cookie': `session=${sessionRaw}`
    },
    body: JSON.stringify(body)
  });

  if (!response.ok) {
    throw new Error(`Zoezi API error: ${response.status}`);
  }

  return response.json();
}
```

### Key API Endpoints

See [ZOEZI_API_REFERENCE.md](ZOEZI_API_REFERENCE.md) for complete endpoint documentation.

**Most Common Endpoints:**

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/api/memberapi/get/current` | GET | Session | Get logged-in user info |
| `/api/user/get?id={id}` | GET | Session | Get specific user |
| `/api/user/change` | POST | API Key | Update user data |
| `/api/member/change` | POST | API Key | Update member status/groups |
| `/api/usergroup/get/all` | GET | Session | List all groups |
| `/api/usergroup/change` | POST | Session | Update group membership |
| `/api/user/session/verify?hash={hash}&user_id={id}` | GET | None | Verify session hash |

### Important Integration Patterns

#### Pattern 1: Ensure Resource Exists

```javascript
async function ensureGroupExists(hostname, sessionRaw, groupName) {
  // Try to find existing group
  const groups = await getAllGroups(hostname, sessionRaw);
  let group = groups.find(g => g.name === groupName);

  // Create if doesn't exist
  if (!group) {
    const result = await zoeziPost(hostname, sessionRaw, '/api/usergroup/add', {
      name: groupName,
      change_permission: false,
      users: [],
      show: true
    });
    group = result.group;
  }

  return group;
}
```

#### Pattern 2: Update User Extra Fields (XF)

Zoezi allows storing custom data in "extra fields" (xf):

```javascript
async function updateUserXf(hostname, userId, fieldName, value) {
  // Get current user data
  const user = await zoeziApiPost(hostname, '/api/user/get', { id: userId });

  // Merge new XF value
  const updatedXf = {
    ...(user.xf || {}),
    [fieldName]: value
  };

  // Save back
  return zoeziApiPost(hostname, '/api/user/change', {
    id: userId,
    xf: updatedXf
  });
}
```

#### Pattern 3: Manage Group Membership

```javascript
async function addUserToGroup(hostname, sessionRaw, groupId, userId) {
  // Get current group
  const group = await zoeziGet(hostname, sessionRaw, `/api/usergroup/get?id=${groupId}`);

  // Check if already member
  if (group.users?.includes(userId)) {
    return { alreadyMember: true };
  }

  // Add user
  await zoeziPost(hostname, sessionRaw, '/api/usergroup/change', {
    id: groupId,
    users: [...(group.users || []), userId]
  });

  return { success: true };
}
```

---

## Supabase Setup

### Database Configuration

**Environment Variables:**
```bash
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-service-role-key
```

**Initialize Client:**
```javascript
const { createClient } = require('@supabase/supabase-js');

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_KEY
);

module.exports = supabase;
```

### Database Schema Patterns

#### Multi-Tenant Table Pattern

**Always include `hostname` for multi-tenancy:**

```sql
CREATE TABLE IF NOT EXISTS your_table_name (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  hostname TEXT NOT NULL,  -- Critical for multi-tenancy
  -- Your fields here
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index on hostname for performance
CREATE INDEX idx_your_table_hostname ON your_table_name(hostname);

-- Row Level Security (RLS)
ALTER TABLE your_table_name ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Service role full access" ON your_table_name
  FOR ALL USING (true);
```

#### Common Field Patterns

```sql
-- User reference
user_id INTEGER NOT NULL,  -- Zoezi user ID

-- Company/group reference
company_id UUID REFERENCES companies(id),
group_id INTEGER,  -- Zoezi group ID

-- Status tracking
status TEXT DEFAULT 'pending',  -- pending/approved/rejected
is_active BOOLEAN DEFAULT true,

-- Audit fields
created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
created_by INTEGER,  -- Zoezi user ID
updated_by INTEGER,  -- Zoezi user ID

-- Unique constraints (always include hostname)
UNIQUE(hostname, some_unique_field)
```

### Supabase Storage

**For file uploads, use Supabase Storage instead of local filesystem:**

```javascript
const BUCKET_NAME = 'your-bucket-name';

// Create bucket (run once on startup)
async function ensureBucket() {
  const { data: buckets } = await supabase.storage.listBuckets();
  const exists = buckets?.some(b => b.name === BUCKET_NAME);

  if (!exists) {
    await supabase.storage.createBucket(BUCKET_NAME, {
      public: false,
      fileSizeLimit: 5 * 1024 * 1024  // 5MB
    });
  }
}

// Upload file
async function uploadFile(fileBuffer, fileName, mimetype, hostname, userId) {
  const path = `${hostname}/${userId}/${Date.now()}_${fileName}`;

  const { data, error } = await supabase.storage
    .from(BUCKET_NAME)
    .upload(path, fileBuffer, {
      contentType: mimetype,
      upsert: false
    });

  if (error) throw error;
  return { path, url: data.path };
}

// Get signed URL (for private files)
async function getDownloadUrl(path, expiresIn = 3600) {
  const { data, error } = await supabase.storage
    .from(BUCKET_NAME)
    .createSignedUrl(path, expiresIn);

  if (error) throw error;
  return data.signedUrl;
}

// Delete file
async function deleteFile(path) {
  const { error } = await supabase.storage
    .from(BUCKET_NAME)
    .remove([path]);

  if (error) throw error;
}
```

---

## Authentication & Security

### Frontend: Session Extraction & Hashing

**In your Vue.js component:**

```javascript
export default {
  methods: {
    // Extract raw session from cookie
    getRawSession() {
      const cookies = document.cookie.split('; ');
      for (const c of cookies) {
        const [name, value] = c.split('=');
        if (name === 'session') return value;
      }
      return null;
    },

    // Hash session using SHA-256
    async hashSession(session) {
      if (!session) return null;

      const encoder = new TextEncoder();
      const data = encoder.encode(session);
      const hashBuffer = await crypto.subtle.digest('SHA-256', data);

      return Array.from(new Uint8Array(hashBuffer))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('');
    },

    // Initialize authentication
    async initAuth() {
      this.hostname = window.location.hostname;
      this.sessionRaw = this.getRawSession();
      this.sessionHash = await this.hashSession(this.sessionRaw);

      // Get current user from Zoezi
      const user = await window.$zoeziapi.get('/api/memberapi/get/current');
      this.userId = user.id;
      this.isStaff = user.staff || false;
    }
  }
};
```

### Backend: Session Verification

**Middleware for authentication:**

```javascript
// src/middleware/auth.js
const zoezi = require('../services/zoezi');

async function verifySession(hostname, userId, sessionHash) {
  const url = `https://${hostname}/api/user/session/verify?hash=${encodeURIComponent(sessionHash)}&user_id=${userId}`;
  const response = await fetch(url);
  const data = await response.json();
  return data.valid === true;
}

// Middleware for any authenticated user
async function authenticateUser(req, res, next) {
  try {
    const { hostname, user_id, session_hash, session_raw } = req.body;

    // Validate required fields
    if (!hostname || !user_id || !session_hash || !session_raw) {
      return res.status(400).json({
        success: false,
        error: 'Missing authentication fields'
      });
    }

    // Verify session with Zoezi
    const isValid = await verifySession(hostname, user_id, session_hash);
    if (!isValid) {
      return res.status(401).json({
        success: false,
        error: 'Invalid or expired session'
      });
    }

    // Attach to request for use in route handlers
    req.gymHostname = hostname;
    req.userId = user_id;
    req.sessionRaw = session_raw;

    next();
  } catch (err) {
    next(err);
  }
}

// Middleware for staff-only endpoints
async function authenticateStaff(req, res, next) {
  try {
    const { hostname, user_id, session_hash, session_raw } = req.body;

    // Validate required fields
    if (!hostname || !user_id || !session_hash || !session_raw) {
      return res.status(400).json({
        success: false,
        error: 'Missing authentication fields'
      });
    }

    // Verify session
    const isValid = await verifySession(hostname, user_id, session_hash);
    if (!isValid) {
      return res.status(401).json({
        success: false,
        error: 'Invalid or expired session'
      });
    }

    // Check staff status
    const user = await zoezi.getUser(hostname, session_raw, user_id);
    if (!user.staff) {
      return res.status(403).json({
        success: false,
        error: 'Staff access required'
      });
    }

    // Attach to request
    req.gymHostname = hostname;
    req.userId = user_id;
    req.sessionRaw = session_raw;
    req.user = user;

    next();
  } catch (err) {
    next(err);
  }
}

module.exports = { authenticateUser, authenticateStaff };
```

### Security Best Practices

1. **Never trust client input** - Always verify session on backend
2. **Multi-tenant isolation** - Always filter by hostname
3. **Origin checking** - Verify requests come from correct hostname
4. **HTTPS only** - Never use HTTP in production
5. **API key security** - Store API keys in environment variables, never in code
6. **File upload validation** - Check file types, sizes, and sanitize names
7. **SQL injection prevention** - Use Supabase parameterized queries (built-in)
8. **XSS prevention** - Sanitize user input before displaying

---

## Frontend Components

### Component Architecture

**Components run inside Zoezi's Vue.js application and have access to:**
- `window.$zoeziapi` - Zoezi API client
- `window.$store` - Vuex store (user state, login status)
- Standard Vue.js features

### Standard Component Template

```html
<template>
  <div class="component-root">
    <!-- Loading State -->
    <div v-if="loading" class="loading-container">
      <div class="spinner"></div>
      <p>Loading...</p>
    </div>

    <!-- Error State -->
    <div v-else-if="error" class="error-container">
      <p class="error-message">{{ error }}</p>
      <button @click="loadData">Retry</button>
    </div>

    <!-- Main Content -->
    <div v-else class="main-content">
      <!-- Your component UI here -->
    </div>
  </div>
</template>

<script>
export default {
  name: 'YourComponentName',

  data() {
    return {
      // Backend URL (UPDATE THIS AFTER DEPLOYMENT!)
      BACKEND_URL: 'https://zoezi-b-2-b-app.replit.app',

      // Authentication
      hostname: null,
      userId: null,
      sessionHash: null,
      sessionRaw: null,
      isStaff: false,

      // State
      loading: true,
      error: null,

      // Component-specific data
      data: null,
    };
  },

  async mounted() {
    await this.initAuth();
    await this.loadData();
  },

  methods: {
    // Initialize authentication
    async initAuth() {
      try {
        this.hostname = window.location.hostname;
        this.sessionRaw = this.getRawSession();
        this.sessionHash = await this.hashSession(this.sessionRaw);

        const user = await window.$zoeziapi.get('/api/memberapi/get/current');
        this.userId = user.id;
        this.isStaff = user.staff || false;
      } catch (err) {
        this.error = 'Failed to initialize authentication';
        console.error(err);
      }
    },

    // Get raw session from cookie
    getRawSession() {
      const cookies = document.cookie.split('; ');
      for (const c of cookies) {
        const [name, value] = c.split('=');
        if (name === 'session') return value;
      }
      return null;
    },

    // Hash session using SHA-256
    async hashSession(session) {
      if (!session) return null;
      const encoder = new TextEncoder();
      const data = encoder.encode(session);
      const hashBuffer = await crypto.subtle.digest('SHA-256', data);
      return Array.from(new Uint8Array(hashBuffer))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('');
    },

    // Make API call to backend
    async apiCall(endpoint, data = {}) {
      const response = await fetch(`${this.BACKEND_URL}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          hostname: this.hostname,
          user_id: this.userId,
          session_hash: this.sessionHash,
          session_raw: this.sessionRaw,
          ...data
        })
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return response.json();
    },

    // Load data from backend
    async loadData() {
      this.loading = true;
      this.error = null;

      try {
        const result = await this.apiCall('/api/your-endpoint');

        if (result.success) {
          this.data = result.data;
        } else {
          this.error = result.error || 'Failed to load data';
        }
      } catch (err) {
        this.error = err.message;
        console.error(err);
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>

<style scoped>
.component-root {
  padding: 20px;
}

.loading-container {
  text-align: center;
  padding: 40px;
}

.spinner {
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin: 0 auto 20px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.error-container {
  background-color: #fee;
  border: 1px solid #fcc;
  border-radius: 4px;
  padding: 20px;
  text-align: center;
}

.error-message {
  color: #c33;
  margin-bottom: 10px;
}
</style>
```

### Component Types

**1. Settings Component** (Staff Only)
- Configure add-on settings
- Manage system-wide configurations
- Protected by staff authentication

**2. Admin/Management Component** (Staff Only)
- Manage entities (companies, users, etc.)
- Perform administrative actions
- CRUD operations

**3. User Component** (Customer-Facing)
- Self-service features
- View personal data
- Submit forms

**4. Auto/Background Component** (Invisible)
- Runs automatically on specific pages
- No UI or minimal UI
- Monitors page state and performs actions

---

## Backend Architecture

### Express Server Setup

**src/server.js:**

```javascript
const express = require('express');
const cors = require('cors');

const app = express();

// Middleware
app.use(cors());  // Allow all origins (components come from various gym domains)
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/settings', require('./routes/settings'));
app.use('/api/admin', require('./routes/admin'));
app.use('/api/user', require('./routes/user'));

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
  res.status(404).json({
    success: false,
    error: 'Endpoint not found'
  });
});

module.exports = app;
```

**index.js:**

```javascript
const app = require('./src/server');

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Route Structure

**Standard route file pattern:**

```javascript
// src/routes/example.js
const express = require('express');
const router = express.Router();
const { authenticateUser, authenticateStaff } = require('../middleware/auth');
const supabase = require('../services/supabase');
const zoezi = require('../services/zoezi');

// Staff-only endpoint
router.post('/admin-action', authenticateStaff, async (req, res, next) => {
  try {
    const { param1, param2 } = req.body;

    // Access authenticated data from middleware
    const hostname = req.gymHostname;
    const userId = req.userId;
    const sessionRaw = req.sessionRaw;

    // Database operation
    const { data, error } = await supabase
      .from('your_table')
      .select('*')
      .eq('hostname', hostname);

    if (error) throw error;

    // Zoezi API operation
    const zoeziData = await zoezi.someFunction(hostname, sessionRaw, param1);

    res.json({ success: true, data });
  } catch (err) {
    next(err);
  }
});

// User endpoint
router.post('/user-action', authenticateUser, async (req, res, next) => {
  try {
    // Implementation
    res.json({ success: true, data: {} });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

---

## Common Patterns

### Multi-Tenant Query Pattern

**Always filter by hostname:**

```javascript
// Get records for specific gym
async function getRecords(hostname) {
  const { data, error } = await supabase
    .from('table_name')
    .select('*')
    .eq('hostname', hostname)
    .order('created_at', { ascending: false });

  if (error) throw error;
  return data || [];
}

// Get single record with safety check
async function getRecord(hostname, id) {
  const { data, error } = await supabase
    .from('table_name')
    .select('*')
    .eq('hostname', hostname)
    .eq('id', id)
    .single();

  if (error) throw error;
  return data;
}
```

### Upsert Pattern

**Create or update in one operation:**

```javascript
async function saveSettings(hostname, settings) {
  const { data, error } = await supabase
    .from('settings')
    .upsert({
      hostname,
      ...settings,
      updated_at: new Date().toISOString()
    }, {
      onConflict: 'hostname',  // Unique constraint column
      returning: 'representation'
    })
    .select()
    .single();

  if (error) throw error;
  return data;
}
```

### File Upload Pattern

**Using Multer:**

```javascript
const multer = require('multer');
const path = require('path');
const { v4: uuidv4 } = require('uuid');

// Configure Multer
const upload = multer({
  dest: path.join(__dirname, '../../uploads/temp'),
  limits: {
    fileSize: 5 * 1024 * 1024  // 5MB max
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = [
      'application/pdf',
      'image/jpeg',
      'image/png',
      'image/webp'
    ];

    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type. Allowed: PDF, JPEG, PNG, WEBP'));
    }
  }
});

// Route with file upload
router.post('/upload', upload.single('file'), async (req, res, next) => {
  try {
    // Auth fields come from form body
    const { hostname, user_id, session_hash, session_raw } = req.body;

    // Verify session manually (since we can't use middleware with multer)
    const isValid = await verifySession(hostname, user_id, session_hash);
    if (!isValid) {
      return res.status(401).json({ success: false, error: 'Invalid session' });
    }

    // File is in req.file
    const file = req.file;

    // Upload to Supabase Storage
    const storedPath = `${hostname}/${user_id}/${Date.now()}_${uuidv4()}${path.extname(file.originalname)}`;

    const fileBuffer = fs.readFileSync(file.path);
    const { data, error } = await supabase.storage
      .from('your-bucket')
      .upload(storedPath, fileBuffer, {
        contentType: file.mimetype,
        upsert: false
      });

    if (error) throw error;

    // Clean up temp file
    fs.unlinkSync(file.path);

    // Save metadata to database
    const { data: record } = await supabase
      .from('uploads')
      .insert({
        hostname,
        user_id,
        original_filename: file.originalname,
        stored_path: storedPath,
        file_size: file.size,
        mime_type: file.mimetype
      })
      .select()
      .single();

    res.json({ success: true, data: record });
  } catch (err) {
    // Clean up temp file on error
    if (req.file) {
      fs.unlinkSync(req.file.path);
    }
    next(err);
  }
});
```

### Error Handling Pattern

**Consistent error responses:**

```javascript
// In route handlers
try {
  // ... logic
  res.json({ success: true, data });
} catch (err) {
  next(err);  // Pass to error middleware
}

// Central error middleware (in server.js)
app.use((err, req, res, next) => {
  console.error('Error:', err);

  // Send user-friendly error
  res.status(err.status || 500).json({
    success: false,
    error: err.message || 'Internal server error'
  });
});
```

---

## Deployment

### Environment Variables

**Required variables:**

```bash
# Supabase
SUPABASE_URL=https://xxxxxxxxxxxxx.supabase.co
SUPABASE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Zoezi
ZOEZI_API_KEY=your-api-key-here

# Server (optional)
PORT=5000
NODE_ENV=production
```

### Replit Deployment

**.replit file:**

```
entrypoint = "index.js"
modules = ["nodejs-22"]

[deployment]
run = ["node", "index.js"]
deploymentTarget = "autoscale"

[[ports]]
localPort = 5000
externalPort = 80
```

**Steps:**
1. Push code to Replit
2. Set environment secrets in Replit Secrets panel
3. Click "Deploy" button
4. Copy deployment URL (e.g., `https://your-app.replit.dev`)
5. Update `BACKEND_URL` in all frontend components

### Alternative Deployment (Railway, Render, etc.)

**Requirements:**
- Node.js 22+
- Public HTTPS endpoint
- Environment variables configured

**package.json scripts:**

```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  }
}
```

### Installing Components in Zoezi

1. Log in to Zoezi admin panel
2. Navigate to "Components" or "Custom Code" section
3. Create new component for each:
   - Settings Component (admin page)
   - Admin Component (admin page)
   - User Component (member area)
   - Auto Component (global or specific pages)
4. **CRITICAL**: Update `BACKEND_URL` in JavaScript to your deployment URL
5. Assign components to appropriate pages
6. Test thoroughly with real session data

---

## Checklist for New Projects

### Planning Phase

- [ ] Define add-on purpose and features
- [ ] Identify required Zoezi API endpoints
- [ ] Design database schema
- [ ] Plan component hierarchy (admin vs user)
- [ ] List required environment variables

### Setup Phase

- [ ] Create project directory structure
- [ ] Initialize `package.json` with dependencies
- [ ] Set up Express server (`index.js`, `src/server.js`)
- [ ] Create Supabase project and get credentials
- [ ] Get Zoezi API key from admin
- [ ] Set up environment variables

### Backend Development

- [ ] Create `src/services/zoezi.js` with API functions
- [ ] Create `src/services/supabase.js` client
- [ ] Implement `src/middleware/auth.js`
- [ ] Create route files for each feature area
- [ ] Implement all endpoints with proper error handling
- [ ] Test endpoints with Postman/curl

### Database Setup

- [ ] Create all tables with proper schemas
- [ ] Add indexes for performance (especially on `hostname`)
- [ ] Enable Row Level Security (RLS)
- [ ] Create storage buckets if needed
- [ ] Test queries and constraints

### Frontend Development

- [ ] Create Vue.js components for each feature
- [ ] Implement authentication helpers
- [ ] Add loading and error states
- [ ] Style components to match Zoezi design
- [ ] Test with real Zoezi session data

### Deployment

- [ ] Push code to hosting platform (Replit, Railway, etc.)
- [ ] Configure environment secrets
- [ ] Deploy and get public URL
- [ ] Update all component `BACKEND_URL` values
- [ ] Install components in Zoezi admin panel
- [ ] Assign components to correct pages

### Testing

- [ ] Test with multiple gym installations (different hostnames)
- [ ] Test as staff user (admin features)
- [ ] Test as regular user (customer features)
- [ ] Test file uploads (if applicable)
- [ ] Test error scenarios (invalid session, network errors, etc.)
- [ ] Verify multi-tenancy (data isolation between gyms)

### Documentation

- [ ] Update README with project-specific details
- [ ] Document API endpoints
- [ ] Create user guide for gym administrators
- [ ] Add code comments for complex logic

---

## Additional Resources

- **Zoezi API Reference**: [ZOEZI_API_REFERENCE.md](ZOEZI_API_REFERENCE.md)
- **Quick Start Template**: [ZOEZI_QUICKSTART.md](ZOEZI_QUICKSTART.md)
- **Supabase Docs**: https://supabase.com/docs
- **Express.js Docs**: https://expressjs.com
- **Vue.js Docs**: https://vuejs.org

---

## Support

For issues specific to this guide or Zoezi integration:
- Check existing code in `src/` for working examples
- Review Zoezi API responses for data structure
- Test authentication flow step-by-step
- Verify multi-tenancy filtering in all queries

---

**Last Updated:** 2026-01-13
**Guide Version:** 1.0
