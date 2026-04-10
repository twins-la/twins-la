# twins.la Operating Principles

## Purpose

twins.la exists to provide digital twins that let developers write code against a twin and then run that code against the real system without scenario-specific changes. Exact API fidelity within supported scenarios is the primary goal.

## Principles

### 1. Fidelity is the top priority

Within a supported scenario, a twin MUST preserve the real system’s externally observable behavior closely enough that code written against the twin works against the real system.

This MUST include, where applicable:

* request and response structure
* authentication and authorization shape
* identifier formats and comparable value structure
* error behavior
* observable state transitions
* runtime semantics relevant to the scenario

A twin MUST NOT require reuse of the real provider’s literal values or bytes. A twin MUST be self-contained and MUST NOT require the caller to interact with the real provider.

### 2. Scenarios define fidelity boundaries

Each twin MUST define one or more scenarios.

A scenario MUST define scope at a high level. It MUST make clear what is in scope and what is out of scope without relying on an exhaustive checklist of individual details.

Scenario documentation is the sole source of fidelity claims. Anything not defined by the scenario documentation MUST be treated as twin-specific and non-authoritative.

### 3. Supported scenarios are normative

If a scenario is declared supported, the twin MUST behave faithfully within that scenario.

Any discovered behavioral delta within a supported scenario MUST be treated as a defect in the twin and MUST be fixed. Such a delta MUST NOT be explained away by documentation alone.

### 4. Out-of-scope behavior may be fabricated

Behavior outside a supported scenario MAY be fabricated, stubbed, approximated, or partially emulated.

Callers MUST assume that out-of-scope values may be fabricated.

Out-of-scope behavior MUST NOT be allowed to weaken fidelity claims for supported scenarios.

### 5. Twins must be self-contained

A twin MUST be usable without visiting, configuring, or depending on the real provider.

An implementer using a twin MUST NOT have to use the original system in order to develop, configure, run, inspect, or test a supported scenario.

### 6. One package, local and cloud

The package run locally and the package deployed to twins.la MUST be the same package.

Behavioral drift between local and twins.la MUST NOT be introduced through special-casing, alternate code paths, or platform-specific logic.

If a true exception is discovered, it MUST be surfaced explicitly.

### 7. Local startup must be trivial

A twin MUST be straightforward to run locally with minimal setup.

The principles MUST NOT prefer local over cloud or cloud over local. Users MAY choose either.

### 8. Twin Plane is standardized

Each twin MUST expose a Twin Plane.

The Twin Plane MUST have a common contract across twins. Its API shape and behavior MUST be consistent enough that tooling built for one twin can operate against another without per-provider rewrites.

Provider-specific extensions MAY exist only if they follow the project’s standard extension pattern.

The Twin Plane MUST expose the supported scenarios for the twin.

### 9. Twin Plane and provider surface are separate concepts

The Twin Plane is part of the twin, not part of the real provider.

A twin MAY omit the real provider’s control plane when the supported scenarios do not require it.

The presence or absence of a provider control plane MUST NOT weaken the fidelity requirement for supported scenarios.

### 10. Logging is mandatory

A twin MUST log all operations, including content.

A twin MUST provide a programmatic interface for retrieving logs.

A twin MUST provide a programmatic interface for settings.

### 11. Documentation must be source-grounded

All twins MUST include documentation.

Documentation MUST cite the authoritative sources used to build the twin, and MUST include retrieval dates for those sources.

The repository is the source of versioning, but the authoritative references MAY exist outside the repository and therefore MUST be cited.

### 12. Minimum project expectations are real

All twins MUST include:

* documentation
* authoritative references with retrieval dates
* smoke tests
* tests for all Twin Plane operations

Repository layout and other implementation conventions MUST follow the project’s existing conventions.

## Interpretation

When there is tension between fidelity and convenience, fidelity MUST win within supported scenarios.

When there is tension between breadth and correctness, correctness within the declared scenario MUST win.

The project SHOULD expand scenario coverage over time, but it MUST NOT dilute the meaning of “supported scenario” in order to appear broader than it is.

