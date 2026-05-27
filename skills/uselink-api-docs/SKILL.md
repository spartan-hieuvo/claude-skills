---
name: uselink-api-docs
description: "Scan backend controllers/routes, generate API documentation as HTML, and publish to uselink. Use when the user wants to share API docs with frontend devs, partners, or stakeholders who need to understand the API contract."
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
---

# API Docs → Uselink

Scan a codebase for API endpoints, generate clean API documentation as HTML, and publish to uselink.

## When to Use

- "Generate API docs and share them"
- "Document our endpoints for the frontend team"
- "Share the API contract with the mobile dev"
- "What endpoints do we have? Publish a reference."

## Prerequisites

```bash
which uselink || echo "MISSING"
test -f ~/.uselink/config.json && echo "OK" || echo "NO_CONFIG"
```

## Workflow

### 1. Detect framework and find endpoints

**Micronaut/Kotlin:**
```bash
grep -rn "@Controller\|@Get\|@Post\|@Put\|@Delete" \
  --include="*.kt" -l | grep -v build | grep -v test
```

**Express/Node:**
```bash
grep -rn "router\.\(get\|post\|put\|delete\)\|app\.\(get\|post\|put\|delete\)" \
  --include="*.ts" --include="*.js" -l | grep -v node_modules | grep -v test
```

**FastAPI/Python:**
```bash
grep -rn "@app\.\(get\|post\|put\|delete\)\|@router\.\(get\|post\|put\|delete\)" \
  --include="*.py" -l | grep -v __pycache__ | grep -v test
```

**Spring/Java:**
```bash
grep -rn "@RestController\|@GetMapping\|@PostMapping\|@RequestMapping" \
  --include="*.java" --include="*.kt" -l | grep -v build | grep -v test
```

### 2. Read each controller

For each controller file:
- Extract the base path (e.g., `@Controller("/api/v1/documents")`)
- Extract each endpoint: method, path, query params, body type, return type
- Extract auth requirements (e.g., `@Secured`, middleware)
- Read the request/response DTOs to get field names and types

### 3. Group endpoints

Group by resource/domain:
- `/api/v1/documents/*` → Documents
- `/api/v1/auth/*` → Authentication
- `/api/v1/comments/*` → Comments

### 4. Generate HTML

Write to `/tmp/uselink-api-docs-<project>-<timestamp>.html`.

```html
<!-- API header -->
<div class="card">
  <h3>{{PROJECT}} API Reference</h3>
  <p class="muted">{{ENDPOINT_COUNT}} endpoints · Generated {{DATE}}</p>
  <p>Base URL: <code>{{BASE_URL}}</code></p>
</div>

<!-- Table of contents -->
<h2>Endpoints</h2>
<table>
  <thead><tr><th>Method</th><th>Path</th><th>Description</th><th>Auth</th></tr></thead>
  <tbody>
    <tr>
      <td><span class="tag tag-done">POST</span></td>
      <td><code>{{PATH}}</code></td>
      <td>{{SHORT_DESC}}</td>
      <td>{{AUTH_REQUIREMENT}}</td>
    </tr>
  </tbody>
</table>

<!-- Per-resource sections -->
<h2>{{RESOURCE_NAME}}</h2>

<!-- Per-endpoint detail -->
<h3><span class="tag tag-done">POST</span> <code>{{PATH}}</code></h3>
<div class="card">
  <p>{{DESCRIPTION}}</p>

  <h4>Query Parameters</h4>
  <table>
    <thead><tr><th>Name</th><th>Type</th><th>Required</th><th>Description</th></tr></thead>
    <tbody>
      <tr><td><code>{{PARAM}}</code></td><td>{{TYPE}}</td><td>{{REQ}}</td><td>{{DESC}}</td></tr>
    </tbody>
  </table>

  <h4>Request Body</h4>
  <pre><code>{{REQUEST_JSON_EXAMPLE}}</code></pre>

  <h4>Response</h4>
  <pre><code>{{RESPONSE_JSON_EXAMPLE}}</code></pre>
</div>
```

Use color-coded method tags:
- `GET` → green (`tag-done`)
- `POST` → blue (`tag-nit`)
- `DELETE` → red (`tag-bug`)

### 5. Publish

```bash
uselink publish /tmp/uselink-api-docs-<project>-<timestamp>.html \
  --title "API Reference -- <project> -- <date>" \
  --format html
```

Return the URL to the user.

## Gotchas

- **NEVER set background or color on body, table, th, td, code, or pre.** uselink's viewer injects theme-adaptive CSS that handles light/dark mode automatically. Any hardcoded color in published HTML overrides the viewer's theme and forces a white background in dark mode. Only style layout (padding, margins, borders, border-radius, font-size, grid). Use `opacity` for muted text instead of `color: #888`.
- **Generate example JSON, don't guess.** Read the actual DTO/model classes to get field names, types, and defaults. Don't invent example values that don't match the schema.
- **Use snake_case in JSON examples if that's the wire format.** Check the project's serialization config (Jackson SNAKE_CASE, etc.) and match the actual API output.
- **Skip internal/admin endpoints unless asked.** If the docs are for external consumers, filter out admin-only routes. Ask the user if unclear.
- **Auth info matters.** Always show whether an endpoint requires authentication and what role/scope is needed. Stakeholders need this to understand access patterns.
- **HTML-escape all code examples.** JSON with angle brackets or ampersands in string values will corrupt the HTML.
- **Large APIs need a TOC.** If there are 20+ endpoints, the table of contents at the top is critical for navigation.
