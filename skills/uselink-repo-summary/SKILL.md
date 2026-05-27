---
name: uselink-repo-summary
description: "Scan a GitHub repo (local or remote), generate an architecture overview as HTML, and publish to uselink. Use when the user wants to share a repo summary, architecture overview, or codebase tour with stakeholders who don't have repo access."
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
---

# Repo Summary → Uselink

Scan a codebase, generate a clean architecture overview as HTML, and publish it to uselink as a shareable link.

## When to Use

- "Summarize this repo and share it"
- "Create an architecture overview for the PM"
- "Share the codebase structure with the new hire"
- User provides a GitHub URL and wants a summary page

## Prerequisites

```bash
which uselink || echo "MISSING"
test -f ~/.uselink/config.json && echo "OK" || echo "NO_CONFIG"
```

If either fails, tell the user to install (`npm install -g uselink`) or configure (`uselink login`). Do not proceed.

## Workflow

### 1. Get the repo

**Local repo (current directory):**
```bash
git remote get-url origin 2>/dev/null
basename "$(git rev-parse --show-toplevel)"
```

**Remote GitHub URL:**
```bash
gh repo clone <owner/repo> /tmp/uselink-repo-<name> -- --depth 1
cd /tmp/uselink-repo-<name>
```

### 2. Gather data

```bash
# Project name and description
gh repo view --json name,description 2>/dev/null

# Languages
gh repo view --json languages 2>/dev/null
find . -type f -not -path "*/node_modules/*" -not -path "*/.git/*" -not -path "*/build/*" \
  | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -10

# Directory structure (top 2 levels)
find . -maxdepth 2 -type d -not -path "*/.git*" -not -path "*/node_modules*" -not -path "*/build*" | sort

# Key files
for f in README.md CLAUDE.md package.json build.gradle.kts settings.gradle.kts Cargo.toml go.mod pyproject.toml Makefile docker-compose.yml; do
  test -f "$f" && echo "FOUND: $f"
done

# Git stats
git log --oneline -20
git shortlog -sn --no-merges | head -10
git log -1 --format="%ai"
```

### 3. Read key files

Read the first 100 lines of:
- `README.md` (project description)
- `CLAUDE.md` (architecture notes, if present)
- Main config file (`package.json`, `build.gradle.kts`, etc.)
- Entry point files (scan for `main`, `index`, `app`)

### 4. Generate HTML

Write to `/tmp/uselink-repo-summary-<name>-<timestamp>.html`.

Use the base HTML template with these sections:

```html
<!-- Summary card -->
<div class="card">
  <div class="summary-grid">
    <div class="summary-stat"><div class="num">{{TOTAL_FILES}}</div><div class="label">Files</div></div>
    <div class="summary-stat"><div class="num">{{CONTRIBUTORS}}</div><div class="label">Contributors</div></div>
    <div class="summary-stat"><div class="num">{{COMMITS}}</div><div class="label">Commits</div></div>
    <div class="summary-stat"><div class="num">{{LAST_ACTIVITY}}</div><div class="label">Last active</div></div>
  </div>
</div>

<!-- Tech Stack -->
<h2>Tech Stack</h2>
<table>
  <thead><tr><th>Layer</th><th>Technology</th></tr></thead>
  <tbody><!-- rows --></tbody>
</table>

<!-- Architecture -->
<h2>Architecture</h2>
<div class="card"><pre><code>{{ASCII_DIAGRAM}}</code></pre></div>

<!-- Directory Structure -->
<h2>Directory Structure</h2>
<div class="card"><pre><code>{{TREE}}</code></pre></div>

<!-- Key Modules -->
<h2>Key Modules</h2>
<!-- One card per module with path + purpose -->

<!-- How to Run -->
<h2>Getting Started</h2>
<div class="card"><pre><code>{{SETUP_COMMANDS}}</code></pre></div>

<!-- Recent Activity -->
<h2>Recent Activity</h2>
<table>
  <thead><tr><th>Commit</th><th>Author</th><th>Date</th></tr></thead>
  <tbody><!-- last 10 commits --></tbody>
</table>
```

Use the same inline CSS from uselink-report's base HTML shell (system font, cards, dark mode support).

### 5. Publish

```bash
uselink publish /tmp/uselink-repo-summary-<name>-<timestamp>.html \
  --title "Repo Summary -- <name>" \
  --format html
```

Return the URL to the user.

## Gotchas

- **NEVER set background or color on body, table, th, td, code, or pre.** uselink's viewer injects theme-adaptive CSS that handles light/dark mode automatically. Any hardcoded color in published HTML overrides the viewer's theme and forces a white background in dark mode. Only style layout (padding, margins, borders, border-radius, font-size, grid). Use `opacity` for muted text instead of `color: #888`.
- **Remote repos need `gh` CLI.** If the user gives a GitHub URL but `gh` is not installed, fall back to `git clone` with HTTPS. If that also fails (private repo), tell the user to clone it locally first.
- **Don't include secrets or env files in the summary.** Skip `.env`, `credentials.json`, and similar files when scanning. Never read or display their contents.
- **Large repos need sampling.** For repos with 1000+ files, show the top-level structure and 3-5 most important modules, not every file. Use `find -maxdepth 2` and read only the key files.
- **HTML must use inline CSS only.** Uselink strips external stylesheets. All styles go in a `<style>` block in `<head>`.
- **HTML-escape all repo-sourced strings.** Commit messages, file paths, and README content can contain `<`, `>`, `&` that corrupt the HTML.
