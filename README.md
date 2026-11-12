# finalrex-skills

Claude Code skills for FinalRex.

## Requirements

- Claude Code
- The FinalRex MCP server connected and signed in

## Install

Run in Claude Code:

```
/plugin marketplace add <owner>/finalrex-skills
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

The first run asks for your workspace, board, rules note, format and name.
Your settings are saved in `~/.claude/finalrex-report.json` on your machine. They are not part of this repo.
