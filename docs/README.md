# Zoezi Addon Development — Documentation Index

> **This is a KNOWLEDGE BASE for AI assistants (Claude, Codex, etc.).**
> Clone this repo when starting a new Zoezi addon project. The documentation here provides all context needed to build a production-grade addon from scratch.

---

## Quick Navigation

### Start Here (Backend + Infrastructure)

| Document | What You Learn |
|----------|----------------|
| **[ZOEZI_QUICKSTART.md](ZOEZI_QUICKSTART.md)** | Get a working addon in 30 minutes |
| **[BACKEND-ARCHITECTURE.md](zoezi-patterns/BACKEND-ARCHITECTURE.md)** | Express setup, auth middleware, routes, webhooks, cron, file uploads |
| **[SUPABASE-PATTERNS.md](zoezi-patterns/SUPABASE-PATTERNS.md)** | Multi-tenant DB design, queries, storage, RLS, migrations |
| **[ZOEZI_API_REFERENCE.md](ZOEZI_API_REFERENCE.md)** | Complete Zoezi API — users, groups, statuses, TODOs, discounts |
| **[ZOEZI_ADDON_GUIDE.md](ZOEZI_ADDON_GUIDE.md)** | Comprehensive guide covering all patterns in one document |

### Zoezi Platform (Frontend Components)

| Document | What You Learn |
|----------|----------------|
| **[ADDON-DEPLOYMENT.md](zoezi-architecture/ADDON-DEPLOYMENT.md)** | Developer Portal structure — config.json, webhooks, widgets, deployment |
| **[COMPONENT-STRUCTURE.md](zoezi-architecture/COMPONENT-STRUCTURE.md)** | Vue 2 component creation — props, metadata, URL sync, styling |
| **[SERVICES-AND-STATE.md](zoezi-architecture/SERVICES-AND-STATE.md)** | Platform services — `$api`, `$store`, `$translate`, `$booking` |
| **[INTEGRATION-PATTERNS.md](zoezi-patterns/INTEGRATION-PATTERNS.md)** | Checkout, cart, auth, products, bookings, multi-site patterns |
| **[COMPONENT-REFERENCE.md](zoezi-components/COMPONENT-REFERENCE.md)** | Built-in Zoezi components — login, shop, checkout, booking, mypage |

### Architecture

| Document | What You Learn |
|----------|----------------|
| **[SYSTEM-OVERVIEW.md](zoezi-architecture/SYSTEM-OVERVIEW.md)** | Zoezi tech stack — Vue 2, Vuetify, Vuex, CSS architecture |

---

## Documentation Structure

```
docs/
├── README.md                             # This file
├── ZOEZI_ADDON_GUIDE.md                  # Comprehensive addon development guide
├── ZOEZI_API_REFERENCE.md                # Complete Zoezi API reference
├── ZOEZI_QUICKSTART.md                   # 30-minute quickstart template
├── zoezi-architecture/
│   ├── SYSTEM-OVERVIEW.md                # Zoezi tech stack and architecture
│   ├── SERVICES-AND-STATE.md             # $api, $store, $translate, $booking
│   ├── COMPONENT-STRUCTURE.md            # Vue 2 component creation guide
│   └── ADDON-DEPLOYMENT.md              # Developer Portal structure + deployment
├── zoezi-patterns/
│   ├── BACKEND-ARCHITECTURE.md           # Express, auth, routes, webhooks, cron
│   ├── SUPABASE-PATTERNS.md              # Multi-tenant DB, queries, storage, RLS
│   └── INTEGRATION-PATTERNS.md           # Checkout, cart, auth, products, bookings
└── zoezi-components/
    └── COMPONENT-REFERENCE.md            # Built-in Zoezi components reference
```

---

## What is Zoezi?

Zoezi is a **Vue.js 2 + Vuetify** gym management platform that provides:

- **Membership sales** (training cards, subscriptions)
- **Group training** booking (classes, workouts)
- **Course & Resource booking** (rooms, equipment, personal training)
- **E-commerce** (webshop with products)
- **User management** (family accounts, payment methods)
- **Addon system** — third-party apps that extend the platform

## What is a Zoezi Addon?

An addon has three parts:
1. **Backend API** — Node.js/Express on Replit, with Supabase for storage
2. **Frontend Components** — Vue 2 components deployed via the Zoezi Developer Portal
3. **Configuration** — Webhooks, settings, widgets, APIs defined in JSON files

Key characteristics:
- **Multi-tenant** — One backend serves many gyms, isolated by hostname
- **Session-based auth** — SHA-256 hash of Zoezi session cookie, verified server-side
- **Two deployment targets** — Backend on Replit + components via git push

See **[CLAUDE.md](../CLAUDE.md)** for the complete AI briefing document.
