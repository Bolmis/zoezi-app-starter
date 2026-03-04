# Supabase Patterns for Zoezi Addons

Production-proven database and storage patterns for multi-tenant Zoezi addons.

---

## Setup

### 1. Create Supabase Project

1. Go to https://supabase.com and create a new project
2. Copy the project URL and **service role key** (not anon key)
3. Add to environment variables:

```bash
SUPABASE_URL=https://xxxxxxxxxxxxx.supabase.co
SUPABASE_KEY=eyJhbGciOiJIUzI1NiIs...  # Service role key
```

### 2. Initialize Client

```javascript
// src/services/supabase.js
const { createClient } = require('@supabase/supabase-js');

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_KEY
);

module.exports = supabase;
```

### 3. MCP Integration (for AI-assisted development)

Create `.mcp.json` in your project root so Claude/Codex can query your database directly:

```json
{
  "mcpServers": {
    "supabase": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-supabase", "--project-ref", "your-project-ref"]
    }
  }
}
```

---

## Multi-Tenant Database Design

### The Hostname Pattern

**Every table MUST include a `hostname` column.** This is the multi-tenancy key — one backend serves many gyms, and `hostname` isolates their data.

```sql
CREATE TABLE IF NOT EXISTS your_settings (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  hostname TEXT NOT NULL UNIQUE,        -- Multi-tenancy key (e.g., "mygym.zoezi.se")
  setting_name TEXT,
  setting_value JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ALWAYS index hostname
CREATE INDEX idx_your_settings_hostname ON your_settings(hostname);
```

### Standard Table Template

```sql
CREATE TABLE IF NOT EXISTS your_table (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  hostname TEXT NOT NULL,               -- Always required
  user_id INTEGER,                      -- Zoezi user ID (integer, not UUID)
  -- Your fields here
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Performance indexes
CREATE INDEX idx_your_table_hostname ON your_table(hostname);
CREATE INDEX idx_your_table_user ON your_table(hostname, user_id);

-- Unique constraints always include hostname
ALTER TABLE your_table ADD CONSTRAINT uq_your_table_unique
  UNIQUE(hostname, some_unique_field);

-- Row Level Security
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Service role full access" ON your_table FOR ALL USING (true);
```

### Common Column Patterns

```sql
-- Zoezi user reference (integer, not UUID!)
user_id INTEGER NOT NULL,

-- Zoezi group reference
zoezi_group_id INTEGER,

-- Status tracking
status TEXT DEFAULT 'pending',  -- pending / approved / rejected / expired

-- Foreign keys to other addon tables
parent_id UUID REFERENCES parent_table(id) ON DELETE CASCADE,

-- Unique per gym
UNIQUE(hostname, secret_code)
```

---

## Query Patterns

### CRITICAL: Always Filter by Hostname

```javascript
// CORRECT - always filter by hostname
const { data, error } = await supabase
  .from('your_table')
  .select('*')
  .eq('hostname', hostname);

// WRONG - never query without hostname
const { data, error } = await supabase
  .from('your_table')
  .select('*');  // This leaks data across gyms!
```

### Get Settings (upsert pattern)

```javascript
async function getSettings(hostname) {
  const { data, error } = await supabase
    .from('settings')
    .select('*')
    .eq('hostname', hostname)
    .maybeSingle();  // Returns null if not found (no error)

  if (error) throw error;
  return data;
}

async function saveSettings(hostname, settings) {
  const { data, error } = await supabase
    .from('settings')
    .upsert({
      hostname,
      ...settings,
      updated_at: new Date().toISOString()
    }, {
      onConflict: 'hostname'
    })
    .select()
    .single();

  if (error) throw error;
  return data;
}
```

### CRUD Operations

```javascript
// List records for a gym
async function listRecords(hostname) {
  const { data, error } = await supabase
    .from('records')
    .select('*')
    .eq('hostname', hostname)
    .order('created_at', { ascending: false });

  if (error) throw error;
  return data || [];
}

// Get single record (with hostname check!)
async function getRecord(hostname, id) {
  const { data, error } = await supabase
    .from('records')
    .select('*')
    .eq('hostname', hostname)
    .eq('id', id)
    .single();

  if (error) throw error;
  return data;
}

// Create record
async function createRecord(hostname, fields) {
  const { data, error } = await supabase
    .from('records')
    .insert({ hostname, ...fields })
    .select()
    .single();

  if (error) throw error;
  return data;
}

// Update record (always verify hostname ownership)
async function updateRecord(hostname, id, updates) {
  const { data, error } = await supabase
    .from('records')
    .update({ ...updates, updated_at: new Date().toISOString() })
    .eq('hostname', hostname)
    .eq('id', id)
    .select()
    .single();

  if (error) throw error;
  return data;
}

// Delete record
async function deleteRecord(hostname, id) {
  const { error } = await supabase
    .from('records')
    .delete()
    .eq('hostname', hostname)
    .eq('id', id);

  if (error) throw error;
}
```

### Join Queries

```javascript
// Get records with related data
async function getRecordsWithRelations(hostname) {
  const { data, error } = await supabase
    .from('companies')
    .select(`
      *,
      discount_mapping:discount_mappings(id, name, percentage)
    `)
    .eq('hostname', hostname)
    .eq('is_active', true)
    .order('company_name');

  if (error) throw error;
  return data || [];
}
```

---

## Supabase Storage

### Setup Storage Bucket

```javascript
const BUCKET_NAME = 'your-files';

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
```

### Upload File

```javascript
async function uploadFile(hostname, file) {
  const { v4: uuidv4 } = require('uuid');
  const storedName = `${uuidv4()}_${file.originalname}`;

  const { data, error } = await supabase.storage
    .from(BUCKET_NAME)
    .upload(storedName, file.buffer, {
      contentType: file.mimetype,
      upsert: false
    });

  if (error) throw error;
  return storedName;
}
```

### Get Signed Download URL

```javascript
async function getSignedUrl(filename, expiresIn = 3600) {
  const { data, error } = await supabase.storage
    .from(BUCKET_NAME)
    .createSignedUrl(filename, expiresIn);

  if (error) throw error;
  return data.signedUrl;
}
```

### Delete File

```javascript
async function deleteFile(filename) {
  const { error } = await supabase.storage
    .from(BUCKET_NAME)
    .remove([filename]);

  if (error) throw error;
}
```

---

## Row Level Security (RLS)

For addons using a **service role key** (recommended), a simple policy works:

```sql
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;

-- Service role bypasses RLS, but this policy ensures safety
CREATE POLICY "Service role full access" ON your_table
  FOR ALL USING (true);
```

The service role key bypasses RLS entirely, but enabling RLS is still good practice — it protects against accidental access from the anon key.

---

## Migration Best Practices

1. **Keep migrations in order** — Name them with timestamps or sequence numbers
2. **Always include hostname** — Every table, every index, every unique constraint
3. **Use UUIDs for primary keys** — `gen_random_uuid()` is built-in
4. **Use INTEGER for Zoezi IDs** — Zoezi user/group/status IDs are integers
5. **Add indexes on hostname** — Every table should have `CREATE INDEX ... ON ...(hostname)`
6. **Enable RLS on every table** — Even with service role access
7. **Use CASCADE for deletions** — `ON DELETE CASCADE` for child records

---

## Example: Complete Settings Table

A typical addon settings table with all best practices:

```sql
-- Settings table (one row per gym)
CREATE TABLE IF NOT EXISTS addon_settings (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  hostname TEXT NOT NULL UNIQUE,
  feature_enabled BOOLEAN DEFAULT false,
  welcome_message TEXT DEFAULT '',
  config JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_addon_settings_hostname ON addon_settings(hostname);
ALTER TABLE addon_settings ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Service role full access" ON addon_settings FOR ALL USING (true);
```

Service layer:

```javascript
async function getSettings(hostname) {
  const { data } = await supabase
    .from('addon_settings')
    .select('*')
    .eq('hostname', hostname)
    .maybeSingle();
  return data;
}

async function saveSettings(hostname, settings) {
  const { data, error } = await supabase
    .from('addon_settings')
    .upsert({
      hostname,
      ...settings,
      updated_at: new Date().toISOString()
    }, { onConflict: 'hostname' })
    .select()
    .single();

  if (error) throw error;
  return data;
}
```
