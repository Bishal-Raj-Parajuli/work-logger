---
name: log-entry
description: >
  Manually add an entry to the daily work log (~/.claude/daily-log.md, or
  $CLAUDE_DAILY_LOG). Use when doing non-code work that the automatic work
  logger won't capture: meetings, emails, research, decisions, blockers, or
  anything you want in your daily summary. Trigger on "/log-entry", "/log",
  "log this", "add to daily log", or when the user says something like
  "note that I did X" or "record that".
argument-hint: <what you did>
allowed-tools: Bash
---

# /log-entry — Manual Daily Log Entry

Append a manual entry to the daily work log. The log file lives at
`$CLAUDE_DAILY_LOG` if set, otherwise `~/.claude/daily-log.md`.

## When this fires

The user types `/log-entry <description>` or says "log this", "add to my log", etc.
The `$ARGUMENTS` variable contains what they want logged.

## What to do

1. Read `$ARGUMENTS` — this is the raw description of what to log.
   If it is empty, ask the user what they want to log and stop.
2. Determine which **project** this belongs to:
   - Code work in the current directory → the current directory's name (`basename "$(pwd)"`).
   - Clearly about a different project → that project's name.
   - Not about code (meeting, email, admin) → `General`.
3. Clean it up into a concise, specific one-liner. Keep a leading `⚠️` for blockers.
4. Check today's log for an entry that already says the same thing:

   ```bash
   bash ~/.claude/scripts/work-logger.sh --show
   ```

   If it is already there, tell the user and stop.
5. Append it with the logger script. Pass the project as the argument and the
   text on stdin through a quoted heredoc, exactly like this, so quotes, `$`,
   and backticks in the text are never interpreted by the shell:

   ```bash
   bash ~/.claude/scripts/work-logger.sh --manual "General" <<'ENTRY'
   Met with team about API redesign
   ENTRY
   ```

   The script adds the timestamp and `📝` marker, files the line under today's
   date and the project heading, and handles locking. Do not edit the log file
   directly.
6. Confirm in one line what was logged and under which project.

## Examples

Input: `/log-entry met with team about API redesign`
→ project `General`, text `Met with team about API redesign`
→ logged as `- **14:32** 📝 Met with team about API redesign`

Input: `/log-entry researched rate-limiting strategies for the gateway` (in `api-service`)
→ project `api-service`, text `Researched rate-limiting strategies for the gateway`

Input: `/log-entry blocker — waiting on infra for staging DB access`
→ current project, text `⚠️ Blocker — waiting on infra for staging DB access`

Input: `/log-entry pushed frontend onboarding fix` (while in `api-service`)
→ project `frontend`, text `Pushed onboarding fix`
