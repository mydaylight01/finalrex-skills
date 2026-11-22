---
name: finalrex-summary
description: Write the Weekly and/or Monthly summary of the configured FinalRex board (e.g. Master Plan) - per-person "Weekly Report: <name>" / "Monthly Report: <name>" notes and the team summary in the Weekly / Monthly note - as tables in the rules note's formats, from the board's tasks and the Daily Report notes of the period. Use when the user says /finalrex-summary, "สรุป weekly", "สรุปรายสัปดาห์", "สรุปรายเดือน", "เขียน weekly report", "monthly report", "weekly summary", "summarize the week / month".
---

# FinalRex weekly / monthly summary

Build the Weekly and/or Monthly reports of the board from its tasks and the Daily Report notes of the period, then write them into the board note tree after one confirmation.

Settings: `~/.claude/finalrex-report.json` (shared with `finalrex-report`). Ask every question with AskUserQuestion, in Thai. Talk to the user in Thai, short. Quoted English strings below are meant to be shown to the user: translate them to Thai when you say them.

## Names, not ids

- In chat, AskUserQuestion questions and options, confirmations, dry-run summaries and reports, refer to workspaces, boards, views, tasks, notes, books, pages, atlases, nodes and people by name or title (plus the url a tool returns when linking). Never show raw ids / UUIDs unless the user asks for them.
- Before every `mcp__finalrex__*` call, write one short Thai line saying what it will do in names and counts, e.g. "Will read 20 tasks from the board Master Plan" or "Will create 3 pages in the book Team Handbook", so the user understands the permission prompt, which shows raw ids.
- Tool calls still pass ids / UUIDs exactly as before. If a name is not known yet, take it from an earlier `list_*` / `get_*` result (look it up first if cheap) instead of printing the id.

## 0. Connect FinalRex

Look for `mcp__finalrex__*` tools (ToolSearch "+finalrex" if deferred). If none exist, stop and tell the user, in Thai, to run `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`, then `/mcp` › finalrex › log in, and run the skill again. Auth error → tell them to sign in the same way. Never ask for tokens or passwords.

## 1. Settings

Read `~/.claude/finalrex-report.json`. Needed: `workspace`, `board`, `rootNote`, `displayName`.

- All present → use them, no question.
- Something missing → ask only for that, the same way `finalrex-report` does (workspace: `list_workspaces`; board: `list_boards`, "Master Plan" first; rules note: `list_notes` top-level notes, "Work Log Reports" first; name: `whoami` display name as default). Save it into the json, keeping the other keys.

## 2. Read the rules

`get_note` the rules note (`rootNote`). Its "FOR AI" section wins over this skill: note tree, table formats, sort rule, **timing** (when a Weekly / Monthly may be written), weeks that span two months, no-duplicate rules.

## 3. Ask what to summarise

Always ask, even if the user named it:

1. "Which summary?" — Weekly / Monthly / Both
2. The period, per type chosen:
   - Weekly: "This week (dd/mm - dd/mm)" (Monday–Sunday, default) / "Last week (dd/mm - dd/mm)"
   - Monthly: "This month (MM/YYYY)" (default) / "Last month (MM/YYYY)"

Check the rules note timing against today. If the chosen period may not be written yet (e.g. "Weekly can be updated from the Monday after the week ends"), say so and ask: "Summarise the previous period instead" / "Show the draft in chat only (do not write to notes)".

## 4. Gather

1. **Board.** `get_board` (Status / Project options, `doneOptionIds`). `list_members` (ids → names).
2. **Tasks.** `list_tasks` with `compact: true`, `limit: 200`, follow `nextCursor`. Filter facts checked live: select values are **option ids**; dates use `due_at` / `start_at` (`{"field":"due_at","op":"between","value":["<start>T00:00:00Z","<end>T23:59:59Z"]}`); `fields` takes `Status`, `Progress`, `Project`, `Remark`, `table.col.assignee`, `table.col.start_at`, `table.col.due_at`. Tasks have no updated / completed time, so:
   - **done in the period** = Status in `doneOptionIds` and `due_at` in the period (rules note: due = the day it reached 100%)
   - **worked on in the period** = named in a Daily Report of the period, or `start_at` / `due_at` in the period
   - still open at the end = Open / In Progress / Hold among those
3. **Daily notes.** `list_notes` with `workspace` (slug) **and** `boardId` (on an all-workspaces connection it fails without `workspace`). Find the Monthly › Weekly › Daily notes of the period and `get_note` every "Daily Report: <name>" under them (and "Standup: dd/mm/yyyy" if present). They say who did what on which day.
4. **People.** One report per person who worked in the period. Use the name that person already uses in that period's note titles (e.g. "Daily Report: Nok"); a name can differ from the member display name (`nok`, `creme`), so match by assignee + task title and ask if unsure. Someone with no earlier note → their member display name.

## 5. Build the tables

Formats from the rules note (if it changes them, follow it):

- **Weekly Report: <name>** / **Monthly Report: <name>**: `| Task | Project | Status | Progress | Remark |`, the person's tasks worked on in the period, Status / Progress / Remark as on the board now.
- **Team summary Weekly** (body of the `Weekly: dd/mm - dd/mm` note): `| Task | Project | Assignee | Status | Progress |`.
- **Team summary Monthly** (body of the `Monthly: MM/YYYY` note): `| Feature | Project | Status | Progress |`, top-level tasks only (one task = one deliverable; subtasks are steps of a task and are left out; never a group task named after a date range), no names.

Rules: task titles exactly as on the board; sort Project A→Z, then Progress high→low, non-% last (Hold / Close last within their Project); escape `|` as `\|`; Thai, short, no process noise. A week that spans two months counts only in the month its Weekly note sits under. Never count the same task twice in one table.

## 6. Find the notes, confirm once, write

Note tree of the board (see `finalrex-report`):

```
Monthly: MM/YYYY                ← team summary Monthly (body)
├─ Weekly: dd/mm - dd/mm        ← team summary Weekly (body)
│  ├─ Daily: dd/mm/yyyy › Daily Report: <name>
│  └─ Weekly Report: <name>
└─ Monthly Reports              ← group note, empty body
   └─ Monthly Report: <name>
```

"Monthly Report: <name>" notes always sit under the `Monthly Reports` group of their month, never directly under the Monthly. The rules note's FOR AI section wins if it says otherwise.

**Order of sibling notes** (every level; never leave a note wherever `create_note` put it):

1. Group notes (notes that hold notes: Monthly, Weekly, Daily, Monthly Reports) come first, by date ascending: Monthly by month, Weekly by start day, Daily by day. A group without a date (`Monthly Reports`) comes after the dated groups.
2. Other notes (Weekly Report, Monthly Report, Daily Report, Standup) come after all groups, A→Z by title.

Steps:

1. Match each target note by title **and** parent. Existing → update it; missing → create it (`create_note` with `boardId` for a Monthly, `parentId` for children). Never create a second note with the same title under the same parent.
   - A Monthly Report needs the month's `Monthly Reports` group: reuse it, or create it (empty body) before the report.
   - A "Monthly Report: <name>" found directly under the Monthly (old layout) → `move_note` it into the group; never re-create it.
2. **Confirm once.** Show a short list and ask "Write this as listed?":
   ```
   Create: Monthly: 11/2026 › Weekly: 09/11 - 15/11 › Weekly Report: Pac (5 rows)
   Create: Monthly: 11/2026 › Monthly Reports › Monthly Report: Pac (8 rows)
   Update: Weekly: 09/11 - 15/11 (team summary 18 rows), Weekly Report: Gas (+2 rows, 3 changed)
   ```
   Do not write before the yes. The user may ask to see a table first.
3. **Write.** For an existing note: `get_note` for the current `version` and body, merge (update the row of the same task in place, add new rows, keep the sort, keep rows of other tasks and any text outside the table), then `update_note` with the whole body. For a new note: `list_notes` again right before `create_note` and reuse a note that appeared meanwhile. `create_note` puts a note last, so place every new or moved note right away: `move_note` with `afterNoteId` = the sibling it follows by the order above (`version` from the create result or `get_note`).
4. **Verify.** `list_notes` once more for every parent you touched: if a parent now has two notes with the same title, merge the bodies into the older one and `delete_note` the newer one; if siblings are out of the order above, `move_note` them into place.

**Always:** never write work into the rules note. No secrets, credentials, tokens or customer personal data (names, emails, phones) in notes.

## 7. Tell the user

Short, in Thai:

```
Summary written (board <board name>)
- Weekly 09/11 - 15/11: team summary (updated), Weekly Report: Gas, Nok, Pac (1 created, 2 updated)
- Monthly 11/2026: …
Skipped: <what and why, e.g. Monthly not due yet per the rules>
```
