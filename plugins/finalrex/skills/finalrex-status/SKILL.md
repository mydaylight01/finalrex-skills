---
name: finalrex-status
description: Read-only status of the configured FinalRex board (e.g. Master Plan) as tables - open and in-progress tasks, overdue, STUCK (no change for 3+ days), Hold, open risks - for me, a person, a project or the whole team. Never writes anything. Use when the user says /finalrex-status, "งานฉันค้างอะไร", "ทีมติดอะไร", "status โปรเจค X", "งานใครOverdue", "what's my status", "what is the team stuck on", "project status".
---

# FinalRex status (read-only)

Answer "what is open / late / stuck" from the board. **This skill never writes**: no `create_*`, `update_*`, `move_*`, `delete_*` and no `batch_tasks` (not even a dry run). If the user asks for a change, tell them to use `finalrex-report`, `finalrex-standup` or `finalrex-plan`.

Settings: `~/.claude/finalrex-report.json` (shared with `finalrex-report`). Ask every question with AskUserQuestion, in Thai. Answer in Thai, short. Quoted English strings below are meant to be shown to the user: translate them to Thai when you say them.

## Names, not ids

- In chat, AskUserQuestion questions and options, confirmations, dry-run summaries and reports, refer to workspaces, boards, views, tasks, notes, books, pages, atlases, nodes and people by name or title (plus the url a tool returns when linking). Never show raw ids / UUIDs unless the user asks for them.
- Before every `mcp__finalrex__*` call, write one short Thai line saying what it will do in names and counts, e.g. "Will read 20 tasks from the board Master Plan" or "Will create 3 pages in the book Team Handbook", so the user understands the permission prompt, which shows raw ids.
- Tool calls still pass ids / UUIDs exactly as before. If a name is not known yet, take it from an earlier `list_*` / `get_*` result (look it up first if cheap) instead of printing the id.

## 0. Connect FinalRex

Look for `mcp__finalrex__*` tools (ToolSearch "+finalrex" if deferred). If none exist, stop and tell the user, in Thai, to run `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`, then `/mcp` › finalrex › log in, and run the skill again. Auth error → tell them to sign in the same way. Never ask for tokens or passwords.

## 1. Settings

Read `~/.claude/finalrex-report.json`. Needed: `workspace`, `board`, `rootNote`. `displayName` / `userId` are needed only for "me".

- All present → use them, no question.
- Something missing → ask only for that, the same way `finalrex-report` does (workspace: `list_workspaces`; board: `list_boards`, "Master Plan" first; rules note: `list_notes` top-level notes, "Work Log Reports" first; name: `whoami` display name as default). Save it into the json, keeping the other keys. Saving settings is the only file this skill writes.

## 2. Read the rules

`get_note` the rules note (`rootNote`). Its "FOR AI" section wins over this skill: Status meanings, done options, sort order, table style.

## 3. Scope

From the user's words, pick one; ask only if unclear ("Whose status?": me / another person / a project / the whole team):

- **me** → `userId` from the json. If missing: `list_members`, match `displayName` or the `whoami` email, save `userId`.
- **a person** → `list_members`; match the name to `displayName` or `name` (names may be Thai or nicknames that differ from report titles, e.g. "Nok" = `nok`, "cream" = `creme`). Ask if unsure.
- **a project** → match to a `Project` option from `get_board` (ask if none or several match).
- **whole team** → no filter.

## 4. Read the board

1. `get_board`: Status / Project option ids, `doneOptionIds` (e.g. Done and Close).
2. `list_tasks` with `compact: true`, `limit: 200`, follow `nextCursor`, and a `filter` (JSON string). Filter facts checked live:
   - select fields take **option ids**, not labels: `{"field":"Status","op":"nin","value":[<doneOptionIds>]}`
   - person: `{"field":"assignee","op":"in","value":["<userId>"]}`
   - project: `{"field":"Project","op":"eq","value":"<option id>"}`
   - dates in a filter: `due_at`, `start_at` (e.g. `{"field":"due_at","op":"lt","value":"<today>T00:00:00Z"}`)
   - the `fields` parameter takes the names as they come back: `Status`, `Progress`, `Project`, `Remark`, `table.col.assignee`, `table.col.start_at`, `table.col.due_at`. Empty values are left out of the result.
3. `list_members` once to turn assignee ids into names.
4. For the task a subtask is a step of, use the `parentId` and the parent row from the same list (`get_task` only if the parent is not in it). Add `Type` to `fields` when the board has a Type field.

## 5. Classify (today = local date)

A task can be in more than one table.

| Table | Rule |
|---|---|
| Pending (Open / In Progress) | Status Open or In Progress |
| Overdue | not done (Status not in `doneOptionIds`) and `due_at` day < today |
| STUCK | Open or In Progress and no change for 3 days or more (see below) |
| Hold | Status Hold |
| Open risks | Type Risk and not done (rules note: open risks = Type Risk tasks, linked Risk of to what they concern; `get_task` shows that link). Board without a Type field: Status Open, Progress 0 or empty |

**STUCK and "last change".** Checked live: `list_tasks`, `get_task` and `search` return **no updated time** for tasks (only `startAt` / `dueAt`). So use the best signal available:

1. **Last report day**: the latest day the task title appears in a "Daily Report: …" or "Standup: …" note. Read only the last 3 days (today, -1, -2): `list_notes` with `workspace` (slug) **and** `boardId` (on an all-workspaces connection `list_notes` fails without `workspace`), find `Daily: dd/mm/yyyy` notes for those days, `get_note` each child report. A task named in any of them changed recently → not STUCK.
2. Otherwise **`start_at`**: started within the last 3 days → not STUCK; started 3+ days ago → STUCK, "No change since" = the last report day if seen, else `start_at`.
3. No report and no `start_at` → not STUCK (age unknown); count them in one line under the tables ("N tasks have no date, cannot tell if they are stuck").

If FinalRex later adds an updated time to tasks, use that instead and say so.

## 6. Show the answer

Sort every table by the rules note sort rule: Project A→Z, then Progress high→low, rows without a % last (Hold / Close last within their Project). Escape `|` in cells as `\|`. Use task titles exactly as on the board. Skip an empty table and say "None" in one line.

```
Status of <scope> — board <board name> (as of dd/mm/yyyy)

Summary: Pending N | Overdue N | STUCK N | Hold N | Risks N

### Overdue
| Task | Project | Assignee | Status | Progress | Due | Days overdue |

### STUCK (no change for 3+ days)
| Task | Project | Assignee | Status | Progress | No change since | Remark |

### Pending (Open / In Progress)
| Task | Project | Assignee | Status | Progress | Due | Remark |

### Hold
| Task | Project | Assignee | Remark |

### Open risks
| Task | Project | Assignee | Remark |
```

- Scope "me": drop the Assignee column.
- Whole team with many rows: show Overdue, STUCK and Open risks in full, and the Pending table only on request ("Show all pending tasks?").
- End with one line on how STUCK was judged (report notes of the last 3 days + `start_at`, because tasks have no updated time).

**Always:** read-only. No secrets or customer personal data in the answer beyond what is on the board.
