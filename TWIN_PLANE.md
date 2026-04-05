# Twin Plane Contract

The Twin Plane is a standardized management API that every twin MUST expose. It provides scenario information, operational logs, settings, and simulation capabilities.

## Base Path

All Twin Plane endpoints are served under `/_twin/`.

## Required Endpoints

Every twin MUST implement these endpoints:

### Health

```
GET /_twin/health
```

Returns:
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

Returns the list of supported scenarios for this twin:
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

Returns operation logs:
```json
{
  "logs": [
    {
      "id": 1,
      "timestamp": "<ISO 8601>",
      "entry": { "operation": "...", ... }
    }
  ],
  "limit": 100,
  "offset": 0
}
```

All twin operations MUST be logged, including content.

### Settings

```
GET /_twin/settings
```

Returns twin configuration:
```json
{
  "twin": "<twin-name>",
  "version": "<twin-version>",
  "base_url": "<base-url>"
}
```

### Account Management

```
POST /_twin/accounts
GET  /_twin/accounts
```

Create and list accounts on the twin. This is a Twin Plane operation, not part of the emulated provider API. Real provider account creation typically requires their console or sales process; the twin makes it a simple API call.

## Twin-Specific Endpoints

Individual twins MAY expose additional endpoints under `/_twin/` for simulation and twin-specific operations.

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

Triggers the full inbound SMS flow: creates message record, delivers webhook to configured URL, parses TwiML response, and creates any reply messages.

## Authentication

For 0.1.0, the Twin Plane is unauthenticated. Future versions may add a management key.

The Twin Plane MUST NOT use the same authentication mechanism as the emulated provider API (e.g., Twilio HTTP Basic Auth). These are separate concerns.

## Version

This contract is at version 0.1.0.
