---
title: "twins.la"
summary: "High-fidelity digital twins of real APIs — write against a twin, deploy against the real thing."
shipped: 2026-03-01
tags: [api, digital-twins, testing, developer-tools, agents]
links:
  - label: "Website"
    url: "https://twins.la"
    primary: true
  - label: "Twilio Twin"
    url: "https://twilio.twins.la"
icon: "../twins.png"
---

## What is it?

twins.la provides drop-in digital twins of real APIs. Write and test your code against a twin, then switch to the real service by changing the hostname and credentials. Each twin runs locally via pip install or in the cloud at its own subdomain — same package, same behavior.

## Key Features

- **Exact API fidelity** — Within supported scenarios, code written against a twin works against the real provider without changes
- **Local and cloud, same package** — `pip install` for local dev, or point at the hosted URL. No behavioral drift between the two
- **Twin Plane** — Standardized management API (`/_twin/`) across all twins for health checks, logs, scenarios, and account creation
- **Agent-ready** — Each twin serves machine-readable instructions at `/_twin/agent-instructions` for LLM agents and automation
- **Scenario-driven** — Twins define explicit supported scenarios with strict fidelity. No ocean-boiling — narrow enough to build, strict enough to trust

---
[View on ishipped.io](https://ishipped.io/card/twins-la/twins-la)
