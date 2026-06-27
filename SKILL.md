---
name: build
description: Initialize a new full-stack application following sequential build phases (FastAPI + React + Docker)
argument-hint: [project-name] [phase-number]
allowed-tools: Bash(*), Read, Write, Edit, Glob, Grep, Agent
---

# Build: $0

## Available Phases
!`ls -1 ${CLAUDE_SKILL_DIR}/phases/ | sed 's/\.md$//'`

## Available References
!`ls -1 ${CLAUDE_SKILL_DIR}/references/ | sed 's/\.md$//'`

## Instructions

You are initializing or continuing work on project **$0**.

### If a phase number is provided ($1):

Read `${CLAUDE_SKILL_DIR}/phases/$1-*.md` (glob for the matching phase file) and execute that phase step-by-step.

### If no phase number is provided:

1. Check the current state of the project directory to determine which phases have been completed
2. Suggest which phase to run next
3. Wait for confirmation before proceeding

### Phase Execution Rules

1. **Read the full phase doc** before starting any work
2. **Ask clarifying questions** if the phase requires decisions (e.g., deployment path in Phase 01)
3. **Execute step-by-step**, following the phase doc instructions exactly
4. **Check off each item** in the phase's checklist before moving to the next
5. **Do not skip ahead** to the next phase — stop and confirm when complete

### Reference Documents

If the project needs file storage, background workers, or Google auth, read the relevant reference doc:
- `${CLAUDE_SKILL_DIR}/references/file-storage.md` — for file upload/download
- `${CLAUDE_SKILL_DIR}/references/background-workers.md` — for async processing
- `${CLAUDE_SKILL_DIR}/references/google-auth.md` — for Google Sign-In and/or mandatory email verification (the exact Toolshed implementation)

Only pull these in when the project actually needs the capability.

### Phase Overview

See `${CLAUDE_SKILL_DIR}/README.md` for the full phase dependency map and descriptions.
