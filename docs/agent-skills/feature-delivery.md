# Feature Delivery Skill

Use this skill for any non-trivial feature, refactor, or bug fix that affects behavior, spans multiple files, or needs review coordination.

Do not use this skill for trivial typos, copy changes, or single-line documentation edits.

## Workflow

1. Read `AGENTS.md` first and stay within the current sprint scope.
2. Inspect existing patterns before proposing new structure.
3. For non-trivial work, write or update a spec in `docs/specs/`.
4. Prefer an existing skill, built-in command, or subagent before inventing a long custom prompt.
5. Write a failing test first whenever behavior changes or a bug is being fixed.
6. Implement the smallest diff that satisfies the spec and test.
7. Run relevant verification:
   - `pnpm typecheck`
   - `pnpm lint`
   - relevant tests only
8. Run an independent review pass and an independent security pass.
9. Update docs, skills, and the decision log if a durable pattern changed.

## Parallelism Rules

- Split work only into independent slices with disjoint write sets.
- Use a separate worktree or branch per parallel task.
- Do not allow parallel agents to edit the same files.
- The builder is not the final certifier. A separate review or QA lane must exist.

## Automation Rules

- Run the workflow interactively before scheduling or backgrounding it.
- Prefer hooks and tests for deterministic checks.
- Review the first few automated runs closely before trusting them.

## Tool Hints

- Claude: prefer `/agents`, `/batch`, `/simplify`, and `/loop` when they fit the task.
- Codex: prefer `/skills`, `$skill-creator`, `$skill-installer`, subagents, and automations when they fit the task.

If a matching repo or system skill exists for the task, invoke it rather than rewriting its instructions inline.
