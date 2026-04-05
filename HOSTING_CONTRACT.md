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

To add a new twin to the platform:

1. Create a new twin package with `create_app(storage, config)`.
2. Define the twin's storage interface (may extend or differ from TwinStorage).
3. Implement storage backends in local/ and cloud/ for the new twin.
4. Add Docker Compose service (local) or Container App (cloud) for the new twin.
5. The twin MUST expose `/_twin/` Twin Plane endpoints.

## Version

This contract is at version 0.1.0 and will evolve as additional twins are added.
