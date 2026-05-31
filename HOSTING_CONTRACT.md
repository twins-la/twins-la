# Hosting Contract

This document defines the interface between a twin package and a host.

## Overview

A **twin** is a Python package that implements a provider emulation and a Twin Plane. A **host** is a runtime environment that loads the twin, provides it with infrastructure (storage, configuration), and serves it over HTTP.

The same twin package runs in both local and cloud hosts. Behavioral differences between hosts come only from infrastructure (e.g., SQLite vs PostgreSQL), never from twin code branching on the host type.

## Twin Package Requirements

A twin package MUST provide:

```python
from flask import Flask
from twins_twilio.storage import TwinStorage  # or equivalent for the twin

def create_app(storage: TwinStorage, config: dict) -> Flask:
    """Create and return the configured Flask application.

    Args:
        storage: A host-provided TwinStorage implementation.
        config: Configuration dict. Required keys:
            - "base_url" (str): The public-facing URL of the twin.

    Returns:
        A Flask app with all routes registered.
    """
```

A twin package MUST:
- Accept a `TwinStorage` implementation from the host
- Accept configuration as a plain dict
- Return a standard Flask (WSGI) application
- NOT embed any storage driver or database dependency
- NOT import host-specific code

## Host Requirements

A host MUST provide:

1. **A `TwinStorage` implementation** appropriate for the environment:
   - Local: `SQLiteStorage` (file-backed, persistent across restarts)
   - Cloud: `PostgresStorage` (server-backed, persistent across restarts)

2. **Configuration** including at minimum `base_url`.

3. **HTTP serving** of the Flask application (e.g., via gunicorn).

4. **Persistence** — state MUST survive host restarts. The host owns the datastore lifecycle.

5. **A WSGI entry point** callable by gunicorn or equivalent:
   ```python
   def create_local_app() -> Flask:  # or create_cloud_app()
   ```

## Storage Interface

The `TwinStorage` ABC is defined in the twin package. Hosts implement it.

```python
class TwinStorage(ABC):
    # Accounts
    def create_account(sid, auth_token, friendly_name) -> dict
    def get_account(sid) -> Optional[dict]
    def list_accounts() -> list[dict]

    # Phone Numbers
    def create_phone_number(data: dict) -> dict
    def get_phone_number(account_sid, sid) -> Optional[dict]
    def get_phone_number_by_number(account_sid, phone_number) -> Optional[dict]
    def list_phone_numbers(account_sid) -> list[dict]
    def update_phone_number(account_sid, sid, updates) -> Optional[dict]

    # Messages
    def create_message(data: dict) -> dict
    def get_message(account_sid, sid) -> Optional[dict]
    def list_messages(account_sid, filters=None) -> list[dict]
    def update_message(account_sid, sid, updates) -> Optional[dict]

    # Logs
    def append_log(entry: dict) -> None
    def list_logs(limit=100, offset=0) -> list[dict]
```

Storage methods use plain dicts for data. The twin defines what keys it expects; the storage implementation handles serialization.

## Adding a New Twin

This is the **canonical, mandatory checklist** for introducing a new twin to the platform. Every step below MUST be completed. A twin that ships its own repo without the cross-cutting surface edits is not "added" — it is invisible to users and to other twins. Reviewers and operators MUST treat any unchecked item as a blocker.

### A. Twin repository (`twins-la/<provider>`)

1. Create the twin package `twins_<provider>/` with `create_app(storage, tenants, config) -> Flask`.
2. Define the twin's storage interface as a `TwinStorage` ABC (may extend the cross-twin shape; never import a DB driver in the twin package).
3. Expose every required `/_twin/` Twin Plane endpoint per [TWIN_PLANE.md](TWIN_PLANE.md): `health`, `scenarios`, `references`, `settings`, `tenants` (POST bootstrap), `logs`, `accounts`, plus twin-specific `/simulate/*` as the scenario warrants. **Also add a catch-all under the provider's documented URL prefix** that returns the provider's documented JSON error envelope on any unmatched path. Without this catch-all, Flask returns its default HTML 404 on every unimplemented endpoint, which breaks SDK consumers that decode `Content-Type: application/json` (silent until a human notices). See `twins-la/aoai`'s `routes/openai_data.py:unknown_openai_path` for the reference impl. Closes the recurrence pattern named in twins-la/twins-la#2.
4. Add `SCENARIOS.md` with cited authoritative URLs and retrieval dates. Mirror the same list at `GET /_twin/references`.
5. Enumerate the provider's documented endpoints. Before scope-locking the twin's v0.1.0, fetch the provider's reference page (the same URL cited in `SCENARIOS.md`) and list every documented endpoint with its current GA / preview status. Mark each as one of: in-scope-for-v0.1.0, deferred-with-target-version, or out-of-scope-with-rationale. Surface the list to whoever approves scope; an unmarked endpoint is a missed endpoint.
6. Add the explainer landing page at `GET /` and `GET /_twin/agent-instructions` per [EXPLAINER_TEMPLATE.md](EXPLAINER_TEMPLATE.md).
7. Implement the local sibling host package `twins_<provider>_local/` (SQLite storage + `python -m twins_<provider>_local`).
8. Add smoke tests under `tests/smoke/` covering: Twin Plane info endpoints, tenant bootstrap + auth, every supported provider-API method (happy + error paths), inbound simulation, log conformance to [LOGGING.md §3.2](LOGGING.md), tenant isolation, admin auth, feedback, **and a parametrized sweep that asserts every unknown path under the provider's URL prefix returns `Content-Type: application/json`, the documented error envelope, and no `<!doctype` HTML leak**. (See `twins-la/twilio` for the reference shape and footprint; `twins-la/aoai`'s `tests/smoke/test_unknown_openai_path.py` for the catch-all sweep pattern.)
9. `README.md` + `LICENSE` (MIT).

### B. Cloud aggregator, deploy, and Terraform (`twins-la/cloud`)

The cloud build, deploy, CI, and Terraform steps for a new twin live in the
private `twins-la/cloud` repo — they reference internal infrastructure that
twin contributors do not need. See the **"Adding a New Twin"** runbook in
`twins-la/cloud`.

### C. Cross-cutting surfaces (MANDATORY — these are the most-missed)

10. **`twins-la/twins-la` README**: add the new twin to the *Available Twins* table AND to the agent-instructions snippet. A twin missing from this README is undiscoverable through the project's front door.
11. **`twins-la/twins-la-website` (`twins.la` landing page)**: add a brand color (`.${provider}` CSS class), a twin card, an entry in the agent-instructions block, and a footer link. A twin missing from the website is invisible to every visitor of `twins.la`.
12. **Build + push** the website image and roll the website Container App. Edits to the website repo do not auto-deploy without an explicit roll (or a new build that updates `:latest`).

### D. Verification (don't sign off without these)

The end-to-end verification steps (health/settings curls against `https://<provider>.twins.la`, the website and README surface checks, and the deploy-side surface-parity enforcement) live alongside the deploy steps in the private `twins-la/cloud` runbook.

A new twin is **not** "added" until the cloud runbook's checklist passes *and* items 10–12 above are complete. Items 10–12 are the most commonly missed: a twin that ships its own repo and even deploys to its own subdomain is still invisible to humans and agents until the meta-repo README and the website list it. A missing edit in `twins-la-website` will fail the build loudly.

### Reviewer checklist

When reviewing a "new twin" change, the reviewer MUST verify items 10–12 by reading the diffs in `twins-la/twins-la` and `twins-la/twins-la-website`. The absence of those diffs is a missing-surface defect, not a follow-up item.

## Version

This contract is at version 0.1.0 and will evolve as additional twins are added.
