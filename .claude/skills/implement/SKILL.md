---
name: implement
description: Implement the next open GitHub issue end-to-end (code, tests, activity-log, commit, push, close). Use when the user runs `/implement`, asks to work on the next issue, or says to start coding the planned backlog.
---

# /implement — Work one issue end-to-end

Pick the next open issue from the backlog and take it from code to closed. One issue per invocation. Always run inside `.venv`.

## Pre-flight

1. **Activate the venv.** All commands must run inside `.venv`. If it is not active, source it before running tests.
2. **Confirm clean tree.** `git status` should be clean (or only contain expected work-in-progress). If not, stop and ask the user.

## Pick the issue

1. List open issues ordered by phase, lowest first:
   ```
   gh issue list --state open --label "phase-1" --json number,title,labels --limit 50
   ```
   Walk phases ascending — only move to `phase-2` once `phase-1` is empty.
2. If multiple issues exist in the current phase, pick the smallest/most foundational and **state your choice to the user before starting.** Allow them to redirect.
3. Read the issue body fully:
   ```
   gh issue view <number>
   ```
   Note its acceptance criteria — these are the definition of done.

## Implement

1. **Start fresh context.** If the conversation already has unrelated state, suggest the user start a new session before continuing.
2. Refer to `doc/prd.md` for product intent and `doc/design-spec-v2.md` (if present) for architecture.
3. Write the code and the tests together. Tests must cover each acceptance criterion.
4. Run the full test suite inside `.venv`. Do not proceed until it is green.
5. If you discover work that is out of scope, **create a new GitHub issue** rather than expanding this one.

## Wrap up

Run these in order — do not skip steps:

1. **Activity log.** Prepend a new entry to the top of `activity-log/` (newest first) with:
   - `YYYY/MM/DD-HHMM` timestamp
   - User's prompt (the `/implement` invocation + any clarifications)
   - Summary of what was done (no full code dumps) + final todo status
   - Issue number(s) closed

2. **Commit.** Reference the issue number:
   ```
   git add <specific files>
   git commit -m "Fix #<N>: <concise summary>"
   ```
   Stage specific files — avoid `git add -A`. Group related issues into one commit only when they truly belong together; otherwise one issue per commit.

3. **Push.**
   ```
   git push
   ```

4. **Close the issue.**
   ```
   gh issue close <N> --comment "Implemented in <commit-sha>. All acceptance criteria met."
   ```

5. **Stop.** After the push + close, end the session and tell the user to start a fresh conversation for the next issue.

## Rules

- One issue per invocation. Do not chain multiple issues in one run.
- No code changes without a referenced issue. If there is no issue, run `/plan` first or create one manually.
- Every commit message includes `#<issue-number>`.
- Never modify `CLAUDE.md` — human-edited only.
- Never use `--no-verify` or skip hooks. Fix the underlying problem.
- If tests fail and the fix is non-trivial, stop and report rather than spiraling.
