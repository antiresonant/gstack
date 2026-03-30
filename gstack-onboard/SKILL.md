---
name: gstack-onboard
version: 1.0.0
description: |
  Onboard any project for gstack. Scans tech stack, creates review checklist,
  scaffolds VERSION/CHANGELOG/TODOS, and adds gstack section to CLAUDE.md.
  Invoke naturally: "set up this project for gstack", "onboard", "prepare for gstack".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

# /gstack-onboard

Prepare any project to work with all gstack skills. This is an interactive flow —
scan first, ask questions, then generate everything the skills need.

---

## Step 1: Scan the project (read-only)

Gather context silently. Do NOT ask questions yet.

### 1a. Detect tech stack

Check for these files (read whichever exist):
```bash
ls package.json Gemfile requirements.txt pyproject.toml go.mod Cargo.toml pom.xml build.gradle composer.json mix.exs deno.json 2>/dev/null
```

From what you find, identify:
- **Language:** JavaScript/TypeScript, Ruby, Python, Go, Rust, Java, PHP, Elixir, etc.
- **Framework:** Next.js, React, Vue, Svelte, Rails, Django, Flask, FastAPI, Express, Hono, etc.
- **Has database:** Look for prisma/, migrations/, db/, schema.rb, alembic/, knex, drizzle, etc.
- **Has LLM/AI code:** Look for imports of openai, anthropic, langchain, or files matching *prompt*, *llm*, *ai_*, *agent*
- **Has web UI:** Look for pages/, app/, components/, templates/, views/, public/, static/

### 1b. Detect commands

```bash
# Test command
cat package.json 2>/dev/null | grep -A2 '"test"'
cat Makefile 2>/dev/null | grep -E '^test:'
ls pytest.ini setup.cfg pyproject.toml 2>/dev/null
ls bin/test* 2>/dev/null

# Dev server
cat package.json 2>/dev/null | grep -A2 '"dev"\|"start"'
cat Procfile 2>/dev/null
ls bin/dev 2>/dev/null
```

### 1c. Check existing gstack artifacts

```bash
ls CLAUDE.md VERSION CHANGELOG.md TODOS.md 2>/dev/null
ls .claude/skills/review/checklist.md 2>/dev/null
ls .context/retros/ 2>/dev/null
ls .gstack/ 2>/dev/null
git remote get-url origin 2>/dev/null
```

### 1d. Check gstack installation

```bash
# Browse binary
ls ~/.claude/skills/gstack/browse/dist/browse* 2>/dev/null || ls .claude/skills/gstack/browse/dist/browse* 2>/dev/null
# Skill symlinks
ls ~/.claude/skills/review/SKILL.md 2>/dev/null || ls .claude/skills/review/SKILL.md 2>/dev/null
```

Collect all findings into a mental model. Proceed to Step 2.

---

## Step 2: Ask the user

Use **one AskUserQuestion call** with up to 4 questions. Adapt questions based on what you detected — skip questions you already have answers to.

**Question 1 — App URL** (only if web UI detected):
- Header: "App URL"
- Question: "What URL should gstack use for QA testing?"
- Options: [detected dev server URL like "http://localhost:3000"], "Staging/preview URL", "No web UI"
- If no web UI detected, skip this question entirely.

**Question 2 — Test command** (only if not confidently detected):
- Header: "Tests"
- Question: "How do you run tests?"
- Options: [detected command], "No tests yet"
- If confidently detected (e.g., `scripts.test` exists), skip and use detected value.

**Question 3 — Authentication** (only if web UI detected):
- Header: "Auth"
- Question: "Does QA testing need authentication?"
- Options: "No auth needed", "Cookie import (/setup-browser-cookies)", "Login credentials"
- multiSelect: false

**Question 4 — Review focus** (always ask):
- Header: "Review focus"
- Question: "What should code reviews focus on?"
- Options: "SQL/database safety", "LLM/AI trust boundaries", "API & auth security", "Standard web app"
- multiSelect: true

---

## Step 3: Create `.claude/skills/review/checklist.md`

**This is the most critical step.** Without this file, `/review` and `/ship` both hard-stop with an error.

First, create the directory:
```bash
mkdir -p .claude/skills/review
```

Then generate `checklist.md` adapted to the project. Use this structure:

```markdown
# Pre-Landing Review Checklist

## Instructions

Review the `git diff origin/main` output for the issues listed below. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

**Two-pass review:**
- **Pass 1 (CRITICAL):** Run critical checks first. These can block `/ship`.
- **Pass 2 (INFORMATIONAL):** Run all remaining categories. These are included in the PR body but do not block.

**Output format:**

\```
Pre-Landing Review: N issues (X critical, Y informational)

**CRITICAL** (blocking /ship):
- [file:line] Problem description
  Fix: suggested fix

**Issues** (non-blocking):
- [file:line] Problem description
  Fix: suggested fix
\```

If no issues found: `Pre-Landing Review: No issues found.`

Be terse. For each issue: one line describing the problem, one line with the fix. No preamble, no summaries, no "looks good overall."

---

## Review Categories

### Pass 1 — CRITICAL

{GENERATE CRITICAL CHECKS BASED ON STACK}

### Pass 2 — INFORMATIONAL

{GENERATE INFORMATIONAL CHECKS BASED ON STACK}

---

## Suppressions — DO NOT flag these

- "X is redundant with Y" when the redundancy is harmless and aids readability
- "Add a comment explaining why this threshold/constant was chosen" — thresholds change during tuning, comments rot
- "This assertion could be tighter" when the assertion already covers the behavior
- Suggesting consistency-only changes that don't fix bugs
- ANYTHING already addressed in the diff you're reviewing — read the FULL diff before commenting
```

### Critical checks to include based on stack:

**If has database (SQL):**
- String interpolation in SQL — use parameterized queries / ORM methods
- TOCTOU races: check-then-set patterns that should be atomic
- N+1 queries: missing eager loading for associations used in loops
- Missing DB indices on columns used in WHERE/JOIN

**If has LLM/AI code:**
- LLM-generated values written to DB without format validation
- Structured tool output accepted without type/shape checks
- Prompt injection vectors: user input concatenated into prompts without sanitization
- Missing output length/format guards on LLM responses

**If has web UI:**
- XSS: user-controlled data rendered without escaping (html_safe, dangerouslySetInnerHTML, v-html, {!! !!})
- CSRF: state-changing endpoints without CSRF tokens
- Open redirects: redirect URLs from user input without allowlist validation

**If has API/auth:**
- Missing authentication checks on new endpoints
- Authorization bypass: checking ownership via user-supplied IDs without server-side validation
- Secrets/tokens in source code or logs

**Always include (all projects):**
- Race conditions: read-check-write without locking or unique constraints
- Dead code: variables assigned but never read
- Version mismatch between PR title and VERSION/CHANGELOG files

### Informational checks to include based on stack:

**If JavaScript/TypeScript:**
- `any` types that could be narrowed
- Missing error boundaries in React components
- Async operations without error handling (unhandled promise rejections)
- Bundle size: large imports that could be tree-shaken or lazy-loaded

**If Python:**
- Bare `except:` clauses that swallow all exceptions
- Mutable default arguments in function signatures
- Missing type hints on public API functions
- `os.system()` or `subprocess.call(shell=True)` — use `subprocess.run()` with explicit args

**If Ruby/Rails:**
- `update_column`/`update_columns` bypassing validations on constrained fields
- Inline `<style>` blocks in partials
- `.select{}` filtering on DB results that could be a `WHERE` clause

**If Go:**
- Unchecked errors (especially from `Close()`, `Write()`, `Exec()`)
- Goroutine leaks: goroutines without cancellation context
- Data races: shared state without mutex or channels

**If has LLM/AI code:**
- Prompt text listing tools/capabilities that don't match what's wired up
- Word/token limits stated in multiple places that could drift
- 0-indexed lists in prompts (LLMs reliably return 1-indexed)

**Always include (all projects):**
- Magic numbers: bare numeric literals used in multiple files
- Test gaps: negative-path tests that assert status but not side effects
- Comments/docstrings that describe old behavior after code changed

---

## Step 4: Scaffold missing files

Only create files that don't already exist. Check first.

### VERSION (if missing)
```bash
# Try to detect from package.json
VERSION=$(cat package.json 2>/dev/null | grep '"version"' | head -1 | sed 's/.*"\([0-9.]*\)".*/\1/')
```

If detected, convert to 4-digit format (append `.0` segments as needed: `1.2.3` → `1.2.3.0`).
If not detected, use `0.1.0.0`.

Write a single line to `VERSION`:
```
0.1.0.0
```

### CHANGELOG.md (if missing)

```markdown
# Changelog

## [0.1.0.0] — {TODAY'S DATE YYYY-MM-DD}

### Added
- Initial project setup
- gstack onboarding (review checklist, VERSION, CHANGELOG)
```

Use the version from VERSION file and today's date.

### TODOS.md (if missing)

```markdown
# TODOs

## Backlog

- [ ] (add items here)

## Deferred

Items explicitly deferred — revisit periodically.
```

### Directories

```bash
mkdir -p .context/retros
mkdir -p .gstack
```

---

## Step 5: Update CLAUDE.md

### If CLAUDE.md exists:

Read it first. Check if it already has a gstack section (search for "gstack" or "/browse" or "/review").
- If gstack section exists: update it with the content below.
- If no gstack section: append the section below.

### If CLAUDE.md does not exist:

Create it with a project header + gstack section.

### gstack section content:

Generate this section, filling in project-specific values from Steps 1-2:

```markdown
## gstack

This project uses [gstack](https://github.com/garrytan/gstack) workflow skills.

### Available skills

- `/plan-ceo-review` — Founder-mode plan review: challenge premises, find the 10-star product
- `/plan-eng-review` — Eng manager plan review: architecture, edge cases, test coverage
- `/review` — Pre-landing code review: find bugs that pass CI but fail in prod
- `/ship` — Automated ship: merge main, test, review, bump version, push, create PR
- `/browse` — Headless browser: navigate, click, fill, screenshot, verify deployments
- `/qa` — Systematic QA: diff-aware testing, full exploration, smoke test, regression
- `/retro` — Weekly retrospective: commit analysis, work patterns, team feedback
- `/setup-browser-cookies` — Import real browser cookies for authenticated QA testing
- `/gstack-onboard` — Re-run project onboarding (updates checklist, scaffolding)

### Browser

Use `/browse` for all web interactions. **Never use `mcp__claude-in-chrome__*` tools.**

{IF APP URL WAS PROVIDED}
App URL for QA: {URL}
{END IF}

{IF AUTH METHOD WAS SPECIFIED}
Authentication: {method — e.g., "Run /setup-browser-cookies first" or "Login with test credentials"}
{END IF}

### Testing

{IF TEST COMMAND KNOWN}
Run tests: `{test command}`
{END IF}

{IF DEV SERVER KNOWN}
Dev server: `{dev command}`
{END IF}

### Setup

If gstack skills aren't working:
\```bash
cd ~/.claude/skills/gstack && ./setup
\```
```

Adapt the section — omit sub-sections that don't apply (e.g., no "Browser" section if no web UI).

---

## Step 6: Verify & summarize

### Verify gstack installation

```bash
# Check browse binary
ls ~/.claude/skills/gstack/browse/dist/browse* 2>/dev/null && echo "BROWSE_OK" || echo "BROWSE_MISSING"

# Check skill registration
ls ~/.claude/skills/review/SKILL.md 2>/dev/null && echo "SKILLS_OK" || echo "SKILLS_MISSING"
```

If `BROWSE_MISSING`: warn "Browse binary not found. Run: `cd ~/.claude/skills/gstack && ./setup`"
If `SKILLS_MISSING`: warn "Skill symlinks not found. Run: `cd ~/.claude/skills/gstack && ./setup`"

### Print summary

```
gstack onboarding complete!

Created:
{list each file/directory that was created, with checkmarks}
✓ .claude/skills/review/checklist.md — review checklist ({N} checks)
✓ VERSION — {version}
✓ CHANGELOG.md
✓ TODOS.md
✓ .context/retros/ — retro history directory
✓ .gstack/ — QA reports directory
✓ CLAUDE.md — gstack section added

Skipped (already existed):
{list any files that already existed}

{IF ANY WARNINGS}
Warnings:
⚠ {warning messages}
{END IF}
```

### Suggest next steps

Based on project state, suggest ONE next action:
- If on a feature branch with changes → "Try `/review` to review your changes before landing."
- If app URL was provided → "Try `/qa {url}` to run a QA sweep."
- If starting fresh → "Try `/plan-ceo-review` to plan your next feature."
- Default → "You're all set. Use `/ship` when you're ready to land a PR."
