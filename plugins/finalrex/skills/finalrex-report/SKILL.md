---
name: finalrex-report
description: Log the work of this session as tasks on the configured FinalRex board (e.g. Master Plan) and write the Daily Report note on that board (Monthly › Weekly › Daily › "Daily Report: <name>"). Use when the user says /finalrex-report, "เขียน daily", "ลง work log", "daily report", "สรุปงานวันนี้ลง finalrex", or asks to report the work done in this session. First run asks for workspace, board, rules note, format and name, then remembers them.
---

# FinalRex daily report

Log what was done in **this session** (and anything the user names) as tasks on the board, then write the Daily Report note of that board.

Settings are saved in `~/.claude/finalrex-report.json` (outside the skill folder, so reinstalling the skill keeps them):

```json
{
  "workspace": { "id": "...", "slug": "...", "name": "..." },
  "board": { "id": "...", "name": "Master Plan" },
  "rootNote": { "id": "...", "title": "Work Log Reports" },
  "displayName": "<your name>",
  "userId": "...",
  "format": { "mode": "default", "custom": null }
}
```

`format.mode` is `default` (the rules note's formats) or `custom` (`format.custom` holds the user's bullet points). `userId` is the user's member id, filled in section 3.

Ask every question with the AskUserQuestion tool, in Thai. It adds an "Other" box on its own, so the user can always type a free answer.

## 0. Connect FinalRex (only if it is missing)

Look for tools named `mcp__finalrex__*` (search with ToolSearch for "+finalrex" if they are deferred; wait for it if the server is still connecting). If none exist, stop and tell the user, in Thai, to connect it, then run the skill again:

```bash
claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp
```

Then sign in: in a terminal `claude` session run `/mcp`, choose **finalrex**, and log in in the browser. In the Claude desktop app, open the connector settings and sign in to finalrex there.

If the tools exist but calls fail with an auth error, tell the user to sign in the same way. Never ask for tokens or passwords in chat.

Once connected, call `whoami`. Its display name is the default for step 5 (the name in the report title).

## 1. Settings: first run or "same as before?"

Read `~/.claude/finalrex-report.json`.

**File exists and is complete** → ask ONE question only:
- "ทำแบบเดิม" — description: `<workspace name> › board <board name> › <root note title> › format <ที่มีอยู่ | กำหนดเอง> › ชื่อ <displayName>`
- "ตั้งค่าใหม่"

"ทำแบบเดิม" → go to section 2. "ตั้งค่าใหม่" → run steps 1-5 below, then save.

**File complete except `board`** → run step 2 only, save, continue (no "same as before?" question this run).

**File missing or broken** (including no `displayName`) → run steps 1-5, then save.

### Step 1 — Workspace
Call `list_workspaces`. Ask "จะเขียนลง workspace ไหน?" with up to 4 workspaces as options (label = name). With only one workspace, still ask, so the user confirms it.

### Step 2 — Board
Call `list_boards` for that workspace. Ask "งานเก็บอยู่ใน board ไหน?" with up to 4 boards (put one named "Master Plan" first). Save `board: { id, name }`.

### Step 3 — Rules note
Call `list_notes` for the workspace and for the board (`boardId`). Candidates = notes with a title and no `parentId` (workspace notes and board notes). Ask "โน้ตกลางที่เก็บกฎ / Standard คือโน้ตไหน?":
- up to 3 candidates (put one named like "Work Log Reports" first)
- "สร้างโน้ตใหม่"

"สร้างโน้ตใหม่" (or a typed name in Other that matches no note) → if no name was typed, ask for the name in chat, default `Work Log Reports`. Create it with `create_note` (workspace, title, empty body; step 4 fills the body). Save it as `rootNote`.

### Step 4 — Format
Ask "จะใช้ format แบบไหน?":
- "ใช้แบบที่มีอยู่" — description: "ตามกฎในโน้ตกลาง (Daily Report: Task / Project / Status / Progress / สิ่งที่ทำวันนั้น)"
- "สร้างเองใหม่" — description: "พิมพ์ bullet point ว่าอยากให้รายงานมีอะไรบ้าง"

"สร้างเองใหม่" → ask in chat: "พิมพ์ bullet point ของ format ที่ต้องการ เช่น หัวข้อ, คอลัมน์, ภาษา, สิ่งที่ต้องมี/ห้ามมี". Save the bullets as-is in `format.custom`. Show a short sample report built from them and ask for OK before saving. The rules note still wins where they conflict.

If the rules note was just created (step 3), write the chosen format into its body as a "FOR AI" block, so the whole team shares one format. Do not overwrite the body of an existing rules note.

### Step 5 — Name
Ask "ชื่อที่จะใช้ใน Daily Report (ชื่อโน้ตจะเป็น `Daily Report: <ชื่อ>`)":
- the `whoami` display name — description: "ชื่อจากบัญชี FinalRex"
- the saved `displayName`, if there is one and it differs
- the user types another name in Other

Trim spaces. Use the name exactly as given (any language). Every member needs a different name, so their reports stay separate.

### Save
Write `~/.claude/finalrex-report.json` with the answers, keeping keys you did not ask about. Then continue.

## 2. Read the rules

`get_note` the rules note (`rootNote`) every run. Its "FOR AI" section is the source of truth: task rules, Status from %, fields, board-note structure, table formats, timing, no-duplicate rules. It wins over this skill and over the saved format. Follow it where this skill is silent or differs.

## 3. Update tasks

1. **Gather** the session's work: each feature and its pieces, with Project, latest %, what is left, and the real work dates. Date = the day the work was actually done, not the day the report is written. Find it from the session transcript timestamps (`~/.claude/projects/<project>/<session-id>.jsonl`, field `timestamp`, convert UTC to local time) and commit/deploy times. The user naming a day overrides this.
2. **Board and people.** `get_board` for fields and select options (Status, Project). If `userId` is missing in the settings, `list_members` once, match `displayName` (or the `whoami` email) to a member (display names may be Thai; ask if unsure), save `userId` in the json.
3. **Find existing tasks** for each piece: `list_tasks` (board, `compact: true`, `filter` on title `contains`, or `parentId`) and/or `search` (kinds `task`). Update an existing task; create only when none matches.
4. **Plan** creates/updates per the rules note: feature = parent task, pieces = subtasks (`parentId`, one level), title with no "(Project)" suffix, `assigneeIds: [userId]`, `startAt` / `dueAt` from real work days, fields `Project`, `Progress`, `Status` (from %), `Remark`. Project must be an existing option; if none fits, ask the user before anything else, never add an option yourself. Open risks / things left to do → their own task with Status Open (under the related feature) and the detail in Remark.
5. **Dry run.** `batch_tasks` with `dryRun: true`. Fix any error.
6. **Confirm once.** Show the user a short list (create / update: title, Status, Progress, parent) and ask "ลง task ตามนี้ไหม?". Do not write before the yes.
7. **Run** `batch_tasks` for real (create ops use `ref` so subtasks can point at a new parent). Then `list_tasks` again to verify; if the batch stopped partway, report which ops were applied and fix the rest.
8. **Standards.** A new reusable rule from this session → append it to the Standard section of the rules note, only if it is not already there (`get_note` for the current `version`, then `update_note` with the full body plus the new line). Never write work into the rules note.

## 4. Find or create the note

Board notes of the saved board (create the Monthly with `create_note` + `boardId`, every child with `parentId`, never duplicate one that exists):

```
<board>
└─ Monthly: MM/YYYY              e.g. Monthly: 11/2026
   └─ Weekly: dd/mm - dd/mm      Monday–Sunday, e.g. Weekly: 09/11 - 15/11
      └─ Daily: dd/mm/yyyy       e.g. Daily: 13/11/2026
         └─ Daily Report: <displayName>
```

- Use `list_notes` with `boardId` to see the tree; match by title and parent. Ignore untitled notes.
- If the work spans several days, split it: each piece goes into the Daily note of the day it was done.
- Several sessions may write at once. Call `list_notes` again right before every `create_note` and reuse a note that appeared meanwhile. After writing, list once more: if the same parent now has two notes with the same title, merge the bodies into the older one and `delete_note` the newer one.

## 5. Write the body

Daily Report = the table from the rules note:

```markdown
| Task | Project | Status | Progress | สิ่งที่ทำวันนั้น |
|---|---|---|---|---|
| <task title exactly as on the board> | <Project> | <Status> | <N>% | <what was done that day, short> |
```

- One row per task touched that day. Sort Project A→Z, then Progress high→low (Hold / Close last within their Project).
- If the note already has content, keep it and merge: update the row of the same task in place, add new rows, keep the sort. Never write the same thing twice. Work continuing from an earlier day → only that day's part.
- Thai, plain language, no process noise (commands run, files read, retries). Escape a literal `|` in a cell as `\|`.
- `update_note` needs the current `version` from `get_note` and replaces the whole body, so send the old body plus your changes. Do not delete other entries.
- **Custom format** (`format.mode = custom`): follow `format.custom` bullets where the rules note does not say otherwise; keep the merge and safety rules.

**Weekly / Monthly:** write "Weekly Report: <displayName>" / "Monthly Report: <displayName>" and the team summaries only when the rules note's timing allows it (judged by today's date), in its formats. Otherwise write only the Daily.

**Always:** no secrets, credentials, tokens, or customer personal data (names, emails, phones) in tasks or notes.

## 6. Tell the user

Short, in Thai. Shape:

```
ลง FinalRex แล้ว

Task (board <board name>)
- สร้าง: <title> — <Status> <N>%
- อัพเดต: <title> — <Status> <N>%

Note
- Monthly: 11/2026 › Weekly: 09/11 - 15/11 › Daily: 13/11/2026 › **Daily Report: <displayName>**

ข้าม: <anything skipped and why, e.g. Weekly ยังไม่ถึงเวลา, Project ไม่มีตัวเลือก>

ต้องการดูรายละเอียดที่เขียนไหม?
```

Only if they say yes, show the content and point out any % you estimated.
