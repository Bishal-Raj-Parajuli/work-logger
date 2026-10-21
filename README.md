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
  - 09:25 ↳ Added Redis-backed token bucket · edited: src/middleware/rateLimit.ts
- **09:35** 🔴 Session ended — uncommitted: 3 files changed, 41 insertions(+), 6 deletions(-) (1 commit by you today)

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
- **Concurrency-safe** — file locking handles two terminals open in different projects at once, and recovers from a lock left behind by a killed run
- **Manual entries** — `/log-entry <text>` for meetings, blockers, or anything code-only logging would miss
- **Clean export** — `/daily-log copy` emits a per-project summary ready to paste anywhere
- **Local-only** — one markdown file under `~/.claude/`. No network calls, no accounts, no opt-in telemetry
- **Safe installer** — backs up `settings.json` first, refuses to touch it if it isn't valid JSON, and re-running never duplicates hooks

---

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed
- `bash` (the 3.2 that ships with macOS is fine), `python3` 3.8+, `git`
- macOS or Linux (Windows via WSL)

---

## Install

```bash
git clone https://github.com/Bishal-Raj-Parajuli/work-logger.git
cd work-logger
bash install.sh
```

The installer:

1. Copies `scripts/work-logger.sh` to `~/.claude/scripts/`
2. Installs `/log-entry` and `/daily-log` as global Claude Code skills under `~/.claude/skills/`
3. Backs up `~/.claude/settings.json`, then merges three hooks into it (safe to re-run; upgrades older Work Logger hooks in place)
4. Creates the log file (`~/.claude/daily-log.md`, or `$CLAUDE_DAILY_LOG`) if it doesn't already exist

If `settings.json` isn't valid JSON, the installer stops and leaves it untouched so you can fix it first.

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

The skill picks the right project (current directory) or files non-code entries under `General`, then hands the entry to the logger script so it gets the same format and locking as the hooks.

### Review and export

```bash
/daily-log          # show today's log, grouped by project
/daily-log copy     # clean per-project summary, ready to paste
/daily-log <name>   # filter to a single project
/daily-log 2025-01-15  # show an earlier day
```

The `copy` mode strips timestamps and emoji and skips session start/end noise, leaving a digest you can paste into a chat tool, standup doc, or follow-on AI prompt for further formatting.

---

## How It Works

| Hook           | When it fires                          | What it logs                                                      |
| -------------- | -------------------------------------- | ----------------------------------------------------------------- |
| `SessionStart` | You open Claude Code in a project      | Project name, git branch, timestamp                               |
| `Stop`         | After each Claude assistant turn       | First line of Claude's last reply, plus files edited in that turn |
| `SessionEnd`   | The session closes                     | Uncommitted changes (`git diff --shortstat HEAD`) and your commits today |

Each entry is filed under `### <project-name>` inside today's `## YYYY-MM-DD` section. Project name = the name of the session's working directory. The `Stop` summary is read from the session transcript Claude Code passes to the hook.

Concurrent writes from sibling sessions are serialized via a portable `mkdir`-based lock next to the log file (works on macOS and Linux without extra dependencies). The lock records its owner's PID, so a lock left by a killed run is cleared automatically. Each write replaces the log file atomically, so an interrupted hook can't leave it half-written.

The uncommitted-changes figure is the state of the working tree when the session ends, not only what that session changed.

---

## Configuration

### Custom log location

```bash
export CLAUDE_DAILY_LOG="$HOME/Documents/work-log.md"
```

Both the hook script and the two skills honor this variable. Set it in the shell you launch Claude Code from (e.g. your `~/.zshrc`) so the hooks see it too.

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

Work Logger writes only to `~/.claude/daily-log.md` (or `$CLAUDE_DAILY_LOG`) and a transient lock directory next to it (`daily-log.md.lock`). It does not make network calls, send analytics, or share data anywhere. The log file is yours — back it up, encrypt it, or `.gitignore` it as you see fit.

The `Stop` hook records the first line of Claude's last reply (≤150 chars) and the paths of files edited in that turn. If you'd rather not record that, delete the `Stop` block from your `~/.claude/settings.json` after install — `SessionStart` and `SessionEnd` are enough for a basic daily timeline.

---

## Troubleshooting

**Nothing is being logged.**
Check that the hooks are present:

```bash
grep -n work-logger ~/.claude/settings.json
```

You should see three lines (for `SessionStart`, `Stop`, and `SessionEnd`) running `~/.claude/scripts/work-logger.sh`. If not, re-run `bash install.sh`.

**Entries are interleaved or out of order.**
Concurrent sessions take turns through a lock. If a write can't get the lock within ~3 seconds, that one entry is skipped rather than risk overwriting another session's write. Stale locks from killed runs are cleared automatically; if entries keep going missing, check for a leftover `daily-log.md.lock` directory next to your log and delete it.

**Stop entries are blank.**
The hook only writes a Stop line when it can extract a meaningful summary (≥6 chars of reply text, or an edited file). Trivial turns are skipped on purpose to keep the log tidy. To see why a hook did nothing, run it by hand with `WORK_LOGGER_DEBUG=1` set.

**I want to wipe today's log and start clean.**
Open `~/.claude/daily-log.md` in any editor and delete today's `## YYYY-MM-DD` section. The hooks recreate it on the next event.

---

## Contributing

Issues and pull requests are welcome. Before opening a PR:

- Keep the runtime dependency surface minimal — `bash`, `python3`, `git`, and POSIX utilities only
- The installer must stay idempotent (no duplicate hook entries on re-run)
- The log file format is human-readable markdown — preserve the date / project heading shape
- Test on macOS and at least one Linux distro before requesting review

---

## License

MIT. See [LICENSE](LICENSE) for details.
