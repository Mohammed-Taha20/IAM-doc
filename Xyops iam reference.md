# xyOps — Identity, Permissions & Access Management
## Reference & Implementation Guide for the Text-Based AI Pipeline Builder

> **Document scope:**  IAM concepts, database schema, and REST API reference for implementing roles and permissions in the Text-Based AI Pipeline Builder..  
---

## 1. Architecture Mental Model

xyOps IAM is **not** a relational database with foreign keys and join tables. It is a **key-value store with JSON blobs**, using slash-delimited path keys as the "schema." Before reading anything else, anchor this model in your head:

```
SQLite table: records
┌─────────────────────────────────────┬──────────────────────────────┐
│ path (TEXT PRIMARY KEY)             │ data (TEXT — JSON blob)      │
├─────────────────────────────────────┼──────────────────────────────┤
│ users/mohammed                      │ { "username": "mohammed", …} │
│ sessions/abc123def456               │ { "id": "abc123…", "ip": …}  │
│ global/roles                        │ { "length": 3, … }           │  ← list header
│ global/roles/0                      │ [ {role1}, {role2}, … ]      │  ← list page 0
│ global/state                        │ { "scheduler": {…}, … }      │
└─────────────────────────────────────┴──────────────────────────────┘
```

**Everything is in this one table.** Users, sessions, roles, API keys — all rows. The "database tables" in xyOps docs are key-prefix naming conventions, not SQL tables.

### IAM component stack

```
┌──────────────────────────────────────────────────────────┐
│  HTTP / WebSocket request                                │
├──────────────────────────────────────────────────────────┤
│  Auth gate:  loadSession()  OR  API key header check     │  lib/api.js
├──────────────────────────────────────────────────────────┤
│  Privilege merge: user.privileges ∪ role[].privileges    │  pixl-server-user
├──────────────────────────────────────────────────────────┤
│  Resource check: categories[] scope, groups[] scope      │  lib/api.js
├──────────────────────────────────────────────────────────┤
│  Business logic runs only if all gates pass              │  lib/*.js
├──────────────────────────────────────────────────────────┤
│  pixl-server-storage → better-sqlite3 → records table   │  node_modules/
└──────────────────────────────────────────────────────────┘
```

---

## 2. Core IAM Concepts

### 2.1 User

A user is the primary identity. One JSON record per human or service account. Stored at `users/<username>`.

**Key design decision:** Passwords are never stored in plaintext. bcrypt hash + separate random salt. The password field in storage always contains the bcrypt output.

**Privilege resolution:** A user's effective permissions are the **additive union** of:
- Their own `privileges{}` object, plus
- The `privileges{}` of every role listed in their `roles[]` array where `enabled: true`.

`admin: true` anywhere in this union **bypasses all other checks**, including category and group scopes.

### 2.2 Role

A reusable bundle of privileges + optional resource restrictions. Assign one role to many users instead of granting privileges one-by-one. Roles can be updated and the change propagates to all assigned users on next request.

Roles also carry `categories[]` and `groups[]` scopes — these **restrict** what resources the role grants access to, not what actions it permits.

### 2.3 Privilege

A named boolean gate. Set to `true` to grant, absent or `false` to deny. The full privilege ID list is fixed in the codebase — you cannot invent new IDs without also adding enforcement logic.

| Privilege ID | What it gates |
|---|---|
| `admin` | Full access, bypasses everything |
| `create_events` | POST new events/workflows |
| `edit_events` | PUT/PATCH existing events |
| `run_events` | Manually trigger job execution |
| `abort_events` | Kill running jobs |
| `create_users` | POST new user accounts |
| `edit_users` | PUT existing user records |
| `delete_users` | DELETE user accounts |
| `view_live_log` | Stream job stdout in real time |
| `create_plugins` | Register new plugins |
| `edit_plugins` | Modify plugin configs |
| `manage_secrets` | Read/write encrypted secrets |
| `manage_api_keys` | Create/revoke API keys |
| `view_history` | Access completed job history |
| `view_servers` | Read server/satellite status |
| `manage_groups` | Create/edit server groups |
| `manage_categories` | Create/edit categories |
| `manage_channels` | Create/edit notification channels |
| `manage_tickets` | Create/edit/assign tickets |
| `manage_snapshots` | Trigger and view server snapshots |

### 2.4 Resource Scopes

Scopes **restrict** which resources a user can see and touch, independent of privileges.

| Scope field | Applies to | Empty array means |
|---|---|---|
| `categories[]` | Events and workflows | Access to all categories |
| `groups[]` | Target server groups for job execution | Access to all groups |

A user with `edit_events` but `categories: ["ml-pipelines"]` can only edit events in the `ml-pipelines` category. An admin ignores both arrays entirely.

### 2.5 API Key

Machine identity. No session, no cookie. Same privilege enforcement as users. Pass as `X-Api-Key` HTTP header or `?api_key=` query parameter.

### 2.6 Session

Created on successful login. Stored at `sessions/<id>`. A cookie containing the session ID is set on the client. Every subsequent API call calls `loadSession()` which reads this record and hydrates the user object.

### 2.7 SSO

Trusted-header SSO via reverse proxy (OAuth2-Proxy, Tailscale, etc.). The proxy injects an identity header after authenticating the user externally. xyOps reads that header, and optionally auto-creates a local account and maps IdP groups to xyOps roles.

---

## 3. Proposed Role Design

| Role | Key Privileges | Category Scope | Who gets it |
|---|---|---|---|
| pipeline-admin | All | all | Project lead |
| pipeline-designer | create_events, edit_events, run_events | nl-pipelines | Team members building flows |
| pipeline-operator | run_events, abort_events, view_live_log | nl-pipelines | Team members running flows |
| pipeline-viewer | view_history | nl-pipelines | Stakeholders / reviewers |
| api-service | run_events, view_history | nl-pipelines | LLM execution engine (API key) |

All pipeline events belong to the `nl-pipelines` category.
This means role scopes work automatically — no per-event permission setup needed.


## 4. Database Schema — Full Field Reference

### 4.1 User record — `users/<username>`

```json
{
  "username":             "mohammed",
  "full_name":            "Mohammed Taha",
  "email":               "mohammed@example.com",
  "active":               true,
  "created":              1704067200,
  "modified":             1716000000,
  "password":             "$2b$10$...",
  "salt":                 "a8f3c2...",
  "force_password_reset": false,

  "roles":       ["pipeline-operator", "viewer"],
  "privileges": {
    "run_events":    true,
    "view_history":  true
  },

  "categories": ["ml-pipelines", "data-ingestion"],
  "groups":     ["gpu-workers"],

  "language":    "en",
  "timezone":    "Africa/Cairo",
  "color_acc":   "#6366f1",
  "theme":       "dark",
  "avatar":      "/files/avatars/mohammed.jpg",

  "remote":     false,
  "sync":       false,
  "sso_roles":  [],

  "fl_date_code": 20240601,
  "fl_count":     0
}
```

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `username` | string | required, unique, normalized to lowercase, alphanumeric + underscore | Used as the storage key suffix |
| `full_name` | string | optional | Display name in UI and emails |
| `email` | string | required | Used for password reset, notifications |
| `active` | boolean | required | `false` = account disabled, login rejected |
| `created` | integer | auto-set, immutable | Unix epoch timestamp |
| `modified` | integer | auto-updated on every write | Unix epoch timestamp |
| `password` | string | bcrypt hash, never plaintext | Stripped from all API responses |
| `salt` | string | random hex string, used as bcrypt salt seed | Stripped from all API responses |
| `force_password_reset` | boolean | admin-settable only | Forces password change on next login |
| `roles[]` | string array | references role IDs in `global/roles` | Merged at request time, not stored |
| `privileges{}` | object | boolean values per privilege ID | Combined with role privileges additively |
| `categories[]` | string array | references category IDs | Empty = all categories |
| `groups[]` | string array | references group IDs | Empty = all groups |
| `language` | string | ISO 639-1 code | UI language preference |
| `timezone` | string | IANA timezone string | Display timezone |
| `color_acc` | string | hex color | UI accent color |
| `theme` | string | `"dark"` or `"light"` | UI theme preference |
| `avatar` | string | relative URL path | Optional profile image |
| `remote` | boolean | auto-set by SSO | `true` = identity managed externally |
| `sync` | boolean | auto-set by SSO | `true` = fields synced from IdP on each login |
| `sso_roles[]` | string array | set by SSO role mapper | Raw IdP group names before mapping |
| `fl_date_code` | integer | internal | Failed-login flood control date counter |
| `fl_count` | integer | internal | Failed-login count for lockout enforcement |

---

### 4.2 Role record — pages inside `global/roles` list

Each item in the roles list:

```json
{
  "id":          "pipeline-operator",
  "title":       "Pipeline Operator",
  "enabled":     true,
  "created":     1704067200,
  "modified":    1716000000,

  "privileges": {
    "run_events":    true,
    "abort_events":  true,
    "view_live_log": true,
    "view_history":  true
  },

  "categories": ["ml-pipelines"],
  "groups":     ["gpu-workers", "cpu-workers"],
  "description": "Can run and monitor ML pipeline workflows. Cannot edit definitions."
}
```

| Field | Type | Constraints | Notes |
|---|---|---|---|
| `id` | string | required, unique, slug format | Referenced in `user.roles[]` |
| `title` | string | required | Human-readable display name |
| `enabled` | boolean | required | `false` = role contributes nothing even if assigned |
| `created` | integer | auto-set | Unix epoch |
| `modified` | integer | auto-updated | Unix epoch |
| `privileges{}` | object | boolean values | Merged additively into assigned users' effective privileges |
| `categories[]` | string array | optional | Restricts which categories the role's privileges apply to. Empty = all. |
| `groups[]` | string array | optional | Restricts which server groups the role covers. Empty = all. |
| `description` | string | optional | Admin-facing documentation |

---

### 4.3 Session record — `sessions/<session_id>`

```json
{
  "id":         "abc123def456ghi789",
  "username":   "mohammed",
  "created":    1716000000,
  "modified":   1716003600,
  "ip":         "41.65.200.1",
  "useragent":  "Mozilla/5.0 (Macintosh; …)",
  "expire":     1717209600
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | string | 18-character random alphanumeric. Also the storage key suffix. |
| `username` | string | Foreign reference to `users/<username>` |
| `created` | integer | Unix epoch — when the session was created |
| `modified` | integer | Unix epoch — updated on every authenticated request |
| `ip` | string | Client IP at login time |
| `useragent` | string | Raw HTTP User-Agent header at login time |
| `expire` | integer | Unix epoch — `created + (session_expire_days * 86400)` |

---

### 4.4 API key record — inside `global/api_keys` list

```json
{
  "id":        "Kp7xR2mQnL8vT4wY9sAj",
  "title":     "Pipeline Builder Service Account",
  "created":   1716000000,
  "modified":  1716000000,
  "active":    true,
  "privileges": {
    "run_events":   true,
    "view_history": true
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | string | 24-character random alphanumeric. This is the actual key value sent in requests. |
| `title` | string | Human label for identification |
| `active` | boolean | `false` = key rejected on all requests |
| `created` | integer | Unix epoch |
| `modified` | integer | Unix epoch |
| `privileges{}` | object | Same format as user/role privileges. No categories or groups scope on API keys. |

---


## 5 — Consume via REST API

Use xyOps as the auth/access layer and call it from your external application.


### Key endpoints for IAM operations

```bash
# --- Users ---
GET    /api/get_users                      # List all users (requires edit_users privilege)
GET    /api/get_user?username=mohammed     # Get one user
POST   /api/create_user                    # Create user (requires create_users)
PUT    /api/update_user                    # Update user (requires edit_users)
DELETE /api/delete_user?username=mohammed  # Delete user (requires delete_users)

# --- User self-service ---
POST   /api/user/login                     # Login → session cookie
POST   /api/user/logout                    # Logout current session
POST   /api/user_settings                  # Update own non-sensitive preferences
POST   /api/logout_all                     # Kill all other sessions (requires current password)
GET    /api/get_user_activity              # Read own security log

# --- Roles ---
GET    /api/get_roles                      # List all roles (requires admin)
POST   /api/create_role                    # Create role (requires admin)
PUT    /api/update_role                    # Update role (requires admin)
DELETE /api/delete_role?id=pipeline-op     # Delete role (requires admin)

# --- API Keys ---
GET    /api/get_api_keys                   # List keys (requires manage_api_keys)
POST   /api/create_api_key                 # Create key (requires manage_api_keys)
DELETE /api/delete_api_key?id=Kp7x…       # Revoke key (requires manage_api_keys)
```


---

