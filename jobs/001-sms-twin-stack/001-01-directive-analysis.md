**Job: 001-sms-twin-stack**

# Directive Analysis

## Original Directive

> Build the first working end-to-end twins.la stack: a Twilio SMS twin (`twilio/`), a local hosting framework (`local/`), and a cloud hosting framework (`cloud/`). The twin must emulate Twilio's SMS API with high fidelity so existing Twilio-targeting code works with only hostname and credential changes. Local hosting must be trivially easy to run. Cloud hosting must include Terraform and Jenkinsfile. All three repos version 0.1.0. State persists across restarts. Smoke tests prove end-to-end SMS flows.

## Restated Intent

Build three working, interconnected repositories that together form a digital twin platform:

1. **`twilio/`** — A Twilio SMS twin that faithfully emulates the subset of Twilio's REST API required for SMS send/receive. An existing application using the Twilio SDK for SMS should be able to point at this twin (changing only hostname and credentials) and have its SMS workflows function correctly. The twin simulates message delivery internally — no real SMS provider.

2. **`local/`** — A local hosting framework that runs the Twilio twin (and future twins) on a developer's machine with minimal setup. One command or near-one-command bring-up. State persists across restarts.

3. **`cloud/`** — A cloud hosting framework that deploys and runs the same twin package in the cloud. Includes Terraform for provisioning and a Jenkinsfile for CI/CD. Endpoint strategy supports future twins (e.g., `twilio.twins.la`). Private repo — no cloud-specific details leak into public repos.

The three repos share a package/runtime contract that must be documented. `twins-la/` holds only cross-repo definitions (Twin Plane contract, shared concepts).

## Assumptions

| # | Assumption | Status | Evidence / Gap |
|---|-----------|--------|----------------|
| 1 | The Twilio SMS API surface to emulate is the 2010-04-01 REST API (`/2010-04-01/Accounts/{AccountSid}/Messages.json`) | Confirmed | Twilio docs — this is the current and only SMS API version |
| 2 | Authentication is HTTP Basic Auth with AccountSid:AuthToken | Confirmed | Twilio docs |
| 3 | SID formats follow Twilio conventions: AC (account), SM (SMS message), PN (phone number) + 32 hex chars | Confirmed | Twilio docs |
| 4 | Webhook signing uses HMAC-SHA1 with AuthToken, delivered via X-Twilio-Signature header | Confirmed | Twilio security docs |
| 5 | The datastore choice is mine; it must support persistence across restarts | Confirmed | Directive explicitly delegates this |
| 6 | SQLite is sufficient for 0.1.0 — it avoids requiring a separate database process while providing persistence | Unresolved | Reasonable for local; cloud may want Postgres. Need to decide. |
| 7 | Python is the implementation language | Confirmed | Directive states this |
| 8 | Docker Compose is an acceptable local bring-up mechanism | Confirmed | Directive mentions it as an option |
| 9 | The cloud target is a Linux VM / container environment reachable at twins.la | Unresolved | Directive doesn't specify cloud provider. Terraform implies a specific provider. |
| 10 | "Smoke tests" means automated tests that exercise the end-to-end SMS flow, not a manual process | Confirmed | Directive says "smoke tests prove the end-to-end SMS scenario works" |
| 11 | The Twin Plane is a standardized management/control API separate from the emulated provider API | Confirmed | PRINCIPLES.md §8-9 |
| 12 | Inbound SMS simulation requires a mechanism to inject messages into the twin as if they arrived from the network | Confirmed | Directive requires simulating message sending and receiving entirely inside the twin |

## Success Criteria

- [ ] `twilio/` exposes `/2010-04-01/Accounts/{AccountSid}/Messages.json` — POST to send, GET to list, GET by SID to fetch
- [ ] `twilio/` exposes `/2010-04-01/Accounts/{AccountSid}/IncomingPhoneNumbers.json` — create/list/fetch/update phone number resources with SMS webhook URL configuration
- [ ] `twilio/` authenticates requests via HTTP Basic Auth (AccountSid:AuthToken)
- [ ] `twilio/` generates Twilio-format SIDs (AC, SM, PN prefixes + 32 hex chars)
- [ ] `twilio/` delivers inbound SMS to configured webhook URLs with correct Twilio webhook parameters and X-Twilio-Signature signing
- [ ] `twilio/` returns Twilio-compatible JSON response shapes for all supported endpoints
- [ ] `twilio/` returns Twilio-compatible error responses (error codes, HTTP status codes)
- [ ] `twilio/` persists all state (accounts, numbers, messages) to a durable store
- [ ] `twilio/` exposes a Twin Plane API for management operations (scenarios, logs, settings, simulation triggers)
- [ ] `local/` runs the Twilio twin with a single bring-up command (e.g., `docker compose up` or `python -m local`)
- [ ] `local/` starts any required datastore automatically
- [ ] `local/` persists twin state across restarts
- [ ] `local/` exposes a clean hosting contract that future twins can implement
- [ ] `cloud/` includes runnable hosting code for the Twilio twin
- [ ] `cloud/` includes Terraform configuration for provisioning
- [ ] `cloud/` includes a Jenkinsfile for deployment
- [ ] `cloud/` serves the twin at a well-known endpoint (e.g., `twilio.twins.la`)
- [ ] `cloud/` persists state across restarts
- [ ] `cloud/` contains no public-repo-leaking details; `local/` and `twilio/` contain no cloud-private details
- [ ] Smoke tests exercise: create account context, provision number, configure webhook, send outbound SMS, receive inbound SMS, verify webhook delivery
- [ ] All implementation repos are versioned 0.1.0
- [ ] Shared package/runtime contract is documented

## Scope

### In Scope

- Twilio SMS Message resource: create (send), fetch, list
- Twilio IncomingPhoneNumber resource: create, fetch, list, update (for SMS URL config)
- Account resource: fetch (at minimum, to validate credentials)
- HTTP Basic Auth matching Twilio's scheme
- Twilio-format SID generation for all resource types
- Webhook delivery for inbound SMS with X-Twilio-Signature signing
- TwiML response parsing for webhook responses (at least `<Message>` verb for reply)
- Message status tracking (queued → sent → delivered simulation)
- Twin Plane API: scenarios, logs, settings, simulation triggers (inject inbound SMS)
- SQLite-based persistence (local), Postgres-based persistence (cloud)
- Docker Compose for local bring-up
- Terraform + Jenkinsfile for cloud deployment
- Smoke tests for both environments
- Cross-repo package/runtime contract documentation

### Out of Scope

- MMS (media messages)
- Voice / video / any non-SMS Twilio APIs
- Twilio subaccounts
- Messaging Services (MessagingServiceSid-based sending)
- Scheduled messages
- Message redaction / deletion
- Real SMS delivery to actual phone networks
- WhatsApp or any non-SMS channel
- User-facing product control plane for cloud
- Billing / usage APIs
- Rate limiting / throttling simulation
- Multi-region cloud deployment

### Ambiguous (requires clarification)

- **Cloud provider**: Terraform needs a target. AWS is the most common, but the directive doesn't specify. I'll assume AWS (ECS/Fargate or EC2) unless directed otherwise.
- **Account bootstrapping**: Should the twin auto-create a default account on first start, or require explicit creation via API? I'll assume auto-bootstrap a default account for ease of use.
- **Status callback webhooks**: Twilio supports StatusCallback for outbound message status updates. This is adjacent to the core SMS scenario. I'll include it as it's important for real-world Twilio integrations.

## Conflicts and Tensions

1. **SQLite vs Postgres**: PRINCIPLES.md §6 says "the package run locally and deployed to twins.la MUST be the same package." If local uses SQLite and cloud uses Postgres, the twin code has two storage backends. Resolution: The twin should define a storage interface. The *host* provides the storage backend. The twin package itself remains identical — only the host configuration differs. This preserves the principle while allowing appropriate datastores per environment.

2. **Fidelity vs implementation cost for webhook signing**: HMAC-SHA1 webhook signing is straightforward to implement and is required for real-world Twilio integration compatibility. No conflict — include it.

3. **Twin Plane scope**: PRINCIPLES.md requires a Twin Plane with a common contract, logs, settings, and scenario listing. This is additional API surface beyond the Twilio emulation. It's in scope per principles but must not interfere with Twilio API fidelity. Resolution: Serve Twin Plane on a separate path prefix (e.g., `/_twin/`).

## PROBLEMS REQUIRING HUMAN ACTION

1. **Cloud provider confirmation**: I need to know the target cloud provider for Terraform. I'll assume AWS unless you redirect me. This affects Terraform resource definitions, networking, and the Jenkinsfile.

2. **DNS / endpoint strategy**: `twilio.twins.la` implies DNS configuration that exists outside these repos. For 0.1.0, should the cloud deployment simply expose an IP/load-balancer endpoint, with DNS as a future step? Or is twins.la DNS already configured?

## PROPOSED NEW ANALYSIS

- None at this time. The scope is well-defined by the directive.
