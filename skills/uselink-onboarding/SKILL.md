---
name: uselink-onboarding
description: "Scan a codebase and generate a new-developer onboarding guide as HTML, then publish to uselink. Use when the user wants to create a getting-started guide, onboard a new team member, or share 'how to work in this repo' docs."
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
---

# Onboarding Guide → Uselink

Scan a codebase, generate a comprehensive getting-started guide for new developers, and publish it to uselink as a shareable link.

## When to Use

- "Create an onboarding guide for the new hire"
- "How do I explain this repo to someone new? Publish it."
- "Generate a getting-started doc and share it"
- "Make a dev setup guide I can send to contractors"

## Prerequisites

```bash
which uselink || echo "MISSING"
test -f ~/.uselink/config.json && echo "OK" || echo "NO_CONFIG"
```

## Workflow

### 1. Scan the codebase

```bash
# Project identity
git remote get-url origin 2>/dev/null
basename "$(git rev-parse --show-toplevel)"

# Build system
for f in Makefile package.json build.gradle.kts Cargo.toml go.mod pyproject.toml docker-compose.yml; do
  test -f "$f" && echo "BUILD: $f"
done

# Environment files
ls .env.example .env.sample .env.template 2>/dev/null
ls .tool-versions .node-version .java-version .python-version .ruby-version 2>/dev/null

# Documentation
ls README.md CLAUDE.md CONTRIBUTING.md docs/ .github/CONTRIBUTING.md 2>/dev/null

# Infrastructure
ls Dockerfile docker-compose.yml 2>/dev/null
ls .github/workflows/*.yml 2>/dev/null

# Directory structure
find . -maxdepth 2 -type d -not -path "*/.git*" -not -path "*/node_modules*" -not -path "*/build*" -not -path "*/__pycache__*" | sort
```

### 2. Read key files

Read (first 150 lines of each):
- `README.md` — existing setup instructions
- `CLAUDE.md` — architecture and conventions
- `Makefile` or equivalent — available commands
- `.env.example` — required environment variables
- `docker-compose.yml` — required services
- Main entry point file

### 3. Generate HTML

Write to `/tmp/uselink-onboarding-<project>-<timestamp>.html`.

Sections:

```html
<!-- Welcome -->
<div class="card">
  <h3>Welcome to {{PROJECT}}</h3>
  <p>{{ONE_LINE_DESCRIPTION}}</p>
  <p class="muted">This guide will get you from zero to running the app locally.</p>
</div>

<!-- Prerequisites -->
<h2>Prerequisites</h2>
<div class="card">
  <table>
    <thead><tr><th>Tool</th><th>Version</th><th>Install</th></tr></thead>
    <tbody>
      <tr><td>{{TOOL}}</td><td>{{VERSION}}</td><td><code>{{INSTALL_CMD}}</code></td></tr>
    </tbody>
  </table>
</div>

<!-- Clone & Setup -->
<h2>Clone & Setup</h2>
<div class="card">
  <pre><code>git clone {{REPO_URL}}
cd {{PROJECT}}
{{SETUP_COMMANDS}}</code></pre>
</div>

<!-- Environment Variables -->
<h2>Environment Variables</h2>
<div class="card">
  <p>Copy the example env file:</p>
  <pre><code>cp .env.example .env</code></pre>
  <table>
    <thead><tr><th>Variable</th><th>Required</th><th>Description</th></tr></thead>
    <tbody>
      <tr><td><code>{{VAR}}</code></td><td>{{REQ}}</td><td>{{DESC}}</td></tr>
    </tbody>
  </table>
</div>

<!-- Running Locally -->
<h2>Running Locally</h2>
<div class="card">
  <pre><code>{{RUN_COMMANDS}}</code></pre>
  <p class="muted">The app should be available at <code>{{LOCAL_URL}}</code></p>
</div>

<!-- Architecture Overview -->
<h2>Architecture</h2>
<div class="card"><pre><code>{{ASCII_DIAGRAM}}</code></pre></div>
<p>{{ARCHITECTURE_DESCRIPTION}}</p>

<!-- Directory Guide -->
<h2>Where Things Live</h2>
<table>
  <thead><tr><th>Path</th><th>What's in it</th></tr></thead>
  <tbody>
    <tr><td><code>{{DIR}}</code></td><td>{{PURPOSE}}</td></tr>
  </tbody>
</table>

<!-- Common Commands -->
<h2>Common Commands</h2>
<table>
  <thead><tr><th>Command</th><th>What it does</th></tr></thead>
  <tbody>
    <tr><td><code>{{CMD}}</code></td><td>{{DESC}}</td></tr>
  </tbody>
</table>

<!-- Conventions -->
<h2>Conventions</h2>
<div class="card">
  <ul>
    <li><strong>{{CONVENTION}}</strong> — {{EXPLANATION}}</li>
  </ul>
</div>

<!-- First Task Suggestions -->
<h2>Good First Tasks</h2>
<div class="card">
  <ol>
    <li>{{TASK}} — {{WHY_ITS_GOOD_FOR_NEW_DEVS}}</li>
  </ol>
</div>

<!-- Who to Ask -->
<h2>Who to Ask</h2>
<div class="card">
  <table>
    <thead><tr><th>Area</th><th>Person</th></tr></thead>
    <tbody>
      <tr><td>{{AREA}}</td><td>{{PERSON_OR_PLACEHOLDER}}</td></tr>
    </tbody>
  </table>
  <p class="muted">Not sure? Check <code>git blame</code> on the file you're working in.</p>
</div>
```

### 4. Publish

```bash
uselink publish /tmp/uselink-onboarding-<project>-<timestamp>.html \
  --title "Onboarding Guide -- <project>" \
  --format html
```

Return the URL to the user.

## Gotchas

- **NEVER set background or color on body, table, th, td, code, or pre.** uselink's viewer injects theme-adaptive CSS that handles light/dark mode automatically. Any hardcoded color in published HTML overrides the viewer's theme and forces a white background in dark mode. Only style layout (padding, margins, borders, border-radius, font-size, grid). Use `opacity` for muted text instead of `color: #888`.
- **Test the setup commands yourself.** Run the commands you're about to document and verify they work. "Clone, install, run" should be copy-pasteable. If a step requires manual intervention (like `uselink login`), call it out explicitly.
- **Don't fabricate team members for "Who to Ask."** If you can't determine team members from git blame or README, use placeholder text like "Check git blame" or ask the user.
- **Environment variables from `.env.example` need real descriptions.** Don't just list the variable names. Read the codebase to understand what each one does and whether it's required or optional.
- **Makefile targets are the best command reference.** If a `Makefile` exists, read it thoroughly — it's usually the canonical list of developer commands.
- **HTML-escape all interpolated values.** File paths and command output can contain special characters.
- **Skip sensitive defaults.** If `.env.example` has API keys or passwords (even dummy ones), note that the user needs to fill them in. Don't publish actual credentials.
