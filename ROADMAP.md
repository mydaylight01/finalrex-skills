# Roadmap

| Skill | Status | What it does |
|-------|--------|--------------|
| `finalrex-report` | Done (0.1.0) | Logs the session's work as tasks and writes the Daily Report note. |
| `finalrex-summary` | Done (0.2.0) | Writes Weekly / Monthly reports per person and the team summary from tasks and Daily Reports. |
| `finalrex-standup` | Done (0.2.0) | Records a standup: yesterday / today / blockers per person, task updates, "Standup: dd/mm/yyyy" note. |
| `finalrex-plan` | Done (0.2.0) | Turns a request or spec into parent tasks and subtasks, without duplicates. |
| `finalrex-status` | Done (0.2.0) | Read-only status: open, overdue, stuck, Hold and open-risk tasks for me / a person / a project / the team. |
| `finalrex-kpi` | On hold | Defines and runs KPIs for a board. |

## finalrex-kpi (on hold)

Planned flow:

1. Pick a board.
2. Analyse what can really be measured on it (fields, statuses, dates, people, history).
3. Ask the user 4-5 questions built from that analysis (goal, period, people or projects, target).
4. Check the KPIs that already exist and offer them first.
5. Before creating a KPI that is an exact duplicate of an existing one, ask again.
6. When the user really wants a separate one, give it a distinct name.

Blocked until the FinalRex MCP has KPI tools: `kpi_capabilities`, and list / get / run / create / update / delete KPI, with duplicate detection.
