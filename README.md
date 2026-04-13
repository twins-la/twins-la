<p align="center">
  <img src="twins.png" alt="twins.la" width="72">
</p>

<h1 align="center">twins.la</h1>

<p align="center">Digital twins for developers and agents.</p>

---

High-fidelity digital twins of real APIs. Write your code against a twin, then switch to the real service by changing the hostname and credentials. No provider accounts needed. No mocks, no stubs — real API behavior within supported scenarios.

Each twin runs locally (`pip install`) or in the cloud (point at the URL). Same package, same behavior.

## Available Twins

| Twin | APIs | URL |
|------|------|-----|
| **facebook**.twins.la | Facebook Login (OAuth 2.0, Graph `/me`, `/debug_token`) | [facebook.twins.la](https://facebook.twins.la) |
| **livekit** | LiveKit rooms, participants, egress, WebSocket signaling | local only — [github.com/twins-la/livekit](https://github.com/twins-la/livekit) |
| **twilio**.twins.la | Twilio SMS, SendGrid Email | [twilio.twins.la](https://twilio.twins.la) |

## How It Works

**Cloud** — Each twin runs at its own subdomain. Create an account via `POST /_twin/accounts`, get credentials, and use the standard provider API against the twin's URL.

**Local** — Install the twin's local package and run it on localhost:

```bash
pip install twins-twilio-local
```

```python
from twins_twilio_local.storage_sqlite import SQLiteStorage
from twins_twilio.app import create_app

app = create_app(storage=SQLiteStorage("twin.db"))
app.run(port=8080)
```

## Twin Plane

Every twin exposes a standardized management API at `/_twin/`:

```
GET  /_twin/health              — Status check
GET  /_twin/scenarios           — Supported scenarios
GET  /_twin/logs                — Operation logs
GET  /_twin/settings            — Configuration
GET  /_twin/agent-instructions  — Agent-specific usage guide
POST /_twin/accounts            — Create credentials
```

Tooling built for one twin works with all of them.

## For Agents

Each twin serves machine-readable instructions at `/_twin/agent-instructions`. Copy this into your agent's system prompt or CLAUDE.md:

```
# twins.la — Digital API Twins

twins.la provides high-fidelity digital twins of real APIs.
Code written against a twin works against the real service
with only hostname and credential changes.

## Available Twins

- facebook.twins.la — Facebook Login (OAuth 2.0 + Graph API)
  Agent instructions: https://facebook.twins.la/_twin/agent-instructions
- livekit — LiveKit WebRTC control plane (local only)
  GitHub: https://github.com/twins-la/livekit
- twilio.twins.la — Twilio SMS + SendGrid Email APIs
  Agent instructions: https://twilio.twins.la/_twin/agent-instructions

## How to Use (any twin)

1. Pick a twin from the list above
2. Create an account: POST https://{twin}/_twin/accounts
3. Use the returned credentials with the standard provider API
4. Check supported scenarios: GET https://{twin}/_twin/scenarios
```

## Documentation

- [Operating Principles](PRINCIPLES.md) — Normative principles governing all twins
- [Twin Plane Contract](TWIN_PLANE.md) — Full Twin Plane API specification

## License

[MIT](LICENSE)
