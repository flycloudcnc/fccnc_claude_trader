---
name: plan
description: Translate `doc/prd.md` into a phased GitHub Issues backlog. Use when the user runs `/plan`, asks to break down the PRD into issues, or wants to (re)generate the project's planning backlog.
---

# /plan — PRD → GitHub Issues

Turn `doc/prd.md` into a tracked, phased backlog of GitHub issues. Run from the repo root.

## Inputs

- **Required:** `doc/prd.md` — the source of truth for scope, requirements, and acceptance criteria.
- **Optional:** `doc/design-spec-v2.md` (or any design doc referenced from the PRD) for additional architectural context.
- **Existing state:** current open issues from `gh issue list --state open --limit 200`.

## Procedure

1. **Read the PRD.** Read `doc/prd.md` end-to-end. Identify phases (Phase 1, 2, 3, …) and the categories used inside them (e.g. data, model, ui, infra, testing).

2. **Inventory existing issues.** Run `gh issue list --state all --limit 200 --json number,title,labels,state` so you do not duplicate tickets that already exist. Note which PRD items are already covered.

3. **Propose the backlog.** Produce a short table for the user:
   - Columns: `Phase | Category | Title | Acceptance criteria (1 line) | Status (NEW / EXISTS #N / SKIP)`
   - Group by phase. Keep each task small enough to finish in one focused conversation; split tasks that bundle unrelated changes.
   - **Stop and confirm with the user before creating any issues.**

4. **Ensure labels exist.** For every phase/category label used, create it if missing:
   ```
   gh label create "phase-1" --color BFD4F2 --description "PRD Phase 1" 2>/dev/null || true
   gh label create "category-data" --color C5DEF5 --description "Data" 2>/dev/null || true
   ```
   Use a consistent prefix: `phase-N` and `category-<name>`.

5. **Create issues.** For each NEW row, run:
   ```
   gh issue create \
     --title "<concise title>" \
     --label "phase-N,category-<name>" \
     --body "$(cat <<'EOF'
   ## Source
   PRD section: <heading or anchor in doc/prd.md>

   ## Description
   <what & why, copied/distilled from the PRD>

   ## Acceptance criteria
   - [ ] <criterion 1>
   - [ ] <criterion 2>

   ## Notes
   <links to related issues, design-spec sections, dependencies>
   EOF
   )"
   ```

6. **Report.** When done, print:
   - Count of issues created, grouped by phase.
   - The `gh issue list --label phase-1` URL/output so the user can verify.
   - Any PRD items intentionally skipped and why.

## Rules

- **Never create issues without confirmation in step 3.**
- One issue = one acceptance-testable outcome. Avoid mega-tickets.
- Every issue references the PRD section it came from.
- Do not modify `CLAUDE.md` — it is human-edited only.
- Do not write code in this workflow. Planning only.
