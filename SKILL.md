---
name: build
description: Build, review, and assess full-stack apps (FastAPI + React + Docker) against a set of conventions. Three modes via the first argument — new (phased build), review (check the branch's changes), assess (whole-repo conformance audit).
argument-hint: [new|review|assess] [project-name | scope]
allowed-tools: Bash(*), Read, Write, Edit, Glob, Grep, Agent
---

# Build: $0

This skill has **three modes**, chosen by the first argument (`$0`):

| `$0` | Mode | What it does | Writes code? |
|------|------|--------------|--------------|
| `new` | **New build** | Scaffold/continue a phased full-stack build | Yes |
| `review` | **Review changes** | Review the current git branch's changes against the build conventions | No (read-only) |
| `assess` | **Assess repo** | Audit the whole repo for conformance with the build conventions | No (read-only) |

If `$0` is empty or unrecognized: if the working directory is an empty/new project, assume **new**; if it's an existing repo, state the three modes in one line and ask which to run. Don't guess between `review` and `assess` — confirm.

The conventions that `review` and `assess` check against are the phase docs in this skill, distilled into `${CLAUDE_SKILL_DIR}/conventions.md` (the audit rubric). The phase docs themselves are canonical.

---

## Mode: `new` — phased build

Remaining args: `$1` = project name (optional), `$2` = phase number (optional).

### Available Phases
!`ls -1 ${CLAUDE_SKILL_DIR}/phases/ | sed 's/\.md$//'`

### Available References
!`ls -1 ${CLAUDE_SKILL_DIR}/references/ | sed 's/\.md$//'`

You are initializing or continuing work on a phased build.

**If a phase number is provided (`$2`):** Read `${CLAUDE_SKILL_DIR}/phases/$2-*.md` (glob for the matching phase file) and execute that phase step-by-step.

**If no phase number is provided:**
1. Check the current state of the project directory to determine which phases have been completed
2. Suggest which phase to run next
3. Wait for confirmation before proceeding

**Phase execution rules:**
1. **Read the full phase doc** before starting any work
2. **Make the relevant decisions** (see the README's "Decisions to Make" table — deployment path, auth strategy, workers, file storage, etc.) and **ask** when a phase needs one
3. **Execute step-by-step**, following the phase doc exactly
4. **Check off each item** in the phase's checklist before moving on
5. **Do not skip ahead** — stop and confirm when a phase is complete

**Reference docs** — pull in only when the app needs the capability:
- `${CLAUDE_SKILL_DIR}/references/file-storage.md` — file upload/download
- `${CLAUDE_SKILL_DIR}/references/background-workers.md` — async processing
- `${CLAUDE_SKILL_DIR}/references/google-auth.md` — Google sign-in and/or mandatory email verification

See `${CLAUDE_SKILL_DIR}/README.md` for the phase map and the decisions table.

---

## Mode: `review` — review the branch's changes

Review **only what changed on the current branch** against the conventions — use this after building, or after an agent has edited the code. **Read-only**: report findings, don't fix them.

Remaining arg: `$1` = scope override (optional). Default scope is the branch diff vs. its base.

1. **Read `${CLAUDE_SKILL_DIR}/conventions.md`** — the audit rubric.
2. **Establish the diff:**
   - Default (no `$1`): diff the branch against `main` — `git diff $(git merge-base HEAD main)...HEAD` (fall back to `master`, or to `git diff` of the working tree if there's no base).
   - `$1 = staged` → `git diff --cached`; `$1 = diff` → `git diff` (working tree); `$1 =` a ref/range → `git diff <that>`.
   - Capture the list of changed files and the changed lines.
3. **Map each changed file to the conventions that govern it** (per `conventions.md`'s sections). Skip capabilities the app doesn't use (one-line "N/A").
4. **Judge each change:** does it conform, introduce a violation, or leave a gap? Cite `file:line` evidence from the diff.
5. **Report** (see Severity & Output) with a verdict: are the changes conformant and safe to keep, conditionally OK (with a must-fix list), or should they be revised?

---

## Mode: `assess` — whole-repo conformance audit

Audit the **entire codebase** against the conventions — a point-in-time conformance check. **Read-only**: report findings, don't fix them.

Remaining arg: `$1` = subtree to scope to (optional; default is the repo root).

1. **Read `${CLAUDE_SKILL_DIR}/conventions.md`** — the audit rubric.
2. **Detect which capabilities are in use** (conventions.md §0) so optional sections are skipped cleanly rather than flagged.
3. **Check every applicable convention** against the code. Prefer Grep/Glob/Read. For a large repo, fan out with the Agent tool — one agent per area (project layout, backend, backend testing, frontend, frontend testing, docker/deploy, optional capabilities) — and synthesize.
4. **Cite concrete evidence** (`file:line`, offending pattern, or missing path) for every finding.
5. **Report** (see Severity & Output) with an overall conformance summary (counts by severity) and the top 3 things to fix.

---

## Severity & Output (`review` and `assess`)

Rate every finding:
- **Violation** — contradicts a stated convention (wrong structure, banned pattern, missing required piece). Must fix.
- **Gap** — a convention is unmet but plausibly intentional/incomplete (e.g. tests missing for a new route). Should address.
- **Note** — minor drift, naming nit, or a conformance-improving suggestion.
- **Conforms** — call out the areas that are correct, so the report is a complete picture, not just a problem list.

Group findings by area (matching `conventions.md`'s sections). For each:

```
[SEVERITY] <area> — <short statement>
  evidence: <file:line | pattern | missing path>
  expected: <the convention, one line>
  fix:      <concrete remediation, one line>
```

End with the mode's verdict (review: keep / conditionally-OK / revise · assess: conformance summary + top 3 fixes). Keep it concrete and skimmable; don't restate conventions the code already follows beyond brief "Conforms" callouts. This is review-only — never edit code in these modes.
