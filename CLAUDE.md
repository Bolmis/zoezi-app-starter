# Claude Code Instructions for Zoezi App Development

This file contains instructions for AI assistants building apps with the Zoezi API.

---

## What This Repository Is

> **This is a STARTER TEMPLATE for building Zoezi-integrated apps.**
>
> Use this as a foundation to create custom applications that fetch data from the Zoezi gym membership platform API and display it in a frontend.
>
> Apps built with this template can have **two deployment targets**:
> 1. **Backend (Express server)** — hosted on Replit or similar, handles API proxying and business logic
> 2. **Zoezi Developer Portal (Addon)** — Vue 2 components, webhooks, cloud functions, and widgets deployed directly to the Zoezi platform via git push

### Key Files

| File/Folder | Purpose |
|-------------|---------|
| `index.js` | Express server - add your API routes here |
| `public/` | Frontend files - build your UI here |
| `Zoezi API docs.json` | Complete Zoezi API reference |
| `docs/` | Architecture and integration documentation |
| `dateextensions.js` | Useful date utility functions |

---

## Quick Start

**Before building, read these documents:**

1. **[Zoezi API docs.json](./Zoezi%20API%20docs.json)** - API endpoints reference
2. **[docs/zoezi-architecture/SERVICES-AND-STATE.md](./docs/zoezi-architecture/SERVICES-AND-STATE.md)** - API services
3. **[docs/zoezi-patterns/INTEGRATION-PATTERNS.md](./docs/zoezi-patterns/INTEGRATION-PATTERNS.md)** - Common patterns
4. **[docs/zoezi-architecture/ADDON-DEPLOYMENT.md](./docs/zoezi-architecture/ADDON-DEPLOYMENT.md)** - Addon structure and deployment

---

## Development Guidelines

### 1. Backend API Routes

Add API proxy routes in `index.js` to communicate with Zoezi:

```javascript
// Proxy endpoint example
app.get('/api/members', async (req, res) => {
  try {
    const response = await fetch(`${process.env.ZOEZI_API_URL}/api/members`, {
      headers: {
        'Authorization': `Bearer ${process.env.ZOEZI_API_KEY}`,
        'Content-Type': 'application/json'
      }
    });
    const data = await response.json();
    res.json(data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### 2. Frontend Development

Build the frontend in `public/`. For simple apps, vanilla HTML/CSS/JS works well:

```html
<!-- public/index.html -->
<script>
  async function loadData() {
    const response = await fetch('/api/members');
    const members = await response.json();
    // Display the data
  }
</script>
```

For more complex apps, consider adding a frontend framework.

### 3. Date Handling

Use the provided `dateextensions.js` for date operations:

```javascript
// Load the extensions
require('./dateextensions.js');

// Use the methods
const today = Date.today();
const formatted = new Date().yyyymmdd();  // "2025-01-30"
const time = new Date().hhmm();            // "14:30"
```

---

## Zoezi API Overview

The Zoezi API provides access to gym management data. Key areas:

### Authentication
- User login/logout
- Session management
- Token-based auth

### Members
- Member profiles
- Subscriptions and memberships
- Payment methods
- Family accounts

### Products
- Shop products
- Membership products
- Pricing and discounts
- Site-specific filtering

### Bookings
- Group training sessions
- Resource bookings (rooms, equipment)
- Course enrollments
- Waitlist management

### Sites
- Multi-location support
- Site-specific settings
- Location filtering

---

## Multi-Site Considerations

Zoezi supports multiple gym locations. When fetching data:

```javascript
// Filter products by site
const siteId = req.query.siteId;
const products = allProducts.filter(p =>
  !p.sites || p.sites.length === 0 || p.sites.includes(siteId)
);
```

---

## Error Handling

Always handle API errors gracefully:

```javascript
app.get('/api/data', async (req, res) => {
  try {
    const response = await fetch(`${API_URL}/api/endpoint`);

    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }

    const data = await response.json();
    res.json(data);
  } catch (error) {
    console.error('API Error:', error);
    res.status(500).json({
      error: 'Failed to fetch data',
      message: error.message
    });
  }
});
```

---

## Environment Variables

Required environment variables:

```env
PORT=3000
ZOEZI_API_URL=https://your-zoezi-instance.com
ZOEZI_API_KEY=your-api-key
```

On Replit, add these as Secrets.

---

## File Structure Convention

When building apps, follow this structure:

```
zoezi-app-starter/
├── index.js                    # Server entry point
├── routes/                     # API route modules (optional)
│   ├── members.js
│   ├── products.js
│   └── bookings.js
├── public/                     # Frontend
│   ├── index.html              # Main HTML
│   ├── css/
│   │   └── styles.css
│   └── js/
│       └── app.js
├── utils/                      # Utilities (optional)
│   └── api.js                  # API helper functions
└── ...
```

---

## Zoezi Addon Deployment

If building a Zoezi addon (components embedded in the Zoezi platform), the addon is managed via a **separate git repo that deploys directly** to the developer portal on push.

See **[docs/zoezi-architecture/ADDON-DEPLOYMENT.md](./docs/zoezi-architecture/ADDON-DEPLOYMENT.md)** for the complete structure.

### Addon repo structure:

```
zoezi-component/
├── config.json          # Addon metadata (name, permissions, state)
├── webhooks.json        # Webhook subscriptions
├── settings.json        # Addon settings schema
├── apis.json            # Custom API endpoints
├── widgets.json         # Staff dashboard widgets
├── components/          # Vue 2 components (code.js, template.html, style.css, config.json each)
└── functions/           # Cloud functions (code.js, config.json each)
```

### Deploy addon changes:

```bash
cd ~/zoezi-component
git pull origin main
# Make changes
git add . && git commit -m "description" && git push origin main
```

Changes are **live immediately** after push.

---

## Zoezi Component Documentation Convention

When documenting Zoezi components, follow the **ONE master file** rule:

1. **ONE file per component** in `docs/zoezi-components/` named `[NAME]-COMPONENT-COMPLETE.md`
2. **FULL working code** in every file — complete HTML, JavaScript, and CSS sections
3. **Update existing files** — never create partial update files or "ADD THESE" style docs
4. **Copy-paste ready** — zero external references, zero partial snippets
5. **Single source of truth** — each file is the definitive version of that component

---

## Checklist Before Deploying

- [ ] Environment variables configured
- [ ] API endpoints tested
- [ ] Error handling in place
- [ ] Frontend displays data correctly
- [ ] Loading states implemented
- [ ] Multi-site filtering (if applicable)
- [ ] Authentication (if required)

---

## Summary for AI

When building Zoezi apps:

1. **Read the API docs** - Check `Zoezi API docs.json` for endpoints
2. **Add backend routes** - Proxy Zoezi API calls through `index.js`
3. **Build the frontend** - Create UI in `public/`
4. **Handle errors** - Always catch and display errors
5. **Consider multi-site** - Filter by site ID when relevant
6. **Use environment variables** - Never hardcode credentials
7. **Test thoroughly** - Verify API responses and UI display
8. **Deploy addon components** - Push to the zoezi-component repo for live Zoezi platform deployment (see ADDON-DEPLOYMENT.md)

**Two deployment targets to remember:**
- **Backend** → Replit (or similar) via `index.js`
- **Addon components/webhooks/functions** → Zoezi Developer Portal via `git push` in the zoezi-component repo

The documentation in `docs/` has detailed examples and patterns for Zoezi integration.
