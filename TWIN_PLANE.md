# Twin Plane Contract

The Twin Plane is a standardized management API that every twin MUST expose. It provides scenario information, operational logs, settings, simulation capabilities, and tenant management.

## Base Path

All Twin Plane endpoints are served under `/_twin/`.

## Authentication

As of 0.2.0, the Twin Plane is authenticated. See PRINCIPLES.md §8 for the normative model.

Three auth layers govern Twin Plane access:

1. **Operator-admin** — platform operators with cross-tenant scope.
   - Credential: the configured `TWIN_ADMIN_TOKEN`.
   - Presented as `Authorization: Bearer <token>` or `X-Twin-Admin-Token: <token>`.
   - Log entries stamp `tenant_id = "__operator_admin__"`.

2. **Tenant** — a twins.la-platform identity, required on every Twin Plane endpoint
   except the public endpoints listed below and the `POST /_twin/tenants` bootstrap.
   - Credential: `tenant_id:tenant_secret` via HTTP Basic Auth.
   - Scope: full access to resources owned by this tenant.

3. **Resource** — per-twin API credentials (Twilio `AccountSid:AuthToken`,
   Facebook `app_id:app_secret`, LiveKit `api_key:api_secret`) — **not** Twin Plane
   auth. Resource credentials govern the emulated provider / Data Plane surface.

**Public (unauthenticated) endpoints**: `GET /_twin/health`, `GET /_twin/scenarios`,
`GET /_twin/references`, `GET /_twin/settings`, and the bootstrap `POST /_twin/tenants`.
All other Twin Plane endpoints require tenant or admin auth.

The Twin Plane MUST NOT use the same authentication mechanism as the emulated
provider API. These are separate concerns.

## Required Endpoints

Every twin MUST implement these endpoints:

### Health

```
GET /_twin/health
```

Public. Returns:
```json
{
  "status": "ok",
  "twin": "<twin-name>",
  "version": "<twin-version>"
}
```

### Scenarios

```
GET /_twin/scenarios
```

Public. Returns the list of supported scenarios for this twin:
```json
{
  "scenarios": [
    {
      "name": "<scenario-name>",
      "status": "supported",
      "description": "<human-readable description>",
      "capabilities": ["<capability>", ...]
    }
  ]
}
```

`status` is one of: `supported`, `partial`, `planned`.

### Logs

```
GET /_twin/logs?limit=100&offset=0
```

Requires tenant or admin auth. Admin sees all logs; tenant sees only entries
stamped with its own `tenant_id`.

Returns operation logs:
```json
{
  "logs": [
    {
      "id": 1,
      "timestamp": "<ISO 8601>",
      "tenant_id": "<tenant-id>",
      "entry": { "operation": "...", ... }
    }
  ],
  "limit": 100,
  "offset": 0
}
```

All twin operations MUST be logged, including content. Every log entry MUST be
stamped with a `tenant_id`: an authenticated tenant's id, `"__operator_admin__"`
for admin operations, or `""` for proxy-level operations with no request-scope
tenant context.

### Settings

```
GET /_twin/settings
```

Public. Returns twin configuration:
```json
{
  "twin": "<twin-name>",
  "version": "<twin-version>",
  "base_url": "<base-url>"
}
```

### References

```
GET /_twin/references
```

Public. Returns the authoritative sources used to build this twin:
```json
{
  "references": [
    {
      "title": "<human-readable title>",
      "url": "<authoritative URL>",
      "retrieved": "<YYYY-MM-DD>"
    }
  ]
}
```

Every twin MUST return at least one reference. This list MAY be hardcoded.

### Tenant Bootstrap

```
POST /_twin/tenants
Content-Type: application/json

{
  "friendly_name": "Optional human label"
}
```

Public (unauthenticated) bootstrap endpoint. Generates a new `tenant_id` (UUID v4)
and `tenant_secret` (URL-safe random token). The secret is hashed (scrypt) before
storage and MUST be returned to the caller exactly once in this response — it is
not retrievable afterward.

Returns `201`:
```json
{
  "tenant_id": "<uuid>",
  "tenant_secret": "<urlsafe>",
  "friendly_name": "Optional human label",
  "created_at": "<ISO 8601>"
}
```

Callers then authenticate subsequent Twin Plane requests with
`Authorization: Basic <base64(tenant_id:tenant_secret)>`.

### Account Management

```
POST /_twin/accounts
GET  /_twin/accounts
```

Requires tenant auth (for mutations) or tenant/admin auth (for reads). Admin
listing spans tenants and redacts secret material.

Create and list accounts on the twin. This is a Twin Plane operation, not part
of the emulated provider API. Real provider account creation typically requires
their console or sales process; the twin makes it a simple API call.

## Twin-Specific Endpoints

Individual twins MAY expose additional endpoints under `/_twin/` for simulation
and twin-specific operations. Mutating endpoints MUST require tenant or admin
auth; read-only endpoints on shared state MAY remain public.

### Twilio Twin: Inbound SMS Simulation

```
POST /_twin/simulate/inbound
Content-Type: application/json

{
  "from": "+15551234567",
  "to": "+15559876543",
  "body": "Hello from outside!"
}
```

Requires tenant or admin auth. Triggers the full inbound SMS flow: creates
message record, delivers webhook to configured URL, parses TwiML response, and
creates any reply messages.

## Version

This contract is at version 0.2.0.
