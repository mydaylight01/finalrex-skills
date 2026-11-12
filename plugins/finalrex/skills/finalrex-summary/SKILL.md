---
name: finalrex-summary
description: Write the Weekly and/or Monthly summary of the configured FinalRex board (e.g. Master Plan) - per-person "Weekly Report: <name>" / "Monthly Report: <name>" notes and the team summary in the Weekly / Monthly note - as tables in the rules note's formats, from the board's tasks and the Daily Report notes of the period. Use when the user says /finalrex-summary, "สรุป weekly", "สรุปรายสัปดาห์", "สรุปรายเดือน", "เขียน weekly report", "monthly report", "weekly summary", "summarize the week / month".
---

# FinalRex weekly / monthly summary

Build the Weekly and/or Monthly reports of the board from its tasks and the Daily Report notes of the period, then write them into the board note tree after one confirmation.

Settings: `~/.claude/finalrex-report.json` (shared with `finalrex-report`). Ask every question with AskUserQuestion, in Thai. Talk to the user in Thai, short.

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

1. "จะสรุปแบบไหน?" — Weekly / Monthly / ทั้งสองแบบ
2. The period, per type chosen:
   - Weekly: "สัปดาห์นี้ (dd/mm - dd/mm)" (Monday–Sunday, default) / "สัปดาห์ก่อน (dd/mm - dd/mm)"
   - Monthly: "เดือนนี้ (MM/YYYY)" (default) / "เดือนก่อน (MM/YYYY)"

Check the rules note timing against today. If the chosen period may not be written yet (e.g. "Weekly อัพเดตได้ตั้งแต่วันจันทร์หลังจบสัปดาห์"), say so and ask: "สรุปช่วงก่อนหน้าแทน" / "ดูร่างในแชทอย่างเดียว (ไม่เขียนลงโน้ต)".

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
- **Team summary Weekly** (body of the `Weekly: dd/mm - dd/mm` note): `| Task | Project | ผู้ทำ | Status | Progress |`.
- **Team summary Monthly** (body of the `Monthly: MM/YYYY` note): `| Feature | Project | Status | Progress |`, parent tasks (features) only, no names.

Rules: task titles exactly as on the board; sort Project A→Z, then Progress high→low, non-% last (Hold / Close last within their Project); escape `|` as `\|`; Thai, short, no process noise. A week that spans two months counts only in the month its Weekly note sits under. Never count the same task twice in one table.

## 6. Find the notes, confirm once, write

Note tree of the board (see `finalrex-report`):

```
Monthly: MM/YYYY                ← team summary Monthly (body)
├─ Monthly Report: <name>
└─ Weekly: dd/mm - dd/mm        ← team summary Weekly (body)
   ├─ Weekly Report: <name>
   └─ Daily: dd/mm/yyyy › Daily Report: <name>
```

1. Match each target note by title **and** parent. Existing → update it; missing → create it (`create_note` with `boardId` for a Monthly, `parentId` for children). Never create a second note with the same title under the same parent.
2. **Confirm once.** Show a short list and ask "เขียนตามนี้ไหม?":
   ```
   สร้าง: Monthly: 11/2026 › Weekly: 09/11 - 15/11 › Weekly Report: Pac (5 แถว)
   อัพเดต: Weekly: 09/11 - 15/11 (สรุปทีม 18 แถว), Weekly Report: Gas (+2 แถว แก้ 3)
   ```
   Do not write before the yes. The user may ask to see a table first.
3. **Write.** For an existing note: `get_note` for the current `version` and body, merge (update the row of the same task in place, add new rows, keep the sort, keep rows of other tasks and any text outside the table), then `update_note` with the whole body. For a new note: `list_notes` again right before `create_note` and reuse a note that appeared meanwhile.
4. **Verify.** `list_notes` once more: if a parent now has two notes with the same title, merge the bodies into the older one and `delete_note` the newer one.

**Always:** never write work into the rules note. No secrets, credentials, tokens or customer personal data (names, emails, phones) in notes.

## 7. Tell the user

Short, in Thai:

```
เขียนสรุปแล้ว (board <board name>)
- Weekly 09/11 - 15/11: สรุปทีม (อัพเดต), Weekly Report: Gas, Nok, Pac (สร้าง 1 อัพเดต 2)
- Monthly 11/2026: …
ข้าม: <what and why, e.g. Monthly ยังไม่ถึงเวลาตามกฎ>
```
