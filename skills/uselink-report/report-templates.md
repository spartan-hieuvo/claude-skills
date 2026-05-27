# Report Templates

HTML templates for each report type. All styles are inline or in a single `<style>` block inside `<head>`.

---

## Data Gathering Commands

Run these before generating the report. Use the output to populate template placeholders.

### Code Review

```bash
# Scope the diff — always use the base branch range so multi-commit PRs are fully covered
git diff --stat origin/main...HEAD
git diff --name-only origin/main...HEAD

# Full diff for analysis (read selectively for large diffs)
git diff origin/main...HEAD -- <file>

# Branch info
git rev-parse --abbrev-ref HEAD
git log -1 --format="%H %s"
```

Then read each changed file and analyze for findings. Classify each finding as:
- `bug` -- broken behavior, crash, data loss, security flaw
- `risk` -- edge case, race, perf cliff, missing guard
- `nit` -- style or naming (include sparingly)

### Architecture Overview

```bash
# Directory structure
find . -type f -name "*.kt" -not -path "*/build/*" | head -100
find . -type f -name "*.tsx" -not -path "*/node_modules/*" -not -path "*/build/*" | head -100

# Key config files
cat settings.gradle.kts 2>/dev/null || cat settings.gradle 2>/dev/null
cat build.gradle.kts 2>/dev/null || cat build.gradle 2>/dev/null

# Git remote for project name
git remote get-url origin 2>/dev/null

# Tech stack markers
grep -r "micronaut" build.gradle.kts --include="*.kts" -l 2>/dev/null
grep -r "react" package.json 2>/dev/null
```

Then read `CLAUDE.md`, key `application.yml` files, and top-level build files.

### Sprint / Daily Summary

```bash
# Commits in range (default 7 days for sprint, 1 day for daily)
git log --since="7 days ago" --format="%H|%an|%s|%ai" --no-merges

# Group by author
git shortlog --since="7 days ago" --no-merges -s -n

# PRs merged (requires gh CLI)
gh pr list --state merged --search "merged:>=$(date -v-7d +%Y-%m-%d)" --limit 50 --json number,title,author,mergedAt 2>/dev/null

# PRs still open
gh pr list --state open --json number,title,author,createdAt 2>/dev/null

# Branch info
git rev-parse --abbrev-ref HEAD
```

### Custom

No fixed commands. Gather data based on what the user describes. Use `git log`, `grep`, file reads, and directory scans as needed.

---

## Base HTML Shell

Every report starts with this wrapper. Replace `{{TITLE}}`, `{{DATE}}`, `{{PROJECT}}`, and `{{BODY}}`.

**HTML-escape all interpolated data.** Commit messages, PR titles, file paths, and findings can contain `<`, `>`, `&`, or quotes that corrupt the HTML or inject markup. Before inserting any repo-sourced string into a template, escape `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`.

**CRITICAL: Do NOT set `background` or `color` on `body`, `table`, `th`, `code`, or `pre`.** uselink's viewer injects theme-adaptive CSS that handles light/dark mode. Any hardcoded color in the published HTML overrides the viewer's theme and breaks dark mode. Only style layout (padding, margins, borders, border-radius, font-size, grid).

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{TITLE}}</title>
<style>
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
    line-height: 1.6;
    padding: 2rem 1rem;
    max-width: 860px;
    margin: 0 auto;
  }
  .report-header {
    border-bottom: 2px solid currentColor;
    border-bottom-color: rgba(128,128,128,0.2);
    padding-bottom: 1rem;
    margin-bottom: 2rem;
  }
  .report-header h1 { font-size: 1.5rem; font-weight: 700; margin-bottom: 0.25rem; }
  .report-header .meta { font-size: 0.85rem; opacity: 0.6; }
  h2 { font-size: 1.15rem; font-weight: 600; margin: 1.75rem 0 0.75rem; }
  h3 { font-size: 1rem; font-weight: 600; margin: 1.25rem 0 0.5rem; }
  .card {
    border: 1px solid rgba(128,128,128,0.2);
    border-radius: 8px;
    padding: 1rem 1.25rem;
    margin-bottom: 1rem;
  }
  table { width: 100%; border-collapse: collapse; font-size: 0.9rem; margin: 0.75rem 0; }
  th, td { text-align: left; padding: 0.5rem 0.75rem; border: 1px solid rgba(128,128,128,0.2); }
  th { font-weight: 600; font-size: 0.8rem; text-transform: uppercase; letter-spacing: 0.03em; opacity: 0.8; }
  code {
    font-family: "SF Mono", "Fira Code", monospace;
    font-size: 0.85em;
    padding: 0.15em 0.4em;
    border-radius: 4px;
  }
  pre { overflow-x: auto; padding: 1rem; margin: 0.75rem 0; border-radius: 6px; }
  pre code { padding: 0; }
  .tag {
    display: inline-block;
    font-size: 0.75rem;
    font-weight: 600;
    padding: 0.15em 0.5em;
    border-radius: 4px;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }
  .tag-bug { background: #fde8e8; color: #b91c1c; }
  .tag-risk { background: #fef3c7; color: #92400e; }
  .tag-nit { background: #e0e7ff; color: #3730a3; }
  .tag-done { background: #d1fae5; color: #065f46; }
  .tag-wip { background: #fef3c7; color: #92400e; }
  .muted { font-size: 0.85rem; opacity: 0.6; }
  ul, ol { margin: 0.5rem 0 0.5rem 1.5rem; }
  li { margin-bottom: 0.25rem; }
  hr { border: none; border-top: 1px solid rgba(128,128,128,0.2); margin: 1.5rem 0; }
  .summary-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
    gap: 0.75rem;
    margin: 1rem 0;
  }
  .summary-stat {
    text-align: center;
    padding: 0.75rem;
  }
  .summary-stat .num { font-size: 1.75rem; font-weight: 700; }
  .summary-stat .label { font-size: 0.8rem; color: #666; margin-top: 0.15rem; }
  .finding { margin-bottom: 0.75rem; }
  .finding-loc { font-size: 0.8rem; color: #666; }
</style>
</head>
<body>

<div class="report-header">
  <h1>{{TITLE}}</h1>
  <div class="meta">{{PROJECT}} &middot; {{DATE}}</div>
</div>

{{BODY}}

</body>
</html>
```

---

## Template: Code Review

Replace `{{BODY}}` with this. Fill in actual data from `git diff` analysis.

```html
<div class="card">
  <div class="summary-grid">
    <div class="summary-stat">
      <div class="num">{{FILES_CHANGED}}</div>
      <div class="label">Files changed</div>
    </div>
    <div class="summary-stat">
      <div class="num">{{ADDITIONS}}</div>
      <div class="label">Additions</div>
    </div>
    <div class="summary-stat">
      <div class="num">{{DELETIONS}}</div>
      <div class="label">Deletions</div>
    </div>
    <div class="summary-stat">
      <div class="num">{{FINDINGS_COUNT}}</div>
      <div class="label">Findings</div>
    </div>
  </div>
</div>

<h2>Files Changed</h2>
<table>
  <thead><tr><th>File</th><th>+/-</th><th>Notes</th></tr></thead>
  <tbody>
    <!-- One row per changed file -->
    <tr><td><code>{{FILE_PATH}}</code></td><td>+{{ADD}} / -{{DEL}}</td><td>{{SHORT_NOTE}}</td></tr>
  </tbody>
</table>

<h2>Findings</h2>

<!-- Repeat this block per finding, grouped by severity: bug first, then risk, then nit -->
<div class="card finding">
  <span class="tag tag-bug">bug</span>
  <strong>{{FINDING_TITLE}}</strong>
  <div class="finding-loc"><code>{{FILE}}:{{LINE}}</code></div>
  <p>{{DESCRIPTION}}</p>
</div>

<!-- When no findings of a severity exist, omit that group entirely. -->

<h2>Recommendations</h2>
<ul>
  <li>{{RECOMMENDATION_1}}</li>
  <li>{{RECOMMENDATION_2}}</li>
</ul>
```

---

## Template: Architecture Overview

Replace `{{BODY}}` with this. Populate from directory scans and config file reads.

```html
<h2>Tech Stack</h2>
<table>
  <thead><tr><th>Layer</th><th>Technology</th><th>Version</th></tr></thead>
  <tbody>
    <tr><td>Language</td><td>{{LANG}}</td><td>{{LANG_VER}}</td></tr>
    <tr><td>Framework</td><td>{{FRAMEWORK}}</td><td>{{FW_VER}}</td></tr>
    <tr><td>Database</td><td>{{DB}}</td><td>{{DB_VER}}</td></tr>
    <tr><td>Frontend</td><td>{{FE}}</td><td>{{FE_VER}}</td></tr>
    <!-- Add rows as needed -->
  </tbody>
</table>

<h2>System Architecture</h2>
<div class="card">
  <pre><code>{{ASCII_DIAGRAM}}</code></pre>
</div>
<p class="muted">{{DIAGRAM_CAPTION}}</p>

<h2>Directory Structure</h2>
<div class="card">
  <pre><code>{{DIRECTORY_TREE}}</code></pre>
</div>

<h2>Key Components</h2>

<!-- Repeat per component/module -->
<h3>{{COMPONENT_NAME}}</h3>
<div class="card">
  <p><strong>Path:</strong> <code>{{COMPONENT_PATH}}</code></p>
  <p><strong>Purpose:</strong> {{PURPOSE}}</p>
  <p><strong>Key files:</strong></p>
  <ul>
    <li><code>{{KEY_FILE_1}}</code> -- {{FILE_1_DESC}}</li>
    <li><code>{{KEY_FILE_2}}</code> -- {{FILE_2_DESC}}</li>
  </ul>
</div>

<h2>Data Flow</h2>
<div class="card">
  <pre><code>{{DATA_FLOW_DIAGRAM}}</code></pre>
</div>
```

---

## Template: Sprint / Daily Summary

Replace `{{BODY}}` with this. Populate from `git log` and `gh pr list`.

```html
<div class="card">
  <div class="summary-grid">
    <div class="summary-stat">
      <div class="num">{{COMMITS}}</div>
      <div class="label">Commits</div>
    </div>
    <div class="summary-stat">
      <div class="num">{{PRS_MERGED}}</div>
      <div class="label">PRs merged</div>
    </div>
    <div class="summary-stat">
      <div class="num">{{PRS_OPEN}}</div>
      <div class="label">PRs open</div>
    </div>
    <div class="summary-stat">
      <div class="num">{{CONTRIBUTORS}}</div>
      <div class="label">Contributors</div>
    </div>
  </div>
</div>

<h2>Merged PRs</h2>
<table>
  <thead><tr><th>#</th><th>Title</th><th>Author</th><th>Merged</th></tr></thead>
  <tbody>
    <!-- One row per merged PR -->
    <tr><td>{{PR_NUM}}</td><td>{{PR_TITLE}}</td><td>{{PR_AUTHOR}}</td><td>{{MERGE_DATE}}</td></tr>
  </tbody>
</table>

<h2>Commits by Author</h2>

<!-- Repeat per author -->
<h3>{{AUTHOR_NAME}} <span class="muted">({{AUTHOR_COMMIT_COUNT}} commits)</span></h3>
<ul>
  <!-- One li per commit -->
  <li><code>{{SHORT_SHA}}</code> {{COMMIT_MSG}} <span class="muted">{{COMMIT_DATE}}</span></li>
</ul>

<h2>In Progress</h2>
<table>
  <thead><tr><th>#</th><th>Title</th><th>Author</th><th>Status</th></tr></thead>
  <tbody>
    <!-- One row per open PR -->
    <tr>
      <td>{{PR_NUM}}</td>
      <td>{{PR_TITLE}}</td>
      <td>{{PR_AUTHOR}}</td>
      <td><span class="tag tag-wip">WIP</span></td>
    </tr>
  </tbody>
</table>
```

---

## Template: Custom Report

For custom reports, use the base shell with a flexible body. Adapt structure to the user's request.

Common building blocks to mix and match:

```html
<!-- Section with prose -->
<h2>{{SECTION_TITLE}}</h2>
<div class="card">
  <p>{{CONTENT}}</p>
</div>

<!-- Key-value pairs -->
<h2>{{SECTION_TITLE}}</h2>
<table>
  <thead><tr><th>Item</th><th>Value</th></tr></thead>
  <tbody>
    <tr><td>{{KEY}}</td><td>{{VALUE}}</td></tr>
  </tbody>
</table>

<!-- Numbered list -->
<h2>{{SECTION_TITLE}}</h2>
<div class="card">
  <ol>
    <li><strong>{{ITEM}}</strong> -- {{DETAIL}}</li>
  </ol>
</div>

<!-- Code block -->
<h2>{{SECTION_TITLE}}</h2>
<div class="card">
  <pre><code>{{CODE}}</code></pre>
</div>

<!-- Status items with tags -->
<h2>{{SECTION_TITLE}}</h2>
<div class="card">
  <p><span class="tag tag-done">done</span> {{ITEM_DONE}}</p>
  <p><span class="tag tag-wip">wip</span> {{ITEM_WIP}}</p>
  <p><span class="tag tag-risk">blocked</span> {{ITEM_BLOCKED}}</p>
</div>
```

---

## Publish Command

After writing the HTML to a temp file:

```bash
uselink publish /tmp/uselink-report-<type>-<timestamp>.html \
  --title "<Type> Report -- <Project> -- <Date>" \
  --format html
```

The CLI prints the public URL to stdout. Return that URL to the user.
