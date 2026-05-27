---
name: uselink-changelog
description: "Generate a changelog or release notes from git history and publish to uselink. Use when the user wants to share release notes, a changelog between versions, or a 'what shipped this week' summary with stakeholders."
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
---

# Changelog → Uselink

Generate formatted release notes from git history and publish as a shareable uselink page.

## When to Use

- "Generate release notes for this version"
- "What shipped this week? Share it."
- "Create a changelog from v1.2 to v1.3"
- "Share what we deployed today"

## Prerequisites

```bash
which uselink || echo "MISSING"
test -f ~/.uselink/config.json && echo "OK" || echo "NO_CONFIG"
```

## Workflow

### 1. Determine range

**Between tags:**
```bash
git tag --sort=-version:refname | head -5
git log <old-tag>..<new-tag> --oneline --no-merges
```

**Time-based (last week, last sprint):**
```bash
git log --since="7 days ago" --oneline --no-merges
```

**Since last deploy (if deploy tags exist):**
```bash
git log $(git describe --tags --abbrev=0)..HEAD --oneline --no-merges
```

If no range specified, default to the last 7 days.

### 2. Gather data

```bash
# All commits in range
git log <range> --format="%H|%an|%s|%ai" --no-merges

# PRs merged in range (if gh is available)
gh pr list --state merged --search "merged:>=<start-date>" --limit 100 \
  --json number,title,author,labels,mergedAt 2>/dev/null

# Group by conventional commit type
git log <range> --oneline --no-merges | grep -c "^[a-f0-9]* feat"
git log <range> --oneline --no-merges | grep -c "^[a-f0-9]* fix"
```

### 3. Categorize changes

Group into sections by commit prefix or PR labels:

| Category | Commit prefix / label | Icon |
|----------|----------------------|------|
| New Features | `feat`, `feature` | ✨ |
| Bug Fixes | `fix`, `bugfix` | 🐛 |
| Improvements | `refactor`, `perf`, `improve` | ⚡ |
| Documentation | `docs` | 📝 |
| Infrastructure | `ci`, `build`, `chore`, `deps` | 🔧 |
| Breaking Changes | `BREAKING` in body, `!` suffix | ⚠️ |

If commits don't follow conventional format, categorize by reading the diff.

### 4. Generate HTML

Write to `/tmp/uselink-changelog-<version>-<timestamp>.html`.

```html
<!-- Version header -->
<div class="card">
  <h3>{{VERSION_OR_TITLE}}</h3>
  <p class="muted">{{DATE_RANGE}} · {{COMMIT_COUNT}} commits · {{CONTRIBUTOR_COUNT}} contributors</p>
</div>

<!-- Highlights (top 3-5 most important changes) -->
<h2>Highlights</h2>
<div class="card">
  <ul>
    <li><strong>{{HIGHLIGHT}}</strong> — {{ONE_SENTENCE_DESCRIPTION}}</li>
  </ul>
</div>

<!-- New Features -->
<h2>✨ New Features</h2>
<div class="card">
  <ul><li>{{FEATURE_DESCRIPTION}} <span class="muted">(#{{PR_NUM}})</span></li></ul>
</div>

<!-- Bug Fixes -->
<h2>🐛 Bug Fixes</h2>
<div class="card">
  <ul><li>{{FIX_DESCRIPTION}} <span class="muted">(#{{PR_NUM}})</span></li></ul>
</div>

<!-- Improvements -->
<h2>⚡ Improvements</h2>
<div class="card">
  <ul><li>{{IMPROVEMENT}}</li></ul>
</div>

<!-- Breaking Changes (if any) -->
<h2>⚠️ Breaking Changes</h2>
<div class="card">
  <ul><li><strong>{{BREAKING}}</strong> — {{MIGRATION_NOTES}}</li></ul>
</div>

<!-- Contributors -->
<h2>Contributors</h2>
<p>{{CONTRIBUTOR_LIST}}</p>
```

### 5. Publish

```bash
uselink publish /tmp/uselink-changelog-<version>-<timestamp>.html \
  --title "Release Notes -- <version or date range>" \
  --format html
```

Return the URL to the user.

## Gotchas

- **NEVER set background or color on body, table, th, td, code, or pre.** uselink's viewer injects theme-adaptive CSS that handles light/dark mode automatically. Any hardcoded color in published HTML overrides the viewer's theme and forces a white background in dark mode. Only style layout (padding, margins, borders, border-radius, font-size, grid). Use `opacity` for muted text instead of `color: #888`.
- **Write for the audience, not the git log.** "Added real-time notifications for comment replies" is better than "feat(notifications): add WebSocket push for DocumentCommentReply events." Translate commit-speak to user-speak.
- **Highlights section is mandatory.** Even for small releases, pick the top 3 changes. Stakeholders read this first and may skip the rest.
- **Omit noise commits.** Skip "fix typo", "lint", "merge branch" commits from the changelog. They're git housekeeping, not release content.
- **If no conventional commits, categorize by diff.** Read the actual changes to determine if something is a feature, fix, or refactor. Don't just dump uncategorized commits.
- **HTML-escape all commit messages and PR titles.** They can contain `<`, `>`, `&`.
