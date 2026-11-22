---
name: finalrex-kpi
description: Define, run, update and delete KPIs for a FinalRex board (Business plan and up) - analyses what the board can really measure, asks 4-5 questions built from that analysis, offers existing KPIs first, dry-runs before saving, and asks again before saving an exact duplicate. Use when the user says /finalrex-kpi, "สร้าง KPI", "วัดผลบอร์ด", "KPI ของทีม", "รัน KPI", "ลบ KPI", "create a KPI", "run a KPI", "measure the team's work".
---

# FinalRex KPI

Define and run KPIs for a board, reusing existing KPIs before creating new ones. Needs the FinalRex Business plan or higher: if a KPI tool answers with a plan error, tell the user, in Thai, that KPIs need the Business plan, and stop.

Settings: `~/.claude/finalrex-report.json` (shared with `finalrex-report`). Ask every question with AskUserQuestion, in Thai. Talk to the user in Thai, short. Quoted English strings below are meant to be shown to the user: translate them to Thai when you say them.

## Names, not ids

- In chat, AskUserQuestion questions and options, confirmations, dry-run summaries and reports, refer to workspaces, boards, views, tasks, notes, books, pages, atlases, nodes, KPIs and people by name or title (plus the url a tool returns when linking). Never show raw ids / UUIDs unless the user asks for them.
- Before every `mcp__finalrex__*` call, write one short Thai line saying what it will do in names and counts, e.g. "Will read the KPIs of the workspace solutions" or "Will check the KPI On-time rate (dry run, nothing saved)", so the user understands the permission prompt, which shows raw ids.
- Tool calls still pass ids / UUIDs exactly as before. If a name is not known yet, take it from an earlier `list_*` / `get_*` result instead of printing the id.
- Links: give the `url` that a result carries (KPI, board, workspace). Never build a link from ids yourself.

## 0. Connect FinalRex

Look for `mcp__finalrex__*` tools (ToolSearch "+finalrex" if deferred). If none exist, stop and tell the user, in Thai, to run `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`, then `/mcp` › finalrex › log in, and run the skill again. Auth error → tell them to sign in the same way. Never ask for tokens or passwords.

## 1. Settings

Read `~/.claude/finalrex-report.json`. Needed: `workspace`, `board`.

- Both present → offer the saved board as the default ("Use the board <board name>?"), otherwise ask.
- Missing → ask only for that (workspace: `list_workspaces`; board: `list_boards`, "Master Plan" first). Save `workspace` / `board` into the json, keeping the other keys.
- The user may pick another board, or several boards for one KPI. Do not overwrite the saved board for a one-off choice.

## 2. Pick the board and read what is measurable

1. `get_board` for the chosen board(s): fields, select options, done options. `list_members` for people.
2. `kpi_capabilities` with `boardIds` (and `workspace`): per metric and template, whether it is feasible and why, how many tasks have an assignee / a due date, done tasks in the last 30 / 90 days, fields usable per metric. Use only what it says is feasible. Never propose a KPI it marks infeasible; tell the user why instead (e.g. "No task has a due date, so on-time rate cannot be measured").
3. `list_kpi_templates` for ready-made starting configs.

## 3. Understand the goal

Ask 4-5 questions with AskUserQuestion, built from the analysis in step 2 (options come from the real fields, statuses, projects and people of the board, not generic ones):

1. Goal: what to measure (e.g. work done, on-time rate, overdue, cycle time, progress, tasks per status, a sum of a number field).
2. Period: this week / this month / last N days / custom range / all time (and a trend over weeks or months if useful).
3. Who or what to compare: per person, per a select field (e.g. Project), or the team as a whole; include unassigned tasks or not.
4. Filter: only some projects / types / statuses, and subtasks included or top-level only.
5. Target or threshold, if the user has one (the KPI config has no target field: mention it in the KPI name or compare it to the result when you report).

Skip a question the user already answered. Ask fewer when the goal is clear.

## 4. Offer existing KPIs first

1. `list_kpis`. Pick the ones on the same board (or workspace-wide) whose name, template or config fits the goal. `get_kpi` for a closer look when unsure.
2. Show them, each with its url, and ask "Use one of these, or create a new KPI?".
3. Use an existing one → `run_kpi` with its `kpiId` (and a `period` only if the user asked for a different one), then step 7. Change to an existing KPI → step 8.

## 5. Build the config

Start from a template when one fits, else from the metrics the user wants. Config shape (fields and options are given by name and matched by name on every board):

- `metrics`: list of `{metric, key, label, field?, values?, basis?, expr?}`. Metrics: `tasks_done`, `on_time_pct`, `overdue_count`, `cycle_time_avg`, `cycle_time_median`, `completion_pct`, `status_count` (needs `field` + `values`), `field_sum` / `field_avg` (a number or manual percent field), `formula` (`expr` over other metric keys).
- `period`: `this_week` | `this_month` | `last_days` (`days`) | `custom` (`from`, `to`) | `all_time`.
- `groupBy`: `assignee` | `field` (with `groupField`, a select field) | `none`.
- `filters`: `{field, op, values}` with op `eq|neq|in|not_in|empty|not_empty`.
- `subtasks`: `include` | `top_level_only` | `only_subtasks`; `boardIds`; `includeUnassigned`; optional `trend: {interval, periods}`.

Pick `scope`: `personal` (default, only the user), `board` (attached to the board, needs manage access) or `workspace` (owner / admin). Ask if the user wants it shared (visibility: private / people / roles). Do not widen visibility on your own.

Optionally `run_kpi` with the ad hoc `config` (and `workspace`) to preview the numbers before saving. This writes nothing.

## 6. Dry run, check duplicates, confirm, save

1. `create_kpi` with `dryRun: true`. Fix any validation error (it names the problem). Read the similar KPIs it returns (`exact`, `close`, `name`).
2. Show the user the KPI in plain words (name, metrics, period, grouping, filters, scope, visibility) and any similar KPIs with their urls.
3. **Exact match** → do not save yet. Ask again: "A KPI with the same settings already exists: <name>. Use it, or really create a separate copy?". Use it → step 4 of section 4. Copy wanted → the new KPI needs a **distinct name** (suggest one, e.g. add the purpose or the period); save with `confirmDuplicate: true`.
4. **Same name** (`name` match) → ask for a different name.
5. **Close match** → mention it and ask once whether to use it instead.
6. **Confirm once:** "Save the KPI <name> (scope <scope>)?". Do not write before the yes. Then `create_kpi` for real with the same arguments (no `dryRun`; `confirmDuplicate: true` only after the user agreed to a copy). If the save fails because a duplicate exists, go back to point 3 instead of forcing it.
7. `run_kpi` with the new `kpiId` to show the first result and check that it works.

## 7. Show the result

From `run_kpi`: show `columns` and `rows` as a table (per assignee, option or team), then the totals (each task counted once). Say which boards were skipped and why, and the time zone. With a trend, show the series per bucket as a table. Say "No data in this period" instead of an empty table. Give the KPI's `url`. If the user gave a target, say how the result compares.

## 8. Change a KPI

`update_kpi` (rename, replace the whole `config`, `visibility`, `charts`). Only what the user asked changes. Because `config` replaces the whole config, start from `get_kpi` and edit that. Show the change in plain words and confirm once before writing. Rights are checked by the server: if it refuses, tell the user who can change it.

## 9. Delete a KPI

`delete_kpi` **cannot be undone**. Never call it on a guess or on a loose "clean up".

1. Only for a KPI the user named. Show its name, scope, creator and url, and say that deleting cannot be undone.
2. Ask with AskUserQuestion: "Delete the KPI <name>? This cannot be undone." Options: delete / keep it. Call `delete_kpi` only on an explicit "delete" answer, one KPI per confirmation.
3. Afterwards say which KPI was deleted.

**Always:** a KPI is only saved after a dry run and one confirmation; never delete without the user's explicit yes; never create tasks or notes (this skill writes only KPIs and the settings json). No secrets or customer personal data in KPI names.
