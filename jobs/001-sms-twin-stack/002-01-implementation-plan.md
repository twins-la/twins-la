**Job: 001-sms-twin-stack**

# Implementation Plan

## Source Understanding(s)
- `001-01-directive-analysis.md`

## Unified Goal Statement

Build a working Twilio SMS digital twin (`twilio/`) and two hosting frameworks (`local/`, `cloud/`) that can run it. The twin must emulate Twilio's SMS REST API with enough fidelity that existing Twilio SDK code works with only hostname and credential changes. Both hosting environments must persist state across restarts. Smoke tests must prove end-to-end SMS flows work.

## Success Criteria

Carried forward from the understanding:

- [ ] `twilio/` exposes `/2010-04-01/Accounts/{AccountSid}/Messages.json` — POST to send, GET to list, GET by SID to fetch
- [ ] `twilio/` exposes `/2010-04-01/Accounts/{AccountSid}/IncomingPhoneNumbers.json` — create/list/fetch/update with SMS webhook URL config
- [ ] `twilio/` authenticates via HTTP Basic Auth (AccountSid:AuthToken)
- [ ] `twilio/` generates Twilio-format SIDs (AC, SM, PN prefixes + 32 hex chars)
- [ ] `twilio/` delivers inbound SMS to configured webhook URLs with correct parameters and X-Twilio-Signature
- [ ] `twilio/` returns Twilio-compatible JSON response shapes
- [ ] `twilio/` returns Twilio-compatible error responses
- [ ] `twilio/` persists all state to a durable store
- [ ] `twilio/` exposes Twin Plane API at `/_twin/`
- [ ] `local/` runs the twin with a single `docker compose up`
- [ ] `local/` starts datastore automatically
- [ ] `local/` persists state across restarts
- [ ] `local/` exposes a clean hosting contract for future twins
- [ ] `cloud/` includes runnable hosting code
- [ ] `cloud/` includes Terraform for Azure Container Apps
- [ ] `cloud/` includes a Jenkinsfile
- [ ] `cloud/` serves at `twilio.twins.la` endpoint
- [ ] `cloud/` persists state across restarts
- [ ] No cloud details leak into `local/` or `twilio/`
- [ ] Smoke tests exercise the full SMS flow in both environments
- [ ] All repos versioned 0.1.0
- [ ] Package/runtime contract documented

## Approach

### Architecture Overview

The system has three layers:

1. **Twin package** (`twilio/`) — A pure Python package that implements the Twilio SMS API emulation and Twin Plane. It defines a storage interface but does NOT embed a datastore. It is a library that a host loads and runs. The package exposes a WSGI/ASGI app factory that a host calls with configuration (including a storage backend).

2. **Host** (`local/` or `cloud/`) — Provides the runtime environment: HTTP server, storage backend, configuration. The host imports the twin package, provides it with a storage backend, and serves its routes. The host owns the datastore lifecycle.

3. **Infrastructure** (`cloud/` for Azure, `website/` for shared platform) — Terraform, Jenkinsfile, Docker images.

### Twin Package Design (`twilio/`)

The twin is a Python package (`twins_twilio`) built on Flask. Flask is the right choice because:
- Twilio's API is synchronous request/response
- Flask is lightweight and well-understood
- The twin needs to make outbound HTTP calls (webhooks) which can be done synchronously for 0.1.0

The package structure:
```
twilio/
  twins_twilio/
    __init__.py          # Package init, version
    app.py               # Flask app factory: create_app(config) → Flask app
    auth.py              # HTTP Basic Auth + Twilio auth validation
    models.py            # Data models (Account, PhoneNumber, Message)
    storage.py           # Abstract storage interface
    sids.py              # Twilio-format SID generation
    routes/
      accounts.py        # Account resource routes
      messages.py        # Message resource routes  
      phone_numbers.py   # IncomingPhoneNumber resource routes
    twin_plane/
      routes.py          # /_twin/ management API
    webhooks.py          # Webhook delivery + signature computation
    twiml.py             # TwiML response parser (Message verb)
    errors.py            # Twilio-compatible error responses
  tests/
    smoke/               # Smoke tests using the Twilio Python SDK
  setup.py / pyproject.toml
```

Key design decisions:
- **Storage interface**: The twin defines `TwinStorage` ABC with methods for CRUD on accounts, numbers, messages. Hosts provide concrete implementations (SQLite, Postgres).
- **App factory**: `create_app(storage, config)` returns a configured Flask app. The host controls how the app is served.
- **SID generation**: `AC` + 32 random hex for accounts, `SM` + 32 for messages, `PN` + 32 for phone numbers. AuthTokens are 32 random hex.
- **Webhook signing**: HMAC-SHA1 with the account's AuthToken, matching Twilio's documented algorithm exactly.
- **Message lifecycle**: POST /Messages → status `queued` → background thread delivers (simulated) → status `sent` → status `delivered`. For inbound simulation via Twin Plane, deliver webhook to configured URL.

### Hosting Contract

The contract between twin and host:

```python
# A twin package must provide:
def create_app(storage: TwinStorage, config: dict) -> Flask:
    """Create and return the configured Flask application."""

# A host must provide:
# 1. A TwinStorage implementation
# 2. Configuration dict with at minimum: {"base_url": "..."}  
# 3. HTTP serving of the returned Flask app
# 4. Lifecycle management (start/stop/persistence)
```

This contract will be documented in `twins-la/HOSTING_CONTRACT.md`.

### Local Host (`local/`)

```
local/
  twins_local/
    __init__.py
    host.py              # Main entry point: loads twin, provides SQLite storage, serves
    storage_sqlite.py    # SQLite TwinStorage implementation
    config.py            # Local configuration
  docker-compose.yml     # Single-command bring-up
  Dockerfile             # Python app image
  setup.py / pyproject.toml
```

- `docker-compose.yml` runs a single container with the local host
- SQLite database file mounted as a Docker volume for persistence
- Port 8080 exposed
- `twins_local.host` is the entry point: imports `twins_twilio`, creates SQLite storage, calls `create_app()`, serves with gunicorn
- Can also run without Docker: `pip install -e ../twilio && pip install -e . && python -m twins_local`

### Cloud Host (`cloud/`)

```
cloud/
  twins_cloud/
    __init__.py
    host.py              # Cloud entry point: loads twin, provides Postgres storage, serves
    storage_postgres.py  # Postgres TwinStorage implementation
    config.py            # Cloud configuration (reads DATABASE_URL from env)
  infra/
    terraform/           # Following TERRAFORM.md gold standard
      backend.tf
      providers.tf
      locals.tf
      main.tf            # Container App for twins, PostgreSQL
      variables.tf
      outputs.tf
      env/
        dev.json
        dev.tfvars
        prod.json
        prod.tfvars
  Jenkinsfile
  Dockerfile
  setup.py / pyproject.toml
```

- Terraform creates the twin Container App and PostgreSQL resources in the existing Azure platform (references existing RG, ACR, Container App Environment from website/ state via data sources or shared variables)
- Jenkinsfile builds Docker image, pushes to ACR, deploys to Container App
- `DATABASE_URL` injected via Key Vault → Container App secrets
- Postgres storage implementation uses psycopg2/asyncpg

### Relationship to website/ Terraform

The `website/` repo already has Terraform that creates shared platform infrastructure (Resource Group, ACR, Container App Environment, Key Vault). It also has a placeholder for the Twilio twin Container App.

For clean separation:
- `website/` Terraform keeps the shared platform resources and the website Container App
- `cloud/` Terraform manages twin-specific resources: the twin Container App(s) and PostgreSQL
- `cloud/` Terraform references shared resources from `website/` via Terraform remote state or input variables

The existing twilio twin placeholder in `website/main.tf` will be removed and moved to `cloud/`.

### Twin Plane API (`/_twin/`)

```
GET  /_twin/scenarios          → list supported scenarios
GET  /_twin/logs               → retrieve operation logs  
GET  /_twin/settings           → get twin settings
PUT  /_twin/settings           → update twin settings
POST /_twin/simulate/inbound   → inject an inbound SMS (triggers webhook delivery)
GET  /_twin/health             → health check
```

The Twin Plane is not authenticated with Twilio credentials — it uses a separate twin management key or is unauthenticated for 0.1.0 local use. Cloud deployment should add basic auth.

## Sequence

### Phase 1: Foundation (twilio/)

1. **Define the storage interface** (`storage.py`) — ABC with methods for all CRUD operations on accounts, phone numbers, and messages. This is the contract that hosts implement.

2. **Implement core models and SID generation** (`models.py`, `sids.py`) — Data classes for Account, PhoneNumber, Message with Twilio-compatible field sets. SID generation matching Twilio format.

3. **Implement auth** (`auth.py`) — HTTP Basic Auth decorator that validates AccountSid:AuthToken against stored accounts.

4. **Implement Twilio API routes** — In order:
   - Account fetch (GET `/2010-04-01/Accounts/{AccountSid}.json`)
   - IncomingPhoneNumbers (POST create, GET list, GET fetch, POST update)
   - Messages (POST send, GET list, GET fetch)

5. **Implement webhook delivery** (`webhooks.py`) — HMAC-SHA1 signature computation, HTTP POST to configured webhook URL with Twilio-format parameters.

6. **Implement TwiML parser** (`twiml.py`) — Parse `<Response><Message>` from webhook responses to handle auto-reply.

7. **Implement Twin Plane** — Management API routes at `/_twin/`.

8. **Implement error responses** (`errors.py`) — Twilio-compatible error JSON format.

### Phase 2: Local Host (local/)

9. **Implement SQLite storage** (`storage_sqlite.py`) — Concrete TwinStorage using SQLite with proper schema.

10. **Implement local host** (`host.py`) — Entry point that wires SQLite storage to twin app, serves with gunicorn.

11. **Create Docker Compose** — Single-service compose file with volume mount for SQLite persistence.

12. **Create Dockerfile** — Python image, installs twilio/ and local/ packages, runs gunicorn.

### Phase 3: Smoke Tests (twilio/)

13. **Write smoke tests** — Python tests using the official `twilio` Python SDK pointed at the local twin. Tests exercise:
    - Create account via Twin Plane
    - Provision phone number
    - Configure webhook URL on number
    - Send outbound SMS
    - Simulate inbound SMS via Twin Plane
    - Verify webhook delivery
    - Verify message status progression

### Phase 4: Cloud Host (cloud/)

14. **Implement Postgres storage** (`storage_postgres.py`) — Concrete TwinStorage using PostgreSQL.

15. **Implement cloud host** (`host.py`) — Entry point that wires Postgres storage, reads config from environment.

16. **Create Terraform** — Following TERRAFORM.md gold standard. Container App, PostgreSQL Flexible Server, Key Vault secrets. References shared platform from website/.

17. **Create Jenkinsfile** — Build, push to ACR, deploy to Container App.

18. **Create Dockerfile** — Similar to local but with Postgres dependencies.

### Phase 5: Documentation and Contract

19. **Document hosting contract** in `twins-la/HOSTING_CONTRACT.md`.

20. **Document Twin Plane contract** in `twins-la/TWIN_PLANE.md`.

21. **Add scenario documentation** to `twilio/`.

**Parallel work**: Phases 2 and 4 (local and cloud hosts) can proceed in parallel after Phase 1. Phase 3 (smoke tests) can begin as soon as Phase 2 has a running local instance.

**Decision points**:
- If the Twilio Python SDK makes assumptions about response shapes that we haven't anticipated, adjust the twin's responses to match.
- If webhook delivery timing needs to be async for the smoke tests to work, add a small async delivery queue.

## Boundaries

**Must NOT**:
- Implement MMS, voice, video, or any non-SMS Twilio API
- Implement subaccounts, Messaging Services, or scheduled messages
- Integrate a real SMS provider
- Add a user-facing product control plane for cloud
- Add rate limiting or billing simulation
- Use any cloud-specific code in `twilio/` or `local/`
- Create infrastructure resources that already exist in `website/` Terraform — reference them instead
- Build speculative platform abstractions for hypothetical future twins

## Verification

1. **Unit verification**: Each Twilio API endpoint returns the correct response shape when tested with curl
2. **SDK verification**: The `twilio` Python SDK can successfully:
   - Create a `Client(account_sid, auth_token, base_url=twin_url)`
   - Call `client.messages.create(to=..., from_=..., body=...)`
   - Call `client.incoming_phone_numbers.create(phone_number=...)`
   - List and fetch messages and phone numbers
3. **Webhook verification**: Inbound SMS simulation triggers webhook POST with correct parameters and valid X-Twilio-Signature
4. **Persistence verification**: Stop and restart the host; verify previously created resources are still present
5. **Smoke test suite**: Automated tests that prove the above, runnable via `pytest`
6. **Cloud deployment**: Terraform applies cleanly; Jenkinsfile builds and deploys; endpoint responds

## Risks and Contingencies

| Risk | Likelihood | If it happens... |
|------|-----------|-------------------|
| Twilio Python SDK makes undocumented assumptions about response fields we haven't implemented | Medium | Add the missing fields — the SDK is open source, read it to discover required fields |
| Twilio SDK requires HTTPS or specific TLS behavior | Low | For local, accept self-signed certs or use HTTP. For cloud, Container Apps provides HTTPS by default |
| webhook delivery blocks the API response (synchronous delivery) | Medium | Move webhook delivery to a background thread with a short queue. For 0.1.0, a threading.Thread per delivery is acceptable |
| SQLite concurrency issues under load | Low | Not a concern for 0.1.0 local dev. WAL mode handles basic concurrent reads |
| Terraform state conflicts between website/ and cloud/ | Medium | Use separate state files. cloud/ references website/ resources via data sources with resource group/name lookups, not remote state |

## PROBLEMS REQUIRING HUMAN ACTION

1. **website/ Terraform cleanup**: The website repo already has a `create_twilio_twin` toggle and twilio twin Container App resource. This should be removed from website/ and moved to cloud/ to maintain clean separation. This requires a change to the website repo. I will proceed with cloud/ having its own Terraform and note the overlap for later cleanup.

2. **DNS for twilio.twins.la**: Custom domain binding on Azure Container Apps requires DNS configuration. For 0.1.0, the twin will be accessible via the auto-generated Container App FQDN. Custom domain binding can be added when DNS is ready.

3. **Jenkins access**: The Jenkinsfile needs Azure credentials configured in Jenkins. This is an operational setup step.

## PROPOSED NEW ANALYSIS

- None.

## FEEDBACK FOR WORKFLOW

- **Category**: Codebase concern
- **Requires human intervention**: Yes (for website/ Terraform cleanup)
- **Should feed back into**: N/A
- **Detail**: The website repo's Terraform already has twilio twin resources. For clean repo separation, these should live in cloud/ only. I'll proceed with cloud/ having its own Terraform; the duplicate in website/ should be cleaned up in a follow-up.
