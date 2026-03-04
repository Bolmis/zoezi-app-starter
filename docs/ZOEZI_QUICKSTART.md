# Zoezi Add-on Quick Start Template

**Purpose:** Minimal boilerplate to get started with a new Zoezi add-on in under 30 minutes.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Setup](#project-setup)
3. [Minimal Backend](#minimal-backend)
4. [Minimal Frontend](#minimal-frontend)
5. [Database Setup](#database-setup)
6. [Testing](#testing)
7. [Deployment](#deployment)
8. [Next Steps](#next-steps)

---

## Prerequisites

- Node.js 22+ installed
- Supabase account (free tier works)
- Zoezi API key from gym admin
- Basic knowledge of Express.js and Vue.js

---

## Project Setup

### 1. Create Project Directory

```bash
mkdir my-zoezi-addon
cd my-zoezi-addon
npm init -y
```

### 2. Install Dependencies

```bash
npm install express cors @supabase/supabase-js
npm install --save-dev nodemon
```

### 3. Create Directory Structure

```bash
mkdir -p src/middleware src/routes src/services
touch index.js .env .env.example
touch src/server.js src/middleware/auth.js src/routes/user.js
touch src/services/zoezi.js src/services/supabase.js
```

### 4. Configure package.json

```json
{
  "name": "my-zoezi-addon",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "@supabase/supabase-js": "^2.87.1",
    "cors": "^2.8.5",
    "express": "^5.2.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.0"
  }
}
```

### 5. Environment Variables

**.env.example:**
```bash
# Supabase
SUPABASE_URL=https://xxxxxxxxxxxxx.supabase.co
SUPABASE_KEY=your-service-role-key

# Zoezi
ZOEZI_API_KEY=your-api-key

# Server
PORT=5000
NODE_ENV=development
```

**Copy and fill in real values:**
```bash
cp .env.example .env
# Edit .env with your actual credentials
```

---

## Minimal Backend

### Entry Point: index.js

```javascript
require('dotenv').config();
const app = require('./src/server');

const PORT = process.env.PORT || 5000;

app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
});
```

### Express Server: src/server.js

```javascript
const express = require('express');
const cors = require('cors');

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Routes
app.use('/api/user', require('./routes/user'));

// Health check
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString()
  });
});

// Error handling
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

### Zoezi Service: src/services/zoezi.js

```javascript
const ZOEZI_API_KEY = process.env.ZOEZI_API_KEY;

// Session authenticated GET
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

// API Key authenticated POST
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

// Get current user
async function getCurrentUser(hostname, sessionRaw) {
  return zoeziGet(hostname, sessionRaw, '/api/memberapi/get/current');
}

// Verify session hash
async function verifySession(hostname, userId, sessionHash) {
  const url = `https://${hostname}/api/user/session/verify?hash=${encodeURIComponent(sessionHash)}&user_id=${userId}`;
  const response = await fetch(url);
  const data = await response.json();
  return data.valid === true;
}

module.exports = {
  getCurrentUser,
  verifySession,
  zoeziGet,
  zoeziApiPost
};
```

### Supabase Service: src/services/supabase.js

```javascript
const { createClient } = require('@supabase/supabase-js');

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_KEY
);

module.exports = supabase;
```

### Auth Middleware: src/middleware/auth.js

```javascript
const zoezi = require('../services/zoezi');

async function authenticateUser(req, res, next) {
  try {
    const { hostname, user_id, session_hash, session_raw } = req.body;

    if (!hostname || !user_id || !session_hash || !session_raw) {
      return res.status(400).json({
        success: false,
        error: 'Missing authentication fields'
      });
    }

    const isValid = await zoezi.verifySession(hostname, user_id, session_hash);
    if (!isValid) {
      return res.status(401).json({
        success: false,
        error: 'Invalid or expired session'
      });
    }

    req.gymHostname = hostname;
    req.userId = user_id;
    req.sessionRaw = session_raw;

    next();
  } catch (err) {
    next(err);
  }
}

module.exports = { authenticateUser };
```

### Example Route: src/routes/user.js

```javascript
const express = require('express');
const router = express.Router();
const { authenticateUser } = require('../middleware/auth');
const supabase = require('../services/supabase');
const zoezi = require('../services/zoezi');

// Get user data
router.post('/info', authenticateUser, async (req, res, next) => {
  try {
    const user = await zoezi.getCurrentUser(
      req.gymHostname,
      req.sessionRaw
    );

    res.json({
      success: true,
      data: {
        id: user.id,
        name: user.name,
        email: user.email
      }
    });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

---

## Minimal Frontend

### Vue.js Component Template

Save this as a `.html` file to paste into Zoezi:

```html
<template>
  <div class="zoezi-addon">
    <div v-if="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <h2>Hello, {{ userName }}!</h2>
      <p>User ID: {{ userId }}</p>
    </div>
  </div>
</template>

<script>
export default {
  name: 'MyZoeziAddon',

  data() {
    return {
      BACKEND_URL: 'http://localhost:5000',  // UPDATE AFTER DEPLOYMENT!
      hostname: null,
      userId: null,
      sessionHash: null,
      sessionRaw: null,
      loading: true,
      error: null,
      userName: null
    };
  },

  async mounted() {
    await this.init();
  },

  methods: {
    async init() {
      try {
        // Get authentication
        this.hostname = window.location.hostname;
        this.sessionRaw = this.getRawSession();
        this.sessionHash = await this.hashSession(this.sessionRaw);

        const user = await window.$zoeziapi.get('/api/memberapi/get/current');
        this.userId = user.id;

        // Get data from backend
        const result = await this.apiCall('/api/user/info');
        if (result.success) {
          this.userName = result.data.name;
        }
      } catch (err) {
        this.error = err.message;
      } finally {
        this.loading = false;
      }
    },

    getRawSession() {
      const cookies = document.cookie.split('; ');
      for (const c of cookies) {
        const [name, value] = c.split('=');
        if (name === 'session') return value;
      }
      return null;
    },

    async hashSession(session) {
      if (!session) return null;
      const encoder = new TextEncoder();
      const data = encoder.encode(session);
      const hashBuffer = await crypto.subtle.digest('SHA-256', data);
      return Array.from(new Uint8Array(hashBuffer))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('');
    },

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
        throw new Error(`HTTP ${response.status}`);
      }

      return response.json();
    }
  }
};
</script>

<style scoped>
.zoezi-addon {
  padding: 20px;
}

.error {
  background-color: #fee;
  border: 1px solid #fcc;
  padding: 15px;
  border-radius: 4px;
  color: #c33;
}
</style>
```

---

## Database Setup

### 1. Create Supabase Project

1. Go to https://supabase.com
2. Create new project
3. Copy project URL and service role key
4. Update `.env` file

### 2. Create Example Table

Run this SQL in Supabase SQL Editor:

```sql
-- Example table for your addon
CREATE TABLE IF NOT EXISTS my_addon_data (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  hostname TEXT NOT NULL,
  user_id INTEGER NOT NULL,
  data JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index for performance (always include hostname!)
CREATE INDEX idx_my_addon_data_hostname ON my_addon_data(hostname);
CREATE INDEX idx_my_addon_data_user ON my_addon_data(hostname, user_id);

-- Row Level Security
ALTER TABLE my_addon_data ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Service role full access" ON my_addon_data
  FOR ALL USING (true);
```

### 3. Use in Your Route

```javascript
// Get user's data
router.post('/data', authenticateUser, async (req, res, next) => {
  try {
    const { data, error } = await supabase
      .from('my_addon_data')
      .select('*')
      .eq('hostname', req.gymHostname)
      .eq('user_id', req.userId)
      .single();

    if (error && error.code !== 'PGRST116') {  // PGRST116 = not found
      throw error;
    }

    res.json({ success: true, data: data || null });
  } catch (err) {
    next(err);
  }
});
```

---

## Testing

### 1. Start Development Server

```bash
npm run dev
```

Server should be running on http://localhost:5000

### 2. Test Health Endpoint

```bash
curl http://localhost:5000/health
```

Expected response:
```json
{
  "status": "ok",
  "timestamp": "2026-01-13T10:30:00.000Z"
}
```

### 3. Test Frontend Locally

1. Open Zoezi in your browser
2. Open browser console
3. Paste the component JavaScript (without `<template>` tags)
4. Call methods manually to test

Or use a local HTML file:

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://cdn.jsdelivr.net/npm/vue@3"></script>
</head>
<body>
  <div id="app"></div>
  <script>
    // Paste your component code here
    Vue.createApp(YourComponent).mount('#app');
  </script>
</body>
</html>
```

---

## Deployment

### Option 1: Replit

1. Create new Repl
2. Upload your code
3. Create `.replit` file:

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

4. Set Secrets (environment variables)
5. Click Deploy
6. Copy deployment URL

### Option 2: Railway

1. Install Railway CLI: `npm i -g @railway/cli`
2. Login: `railway login`
3. Initialize: `railway init`
4. Add environment variables: `railway variables set SUPABASE_URL=...`
5. Deploy: `railway up`
6. Get URL: `railway domain`

### Option 3: Render

1. Push code to GitHub
2. Create new Web Service on Render
3. Connect GitHub repo
4. Set environment variables
5. Deploy
6. Copy deployment URL

---

## Next Steps

### After Deployment

1. **Update Frontend URLs**
   - Replace `http://localhost:5000` with your deployment URL
   - Update in ALL components

2. **Install Components in Zoezi**
   - Log in to Zoezi admin
   - Go to Components section
   - Create new component
   - Paste HTML, JavaScript, CSS separately
   - Assign to appropriate pages

3. **Test with Real Users**
   - Test as regular user
   - Test as staff user
   - Verify multi-tenancy (different gyms)

### Expand Your Add-on

Now that you have a working foundation:

- Add more routes in `src/routes/`
- Create more database tables in Supabase
- Add more frontend components
- Implement file uploads with Multer + Supabase Storage
- Integrate more Zoezi API endpoints (see ZOEZI_API_REFERENCE.md)

### Learn More

- **Full Guide**: [ZOEZI_ADDON_GUIDE.md](ZOEZI_ADDON_GUIDE.md)
- **API Reference**: [ZOEZI_API_REFERENCE.md](ZOEZI_API_REFERENCE.md)
- **Example Code**: Explore `src/` directory in StrongSales-B2B-Addon

---

## Troubleshooting

### "Invalid or expired session"

- Check that you're hashing the session correctly
- Verify session cookie exists (check browser cookies)
- Make sure user is logged into Zoezi

### "Supabase error"

- Verify SUPABASE_URL and SUPABASE_KEY in `.env`
- Check that table exists in Supabase
- Verify RLS policies allow service role access

### "Zoezi API error: 401"

- Check ZOEZI_API_KEY is correct
- Verify you're using correct auth method (API key vs session)
- Some endpoints require staff users

### CORS errors

- Make sure `app.use(cors())` is in server.js
- Check that backend URL is correct in frontend
- Verify deployment is publicly accessible

### Component not loading in Zoezi

- Check browser console for JavaScript errors
- Verify BACKEND_URL is set correctly
- Make sure component is assigned to correct page
- Check that user is logged in

---

## Quick Reference Commands

```bash
# Development
npm run dev              # Start dev server with auto-reload
npm start                # Start production server

# Testing
curl http://localhost:5000/health                    # Health check
curl -X POST http://localhost:5000/api/user/info \   # Test endpoint
  -H "Content-Type: application/json" \
  -d '{"hostname":"test.zoezi.com","user_id":1,...}'

# Deployment
git push                 # Push to repo (Railway, Render)
railway up               # Deploy to Railway
replit deploy           # Deploy to Replit (in Replit IDE)
```

---

**You're Ready to Build!**

This quick start gives you everything you need to create a functional Zoezi add-on. For more advanced features, consult the full guides.

Happy coding! 🚀

---

**Last Updated:** 2026-01-13
**Version:** 1.0
