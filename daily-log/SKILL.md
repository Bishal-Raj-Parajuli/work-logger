---
name: daily-log
description: >
  Show today's daily work log collected by the work logger. Use when the user
  says "/daily-log", "show my log", "what did I do today", "show daily log",
  "review my log", or "get my summary ready". Reads ~/.claude/daily-log.md and
  displays today's entries grouped by project. Can also export a clean per-project
  summary suitable for pasting into a standup doc, chat, or follow-on AI prompt.
command: /daily-log
allowed-tools: Bash, Read
---

# /daily-log — View Today's Work Log

Read and display today's entries from `~/.claude/daily-log.md`, grouped by project.

## What to do

1. Read `~/.claude/daily-log.md`
2. Find today's section (header `## YYYY-MM-DD`)
3. Display all entries under today's date, organized by project subheadings
4. If `$ARGUMENTS` contains "copy" or "export", format it as a clean per-project
   summary suitable for pasting into a standup doc, chat, or follow-on AI prompt.

## Steps

```bash
TODAY=$(date '+%Y-%m-%d')
LOG_FILE="$HOME/.claude/daily-log.md"

if [ ! -f "$LOG_FILE" ]; then
    echo "No daily log found. The work logger may not be set up yet."
    exit 0
fi

# Extract today's section (from ## TODAY to the next ## date or EOF).
# `sed '$d'` drops the trailing next-date header captured by the range
# match. Portable across GNU and BSD sed (macOS), unlike `head -n -1`.
sed -n "/^## $TODAY/,/^## [0-9]/p" "$LOG_FILE" | sed '$d'
```

## Output modes

### Default: show the raw log
Display the entries as-is, grouped by project. The user can review what was captured.

### Export mode (`/daily-log copy`):
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

### Filter mode (`/daily-log <project-name>`):
If the argument matches a project name, show only entries for that project.

## Rules

1. If no entries exist for today, say so — "No entries logged today yet."
2. Don't editorialize. Just show the log.
3. For export mode, group by project name, strip metadata, and skip start/end markers.
4. If the user asks for a specific project, only show that project's entries.
