# docs/specs/agent-governance.md

## Overview

Establish a single-source-of-truth agent workflow for this repository, reduce drift between `AGENTS.md` and `CLAUDE.md`, and promote reusable repo skills for Claude and Codex.

## User Stories

- As a maintainer, I want one canonical instruction file so that agent behavior does not drift across tools.
- As a Claude or Codex user, I want project skills to encode repeatable workflows so that features are delivered more consistently.
- As a reviewer, I want explicit automation and review gates so that agents cannot self-certify unsafe or incomplete work.

## Data Model Changes

None.

## API Procedures

| Procedure | Input (Zod) | Output | Auth |
|-----------|-------------|--------|------|
| None | None | None | None |

## UI Components

None.

## Edge Cases

- What happens if Claude and Codex need the same skill content? The canonical instructions live in `docs/agent-skills/` and thin wrappers exist for each tool.
- What if a tool has session-local skills unavailable in another tool? Repo guidance should prefer them when available but never depend exclusively on them.
- What if automation is added later? The workflow requires an interactive dry run before any recurring or unattended automation.

## Acceptance Criteria

- [ ] `AGENTS.md` contains the canonical shared workflow and guardrails.
- [ ] `CLAUDE.md` imports `AGENTS.md` and remains tool-specific and thin.
- [ ] Repo skills exist for both Claude and Codex and point to a shared canonical instruction body.
- [ ] Local-only Claude settings are ignored by Git.
- [ ] The decision log records the new governance pattern.

## Test Plan

- Unit: N/A
- Integration: Verify Claude and Codex skill wrapper paths resolve to canonical docs.
- E2E: Manually inspect `git status` and repo files to confirm the new layout is coherent and local settings stay ignored.
