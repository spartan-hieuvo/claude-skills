---
name: uselink-pr-digest
description: "Summarize a pull request as a stakeholder-friendly HTML page and publish to uselink. Use when the user wants to share PR changes with PMs, reviewers, or business stakeholders who don't read diffs."
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
---

# PR Digest → Uselink

Turn a pull request into a clean, readable summary page and publish it to uselink. Stakeholders see what changed and why — without reading diffs.

## When to Use

- "Summarize this PR for the PM"
- "Share PR #42 with the team"
- "Create a digest of what this branch does"
- User provides a PR number or URL

## Prerequisites

```bash
which uselink || echo "MISSING"
test -f ~/.uselink/config.json && echo "OK" || echo "NO_CONFIG"
which gh || echo "GH_MISSING"
```

If `uselink` or its config is missing, tell the user to install/configure it. `gh` CLI is needed for PR metadata.

## Workflow

### 1. Get PR data

**From PR number:**
```bash
gh pr view <number> --json title,body,author,state,additions,deletions,changedFiles,baseRefName,headRefName,mergedAt,createdAt,labels,reviewDecision
gh pr diff <number> --stat
gh pr diff <number> --name-only
```

**From current branch (no PR number given):**
```bash
gh pr view --json title,body,author,state,additions,deletions,changedFiles,baseRefName,headRefName,mergedAt,createdAt,labels,reviewDecision
git diff origin/main...HEAD --stat
git diff origin/main...HEAD --name-only
```

**From URL:**
Extract `owner/repo` and `number` from the URL pattern `github.com/<owner>/<repo>/pull/<number>`.

### 2. Analyze changes

```bash
# Get the full diff for analysis
gh pr diff <number>

# Or for current branch
git diff origin/main...HEAD
```

Read each changed file and categorize changes:
- **New features** — new files, new endpoints, new components
- **Bug fixes** — error handling changes, condition fixes
- **Refactoring** — renamed files, restructured code, no behavior change
- **Config/infra** — CI, Docker, dependencies, migrations

### 3. Generate HTML

Write to `/tmp/uselink-pr-digest-<number>-<timestamp>.html`.

```html
<!-- PR header -->
<div class="card">
  <h3>{{PR_TITLE}}</h3>
  <p class="muted">{{AUTHOR}} · {{BASE}} ← {{HEAD}} · {{DATE}}</p>
  <div class="summary-grid">
    <div class="summary-stat"><div class="num">{{FILES}}</div><div class="label">Files changed</div></div>
    <div class="summary-stat"><div class="num" style="color:#22c55e">+{{ADDITIONS}}</div><div class="label">Additions</div></div>
    <div class="summary-stat"><div class="num" style="color:#ef4444">-{{DELETIONS}}</div><div class="label">Deletions</div></div>
  </div>
</div>

<!-- What changed (plain English) -->
<h2>What Changed</h2>
<div class="card">
  <ul>
    <li><strong>{{CHANGE_1_TITLE}}</strong> — {{CHANGE_1_DESCRIPTION}}</li>
    <!-- One bullet per logical change, not per file -->
  </ul>
</div>

<!-- Why (from PR description or commit messages) -->
<h2>Why</h2>
<div class="card"><p>{{MOTIVATION}}</p></div>

<!-- Files changed by category -->
<h2>Files Changed</h2>
<table>
  <thead><tr><th>File</th><th>Category</th><th>+/-</th></tr></thead>
  <tbody>
    <tr><td><code>{{FILE}}</code></td><td><span class="tag tag-done">feature</span></td><td>+{{A}}/-{{D}}</td></tr>
  </tbody>
</table>

<!-- Testing notes -->
<h2>How to Test</h2>
<div class="card">
  <ol><li>{{TEST_STEP}}</li></ol>
</div>

<!-- Risks / things to watch -->
<h2>Risks</h2>
<div class="card">
  <ul><li>{{RISK}}</li></ul>
</div>
```

### 4. Publish

```bash
uselink publish /tmp/uselink-pr-digest-<number>-<timestamp>.html \
  --title "PR #<number> -- <short title> -- <date>" \
  --format html
```

Return the URL to the user.

## Gotchas

- **NEVER set background or color on body, table, th, td, code, or pre.** uselink's viewer injects theme-adaptive CSS that handles light/dark mode automatically. Any hardcoded color in published HTML overrides the viewer's theme and forces a white background in dark mode. Only style layout (padding, margins, borders, border-radius, font-size, grid). Use `opacity` for muted text instead of `color: #888`.
- **Write for non-engineers.** The "What Changed" section should use plain English, not code jargon. "Added email verification to signup" not "Added EmailVerificationService injection to AuthManager."
- **Group by logical change, not by file.** A feature that touches 8 files is ONE bullet in "What Changed" with context, not 8 bullets listing filenames.
- **PR description might be empty.** Infer the "Why" from commit messages if the PR body is blank. If both are empty, ask the user.
- **Large PRs need summarization.** For PRs with 50+ changed files, show the top 10-15 most important files in the table and summarize the rest as "and N other files (config/test changes)."
- **HTML-escape all strings from git/GitHub.** PR titles and commit messages can contain `<`, `>`, `&`.
