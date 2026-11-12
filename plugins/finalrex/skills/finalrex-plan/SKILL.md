---
name: finalrex-plan
description: Turn a request or spec (text, a file, or the current conversation) into tasks on the configured FinalRex board (e.g. Master Plan) - parent tasks and subtasks with Project, Assignee, Start/Due, Status Open and Remark - after checking for duplicates, with one confirmation. Use when the user says /finalrex-plan, "แตก task", "วางแผนงานลง finalrex", "สร้าง task จาก spec", "แปลง requirement เป็น task", "plan this into tasks", "break this down into tasks".
---

# FinalRex plan

Break a request into parent tasks and subtasks on the board, without duplicating existing work.

Settings: `~/.claude/finalrex-report.json` (shared with `finalrex-report`). Ask every question with AskUserQuestion, in Thai. Talk to the user in Thai, short.

## 0. Connect FinalRex

Look for `mcp__finalrex__*` tools (ToolSearch "+finalrex" if deferred). If none exist, stop and tell the user, in Thai, to run `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`, then `/mcp` › finalrex › log in, and run the skill again. Auth error → tell them to sign in the same way. Never ask for tokens or passwords.

## 1. Settings

Read `~/.claude/finalrex-report.json`. Needed: `workspace`, `board`, `rootNote`, `displayName`.

- All present → use them, no question.
- Something missing → ask only for that, the same way `finalrex-report` does (workspace: `list_workspaces`; board: `list_boards`, "Master Plan" first; rules note: `list_notes` top-level notes, "Work Log Reports" first; name: `whoami` display name as default). Save it into the json, keeping the other keys.

## 2. Read the rules

`get_note` the rules note (`rootNote`). Its "FOR AI" section wins over this skill: feature = parent, pieces = subtasks (one level), short titles with no "(Project)" suffix, required fields, Status from %, open risks = Open tasks, and the Standard section (use it to add the steps a standard requires, e.g. translations th + en, staging before main).

## 3. Get the request

- Input: text in the message, a file path (read it), or "จากที่คุยกัน" (use the current conversation). Nothing given → ask in chat: "วาง requirement / spec หรือบอก path ไฟล์ได้เลย".
- Pull out: goal, deliverables, people named, dates or deadline, Project.

## 4. Check what exists

1. `get_board` (fields, Project / Status options, `doneOptionIds`), `list_members` (names → user ids; names may be Thai or nicknames, ask if unsure).
2. For each planned feature and piece, look for an existing task: `list_tasks` (`compact: true`, filter `{"field":"title","op":"contains","value":"<keyword>"}`; select filters take **option ids**) and `search` (kinds `task`) with 2-3 keywords each.
3. Existing match → do not create it again. Use it as the parent for new pieces, or leave it as is and show it as "มีอยู่แล้ว". Unsure if it is the same work → ask.

## 5. Plan

- **Parent task** per feature, **subtasks** per piece (one level; if the matching parent is itself a subtask, use its parent. Checked live: `batch_tasks` accepts deeper nesting, so enforce this yourself).
- Fields: `Project` (existing option only; none fits → ask; never add an option), `assigneeIds` (asked or named people; unknown → leave empty and say so), `startAt` / `dueAt` (RFC 3339, from the request; no date given → leave empty, do not invent deadlines), `Status` Open, `Progress` 0, `Remark` one line (scope / acceptance / what it waits on).
- Known risks or open questions in the request → their own Open task under the feature, detail in Remark.
- Titles short and clear, in the language of the request.

Show the plan as a table and ask for changes before the dry run is final:

```
| # | Task | ใต้ | Project | ผู้ทำ | Start | Due | Remark |
|---|---|---|---|---|---|---|---|
| 1 | <parent> | - | Gocprime | cream | 16/11 | 20/11 | ... |
| 1.1 | <subtask> | 1 | Gocprime | Ky | 16/11 | 17/11 | ... |
```

Also list "มีอยู่แล้ว (ไม่สร้างซ้ำ): <title>".

## 6. Dry run, confirm once, run, verify

1. `batch_tasks` with `dryRun: true`. Each new parent is a `create` op with a `ref`; its subtasks use that ref as `parentId`. Fix any error (an unknown option fails with the list of valid options).
2. **Confirm once:** "สร้าง task ตามนี้ไหม? (parent N, subtask N)". Do not write before the yes.
3. Run `batch_tasks` for real with the same ops.
4. Verify with `list_tasks` (`parentId` of each new parent, or a title filter). If the batch stopped partway, report which ops were applied and fix the rest without duplicating.

**Always:** no notes are written by this skill. No secrets, credentials, tokens or customer personal data (names, emails, phones) in tasks.

## 7. Tell the user

Short, in Thai:

```
สร้าง task แล้ว (board <board name>)
- <parent> — Open, <Project>, <ผู้ทำ>
  - <subtask> — <ผู้ทำ>, due dd/mm
มีอยู่แล้ว (ไม่สร้างซ้ำ): <title>
ยังไม่มี: <missing assignees / dates>
```
