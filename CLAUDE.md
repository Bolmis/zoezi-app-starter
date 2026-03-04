# Zoezi Addon Development — AI Context Guide

> **This repository is a KNOWLEDGE BASE for AI assistants (Claude, Codex, etc.).**
> It is NOT a code scaffold. You clone this repo, then build your addon from scratch using the context provided here. The AI has everything it needs to build a production-grade Zoezi addon right from the start.
>
> **If docs contradict each other, THIS FILE (CLAUDE.md) and BACKEND-ARCHITECTURE.md are the authoritative sources.** The other docs were adapted from a production addon and may have minor inconsistencies.

---

## What is a Zoezi Addon?

A Zoezi addon extends the Zoezi gym management platform. It consists of three parts:

1. **Backend API** — Node.js/Express server hosted on Replit (or Railway/Render)
2. **Frontend Components** — Vue 2 components embedded directly into Zoezi pages via the Developer Portal
3. **Database** — Supabase (PostgreSQL + Storage)

### Architecture

```
Zoezi Platform (Vue 2 components in the browser)
    |
    | HTTPS POST with { hostname, user_id, session_hash, ...data }
    v
Express.js Backend (hosted on Replit)
    |
    ├── Auth middleware verifies session with Zoezi
    ├── Routes handle business logic
    ├── Supabase for data persistence (multi-tenant, hostname-filtered)
    └── Zoezi API for platform operations (groups, statuses, TODOs, etc.)
```

### Key Characteristics

- **Multi-Tenant** — One backend serves many gyms, isolated by `hostname`
- **Session-Based Auth** — Frontend hashes the Zoezi session cookie (SHA-256), backend verifies with Zoezi's API
- **Two Deployment Targets** — Backend on Replit + Components via git push to Zoezi Developer Portal
- **Stateless Backend** — Every request includes full auth context

---

## MUST-READ Documentation (Priority Order)

Read these docs in order before building anything:

| Priority | Document | What You Learn |
|----------|----------|----------------|
| 1 | **[ZOEZI_QUICKSTART.md](docs/ZOEZI_QUICKSTART.md)** | Get a working addon in 30 minutes — minimal boilerplate for backend + frontend |
| 2 | **[BACKEND-ARCHITECTURE.md](docs/zoezi-patterns/BACKEND-ARCHITECTURE.md)** | Express setup, auth middleware, route patterns, webhook handlers, cron jobs, file uploads |
| 3 | **[SUPABASE-PATTERNS.md](docs/zoezi-patterns/SUPABASE-PATTERNS.md)** | Multi-tenant database design, query patterns, storage, RLS, migrations |
| 4 | **[ZOEZI_API_REFERENCE.md](docs/ZOEZI_API_REFERENCE.md)** | Complete Zoezi API endpoints — users, groups, statuses, TODOs, discounts, session verification |
| 5 | **[ZOEZI_ADDON_GUIDE.md](docs/ZOEZI_ADDON_GUIDE.md)** | Comprehensive development guide — all patterns in one document |
| 6 | **[ADDON-DEPLOYMENT.md](docs/zoezi-architecture/ADDON-DEPLOYMENT.md)** | Zoezi Developer Portal structure — config.json, webhooks.json, component 4-file format |
| 7 | **[COMPONENT-STRUCTURE.md](docs/zoezi-architecture/COMPONENT-STRUCTURE.md)** | Vue 2 component creation — props, zoezi metadata, URL sync, styling |
| 8 | **[SERVICES-AND-STATE.md](docs/zoezi-architecture/SERVICES-AND-STATE.md)** | Zoezi platform services — `$api`, `$store`, `$translate`, `$booking`, `$router` |
| 9 | **[INTEGRATION-PATTERNS.md](docs/zoezi-patterns/INTEGRATION-PATTERNS.md)** | Checkout, shopping cart, authentication, products, bookings, multi-site |
| 10 | **[COMPONENT-REFERENCE.md](docs/zoezi-components/COMPONENT-REFERENCE.md)** | Built-in Zoezi components — login, shop, checkout, booking, mypage |

Additional references:
- **[Zoezi API docs.json](Zoezi%20API%20docs.json)** — Raw API endpoint reference
- **[dateextensions.js](dateextensions.js)** — Date utility functions used throughout Zoezi

---

## Two Deployment Targets

### 1. Backend (Express Server) → Replit

The backend handles API logic, database operations, and Zoezi API integration.

**Tech stack:**
- Express 5.x with CORS, rate limiting
- Supabase client (database + storage)
- Zoezi API wrapper service
- Authentication middleware (session hash verification)
- Optional: multer (file uploads), node-cron (scheduled jobs)

**Deployment:**
- Host on Replit (import from GitHub, set Secrets, click Deploy)
- Or Railway/Render/any Node.js host with public HTTPS URL
- Backend URL must be updated in ALL frontend components

**Environment variables required:**
```bash
SUPABASE_URL=https://xxxxxxxxxxxxx.supabase.co
SUPABASE_KEY=your-service-role-key
ZOEZI_API_KEY=your-addon-api-key
WEBHOOK_SECRET=your-webhook-secret    # For webhook authentication
PORT=5000
```

**.replit config:**
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

### 2. Zoezi Developer Portal (Addon) → Git Push

Frontend components, webhooks, functions, and settings are deployed via a **separate git repo** that pushes directly to the Zoezi Developer Portal.

**Repo location:** `~/zoezi-component/` (create a new one for each addon)
**Remote:** `ssh://git@developer.zoezi.se:51749/org-{ID}/{addon-id}.git`

**Deploy command:**
```bash
cd ~/zoezi-component
git pull origin main    # Always pull first (web interface may have made changes)
# Make changes
git add . && git commit -m "description" && git push origin main
```

**Changes are LIVE immediately after push.** No build step, no CI pipeline.

**Addon repo structure:**
```
zoezi-component/
├── config.json          # Addon metadata (name, permissions, state, test_hostname)
├── webhooks.json        # Webhook subscriptions (events → URL or function)
├── settings.json        # Addon settings schema (admin-configurable)
├── apis.json            # Custom API endpoints (through Zoezi platform)
├── widgets.json         # Staff dashboard widgets
├── components/          # Vue 2 components (4 files each — NOT SFC format!)
│   └── MY-COMPONENT/
│       ├── config.json      # { id, name, type: "vue2" }
│       ├── code.js          # export default { data(), methods, ... } (no <script> tags)
│       ├── template.html    # Pure HTML only (no <template> wrapper, no <div> root required)
│       └── style.css        # Scoped styles under a root class (no <style> tags)
└── functions/           # Cloud functions
    └── my-function/
        ├── config.json      # { name, language: "nodejs20.x", template: "api" }
        └── code.js          # module.exports = async function(event, context) { ... }
```

---

## Authentication Flow

This is the most important pattern to understand. Every addon uses it.

### Frontend (in Zoezi component)

```javascript
// 1. Get hostname from browser
this.hostname = window.location.hostname;

// 2. Extract raw session cookie
getRawSession() {
  const cookies = document.cookie.split('; ');
  for (const c of cookies) {
    const [name, value] = c.split('=');
    if (name === 'session') return value;
  }
  return null;
}

// 3. Hash session with SHA-256
async hashSession(session) {
  if (!session) return null;
  const encoder = new TextEncoder();
  const data = encoder.encode(session);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  return Array.from(new Uint8Array(hashBuffer))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}

// 4. Get current user from Zoezi
const user = await window.$zoeziapi.get('/api/memberapi/get/current');
this.userId = user.id;

// 5. Every API call to your backend includes auth context
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
  return response.json();
}
```

### Backend (auth middleware)

```javascript
// Verify with Zoezi that the session hash is valid
const url = `https://${hostname}/api/user/session/verify?hash=${sessionHash}&user_id=${userId}`;
const response = await fetch(url);
const data = await response.json();
// data.valid === true means authenticated

// For staff-only endpoints: also check user.staff === true
const user = await zoezi.getUser(hostname, userId);
if (!user.staff) return res.status(403).json({ error: 'Staff access required' });
```

See [BACKEND-ARCHITECTURE.md](docs/zoezi-patterns/BACKEND-ARCHITECTURE.md) for the complete middleware code.

---

## Multi-Tenancy Pattern

One backend serves many gym installations. **Every database query MUST filter by hostname.**

```javascript
// CORRECT — always filter by hostname
const { data } = await supabase
  .from('settings')
  .select('*')
  .eq('hostname', req.gymHostname);  // Hostname from auth middleware

// WRONG — leaks data across gyms
const { data } = await supabase
  .from('settings')
  .select('*');
```

Database tables always include:
```sql
hostname TEXT NOT NULL,  -- Multi-tenancy key
CREATE INDEX idx_table_hostname ON table(hostname);
UNIQUE(hostname, unique_field);  -- Unique constraints include hostname
```

---

## Zoezi API — Two Auth Methods

### 1. API Key (backend operations)

For creating resources, updating system data (TODOs, statuses, discount types):

```javascript
const response = await fetch(`https://${hostname}/api/todo/add`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
  },
  body: JSON.stringify({ todo: 'Review proof', user_id: userId })
});
```

### 2. Session Cookie (user context)

For reading user data, managing groups, user-specific operations:

```javascript
const response = await fetch(`https://${hostname}/api/memberapi/get/current`, {
  headers: {
    'Content-Type': 'application/json',
    'Cookie': `session=${sessionRaw}`
  }
});
```

### Key API Endpoints

**Note:** The production pattern uses **API Key auth for ALL backend calls**. Session cookie auth is only used by frontend components calling `window.$zoeziapi`. Some older docs show session auth for backend calls — ignore those, use API Key.

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/api/memberapi/get/current` | GET | Session* | Get logged-in user (*frontend only*) |
| `/api/user/get?id={id}` | GET | API Key | Get specific user |
| `/api/user/change` | POST | API Key | Update user data/xf |
| `/api/member/change` | POST | API Key | Update status/groups |
| `/api/usergroup/get/all` | GET | API Key | List all groups |
| `/api/usergroup/get?id={id}` | GET | API Key | Get specific group |
| `/api/usergroup/add` | POST | API Key | Create group |
| `/api/usergroup/change` | POST | API Key | Update group members |
| `/api/member/status/get/all` | GET | API Key | List statuses |
| `/api/member/status/add` | POST | API Key | Create status |
| `/api/todo/add` | POST | API Key | Create TODO |
| `/api/todo/getById?id={id}` | GET | API Key | Get TODO by ID |
| `/api/trainingcard/type/add` | POST | API Key | Create discount type |
| `/api/trainingcard/type/get/all` | GET | API Key | List all trainingcard types |
| `/api/settings/fields/get` | GET | API Key | Get custom fields |
| `/api/user/session/verify` | GET | None | Verify session hash (public) |

*Session auth (`Cookie: session=...`) is used by the **frontend** `window.$zoeziapi` calls. The **backend** uses API Key auth (`Authorization: Zoezi {KEY}`) for everything.

See [ZOEZI_API_REFERENCE.md](docs/ZOEZI_API_REFERENCE.md) for complete documentation with request/response examples.

### Common Zoezi Integration Patterns

```javascript
// Ensure a group exists (find or create)
async function ensureGroupExists(hostname, groupName) {
  const groups = await zoeziApiGet(hostname, '/api/usergroup/get/all');
  let group = groups.find(g => g.name === groupName);
  if (!group) {
    group = await zoeziApiPost(hostname, '/api/usergroup/add', {
      name: groupName, change_permission: false, users: [], show: false
    });
  }
  return group;
}

// Add user to group (preserving existing members)
async function addUserToGroup(hostname, groupId, userId) {
  const group = await zoeziApiGet(hostname, `/api/usergroup/get?id=${groupId}`);
  if (group.users?.includes(userId)) return;
  await zoeziApiPost(hostname, '/api/usergroup/change', {
    id: groupId,
    users: [...(group.users || []), userId]
  });
}

// Update user extra fields (XF)
async function updateUserXf(hostname, userId, fieldName, value) {
  const user = await zoeziApiGet(hostname, `/api/user/get?id=${userId}`);
  await zoeziApiPost(hostname, '/api/user/change', {
    id: userId,
    xf: { ...(user.xf || {}), [fieldName]: value }
  });
}
```

---

## Component Types

Every addon typically has some combination of these:

### 1. Settings Component (Staff Only)
- Configure addon settings per gym
- Protected by `authenticateStaff` middleware
- Placed in admin/settings area of Zoezi

### 2. Admin/Management Component (Staff Only)
- CRUD operations for managing entities
- Data tables, forms, dialogs
- Staff dashboard widget integration

### 3. User Component (Customer-Facing)
- Self-service features for gym members
- Submit forms, view status, take actions
- Placed in customer portal/profile section

### 4. Auto/Background Component (Invisible)
- Runs automatically on specific pages (e.g., checkout)
- No visible UI — monitors page state and acts
- Example: auto-applying discount codes during checkout

---

## Webhook Integration

### Backend: Webhook Handler

```javascript
// src/routes/webhooks.js
router.post('/todo-updated', authenticateInternal, async (req, res, next) => {
  try {
    const events = req.body.events || [];
    for (const event of events) {
      if (event.action === 'update') {
        // Process the webhook event
      }
    }
    res.json({ success: true });
  } catch (err) {
    next(err);
  }
});
```

### Zoezi Developer Portal: webhooks.json

```json
[
  {
    "name": "todo",
    "useFunction": false,
    "url": "https://your-backend.replit.app/api/webhooks/todo-updated?secret=YOUR_SECRET",
    "events": ["todo"],
    "actions": ["update"]
  }
]
```

Common webhook events: `"todo"`, `"member"`, `"booking"`, `"payment"`, `"trainingcard"`

---

## Zoezi Component Documentation Convention

When building components, follow the **ONE master file** pattern:

1. **ONE file per component** in `docs/zoezi-components/` named `[NAME]-COMPONENT-COMPLETE.md`
2. **FULL working code** — complete HTML, JavaScript, AND CSS in every file
3. **Copy-paste ready** — zero external references, zero partial snippets
4. **Update existing files** — never create partial update docs, always edit the COMPLETE file
5. **Single source of truth** — each file IS the definitive version

### CRITICAL RULE: Dual Update

When making changes, ALWAYS update BOTH:
1. Master docs in your backend repo (`docs/zoezi-components/*-COMPLETE.md`)
2. LIVE component files in `~/zoezi-component/components/` (and `webhooks.json` if applicable)

Then push BOTH repos. The `~/zoezi-component/` push is what deploys — skipping it means changes are docs-only, not live.

---

## Development Checklist for New Addons

### Planning
- [ ] Define addon purpose and features
- [ ] Identify required Zoezi API endpoints
- [ ] Design database schema (with hostname multi-tenancy)
- [ ] Plan component hierarchy (settings / admin / user / background)

### Backend Setup
- [ ] Create project with `npm init`
- [ ] Install dependencies: `express cors @supabase/supabase-js express-rate-limit multer node-cron uuid`
- [ ] Create directory structure: `src/middleware/`, `src/routes/`, `src/services/`
- [ ] Implement auth middleware (copy from BACKEND-ARCHITECTURE.md)
- [ ] Create `src/services/zoezi.js` — Zoezi API wrapper
- [ ] Create `src/services/supabase.js` — Database client
- [ ] Create route files for each feature area
- [ ] Add `.env.example` with all required variables
- [ ] Add `.replit` config

### Database Setup
- [ ] Create Supabase project
- [ ] Create tables with `hostname` column and indexes
- [ ] Enable RLS on every table
- [ ] Create storage buckets if needed
- [ ] Add `.mcp.json` for AI-assisted development

### Frontend Components
- [ ] Create `~/zoezi-component/` addon repo
- [ ] Configure `config.json` (permissions, test_hostname)
- [ ] Build components in 4-file format (config.json, code.js, template.html, style.css)
- [ ] Implement auth helpers (getRawSession, hashSession, apiCall)
- [ ] Add loading and error states
- [ ] Configure `webhooks.json` if needed
- [ ] Configure `widgets.json` for staff dashboard widgets

### Deployment
- [ ] Deploy backend to Replit (set Secrets, click Deploy)
- [ ] Copy deployment URL
- [ ] Update `BACKEND_URL` in ALL component code.js files
- [ ] Push zoezi-component repo (`git push origin main`)
- [ ] Test with real Zoezi session on test hostname
- [ ] Test as staff user AND regular user
- [ ] Verify multi-tenancy (different gym hostnames)

---

## Existing Documentation in This Repo

```
docs/
├── ZOEZI_ADDON_GUIDE.md              # Comprehensive addon development guide
├── ZOEZI_API_REFERENCE.md            # Complete Zoezi API endpoint reference
├── ZOEZI_QUICKSTART.md               # 30-minute quickstart template
├── zoezi-architecture/
│   ├── SYSTEM-OVERVIEW.md            # Zoezi tech stack and architecture
│   ├── SERVICES-AND-STATE.md         # $api, $store, $translate, $booking services
│   ├── COMPONENT-STRUCTURE.md        # Vue 2 component creation guide
│   └── ADDON-DEPLOYMENT.md           # Developer Portal structure and deployment
├── zoezi-patterns/
│   ├── BACKEND-ARCHITECTURE.md       # Express setup, auth, routes, webhooks, cron
│   ├── SUPABASE-PATTERNS.md          # Multi-tenant DB, queries, storage, RLS
│   └── INTEGRATION-PATTERNS.md       # Checkout, cart, auth, products, bookings
└── zoezi-components/
    └── COMPONENT-REFERENCE.md        # Built-in Zoezi components reference
```

---

## Summary for AI

When building a new Zoezi addon from this repo:

1. **Read ZOEZI_QUICKSTART.md first** — Get the minimal boilerplate running
2. **Set up the backend** — Express + auth middleware + Supabase + Zoezi API service (see BACKEND-ARCHITECTURE.md)
3. **Design the database** — Multi-tenant tables with hostname isolation (see SUPABASE-PATTERNS.md)
4. **Build components** — Vue 2 in 4-file format, with session hash auth (see COMPONENT-STRUCTURE.md)
5. **Deploy** — Backend to Replit, components via git push to Developer Portal
6. **Always filter by hostname** — Every query, every operation
7. **Always verify sessions** — Never trust client input
8. **Use the ONE master file convention** — One complete doc per component, copy-paste ready
