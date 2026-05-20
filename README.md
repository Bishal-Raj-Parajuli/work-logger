# Work Logger

**Automatic daily work logging for [Claude Code](https://docs.claude.com/en/docs/claude-code) — across all your projects, with zero effort.**

Work Logger uses Claude Code **lifecycle hooks** to capture what you do as you do it, then writes a clean, human-readable markdown log grouped by **date → project**. No databases, no cloud, no telemetry. Just one file on your machine.

---

## Why?

If you work across multiple Claude Code sessions and projects in a day, end-of-day recall is hard. Sessions don't talk to each other, and git logs only catch what you committed. Work Logger fills the gap with a passive, append-only daily journal.

---

## Example Output

```markdown
## 2025-01-15

### api-service
- **09:15** 🟢 Session started (branch: `feat/rate-limiting`)
  - 09:18 ↳ Edited: src/middleware/rateLimit.ts
  - 09:25 ↳ Added Redis-backed token bucket
- **09:35** 🔴 Session ended — 3 files changed (1 commit today)

### frontend
- **10:00** 🟢 Session started (branch: `main`)
  - 10:12 ↳ Updated auth provider in onboarding flow
- **10:30** 🔴 Session ended

### docs-site
- **19:00** 🟢 Session started (branch: `feature/search`)
  - 19:15 ↳ Wired up FlexSearch index
- **20:00** 🔴 Session ended

### General
- **14:32** 📝 Standup notes synced to team channel
- **16:00** 📝 ⚠️ Blocker — waiting on infra for staging DB access
```

---

## Features

- **Zero-effort capture** — three lifecycle hooks (`SessionStart`, `Stop`, `SessionEnd`) record activity automatically
- **Multi-project aware** — entries group under `### <project>` based on the directory you're working in
- **Concurrency-safe** — file locking handles two terminals open in different projects at once
- **Manual entries** — `/log-entry <text>` for meetings, blockers, or anything code-only logging would miss
- **Clean export** — `/daily-log copy` emits a per-project summary ready to paste anywhere
- **Local-only** — one markdown file under `~/.claude/`. No network calls, no accounts, no opt-in telemetry
- **Idempotent installer** — re-running `install.sh` never duplicates hooks

---

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed
- `bash` 4+, `python3`, `git`
- macOS or Linux (Windows via WSL)

---

## Install

```bash
git clone https://github.com/<your-user>/work-logger.git
cd work-logger
bash install.sh
```

The installer:

1. Copies `scripts/work-logger.sh` to `~/.claude/scripts/`
2. Installs `/log-entry` and `/daily-log` as global Claude Code skills under `~/.claude/skills/`
3. Merges three hooks into `~/.claude/settings.json` (idempotent — safe to re-run)
4. Creates `~/.claude/daily-log.md` if it doesn't already exist

Open Claude Code in any project after install. Logging starts on the next session.

---

## Usage

### Automatic — just work normally

The hooks fire on their own. `~/.claude/daily-log.md` fills up as you go, grouped by project.

### Manual entries

```bash
/log-entry met with team about API redesign
/log-entry blocker — waiting on infra for staging DB access
/log-entry shipped onboarding fix to staging
```

The skill picks the right project (current directory) or files non-code entries under `General`.

### Review and export

```bash
/daily-log          # show today's log, grouped by project
/daily-log copy     # clean per-project summary, ready to paste
/daily-log <name>   # filter to a single project
```

The `copy` mode strips timestamps and emoji and skips session start/end noise, leaving a digest you can paste into a chat tool, standup doc, or follow-on AI prompt for further formatting.

---

## How It Works

| Hook           | When it fires                          | What it logs                                                      |
| -------------- | -------------------------------------- | ----------------------------------------------------------------- |
| `SessionStart` | You open Claude Code in a project      | Project name, git branch, timestamp                               |
| `Stop`         | After each Claude assistant turn       | One-line summary (file edited, or first line of the last reply)   |
| `SessionEnd`   | The session closes                     | `git diff --stat HEAD` summary and today's commit count           |

Each entry is filed under `### <project-name>` inside today's `## YYYY-MM-DD` section. Project name = `basename "$(pwd)"`. Concurrent writes from sibling sessions are serialized via a portable `mkdir`-based lock (works on macOS and Linux without extra dependencies).

---

## Configuration

### Custom log location

```bash
export CLAUDE_DAILY_LOG="$HOME/Documents/work-log.md"
```

Both the hook script and the two skills honor this variable.

### Disable temporarily

In Claude Code: `/hooks` → find the work-logger entries → disable.

To remove permanently, delete the matching entries from `~/.claude/settings.json` under the `hooks` key and remove `~/.claude/scripts/work-logger.sh`.

---

## File Layout (after install)

```
~/.claude/
├── scripts/
│   └── work-logger.sh             # the hook script
├── skills/
│   ├── log-entry/SKILL.md         # /log-entry command
│   └── daily-log/SKILL.md         # /daily-log command
├── daily-log.md                   # your daily log (auto-created, append-only)
└── settings.json                  # hooks merged here (safely)
```

---

## Privacy

Work Logger writes only to `~/.claude/daily-log.md` (or `$CLAUDE_DAILY_LOG`) and a transient lock file at `/tmp/work-logger.lock`. It does not make network calls, send analytics, or share data anywhere. The log file is yours — back it up, encrypt it, or `.gitignore` it as you see fit.

The `Stop` hook may include the first non-heading line of Claude's last reply (≤150 chars). If you'd rather not record that, delete the `Stop` block from your `~/.claude/settings.json` after install — `SessionStart` and `SessionEnd` are enough for a basic daily timeline.

---

## Troubleshooting

**Nothing is being logged.**
Check that the hooks are present:

```bash
jq '.hooks' ~/.claude/settings.json
```

You should see entries for `SessionStart`, `Stop`, and `SessionEnd` pointing at `~/.claude/scripts/work-logger.sh`. If not, re-run `bash install.sh`.

**Entries are interleaved or out of order.**
Concurrent sessions race for the lock. Under heavy load the script falls through after a ~5-second retry budget rather than blocking the hook past its timeout. If you see corruption, check that no other process is holding `${TMPDIR:-/tmp}/work-logger.lock.d`.

**Stop entries are blank.**
The hook only writes a Stop line when it can extract a meaningful summary (file edit path, or ≥6 chars of assistant text). Trivial turns are skipped on purpose to keep the log tidy.

**I want to wipe today's log and start clean.**
Open `~/.claude/daily-log.md` in any editor and delete today's `## YYYY-MM-DD` section. The hooks recreate it on the next event.

---

## Contributing

Issues and pull requests are welcome. Before opening a PR:

- Keep the runtime dependency surface minimal — `bash`, `python3`, `git`, and POSIX utilities only
- The installer must stay idempotent (no duplicate hook entries on re-run)
- The log file format is human-readable markdown — preserve the date / project heading shape
- Test on macOS and at least one Linux distro before requesting review

See `_context/` (if present in your clone) for the project's internal context files used during AI-assisted development.

---

## License

MIT. See [LICENSE](LICENSE) for details.
