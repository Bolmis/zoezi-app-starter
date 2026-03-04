# Zoezi API Reference

**Version:** 1.0
**Last Updated:** 2026-01-13
**Purpose:** Complete reference of Zoezi API endpoints used in add-on development

---

## Table of Contents

1. [Authentication](#authentication)
2. [User Management](#user-management)
3. [Member Management](#member-management)
4. [Group Management](#group-management)
5. [Custom Fields](#custom-fields)
6. [TODO Management](#todo-management)
7. [Discount/Trainingcard Management](#discounttrainingcard-management)
8. [Session Management](#session-management)
9. [Response Formats](#response-formats)
10. [Error Handling](#error-handling)

---

## Authentication

Zoezi supports two authentication methods:

### API Key Authentication

Used for backend-initiated operations (creating resources, updating system data).

**Header Format:**
```
Authorization: Zoezi {API_KEY}
```

**Example:**
```javascript
const headers = {
  'Content-Type': 'application/json',
  'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
};
```

### Session Cookie Authentication

Used for user-context operations (reading user data, managing user's groups).

**Header Format:**
```
Cookie: session={sessionRaw}
```

**Example:**
```javascript
const headers = {
  'Content-Type': 'application/json',
  'Cookie': `session=${sessionRaw}`
};
```

---

## User Management

### Get Current User

Get the currently logged-in user.

**Endpoint:** `GET /api/memberapi/get/current`
**Auth:** Session Cookie
**Parameters:** None

**Response:**
```json
{
  "id": 12345,
  "name": "John Doe",
  "email": "john@example.com",
  "staff": false,
  "groups": [1, 2, 5],
  "status": [3, 7],
  "xf": {
    "custom_field_1": "value1",
    "custom_field_2": "value2"
  },
  "phone": "+46701234567",
  "address": "Street 123",
  "city": "Stockholm",
  "zipcode": "12345",
  "country": "Sweden"
}
```

**Example:**
```javascript
async function getCurrentUser(hostname, sessionRaw) {
  const response = await fetch(`https://${hostname}/api/memberapi/get/current`, {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json',
      'Cookie': `session=${sessionRaw}`
    }
  });
  return response.json();
}
```

---

### Get Specific User

Get a specific user by ID.

**Endpoint:** `GET /api/user/get?id={userId}`
**Auth:** Session Cookie
**Parameters:**
- `id` (integer) - User ID

**Response:**
```json
{
  "id": 12345,
  "name": "John Doe",
  "email": "john@example.com",
  "staff": false,
  "groups": [1, 2, 5],
  "status": [3, 7],
  "xf": {
    "custom_field_1": "value1"
  }
}
```

**Example:**
```javascript
async function getUser(hostname, sessionRaw, userId) {
  const response = await fetch(
    `https://${hostname}/api/user/get?id=${userId}`,
    {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': `session=${sessionRaw}`
      }
    }
  );
  return response.json();
}
```

---

### Update User

Update user information, groups, and extra fields.

**Endpoint:** `POST /api/user/change`
**Auth:** API Key
**Method:** POST

**Request Body:**
```json
{
  "id": 12345,
  "name": "Updated Name",
  "email": "newemail@example.com",
  "groups": [1, 2, 5, 8],
  "xf": {
    "custom_field_1": "new_value",
    "custom_field_2": "another_value"
  }
}
```

**Fields:**
- `id` (integer, required) - User ID
- `name` (string, optional) - User's full name
- `email` (string, optional) - Email address
- `groups` (array, optional) - Array of group IDs
- `xf` (object, optional) - Extra fields (custom data)
- Other user fields as needed

**Response:**
```json
{
  "success": true,
  "user": {
    "id": 12345,
    "name": "Updated Name",
    "email": "newemail@example.com"
  }
}
```

**Example:**
```javascript
async function updateUser(hostname, userId, updates) {
  const response = await fetch(`https://${hostname}/api/user/change`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
    },
    body: JSON.stringify({
      id: userId,
      ...updates
    })
  });
  return response.json();
}
```

---

## Member Management

### Get All Member Statuses

Get all available member statuses.

**Endpoint:** `GET /api/member/status/get/all`
**Auth:** API Key
**Parameters:** None

**Response:**
```json
[
  {
    "id": 1,
    "name": "Active Member",
    "color": "#00ff00"
  },
  {
    "id": 2,
    "name": "Inactive",
    "color": "#cccccc"
  },
  {
    "id": 3,
    "name": "discount(CORP10)",
    "color": "#ff9900"
  }
]
```

**Example:**
```javascript
async function getAllStatuses(hostname) {
  const response = await fetch(
    `https://${hostname}/api/member/status/get/all`,
    {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
      }
    }
  );
  return response.json();
}
```

---

### Create Member Status

Create a new member status.

**Endpoint:** `POST /api/member/status/add`
**Auth:** API Key
**Method:** POST

**Request Body:**
```json
{
  "name": "discount(CORP10)",
  "color": "#ff9900"
}
```

**Response:**
```json
{
  "success": true,
  "status": {
    "id": 15,
    "name": "discount(CORP10)",
    "color": "#ff9900"
  }
}
```

**Example:**
```javascript
async function createStatus(hostname, name, color = '#cccccc') {
  const response = await fetch(
    `https://${hostname}/api/member/status/add`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
      },
      body: JSON.stringify({ name, color })
    }
  );
  return response.json();
}
```

---

### Update Member

Update member status, groups, and extra fields.

**Endpoint:** `POST /api/member/change`
**Auth:** API Key
**Method:** POST

**Request Body:**
```json
{
  "id": 12345,
  "status": [1, 3, 7],
  "groups": [2, 5],
  "xf": {
    "employment_verified": "2026-01-13"
  }
}
```

**Fields:**
- `id` (integer, required) - User ID
- `status` (array, optional) - Array of status IDs
- `groups` (array, optional) - Array of group IDs
- `xf` (object, optional) - Extra fields

**Response:**
```json
{
  "success": true,
  "member": {
    "id": 12345,
    "status": [1, 3, 7],
    "groups": [2, 5]
  }
}
```

**Example:**
```javascript
async function updateMember(hostname, userId, updates) {
  const response = await fetch(`https://${hostname}/api/member/change`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
    },
    body: JSON.stringify({
      id: userId,
      ...updates
    })
  });
  return response.json();
}
```

---

## Group Management

### Get All Groups

Get all groups in the system.

**Endpoint:** `GET /api/usergroup/get/all`
**Auth:** Session Cookie
**Parameters:** None

**Response:**
```json
[
  {
    "id": 1,
    "name": "Staff",
    "users": [1, 2, 3],
    "responsible": [1],
    "change_permission": false,
    "show": true,
    "teamsport_id": null
  },
  {
    "id": 5,
    "name": "Company ABC",
    "users": [10, 15, 20],
    "responsible": [],
    "change_permission": false,
    "show": true,
    "teamsport_id": null
  }
]
```

**Example:**
```javascript
async function getAllGroups(hostname, sessionRaw) {
  const response = await fetch(
    `https://${hostname}/api/usergroup/get/all`,
    {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': `session=${sessionRaw}`
      }
    }
  );
  return response.json();
}
```

---

### Get Specific Group

Get a specific group by ID.

**Endpoint:** `GET /api/usergroup/get?id={groupId}`
**Auth:** Session Cookie
**Parameters:**
- `id` (integer) - Group ID

**Response:**
```json
{
  "id": 5,
  "name": "Company ABC",
  "users": [10, 15, 20],
  "responsible": [],
  "change_permission": false,
  "show": true,
  "teamsport_id": null
}
```

**Example:**
```javascript
async function getGroup(hostname, sessionRaw, groupId) {
  const response = await fetch(
    `https://${hostname}/api/usergroup/get?id=${groupId}`,
    {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': `session=${sessionRaw}`
      }
    }
  );
  return response.json();
}
```

---

### Create Group

Create a new user group.

**Endpoint:** `POST /api/usergroup/add`
**Auth:** Session Cookie
**Method:** POST

**Request Body:**
```json
{
  "name": "New Company Group",
  "change_permission": false,
  "users": [],
  "responsible": [],
  "show": true,
  "teamsport_id": null
}
```

**Fields:**
- `name` (string, required) - Group name
- `change_permission` (boolean, optional) - Can users change their own membership?
- `users` (array, optional) - Initial user IDs
- `responsible` (array, optional) - User IDs of group managers
- `show` (boolean, optional) - Show group in UI
- `teamsport_id` (integer, optional) - Link to teamsport team

**Response:**
```json
{
  "success": true,
  "group": {
    "id": 25,
    "name": "New Company Group",
    "users": [],
    "change_permission": false,
    "show": true
  }
}
```

**Example:**
```javascript
async function createGroup(hostname, sessionRaw, name) {
  const response = await fetch(
    `https://${hostname}/api/usergroup/add`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': `session=${sessionRaw}`
      },
      body: JSON.stringify({
        name,
        change_permission: false,
        users: [],
        responsible: [],
        show: true
      })
    }
  );
  return response.json();
}
```

---

### Update Group

Update group information and membership.

**Endpoint:** `POST /api/usergroup/change`
**Auth:** Session Cookie
**Method:** POST

**Request Body:**
```json
{
  "id": 25,
  "name": "Updated Group Name",
  "users": [10, 15, 20, 25],
  "responsible": [1],
  "show": true
}
```

**Fields:**
- `id` (integer, required) - Group ID
- `name` (string, optional) - New group name
- `users` (array, optional) - Complete array of user IDs (replaces existing)
- `responsible` (array, optional) - Group managers
- `show` (boolean, optional) - Visibility
- Other group fields as needed

**Response:**
```json
{
  "success": true,
  "group": {
    "id": 25,
    "name": "Updated Group Name",
    "users": [10, 15, 20, 25]
  }
}
```

**Example:**
```javascript
async function updateGroup(hostname, sessionRaw, groupId, updates) {
  const response = await fetch(
    `https://${hostname}/api/usergroup/change`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': `session=${sessionRaw}`
      },
      body: JSON.stringify({
        id: groupId,
        ...updates
      })
    }
  );
  return response.json();
}
```

**Note:** The `users` array is replaced completely, not appended to. To add a user to a group:
1. Get current group data
2. Merge new user ID into existing users array
3. Send complete array in update

---

## Custom Fields

### Get All Custom Fields

Get all custom field definitions for the system.

**Endpoint:** `GET /api/settings/fields/get`
**Auth:** Session Cookie
**Parameters:** None

**Response:**
```json
{
  "user": [
    {
      "name": "employment_verified",
      "label": "Employment Verified Date",
      "type": "text",
      "required": false
    },
    {
      "name": "company_code",
      "label": "Company Code",
      "type": "text",
      "required": false
    }
  ],
  "booking": [],
  "membership": [],
  "product": []
}
```

**Example:**
```javascript
async function getAllFields(hostname, sessionRaw) {
  const response = await fetch(
    `https://${hostname}/api/settings/fields/get`,
    {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': `session=${sessionRaw}`
      }
    }
  );
  return response.json();
}
```

---

## TODO Management

### Create TODO

Create a TODO task assigned to a user.

**Endpoint:** `POST /api/todo/add`
**Auth:** API Key
**Method:** POST

**Request Body:**
```json
{
  "todo": "Review employment proof for John Doe",
  "dueDate": "2026-01-20",
  "user_id": 5,
  "description": "<p>User John Doe has uploaded employment proof.</p><p>File: <a href='https://example.com/file'>Download</a></p>"
}
```

**Fields:**
- `todo` (string, required) - TODO title
- `dueDate` (string, optional) - Due date in YYYY-MM-DD format
- `user_id` (integer, required) - Assignee user ID (staff member)
- `description` (string, optional) - HTML description with details/links

**Response:**
```json
{
  "success": true,
  "todo": {
    "id": 123,
    "todo": "Review employment proof for John Doe",
    "dueDate": "2026-01-20",
    "user_id": 5
  }
}
```

**Example:**
```javascript
async function createTodo(hostname, title, userId, dueDate, description) {
  const response = await fetch(`https://${hostname}/api/todo/add`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
    },
    body: JSON.stringify({
      todo: title,
      dueDate,
      user_id: userId,
      description
    })
  });
  return response.json();
}
```

---

## Discount/Trainingcard Management

### Create Discount Type (Trainingcard Type)

Create a new discount/trainingcard type.

**Endpoint:** `POST /api/trainingcard/type/add`
**Auth:** API Key
**Method:** POST

**Request Body:**
```json
{
  "name": "Corporate Discount 10%",
  "discount_code": "CORP10",
  "discount_percent": "10",
  "discount_condition": {
    "trainingcards": true,
    "memberships": true,
    "punchcards": true,
    "products": true,
    "courses": true,
    "services": true,
    "user_status": [15, 20]
  },
  "discount": true,
  "validFromDate": "2026-01-01"
}
```

**Fields:**
- `name` (string, required) - Display name
- `discount_code` (string, required) - Code users enter at checkout
- `discount_percent` (string, required) - Percentage as string (e.g., "10")
- `discount_condition` (object, required) - What the discount applies to
  - `trainingcards` (boolean) - Apply to trainingcards
  - `memberships` (boolean) - Apply to memberships
  - `punchcards` (boolean) - Apply to punchcards
  - `products` (boolean) - Apply to products
  - `courses` (boolean) - Apply to courses
  - `services` (boolean) - Apply to services
  - `user_status` (array) - Only for users with these status IDs
- `discount` (boolean, required) - Must be true for discounts
- `validFromDate` (string, optional) - Start date YYYY-MM-DD

**Response:**
```json
{
  "success": true,
  "trainingcard_type": {
    "id": 45,
    "name": "Corporate Discount 10%",
    "discount_code": "CORP10",
    "discount_percent": "10"
  }
}
```

**Example:**
```javascript
async function createDiscount(hostname, name, code, percent, statusIds) {
  const response = await fetch(
    `https://${hostname}/api/trainingcard/type/add`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Zoezi ${process.env.ZOEZI_API_KEY}`
      },
      body: JSON.stringify({
        name,
        discount_code: code,
        discount_percent: String(percent),
        discount_condition: {
          trainingcards: true,
          memberships: true,
          punchcards: true,
          products: true,
          courses: true,
          services: true,
          user_status: statusIds
        },
        discount: true,
        validFromDate: new Date().toISOString().split('T')[0]
      })
    }
  );
  return response.json();
}
```

---

## Session Management

### Verify Session

Verify that a session hash is valid for a user.

**Endpoint:** `GET /api/user/session/verify?hash={sessionHash}&user_id={userId}`
**Auth:** None (public endpoint)
**Parameters:**
- `hash` (string) - SHA-256 hash of session cookie
- `user_id` (integer) - User ID to verify

**Response:**
```json
{
  "valid": true
}
```

Or if invalid:
```json
{
  "valid": false
}
```

**Example:**
```javascript
async function verifySession(hostname, userId, sessionHash) {
  const url = `https://${hostname}/api/user/session/verify?hash=${encodeURIComponent(sessionHash)}&user_id=${userId}`;
  const response = await fetch(url);
  const data = await response.json();
  return data.valid === true;
}
```

---

## Response Formats

### Success Response

Standard successful response format:

```json
{
  "success": true,
  "data": { /* response data */ },
  "message": "Operation completed successfully"
}
```

### Error Response

Standard error response format:

```json
{
  "success": false,
  "error": "Error message describing what went wrong"
}
```

---

## Error Handling

### Common HTTP Status Codes

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 200 | OK | Successful request |
| 400 | Bad Request | Missing or invalid parameters |
| 401 | Unauthorized | Invalid or expired session |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 500 | Server Error | Internal error on Zoezi side |

### Error Handling Pattern

```javascript
async function safeApiCall(hostname, sessionRaw, endpoint) {
  try {
    const response = await fetch(`https://${hostname}${endpoint}`, {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': `session=${sessionRaw}`
      }
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const data = await response.json();

    if (data.success === false) {
      throw new Error(data.error || 'API returned error');
    }

    return data;
  } catch (err) {
    console.error('API call failed:', err);
    throw err;
  }
}
```

---

## Helper Functions

### Complete Zoezi Service Module

Here's a complete service module with all common operations:

```javascript
// src/services/zoezi.js
const ZOEZI_API_KEY = process.env.ZOEZI_API_KEY;

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

// Session authenticated POST
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

// User operations
async function getCurrentUser(hostname, sessionRaw) {
  return zoeziGet(hostname, sessionRaw, '/api/memberapi/get/current');
}

async function getUser(hostname, sessionRaw, userId) {
  return zoeziGet(hostname, sessionRaw, `/api/user/get?id=${userId}`);
}

async function updateUser(hostname, userId, updates) {
  return zoeziApiPost(hostname, '/api/user/change', {
    id: userId,
    ...updates
  });
}

// Group operations
async function getAllGroups(hostname, sessionRaw) {
  return zoeziGet(hostname, sessionRaw, '/api/usergroup/get/all');
}

async function getGroup(hostname, sessionRaw, groupId) {
  return zoeziGet(hostname, sessionRaw, `/api/usergroup/get?id=${groupId}`);
}

async function createGroup(hostname, sessionRaw, name) {
  return zoeziPost(hostname, sessionRaw, '/api/usergroup/add', {
    name,
    change_permission: false,
    users: [],
    responsible: [],
    show: true
  });
}

async function updateGroup(hostname, sessionRaw, groupId, updates) {
  return zoeziPost(hostname, sessionRaw, '/api/usergroup/change', {
    id: groupId,
    ...updates
  });
}

// Member status operations
async function getAllStatuses(hostname) {
  return zoeziApiPost(hostname, '/api/member/status/get/all', {});
}

async function createStatus(hostname, name, color = '#cccccc') {
  return zoeziApiPost(hostname, '/api/member/status/add', { name, color });
}

async function updateMember(hostname, userId, updates) {
  return zoeziApiPost(hostname, '/api/member/change', {
    id: userId,
    ...updates
  });
}

// TODO operations
async function createTodo(hostname, title, userId, dueDate, description) {
  return zoeziApiPost(hostname, '/api/todo/add', {
    todo: title,
    dueDate,
    user_id: userId,
    description
  });
}

// Discount operations
async function createDiscount(hostname, name, code, percent, statusIds) {
  return zoeziApiPost(hostname, '/api/trainingcard/type/add', {
    name,
    discount_code: code,
    discount_percent: String(percent),
    discount_condition: {
      trainingcards: true,
      memberships: true,
      punchcards: true,
      products: true,
      courses: true,
      services: true,
      user_status: statusIds
    },
    discount: true,
    validFromDate: new Date().toISOString().split('T')[0]
  });
}

// Session verification
async function verifySession(hostname, userId, sessionHash) {
  const url = `https://${hostname}/api/user/session/verify?hash=${encodeURIComponent(sessionHash)}&user_id=${userId}`;
  const response = await fetch(url);
  const data = await response.json();
  return data.valid === true;
}

module.exports = {
  getCurrentUser,
  getUser,
  updateUser,
  getAllGroups,
  getGroup,
  createGroup,
  updateGroup,
  getAllStatuses,
  createStatus,
  updateMember,
  createTodo,
  createDiscount,
  verifySession
};
```

---

## Rate Limiting

Zoezi may implement rate limiting on API endpoints. Best practices:

- **Batch operations** when possible
- **Cache responses** that don't change frequently (groups, statuses)
- **Avoid polling** - use webhooks if available
- **Implement exponential backoff** on failures
- **Respect HTTP 429** (Too Many Requests) responses

---

## Testing

### Testing with Real Sessions

```javascript
// Test helper to get real session for development
// WARNING: Only use in development, never commit real sessions
async function testWithRealSession() {
  const hostname = 'your-gym.zoezi.com';
  const sessionRaw = 'paste_real_session_here';
  const userId = 123;

  const user = await getCurrentUser(hostname, sessionRaw);
  console.log('Current user:', user);
}
```

### Mocking Zoezi API

```javascript
// Mock for unit tests
const mockZoeziApi = {
  getCurrentUser: jest.fn().mockResolvedValue({
    id: 123,
    name: 'Test User',
    staff: false
  }),

  getAllGroups: jest.fn().mockResolvedValue([
    { id: 1, name: 'Test Group', users: [] }
  ])
};
```

---

**Last Updated:** 2026-01-13
**Version:** 1.0
