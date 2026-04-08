# Explainer Page Template

Every twin hosted on twins.la MUST have an explainer page and an agent-instructions endpoint. This document defines what they must contain.

## Explainer Page (`GET /`)

The root URL of each twin MUST return an HTML page that serves two audiences:

### Human Section
The page MUST include:

1. **Title** — the twin's name and domain (e.g., "twilio.twins.la")
2. **What it is** — one-paragraph explanation of what this twin does, written for a developer with no prior context
3. **Supported scenarios** — what the twin can do, stated concretely (e.g., "Send and receive SMS via the Twilio REST API")
4. **How to use it** — brief instructions for both cloud (point at the URL) and local (pip install and run)
5. **Links** — GitHub repo, detailed docs, the main twins.la page

### Agent Section
The page MUST include a clearly labeled, copyable code block containing the agent instructions (the same text served by `/_twin/agent-instructions`). The block should be introduced with text like "Copy this into your agent's system prompt or configuration."

### Design
The page MUST follow the twins.la visual design:
- Background: `#f8f8f8`
- Font: Inter (with system fallbacks)
- Text: dark navy (`#1a2e4a`) for headings, gray (`#6b7280`) for body
- Accent: teal (`#0e7490`) for links on the landing page; each twin page uses its own accent color reflecting the provider's brand (e.g., Twilio rose `#e11d48`)
- Cards/code blocks: white (`#ffffff`) with `#e5e7eb` border and subtle shadow
- Favicon: `https://twins.la/twins.png`
- Max content width: `700px`, centered
- Responsive (works on mobile)

Each twin's explainer page SHOULD use a thematic accent color from the provider's brand for:
- The provider name in the h1 (e.g., `<span class="twilio">twilio</span>.twins.la`)
- Links and interactive highlights
- List item markers

The page SHOULD be self-contained HTML with inline CSS (no external stylesheets beyond the font import). This keeps pages independent and avoids cross-origin dependencies.

## Agent Instructions Endpoint (`GET /_twin/agent-instructions`)

Each twin MUST expose a `GET /_twin/agent-instructions` endpoint that returns `text/plain` content.

### Content Requirements
The response MUST include:

1. **Identity** — what the twin is and its URL
2. **Authentication** — how to authenticate requests
3. **Key endpoints** — the most important API endpoints with brief descriptions
4. **Quick start** — minimal steps to make a first successful API call
5. **Reference** — link to detailed documentation on GitHub

### Format
- Plain text, not JSON or HTML
- Self-contained — an agent reading only this text can make API calls
- Concise — under 80 lines, under 3000 characters
- No marketing language — direct, technical, actionable

### Example Structure
```
# {Twin Name} — {domain}

{One-line description}

## Authentication
{How to authenticate}

## Key Endpoints
{Endpoint list with methods and descriptions}

## Quick Start
{Minimal steps to first API call}

## Local Usage
{How to run locally}

## Reference
{Link to GitHub docs}
```

## twins.la Landing Page

The main `twins.la` page follows the same design but has additional content:
- Project overview (what twins.la is as a whole)
- List of available twins with links
- Getting started guide (local and cloud)
- Agent snippet for the project as a whole (not twin-specific)

The landing page is maintained in the `website` repo, not in individual twin repos.
