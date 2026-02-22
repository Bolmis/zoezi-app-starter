# Zoezi Addon Deployment

This document explains how Zoezi addons are structured, configured, and deployed via the Zoezi Developer Portal.

## Overview

Zoezi addons are deployed via a **git repository that pushes directly to the developer portal**. Changes are live immediately after `git push`.

```
Remote: ssh://git@developer.zoezi.se:51749/org-{ID}/{addon-id}.git
Deploy: git add . && git commit -m "description" && git push origin main
```

Always `git pull origin main` first to get any changes made via the web interface.

---

## Repository Structure

```
zoezi-component/
├── config.json          # Addon metadata (name, permissions, state)
├── webhooks.json        # Webhook subscriptions
├── settings.json        # Addon settings schema
├── apis.json            # Custom API endpoints
├── widgets.json         # Staff dashboard widgets
├── components/          # Vue 2 components
│   └── MY-COMPONENT/
│       ├── config.json      # Component metadata
│       ├── code.js          # JavaScript (Vue export default)
│       ├── template.html    # HTML template
│       └── style.css        # CSS styles
└── functions/           # Cloud functions
    └── my-function/
        ├── config.json      # Function metadata
        └── code.js          # Function code
```

---

## config.json — Addon Metadata

Top-level addon configuration.

```json
{
    "id": 1270,
    "addon_id": "e9fa25bf23d3a289",
    "name": "My Addon Name",
    "supplier_mail": "you@example.com",
    "supplier_name": "Your Company AB",
    "description": null,
    "state": "test",
    "permissions_granted": [
        "Customer handling",
        "Show customers",
        "Settings",
        "Admin"
    ],
    "modules": [],
    "test_hostname": "addon111.zoezi.se",
    "unique_apikey": false,
    "preview": {
        "pages": [
            {
                "left": {
                    "header": "Feature Overview",
                    "content": "Description text"
                },
                "right": {
                    "type": "list",
                    "list": ["Feature 1", "Feature 2"]
                }
            }
        ]
    }
}
```

**Key fields:**
- `state`: `"test"` or `"published"` — controls visibility
- `permissions_granted`: Array of Zoezi permissions the addon requests
- `test_hostname`: The Zoezi instance used for testing
- `preview.pages`: Marketing pages shown in the addon store

**Common permissions:**
`"Customer handling"`, `"Change homepage"`, `"Show statistics"`, `"Show staffs"`, `"Show customers"`, `"Show customer journal"`, `"Show payments"`, `"Show all ToDos"`, `"Training card handling"`, `"Resource booking"`, `"Entry handling"`, `"Todo handling"`, `"Change customer journal"`, `"Settings"`, `"Staff handling"`, `"Admin"`

---

## components/ — Vue 2 Components

Each component lives in its own directory with 4 files:

### config.json

```json
{
    "id": "0b2465dbb155e57afbcb5de372ed9b0e93104f0120e0f156337011b5a2a9a721",
    "name": "MY-COMPONENT",
    "dependencies": null,
    "type": "vue2"
}
```

- `id`: Auto-generated hash (assigned by the platform)
- `name`: Component directory name
- `type`: Always `"vue2"` for now

### code.js

Standard Vue 2 component export:

```javascript
export default {
    data() {
        return {
            loading: false,
            items: []
        };
    },

    computed: {
        user() {
            return this.$store.state.user;
        }
    },

    watch: {
        '$store.state.user': {
            immediate: true,
            handler(user) {
                if (user) this.loadData();
            }
        }
    },

    methods: {
        async loadData() {
            this.loading = true;
            try {
                this.items = await this.$api.get('/api/endpoint');
            } finally {
                this.loading = false;
            }
        }
    }
};
```

### template.html

Pure HTML template (no `<template>` wrapper):

```html
<div class="my-component">
    <div v-if="loading" class="text-center pa-4">
        <v-progress-circular indeterminate color="primary" />
    </div>
    <div v-else>
        <!-- Component content using Vuetify components -->
    </div>
</div>
```

### style.css

Scoped styles under a component class:

```css
.my-component {
    padding: 16px;
}
.my-component .header {
    font-size: 1.5em;
    font-weight: bold;
}
```

---

## webhooks.json — Webhook Subscriptions

Array of webhook configurations that subscribe to Zoezi events.

```json
[
    {
        "name": "todo",
        "useFunction": false,
        "functionName": null,
        "url": "https://your-backend.com/api/webhooks/todo-updated",
        "events": ["todo"],
        "actions": ["update"]
    }
]
```

**Fields:**
- `name`: Identifier for the webhook
- `useFunction`: `true` to route to a cloud function instead of URL
- `functionName`: Cloud function name (if `useFunction: true`)
- `url`: External URL to receive the webhook (if `useFunction: false`)
- `events`: Array of event types to subscribe to
- `actions`: Array of action types (e.g., `"create"`, `"update"`, `"delete"`)

**Common events:** `"todo"`, `"member"`, `"booking"`, `"payment"`, `"trainingcard"`

---

## apis.json — Custom API Endpoints

Define custom API endpoints that route through the Zoezi platform.

```json
[
    {
        "name": "my-proxy",
        "path": "/my-addon/proxy",
        "method": "POST",
        "functionName": "abc123...",
        "description": null,
        "session": true,
        "permission": null,
        "params": []
    }
]
```

**Fields:**
- `name`: API identifier
- `path`: URL path on the Zoezi platform
- `method`: HTTP method (`"GET"`, `"POST"`, etc.)
- `functionName`: Hash of the cloud function that handles the request
- `session`: `true` to require an authenticated Zoezi session
- `permission`: Required permission (e.g., `"Admin"`) or `null` for any authenticated user
- `params`: Array of parameter definitions

---

## widgets.json — Staff Dashboard Widgets

Register components as staff dashboard widgets.

```json
[
    {
        "name": "My Admin Widget",
        "type": "staffdashboard",
        "subtype": "general",
        "invocationType": "component",
        "functionName": null,
        "url": null,
        "component": "abc123...",
        "permission": "Show customers",
        "settings": [],
        "id": "def456..."
    }
]
```

**Fields:**
- `name`: Display name in the staff dashboard
- `type`: Widget type (`"staffdashboard"`)
- `subtype`: `"general"` for standard widgets
- `invocationType`: `"component"` to render a Vue component
- `component`: Hash of the component (from `components/{name}/config.json` id)
- `permission`: Required Zoezi permission to see the widget
- `settings`: Array of configurable settings

---

## settings.json — Addon Settings Schema

Define addon-level settings configurable by gym admins. Empty array if no settings needed.

```json
[]
```

---

## functions/ — Cloud Functions

Each function has a directory with `config.json` and `code.js`.

### config.json

```json
{
    "name": "my-function",
    "language": "nodejs20.x",
    "template": "api",
    "timeout": "6000",
    "id": "abc123...",
    "testInput": null
}
```

- `language`: Runtime (`"nodejs20.x"`)
- `template`: Function type (`"api"` for API-backed functions)
- `timeout`: Max execution time in milliseconds

### code.js

Cloud function handler:

```javascript
module.exports = async function(event, context) {
    // event.body - request body
    // event.headers - request headers
    // context.api - Zoezi API client
    // context.session - authenticated session

    const result = await context.api.get('/api/endpoint');

    return {
        statusCode: 200,
        body: JSON.stringify(result)
    };
};
```

---

## Deployment Workflow

```bash
cd ~/zoezi-component

# 1. Always pull first (web interface may have made changes)
git pull origin main

# 2. Make your changes to components, webhooks, settings, etc.

# 3. Deploy
git add . && git commit -m "Add new component" && git push origin main
```

Changes are **live immediately** after push. There is no build step or CI pipeline — the developer portal reads the repo directly.
