# fccnc-claude-trader Development Guidelines

- CLAUDE.md should be updated manually only by a human developer!!!!

- Always run in `.venv`.

- When the conversation becomes long (many tool calls, large code outputs), proactively commit and push changes, then suggest the user start a fresh session.

- **Planning & Implementing**: Use the project skills.
   - `/plan` — translate `doc/prd.md` into GitHub issues with phase/category labels. See `.claude/skills/plan/SKILL.md`.
   - `/implement` — pick the next open issue (lowest phase first), code + test in `.venv`, update activity-log, commit with `Fix #<N>:`, push, close the issue, then stop. See `.claude/skills/implement/SKILL.md`.
   - No code changes without a referenced issue. For overall design, refer to `docs/design-spec-v2.md`.

- When doing DDD modeling, use the templates in `docs/ddd/templates/`.

- **Logging, commit and push to remote**: Always add conversation log to the top of activity-log with format:
   - datetime with format YYYY/MM/DD-HHMM.
   - User's prompt
   - Claude output summary (no detailed output) + todo list status.
   - commit and push the changes to remote.
   - after each commit/push, please stop.