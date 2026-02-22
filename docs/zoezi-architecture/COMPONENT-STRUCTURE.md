# Zoezi Component Structure

This document explains how to create and structure Zoezi Vue components.

## Component Template

Every Zoezi component follows this structure:

```vue
<style lang="scss">
.zoezi-mycomponent {
  /* Component-specific styles */
}
</style>

<template>
  <div class="zoezi-mycomponent">
    <!-- Component markup -->
  </div>
</template>

<script>
export default {
  name: 'zoezi-mycomponent',  // REQUIRED: Must start with 'zoezi-'

  // Zoezi metadata - makes component available in page builder
  zoezi: {
    title: 'My Component',      // Display name in editor
    icon: 'mdi-star',           // Material Design Icon
    addon: 'ModuleName',        // Optional: Required module
    getDefaultStyle: () => ({   // Optional: URL parameter mapping
      pageParams: [
        { id: 'tab', prop: 'activeTab' }
      ]
    })
  },

  // Props are configurable in the page builder
  props: {
    // Simple prop
    showHeader: {
      title: 'Show header',     // Label in editor
      type: Boolean,
      default: true
    },

    // Select prop (dropdown)
    displayMode: {
      title: 'Display mode',
      type: String,
      fulltype: 'select',
      typeinfo: {
        getItems() {
          return [
            { id: 'grid', name: 'Grid view' },
            { id: 'list', name: 'List view' }
          ];
        },
        text: 'name',
        value: 'id'
      },
      default: 'grid'
    },

    // Multi-select with API data
    filterCategories: {
      title: 'Filter by categories',
      type: Array,
      fulltype: 'select',
      typeinfo: {
        getItems() {
          return window.$zoeziapi.get('/api/public/settings/get')
            .then(settings => settings.webshopcategories);
        },
        text: 'name',
        value: 'id',
        multiple: true,
        placeholder: 'Show all'
      },
      default: () => []
    },

    // Page link prop
    linkAfterAction: {
      title: 'Go to page after action',
      type: String,
      fulltype: 'page',    // Shows page picker in editor
      default: '/'
    }
  },

  components: {
    // Register child components
  },

  data: () => ({
    // Component state
    loading: false,
    items: []
  }),

  computed: {
    // Computed properties
    filteredItems() {
      return this.items.filter(/* ... */);
    }
  },

  watch: {
    // Watchers
    '$store.state.user': {
      immediate: true,
      handler(user) {
        if (user) this.loadData();
      }
    }
  },

  methods: {
    // Methods
    async loadData() {
      this.loading = true;
      try {
        this.items = await this.$api.get('/api/endpoint');
      } finally {
        this.loading = false;
      }
    }
  },

  mounted() {
    // Lifecycle hook
    this.loadData();
  }
}
</script>
```

## Zoezi Metadata Object

The `zoezi` object makes components discoverable in the page builder:

```javascript
zoezi: {
  // REQUIRED: Display name
  title: 'My Component',

  // REQUIRED: Icon from Material Design Icons
  // Browse at: https://pictogrammers.com/library/mdi/
  icon: 'mdi-account-circle',

  // OPTIONAL: Required addon/module
  // Component only shows if customer has this license
  addon: 'WebShop',

  // OPTIONAL: URL parameter mapping
  // Allows component state to be reflected in URL
  getDefaultStyle: () => ({
    pageParams: [
      {
        id: 'tab',           // URL param name (?tab=value)
        prop: 'activeTab'    // Component prop to sync
      }
    ]
  })
}
```

## Prop Types

### Basic Types

```javascript
props: {
  // Boolean - renders as checkbox
  enabled: {
    title: 'Enable feature',
    type: Boolean,
    default: false
  },

  // String - renders as text input
  title: {
    title: 'Title text',
    type: String,
    default: ''
  },

  // Number - renders as number input
  limit: {
    title: 'Item limit',
    type: Number,
    default: 10
  },

  // Array - usually with fulltype: 'select'
  items: {
    title: 'Selected items',
    type: Array,
    default: () => []  // NOTE: Arrays/Objects need factory function
  }
}
```

### Select Type (Dropdown)

```javascript
props: {
  // Static options
  sortOrder: {
    title: 'Sort order',
    type: String,
    fulltype: 'select',
    typeinfo: {
      getItems() {
        return [
          { id: 'name-asc', name: 'Name (A-Z)' },
          { id: 'name-desc', name: 'Name (Z-A)' },
          { id: 'price-asc', name: 'Price (Low to High)' }
        ];
      },
      text: 'name',     // Property to display
      value: 'id'       // Property for value
    },
    default: 'name-asc'
  },

  // Dynamic options from API
  selectedSites: {
    title: 'Filter by sites',
    type: Array,
    fulltype: 'select',
    typeinfo: {
      getItems() {
        return window.$zoeziapi.get('/api/public/settings/get')
          .then(settings => settings.sites);
      },
      text: 'name',
      value: 'id',
      multiple: true,           // Allow multiple selection
      placeholder: 'All sites'  // Placeholder text
    },
    default: () => []
  },

  // Conditional visibility
  siteFilter: {
    title: 'Site filter',
    type: Array,
    fulltype: 'select',
    typeinfo: {
      getItems() { /* ... */ },
      visible: '!useGlobalSite'  // Only show if useGlobalSite is false
    },
    default: () => []
  }
}
```

### Page Link Type

```javascript
props: {
  redirectPage: {
    title: 'Redirect to page',
    type: String,
    fulltype: 'page',  // Opens page picker
    default: '/'
  }
}
```

## URL Parameter Syncing

Sync component state with URL for shareable links:

```javascript
export default {
  zoezi: {
    getDefaultStyle: () => ({
      pageParams: [
        { id: 'tab', prop: 'activeTab' },
        { id: 'p', prop: 'productId' }
      ]
    })
  },

  props: {
    activeTab: {
      title: 'Active tab',
      type: String,
      default: 'overview'
    },
    productId: {
      title: 'Product ID',
      type: Number
    }
  },

  watch: {
    activeTab(value) {
      // Update URL when prop changes
      this.setProp('activeTab', value);
    }
  },

  methods: {
    selectTab(tab) {
      // setProp is provided by a mixin
      this.setProp('activeTab', tab);
    }
  }
}
```

URL will be: `/page?tab=payments&p=123`

## Common Patterns

### Loading State

```vue
<template>
  <div>
    <div v-if="loading" class="text-center">
      {{ $translate('Loading...') }}
      <v-progress-circular indeterminate color="primary" />
    </div>
    <div v-else>
      <!-- Content -->
    </div>
  </div>
</template>

<script>
export default {
  data: () => ({
    loading: false,
    items: []
  }),

  async mounted() {
    this.loading = true;
    try {
      this.items = await this.$api.get('/api/items');
    } finally {
      this.loading = false;
    }
  }
}
</script>
```

### Authentication Check

```vue
<template>
  <div>
    <!-- Show content only when logged in -->
    <div v-if="$store.state.user">
      <h1>Welcome, {{ $store.state.user.firstname }}</h1>
    </div>

    <!-- Show login when not authenticated -->
    <zoezi-identification v-else :title="$translate('Please log in')" />
  </div>
</template>
```

### Reacting to User State

```javascript
export default {
  watch: {
    '$store.state.user': {
      immediate: true,
      handler(user) {
        if (user) {
          this.loadUserData();
        } else {
          this.clearData();
        }
      }
    }
  }
}
```

### Multi-Site Filtering

```javascript
export default {
  computed: {
    filteredProducts() {
      const siteId = this.$store.state.selectedSiteId;
      return this.products.filter(p => {
        // Products with no site restriction or matching site
        return !p.sites || p.sites.length === 0 || p.sites.includes(siteId);
      });
    }
  }
}
```

### Dialog Pattern

```vue
<template>
  <div>
    <v-btn @click="showDialog = true">Open Dialog</v-btn>

    <v-dialog v-model="showDialog" max-width="600">
      <v-card>
        <v-card-title class="headline pa-2 primary primarytext--text d-flex">
          {{ $translate('Dialog Title') }}
          <v-spacer />
          <v-btn icon @click="showDialog = false">
            <v-icon class="primarytext--text">mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        <v-card-text>
          <!-- Dialog content -->
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn color="primary" @click="confirm">
            {{ $translate('Confirm') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
export default {
  data: () => ({
    showDialog: false
  }),

  methods: {
    confirm() {
      // Handle confirmation
      this.showDialog = false;
    }
  }
}
</script>
```

## Deployed Component File Structure

When components are deployed to the Zoezi Developer Portal (via the addon git repo), each component is split into **4 separate files** in its own directory:

```
components/MY-COMPONENT/
├── config.json      # Component metadata (id, name, type)
├── code.js          # JavaScript — Vue export default { ... }
├── template.html    # HTML template (no <template> wrapper)
└── style.css        # CSS styles (no <style> tag, plain CSS)
```

### config.json
```json
{
    "id": "auto-generated-hash",
    "name": "MY-COMPONENT",
    "dependencies": null,
    "type": "vue2"
}
```

### code.js
Standard Vue 2 export — same as the `<script>` section of a `.vue` file:
```javascript
export default {
    data() {
        return { loading: false, items: [] };
    },
    methods: {
        async loadData() {
            this.items = await this.$api.get('/api/endpoint');
        }
    }
};
```

### template.html
Pure HTML — same as the contents inside `<template>` in a `.vue` file:
```html
<div class="my-component">
    <div v-if="loading" class="text-center pa-4">
        <v-progress-circular indeterminate color="primary" />
    </div>
    <div v-else>
        <!-- Content -->
    </div>
</div>
```

### style.css
Plain CSS — same as the contents inside `<style>` in a `.vue` file:
```css
.my-component {
    padding: 16px;
}
.my-component .header {
    font-size: 1.5em;
}
```

> **Note:** The deployed format splits what would be a single `.vue` file into 4 files. The `code.js` / `template.html` / `style.css` map directly to the `<script>` / `<template>` / `<style>` sections. SCSS is not supported in `style.css` — use plain CSS.

---

## Component Registration

Components are automatically registered when:

1. Named with `zoezi-` prefix
2. Exported from a `.vue` file in the components directory
3. Included in `component.js` registry

The registration happens in `main.js`:

```javascript
components.registerVueComponents(homepage.components, config, homepage_type);
```

## Styling Guidelines

### Use Scoped Class Names

```vue
<style lang="scss">
.zoezi-mycomponent {
  // All styles scoped under component class

  .header {
    font-size: 1.5em;
  }

  .item {
    padding: 10px;

    &:hover {
      background: rgba(0, 0, 0, 0.05);
    }
  }
}
</style>

<template>
  <div class="zoezi-mycomponent">
    <div class="header">...</div>
    <div class="item">...</div>
  </div>
</template>
```

### Responsive Design

```scss
.zoezi-mycomponent {
  // Mobile first
  .grid {
    display: grid;
    grid-template-columns: 1fr;
    gap: 10px;
  }

  // Tablet
  @media (min-width: 600px) {
    .grid {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  // Desktop
  @media (min-width: 960px) {
    .grid {
      grid-template-columns: repeat(4, 1fr);
    }
  }
}
```

### Use Vuetify Classes

Prefer Vuetify utility classes over custom CSS:

```vue
<template>
  <!-- Spacing -->
  <div class="pa-4 ma-2 mx-auto">

  <!-- Flexbox -->
  <div class="d-flex flex-column align-center justify-space-between">

  <!-- Grid -->
  <v-row>
    <v-col cols="12" md="6" lg="4">

  <!-- Typography -->
  <div class="text-h4 font-weight-bold text--secondary">

  <!-- Colors -->
  <div class="primary--text">
  <div class="error white--text">
</template>
```
