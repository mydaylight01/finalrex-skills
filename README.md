# finalrex-skills

Claude Code skills for FinalRex.

## Requirements

- Claude Code
- The FinalRex MCP server connected and signed in

## Install

Run in Claude Code:

```
/plugin marketplace add mydaylight01/finalrex-skills
/plugin install finalrex@finalrex-skills
```

## Update

```
/plugin marketplace update finalrex-skills
```

## Skills

| Skill | What it does |
|-------|--------------|
| `finalrex-report` | Logs the work of the session as tasks on your FinalRex board and writes the Daily Report note. Say `/finalrex-report` or "เขียน daily". |
| `finalrex-summary` | Writes Weekly / Monthly reports per person and the team summary, as tables, from the board's tasks and Daily Reports. Say `/finalrex-summary` or "สรุป weekly". |
| `finalrex-standup` | Records a standup: yesterday / today / blockers per person, updates tasks (blockers become Open risk tasks) and writes the "Standup: dd/mm/yyyy" note. Say `/finalrex-standup` or "บันทึก standup". |
| `finalrex-plan` | Turns a request or spec into parent tasks and subtasks, checking for duplicates first. Say `/finalrex-plan` or "แตก task". |
| `finalrex-status` | Read-only: open, overdue, stuck, Hold and open-risk tasks for you, a person, a project or the team. Say `/finalrex-status` or "งานฉันค้างอะไร". |
| `finalrex-kpi` | Planned (on hold until the FinalRex MCP has KPI tools). See [ROADMAP.md](ROADMAP.md). |

Every skill reads the rules note on your board first; its rules win over the skill text.
Skills that write show a dry run and ask once before writing.

The first run asks for your workspace, board, rules note, format and name.
Your settings are saved in `~/.claude/finalrex-report.json` on your machine and shared by all the skills. They are not part of this repo.
