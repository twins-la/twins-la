# twins.la Logging Principles and Normative Specification

## Purpose

This document defines the logging contract for every twin in the twins.la system. It is both a philosophical guide — stating why logs exist and what they must enable — and a normative specification that engineers implement against directly.

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "MAY", "REQUIRED", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

This document expands on Principle 11 (Logging is mandatory and consistent) and Principle 8 (Tenancy) from [PRINCIPLES.md](PRINCIPLES.md). Where this document adds detail, it is authoritative; where it conflicts with PRINCIPLES.md, PRINCIPLES.md wins.

---

## 1. Core Principles

### 1.1 Logs MUST be self-sufficient

A reader MUST be able to determine what happened and why — including the specific reason for any failure — without access to the originating request, the caller's inputs, or any out-of-band context. A log record that requires the caller's state to be interpretable is a defect.

### 1.2 Logs MUST enable complete narrative reconstruction

The sequence of log records for a given operation or related set of operations MUST be sufficient to reconstruct the complete operational narrative: what was attempted, in what order, with what outcome, and why. Partial logging that captures outcomes without antecedents, or antecedents without outcomes, is non-conformant.

### 1.3 Logs MUST be consistent in structure and semantics

The normative log format defined in §3 MUST be identical across:

* all twin libraries (e.g., Facebook, Twilio, LiveKit, and any future twin)
* all deployment environments (local and cloud)

Consistency is structural (field names, types, shape) and semantic (a given `operation` value means the same thing everywhere). A twin MUST NOT redefine normative field names or vocabulary.

### 1.4 Logs MUST be clear, deterministic, and programmatically readable

Log records MUST be machine-parseable. They MUST NOT depend on natural-language interpretation for fields that are used for correlation, filtering, or reconstruction. Free-text fields are permitted only where explicitly allowed by the normative format.

---

## 2. Logging Scope and Coverage

Every twin MUST log the following categories of activity. A twin that omits a category is non-conformant.

### 2.1 Resource operations

Operations that create, configure, or mutate resources managed by the twin. This includes, but is not limited to:

* resource provisioning (creation)
* resource configuration (metadata, policy, routing)
* resource mutation (updates, state changes, deletion)

### 2.2 Runtime operations

Operations invoked against a twin's Control Plane or Data Plane as defined in PRINCIPLES.md. This includes, but is not limited to:

* API calls received by the twin
* internal execution events that materially affect outcome
* responses and outcomes returned to the caller

### 2.3 Failure details

Every failure MUST be logged with sufficient detail to diagnose it without reference to the caller's input. This includes:

* validation failures (malformed requests, schema errors, missing required fields)
* execution failures (runtime errors, downstream failures, state conflicts)
* an explicit, machine-readable reason for the failure

A failure logged only as "error" or "failed" is non-conformant.

---

## 3. Normative Log Format Specification

### 3.1 Record shape

Each log record MUST be a JSON object containing, at minimum, the fields listed in §3.2. A twin MUST NOT omit any required field. A twin MUST NOT rename any required field. A twin MUST NOT change the type or semantics of any required field.

### 3.2 Required fields

| Field | Type | Requirement | Description |
|-------|------|-------------|-------------|
| `timestamp` | string | MUST | UTC timestamp in RFC 3339 / ISO 8601 format with at least millisecond precision and a `Z` suffix (e.g., `2026-04-12T14:05:03.123Z`). Local time MUST NOT be used. |
| `twin` | string | MUST | Stable short identifier of the twin library emitting the record (e.g., `facebook`, `twilio`, `livekit`). |
| `tenant_id` | string | MUST | Tenant scope of the operation. In local deployments this MUST be `"default"`. In cloud deployments this MUST NOT be `"default"`. |
| `correlation_id` | string | MUST | Identifier that groups related records belonging to a single logical operation or operation chain. Records emitted in service of the same inbound request MUST share a `correlation_id`. |
| `plane` | string | MUST | One of `"twin"`, `"control"`, `"data"`, or `"runtime"`. Identifies which plane (per PRINCIPLES.md) the operation belongs to. |
| `operation` | string | MUST | Dotted, lowercase, machine-readable operation type (e.g., `resource.create`, `message.send`, `token.validate`). The vocabulary is normative; new values MUST be introduced through the governance process in §7. |
| `resource` | object or null | MUST | `{ "type": "<resource-type>", "id": "<resource-id>" }` for operations bound to a resource; `null` for operations not bound to a resource. |
| `outcome` | string | MUST | Exactly one of `"success"` or `"failure"`. No other values are permitted. |
| `reason` | string or null | MUST | For `outcome = "failure"`, a non-empty human-readable explanation of the failure sufficient to diagnose it without the caller's input. For `outcome = "success"`, MAY be `null` or a short explanatory string. |
| `details` | object | MUST | Structured additional context. MUST be present (possibly empty `{}`). Extension point for verbose mode (§4). |

### 3.3 Record identity and ordering

Each record SHOULD carry a stable, monotonically increasing identifier within the scope of its tenant to support pagination and deterministic ordering. Implementations MAY provide this via an envelope field (e.g., `id`) returned by the log read endpoint without altering the normative record shape.

### 3.4 Extensibility

The format is extensible through the `details` object and through future additions to required fields under §7. Twins MAY add new keys inside `details` without governance approval, provided:

* new keys MUST NOT duplicate or shadow required field names
* new keys MUST NOT be required for correct interpretation of the record
* additions MUST NOT remove, rename, or redefine required fields

### 3.5 Format invariants

The normative format:

* MUST be uniform across all twins
* MUST NOT depend on any API shape specific to a given twin or upstream provider
* SHOULD be extensible without breaking existing consumers

### 3.6 Cross-twin query compatibility

Because the format is uniform, tooling that queries, filters, aggregates, or reconstructs operations MUST be able to operate across twins without per-twin branching on required fields. A twin that forces consumers to branch on twin identity for required fields is non-conformant.

---

## 4. Operational Modes

### 4.1 Standard (normative) view

Every twin MUST provide a standard view that emits exactly the normative format defined in §3. The standard view MUST be consistent across twins and environments. Programmatic consumers SHOULD target the standard view.

### 4.2 Verbose view

A twin MAY implement a verbose view. Verbose view requirements:

* Verbose records MAY include additional keys inside `details` (or, where appropriate, additional top-level keys that do not collide with §3.2).
* Verbose records MUST continue to satisfy the normative format — every required field from §3.2 MUST still be present with its required type and semantics.
* Verbose output MUST NOT rename, remove, or repurpose any required field.
* Enabling verbose mode MUST NOT change the meaning of any record visible in standard view.

### 4.3 Mode selection

How a consumer selects standard vs verbose view is implementation-defined (e.g., a query parameter on the log read endpoint). Mode selection MUST NOT grant access to records or fields that the caller would not otherwise be authorized to see.

---

## 5. Retention and Environment Differences

### 5.1 Cloud deployments

In cloud deployments:

* logs MUST be treated as ephemeral
* logs MAY be deleted at any time, without notice
* retention is NOT guaranteed and MAY be less than 24 hours

Consumers MUST NOT rely on cloud logs as a system of record. Any consumer that requires durable retention MUST export and persist records to its own store.

### 5.2 Local deployments

In local deployments:

* logs are ephemeral by default
* retention MAY be configurable by the operator

Regardless of retention configuration, logs in local deployments MUST conform to the normative format defined in §3.

### 5.3 Consistency across environments

Retention differs between environments. Format MUST NOT. A record emitted locally and a record emitted in cloud MUST be indistinguishable in structure given the same operation.

---

## 6. Security and Tenancy

### 6.1 Strict tenant isolation

Every log record MUST carry a `tenant_id` and MUST be accessible only within the scope of that tenant. Tenant isolation of logs is a strict rule: cross-tenant access MUST NOT be possible under any circumstances through the normative log interface.

### 6.2 Administrator-scoped access

Access to logs MUST be limited to administrators of the same tenant. A non-administrator caller within a tenant MUST NOT be able to read that tenant's logs through the normative log interface. Authentication and authorization MUST be enforced at the log read path; an unauthenticated or broadly-authorized log endpoint is non-conformant.

### 6.3 No cross-tenant leakage

A log read request MUST return only records whose `tenant_id` matches the authenticated tenant. Filtering MUST be enforced server-side. Client-side filtering of a cross-tenant result set is non-conformant.

### 6.4 Sensitive content

Log records MUST include sufficient content for narrative reconstruction (§1.1, §1.2). Twins SHOULD avoid capturing material that is not useful for reconstruction (e.g., full authentication secrets). Where content is captured, access controls defined in §6.1–§6.3 are the enforcement boundary.

---

## 7. Governance and Evolution

### 7.1 Normative format is authoritative

All twins MUST adhere to the normative log format defined in §3. A twin that diverges from the normative format is non-conformant regardless of internal justification.

### 7.2 Changes are centralized

Any required change or extension to the normative format:

* MUST be surfaced for inclusion in this specification
* MUST NOT be implemented unilaterally by an individual twin
* MUST be approved by the Twin system owner before adoption

Unilateral twin-specific changes to required fields are non-conformant. Verbose-mode additions inside `details` per §3.4 and §4.2 are the only sanctioned extension path that does not require approval.

### 7.3 Backward compatibility

Changes to the normative format SHOULD preserve backward compatibility with existing consumers. Where a breaking change is unavoidable, it MUST be versioned and announced; twins MUST NOT silently change required field semantics.

### 7.4 Conformance

A twin MUST be able to demonstrate conformance to this specification. Smoke tests and Twin Plane operation tests (per PRINCIPLES.md §14) SHOULD include coverage that the log read path returns records matching the normative format.

---

## 8. Interpretation

Where there is tension between completeness and brevity in a log record, completeness sufficient for narrative reconstruction (§1.1, §1.2) MUST win.

Where there is tension between cross-twin uniformity and twin-specific convenience, uniformity (§1.3, §3.5) MUST win. Twin-specific detail belongs in `details` or verbose mode.

Where there is tension between operator flexibility in local deployments and format stability, format stability MUST win. Retention is configurable; format is not.
