# Hospital ERP Manager

@AGENTS.md

## Claude-Specific Notes

- Keep this file thin. Put shared rules in `AGENTS.md`.
- Prefer project skills in `.claude/skills/` over long ad-hoc prompts.
- Keep machine-local settings in `.claude/settings.local.json`; do not commit them.
- If you add Claude-specific automation, prefer project hooks or subagents over copying workflow text into this file.
