---
name: daily-log
description: >
  Show today's daily work log collected by the work logger. Use when the user
  says "/work-logger:daily-log", "/daily-log", "show my log", "what did I do today", "show daily log",
  "review my log", or "get my summary ready". Reads the log
  (~/.claude/daily-log.md, or $CLAUDE_DAILY_LOG) and displays today's entries
  grouped by project. Can also export a clean per-project summary suitable for
  pasting into a standup doc, chat, or follow-on AI prompt.
argument-hint: "[copy | <project-name> | YYYY-MM-DD]"
allowed-tools: Bash
---

# /daily-log — View Today's Work Log

Show a day's entries from the daily work log, grouped by project.

## Steps

Print the day's section with the logger script. It finds the log file
(`$CLAUDE_DAILY_LOG` or `~/.claude/daily-log.md`) and returns everything from
`## YYYY-MM-DD` up to the next date:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/work-logger.sh" --show              # today
bash "${CLAUDE_PLUGIN_ROOT}/scripts/work-logger.sh" --show 2025-01-15   # a specific day
```

Then pick the output mode from `$ARGUMENTS`:

- contains `copy` or `export` → export mode
- a `YYYY-MM-DD` date → show that day instead of today
- matches a project heading → filter mode
- empty → default mode

## Output modes

### Default: show the raw log
Display the entries as-is, grouped by project. The user can review what was captured.

### Export mode (`/daily-log copy`)
Strip the timestamps and emoji, collapse into a clean summary grouped by project.
Only include substantive entries (skip session start/end noise).
Output something like:

```
api-service:
- Implemented Redis-backed rate limiter
- Fixed off-by-one in token bucket refill
- Added integration tests for burst behavior

frontend:
- Updated onboarding flow UI

General / Non-dev:
- Met with team about API redesign
- Blocker: waiting on infra for staging DB access

Next:
- [leave blank for user to fill or skip]
```

This is ready to paste into a standup doc, chat thread, or follow-on AI prompt for further formatting.

### Filter mode (`/daily-log <project-name>`)
Show only the entries under that project's `### <project-name>` heading.

## Rules

1. If the script says there are no entries, pass that on — "No entries logged today yet."
2. Don't editorialize. Just show the log.
3. For export mode, group by project name, strip metadata, and skip start/end markers.
4. If the user asks for a specific project, only show that project's entries.
