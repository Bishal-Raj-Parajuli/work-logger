---
name: log-entry
description: >
  Manually add an entry to the daily work log (~/.claude/daily-log.md).
  Use when doing non-code work that the automatic work logger won't capture:
  meetings, emails, research, decisions, blockers, or anything you want in your
  daily summary. Trigger on "/log-entry", "/log", "log this", "add to daily log",
  or when the user says something like "note that I did X" or "record that".
command: /log-entry
argument-hint: <what you did>
allowed-tools: Bash, Read, Write
---

# /log-entry — Manual Daily Log Entry

Append a manual entry to the daily work log at `~/.claude/daily-log.md`.

## When this fires

The user types `/log-entry <description>` or says "log this", "add to my log", etc.
The `$ARGUMENTS` variable contains what they want logged.

## What to do

1. Read `$ARGUMENTS` — this is the raw description of what to log.
2. Determine which **project** this belongs to:
   - If the user is in a project directory, default to that project name.
   - If the entry is about a different project, use that name.
   - If it's general (meeting, email, admin), use `General`.
3. Clean it up into a concise, specific one-liner.
4. Append it under today's date AND the correct project heading in `~/.claude/daily-log.md`.

## Exact steps

```bash
TODAY=$(date '+%Y-%m-%d')
NOW=$(date '+%H:%M')
LOG_FILE="$HOME/.claude/daily-log.md"
PROJECT_NAME=$(basename "$(pwd)")
```

Then use Python to:
1. Ensure `## YYYY-MM-DD` header exists
2. Ensure `### <project>` subheading exists under today
3. Append the entry under the correct project subheading

If the entry is clearly not about the current project (e.g., user says "log meeting with the design team"
while in a code project), categorize under `General` instead.

## Rules

1. **Keep it short** — one line, max two. Technical and specific.
2. **Categorize correctly** — code work goes under the project name, non-code under `General`.
3. **Don't duplicate** — if the same thing was already logged (check the file), skip it.
4. **Confirm** — after appending, tell the user what was logged and under which project. One line.
5. If `$ARGUMENTS` is empty, ask the user what they want to log.

## Examples

Input: `/log-entry met with team about API redesign`
→ Appended under `### General`:
`- **14:32** 📝 Met with team about API redesign`

Input: `/log-entry researched rate-limiting strategies for the gateway`
→ Appended under `### api-service` (current project):
`- **15:10** 📝 Researched rate-limiting strategies for the gateway`

Input: `/log-entry blocker — waiting on infra for staging DB access`
→ Appended under current project:
`- **16:00** 📝 ⚠️ Blocker — waiting on infra for staging DB access`

Input: `/log-entry pushed frontend onboarding fix` (while in api-service)
→ Appended under `### frontend`:
`- **16:30** 📝 Pushed onboarding fix`
