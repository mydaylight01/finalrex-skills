---
name: finalrex-standup
description: Record a standup meeting on the configured FinalRex board (e.g. Master Plan) - split pasted notes or a transcript per person into yesterday / today / blockers, update or create the matching tasks (blockers become Open risk tasks) with one confirmation, and write the "Standup: dd/mm/yyyy" note under that day's Daily note. Use when the user says /finalrex-standup, "บันทึก standup", "ลง standup", "จด daily standup", "สรุปประชุม standup", "record standup", "standup notes".
---

# FinalRex standup

Turn standup notes into task updates and one "Standup: dd/mm/yyyy" note.

Settings: `~/.claude/finalrex-report.json` (shared with `finalrex-report`). Ask every question with AskUserQuestion, in Thai. Talk to the user in Thai, short.

## 0. Connect FinalRex

Look for `mcp__finalrex__*` tools (ToolSearch "+finalrex" if deferred). If none exist, stop and tell the user, in Thai, to run `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`, then `/mcp` › finalrex › log in, and run the skill again. Auth error → tell them to sign in the same way. Never ask for tokens or passwords.

## 1. Settings

Read `~/.claude/finalrex-report.json`. Needed: `workspace`, `board`, `rootNote`, `displayName`.

- All present → use them, no question.
- Something missing → ask only for that, the same way `finalrex-report` does (workspace: `list_workspaces`; board: `list_boards`, "Master Plan" first; rules note: `list_notes` top-level notes, "Work Log Reports" first; name: `whoami` display name as default). Save it into the json, keeping the other keys.

## 2. Read the rules

`get_note` the rules note (`rootNote`). Its "FOR AI" section wins over this skill: task rules (find before create, one subtask level, required fields, Status from %), open risks = Open tasks, note tree, no duplicates.

## 3. Get the standup

- Input = notes or a transcript the user pasted (or a file path they gave). If there is none, ask in chat: "วางโน้ตหรือ transcript ของ standup ได้เลย".
- Meeting date: from the text, else today. Show it in the confirm step.
- Do not keep raw transcript text in tasks or notes; write short summaries.

## 4. Split per person

1. `list_members` once. Map every speaker to a member by `displayName` or `name`. Names may be Thai or nicknames that differ from the member name (e.g. "Nok" = `nok`, "cream" = `creme`); also look at the names already used in "Daily Report: <name>" note titles. Unsure → ask ("<ชื่อ> คือใคร?" with up to 4 likely members).
2. Per person: **เมื่อวาน** (done), **วันนี้** (plan), **ติดอะไร** (blockers). Empty → `-`.

## 5. Match and plan task changes

1. `get_board` for fields and option labels, `doneOptionIds`.
2. Find existing tasks for each item: `list_tasks` (`compact: true`, filter `{"field":"title","op":"contains","value":"…"}`, or `{"field":"assignee","op":"in","value":["<userId>"]}` with Status not done — select filters take **option ids**) and/or `search` (kinds `task`). Prefer the person's own open tasks.
3. Plan per the rules note:
   - **Progress said / work done** → update the task: `Progress`, `Status` from % (100 = Done, 1-99 = In Progress, 0 = Open), `Remark` (one line: what is left), `dueAt` = meeting day for work done that day. Never lower Progress unless the person said so.
   - **New work** with no matching task → create it under the related feature (`parentId`), `assigneeIds`, `startAt` = meeting day, Project (existing option only), Status, Progress, Remark.
   - **Blocker** → its own task with Status **Open**, Progress 0, under the related feature, detail in Remark ("ติด: …, รอ …"). Reuse an existing open risk task with the same meaning.
   - **One subtask level.** If the related task is itself a subtask, use its parent as `parentId`. (Checked live: `batch_tasks` accepts a subtask under a subtask, so the skill must enforce this rule itself.)
   - Project with no fitting option → ask before anything else; never add an option.
4. **Dry run.** `batch_tasks` with `dryRun: true`. Fix any error (an unknown option fails with the list of valid options).
5. **Confirm once.** Show:
   - the per-person table (section 7) and
   - the task changes: `สร้าง / อัพเดต: <title> — <Status> <N>% (ใต้ <parent>)`
   and ask "ลง task และเขียนโน้ต Standup ตามนี้ไหม?". Do not write before the yes.
6. **Run** `batch_tasks` for real (create ops use `ref` so later ops can point at them). Then `list_tasks` again to verify; if the batch stopped partway, report which ops were applied and fix the rest.

## 6. Find or create the note

```
<board>
└─ Monthly: MM/YYYY
   └─ Weekly: dd/mm - dd/mm      Monday–Sunday
      └─ Daily: dd/mm/yyyy
         └─ Standup: dd/mm/yyyy
```

- `list_notes` with `workspace` (slug) **and** `boardId` (on an all-workspaces connection it fails without `workspace`). Match by title and parent; ignore untitled notes.
- Create missing parents like `finalrex-report`: Monthly with `create_note` + `boardId`, every child with `parentId`. Call `list_notes` again right before every `create_note` and reuse a note that appeared meanwhile.
- A "Standup: dd/mm/yyyy" note that already exists → `get_note` (version + body), merge (update the row of the same person, add new people), `update_note` with the whole body.
- After writing, list once more: two notes with the same title under one parent → merge into the older one and `delete_note` the newer one.

## 7. Note body

```markdown
| คน | เมื่อวาน | วันนี้ | ติดอะไร | Task ที่เกี่ยวข้อง |
|---|---|---|---|---|
| <name as in Daily Report titles> | <short> | <short> | <short or -> | <task titles exactly as on the board, comma separated> |
```

- One row per person, sorted by name. Escape `|` as `\|`. Thai, short.

**Always:** never write into the rules note. No secrets, credentials, tokens or customer personal data (names, emails, phones) in tasks or notes.

## 8. Tell the user

Short, in Thai:

```
ลง standup แล้ว (dd/mm/yyyy)
Task: สร้าง N, อัพเดต N, ความเสี่ยงใหม่ N
- <title> — <Status> <N>%
Note: Monthly: MM/YYYY › Weekly: … › Daily: dd/mm/yyyy › **Standup: dd/mm/yyyy**
ข้าม: <anything skipped and why>
```
