---
name: finalrex-plan
description: Turn a request or spec (text, a file, or the current conversation) into tasks on the configured FinalRex board (e.g. Master Plan) - one task per deliverable with Project, Type, Assignee, Start/Due, Status Open and Remark, subtasks only for its steps, and links between tasks - after checking for duplicates, with one confirmation. Use when the user says /finalrex-plan, "แตก task", "วางแผนงานลง finalrex", "สร้าง task จาก spec", "แปลง requirement เป็น task", "plan this into tasks", "break this down into tasks".
---

# FinalRex plan

Break a request into tasks on the board (one per deliverable, with their steps as subtasks and links between them), without duplicating existing work.

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

`get_note` the rules note (`rootNote`). Its "FOR AI" section wins over this skill: one task per deliverable (no group task named after a date range or batch, no title joining several things with "+"), Project + Type on every task, subtasks only for steps of the same task (one level), relations as links (blocks, relates, custom link types), releases as Type Release + Includes links, open risks as Type Risk + Risk of links, short titles with no "(Project)" suffix, required fields, Status from %, and the Standard section (use it to add the steps a standard requires, e.g. translations th + en, staging before main).

## 3. Get the request

- Input: text in the message, a file path (read it), or "from this conversation" (use the current conversation). Nothing given → ask in chat: "Paste the requirement / spec, or give a file path.".
- Pull out: goal, deliverables (each a thing that can be handed over: feature, bug fix, decision, release, research, doc) with their Project and Type, the steps of each (spec, build parts, test, meeting/decision, CR, ER), how they relate (what waits for what, what ships together, what replaces what), people named, dates or deadline.

## 4. Check what exists

1. `get_board` (fields, Project / Type / Status options, `doneOptionIds`), `list_link_types` (the workspace's custom link types; built-in: blocks, relates), `list_members` (names → user ids; names may be Thai or nicknames, ask if unsure).
2. For each planned task and step, look for an existing task: `search` (kinds `task`) with 2-3 keywords and any F / ADR code, and `list_tasks` (`compact: true`, filter `{"field":"title","op":"contains","value":"<keyword>"}`; select filters take **option ids**).
3. Existing match → do not create it again; show it as "already exists". If the new item is a step of it, put the step under it as a subtask; if the new item only relates to it, link to it instead. Unsure if it is the same work → ask.

## 5. Plan

- **One task per deliverable** at the top level. Never a group task named after a date range, a sprint or a batch, and never one title joining several things with "+" or ",": split them.
- **Subtasks** (one level) only for steps of that same task: spec / design, build parts of the same deliverable, test / QA, meeting or decision about it, CR / ER within its scope, deploy check. If the matching task is itself a subtask, use its parent. Checked live: `batch_tasks` accepts deeper nesting, so enforce this yourself.
- **Links** for relations between tasks: blocks (the from-task must finish first), relates, or a custom link type by name from `list_link_types` (e.g. Includes, Found in, Supersedes, Risk of). Only from what the request says; never guess. A link type the rules note names but the workspace lacks → ask before creating it.
- A planned release round → a task with Type Release, linked Includes to what it ships. Known risks or open questions → a task with Type Risk, Status Open, linked Risk of to the task it concerns, detail in Remark.
- Fields: `Project` and `Type` (existing options only; none fits → ask; never add an option), `assigneeIds` (asked or named people; unknown → leave empty and say so), `startAt` / `dueAt` (RFC 3339, from the request; no date given → leave empty, do not invent deadlines), `Status` Open, `Progress` 0, `Remark` one line (scope / acceptance / what it waits on).
- Titles short and clear, `<what> (<F / ADR code if any>)`, in the language of the request.

Show the plan as a table and ask for changes before the dry run is final:

```
| # | Task | Type | Under | Project | Assignee | Start | Due | Remark |
|---|---|---|---|---|---|---|---|---|
| 1 | <task> | Feature | - | Gocprime | Cream | 16/11 | 20/11 | ... |
| 1.1 | <step of task 1> | Test | 1 | Gocprime | Ky | 19/11 | 20/11 | ... |
| 2 | <task> | Decision | - | Gocprime | Oum | 16/11 | 16/11 | ... |
```

Then the links, one per line: "2 → blocks → 1". Also list "Already exists (not created again): <title>".

## 6. Dry run, confirm once, run, verify

1. `batch_tasks` with `dryRun: true`. Each new task is a `create` op with a `ref`; its subtasks use that ref as `parentId`; links are `link` ops whose from / to may be refs or existing task ids. Fix any error (an unknown option fails with the list of valid options).
2. **Confirm once:** "Create these tasks? (N tasks, N subtasks, N links)". Do not write before the yes.
3. Run `batch_tasks` for real with the same ops. If the connector's `batch_tasks` has no `link` op, create the tasks first, then each link with `create_task_link`.
4. Verify with `list_tasks` (`parentId` of each new task, or a title filter) and `get_task` on linked tasks. If the batch stopped partway, report which ops were applied and fix the rest without duplicating.

**Always:** no notes are written by this skill. No secrets, credentials, tokens or customer personal data (names, emails, phones) in tasks.

## 7. Tell the user

Short, in Thai:

```
Tasks created (board <board name>)
- <task> — <Type>, Open, <Project>, <assignee>
  - <subtask> — <Type>, <assignee>, due dd/mm
Links: <task> → <link type> → <task>
Already exists (not created again): <title>
Still missing: <missing assignees / dates>
```
