# Zoezi Component Development Documentation

> **Purpose of this Repository**
>
> This is a **reference repository** containing Zoezi framework source code and documentation for AI assistants. The files here provide context for building custom apps and components that integrate with the Zoezi gym membership platform.
>
> **You don't modify these files** - you use them to understand how Zoezi works so you can create new integrated components.

## Quick Navigation

| Document | Purpose |
|----------|---------|
| [System Overview](./zoezi-architecture/SYSTEM-OVERVIEW.md) | Architecture, tech stack, core concepts |
| [Component Structure](./zoezi-architecture/COMPONENT-STRUCTURE.md) | How to create Zoezi components (Vue SFC + deployed format) |
| [Addon Deployment](./zoezi-architecture/ADDON-DEPLOYMENT.md) | Addon repo structure, webhooks, APIs, widgets, deployment |
| [Services & State](./zoezi-architecture/SERVICES-AND-STATE.md) | Available services, Vuex store, APIs |
| [Integration Patterns](./zoezi-patterns/INTEGRATION-PATTERNS.md) | Common patterns for checkout, cart, auth |
| [Component Reference](./zoezi-components/COMPONENT-REFERENCE.md) | Key components and their props |

## Repository Contents

| Content | Purpose | Do I Modify? |
|---------|---------|--------------|
| `components/` | Zoezi framework Vue components | **NO** - Reference only |
| `main.js` | Zoezi app initialization | **NO** - Reference only |
| `router.js` | Zoezi routing system | **NO** - Reference only |
| `dateextensions.js` | Zoezi date utilities | **NO** - Reference only |
| `docs/` | Documentation for AI assistants | **NO** - Reference only |
| `Fysiken/`, `Centralbadet/`, etc. | Brand-specific custom components | **YES** - Examples of custom integrations |
| `claude.md` | AI coding instructions | Update as needed |

**When building new Zoezi integrations:**
- Study the framework files to understand patterns
- Create new components following the patterns you see
- Place custom implementations in brand-specific folders (e.g., `Fysiken/`)

---

## What is Zoezi?

Zoezi is a **Vue.js 2 + Vuetify** gym management platform that provides:

- **Membership sales** (training cards, subscriptions)
- **Group training** booking (classes, workouts)
- **Course booking** (multi-session programs)
- **Resource booking** (equipment, rooms, personal training)
- **E-commerce** (webshop with products)
- **User management** (family accounts, payment methods)

## Key Principles

### 1. Configuration-Driven
Components are configured through the Zoezi page builder. Props become configurable options.

### 2. Multi-Tenant
The system supports multiple sites/locations. Always consider `$store.state.selectedSiteId`.

### 3. Component Naming
All Zoezi components MUST be named with the `zoezi-` prefix (e.g., `zoezi-shop`, `zoezi-mypage`).

### 4. Service Access
Components access backend data through injected services:
- `this.$api` - API calls
- `this.$store` - Vuex state
- `this.$translate` - Localization
- `this.$booking` - Booking utilities

## Quick Start Example

```vue
<template>
  <div v-if="$store.state.user">
    <h1>{{ $translate('Welcome') }}, {{ $store.state.user.firstname }}</h1>
    <div v-for="product in products" :key="product.id">
      {{ product.name }} - {{ product.price | price }}
    </div>
  </div>
  <zoezi-identification v-else title="Please log in" />
</template>

<script>
export default {
  name: 'zoezi-my-component',

  zoezi: {
    title: 'My Component',
    icon: 'mdi-star'
  },

  props: {
    showPrices: {
      title: 'Show prices',
      type: Boolean,
      default: true
    }
  },

  data: () => ({
    products: []
  }),

  mounted() {
    this.$api.get('/api/public/trainingcard/type/get').then(data => {
      this.products = data;
    });
  }
}
</script>
```

## File Organization

```
StrongSales-Zoezi/
├── docs/                            # Documentation (READ FIRST)
│   ├── README.md                    # This file
│   ├── zoezi-architecture/
│   │   ├── SYSTEM-OVERVIEW.md       # Architecture & tech stack
│   │   ├── COMPONENT-STRUCTURE.md   # Component creation guide
│   │   └── SERVICES-AND-STATE.md    # Services & Vuex store
│   ├── zoezi-patterns/
│   │   └── INTEGRATION-PATTERNS.md  # Common integration patterns
│   └── zoezi-components/
│       └── COMPONENT-REFERENCE.md   # Key component documentation
│
├── components/                      # Vue Components (75 total)
│   ├── README.md                    # Component organization guide
│   ├── auth/                        # Authentication (4 components)
│   │   ├── Identification.vue
│   │   ├── Login.vue
│   │   ├── Logout.vue
│   │   └── ResetPassword.vue
│   ├── checkout/                    # Payments (6 components)
│   │   ├── Checkout.vue             # Main checkout - USE THIS
│   │   └── ...
│   ├── shop/                        # E-commerce (6 components)
│   │   ├── Shop.vue
│   │   └── ...
│   ├── booking/                     # All booking features
│   │   ├── group-training/          # Group classes (3)
│   │   ├── courses/                 # Courses (8)
│   │   └── resources/               # Resource booking (6)
│   ├── user/                        # User dashboard (12 components)
│   │   ├── MyPage.vue
│   │   └── ...
│   ├── layout/                      # UI/Layout (13 components)
│   ├── dialogs/                     # Dialogs (5 components)
│   ├── admin/                       # Admin tools (5 components)
│   └── misc/                        # Miscellaneous (7 components)
│
├── main.js                          # Application entry point
├── router.js                        # Route configuration
├── dateextensions.js                # Date utilities
│
├── Fysiken/                         # Brand-specific implementations
├── Centralbadet/
├── Pelvic Lab/
├── Sturebadet/
│
└── claude.md                        # AI coding instructions
```

## Important Files to Read

When starting work on Zoezi components, read these files:

1. **`claude.md`** - AI-specific instructions and rules
2. **`components/README.md`** - Component organization
3. **`main.js`** - Application initialization and configuration
4. **`router.js`** - Route handling
5. **Target component `.vue` files** - Existing patterns to follow

## Key Components by Feature

| Feature | Component Path |
|---------|---------------|
| **Checkout** | `components/checkout/Checkout.vue` |
| **Shop** | `components/shop/Shop.vue` |
| **Login** | `components/auth/Identification.vue` |
| **User Dashboard** | `components/user/MyPage.vue` |
| **Group Training** | `components/booking/group-training/GroupTraining.vue` |
| **Resource Booking** | `components/booking/resources/ResourceBooking.vue` |
| **Course Booking** | `components/booking/courses/CourseBooking.vue` |
