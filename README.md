<p align="center">
  <img src="assets/logo.png" alt="Work Logger logo" width="128" height="128">
</p>

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
- **Manual entries** — `/work-logger:log-entry <text>` for meetings, blockers, or anything code-only logging would miss
- **Clean export** — `/work-logger:daily-log copy` emits a per-project summary ready to paste anywhere
- **Local-only** — one markdown file under `~/.claude/`. No network calls, no accounts, no opt-in telemetry
- **Plain plugin** — installs and uninstalls with Claude Code's plugin manager; nothing edits your `settings.json`

---

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) with plugin support
- `bash` (the 3.2 that ships with macOS is fine), `python3` 3.8+, `git`
- macOS or Linux (Windows via WSL)

---

## Install

This repo is its own plugin marketplace. In Claude Code:

```
/plugin marketplace add Bishal-Raj-Parajuli/work-logger
/plugin install work-logger@work-logger
```

Or from a terminal:

```bash
claude plugin marketplace add Bishal-Raj-Parajuli/work-logger
```

```bash
claude plugin install work-logger@work-logger
```

Logging starts with your next Claude Code session. The log file is created on the first entry.

To try it without installing, load it for one session from a clone:

```bash
claude --plugin-dir /path/to/work-logger
```

### Upgrading from the old `install.sh` version

Earlier versions copied files into `~/.claude/` and added hooks to `~/.claude/settings.json`. Remove those before installing the plugin, or every event will be logged twice:

1. Delete the three hook entries in `~/.claude/settings.json` whose command runs `~/.claude/scripts/work-logger.sh`
2. Delete `~/.claude/scripts/work-logger.sh`, `~/.claude/skills/log-entry/`, and `~/.claude/skills/daily-log/`

Your existing `~/.claude/daily-log.md` is kept and the plugin keeps appending to it.

---

## Usage

### Automatic — just work normally

The hooks fire on their own. `~/.claude/daily-log.md` fills up as you go, grouped by project.

### Manual entries

```
/work-logger:log-entry met with team about API redesign
/work-logger:log-entry blocker — waiting on infra for staging DB access
/work-logger:log-entry shipped onboarding fix to staging
```

You can also just say "log that I met with the design team" — Claude picks up the skill from the request.

The skill picks the right project (current directory) or files non-code entries under `General`, then hands the entry to the logger script so it gets the same format and locking as the hooks.

### Review and export

```
/work-logger:daily-log             # show today's log, grouped by project
/work-logger:daily-log copy        # clean per-project summary, ready to paste
/work-logger:daily-log <name>      # filter to a single project
/work-logger:daily-log 2025-01-15  # show an earlier day
```

The `copy` mode strips timestamps and emoji and skips session start/end noise, leaving a digest you can paste into a chat tool, standup doc, or follow-on AI prompt for further formatting.

---

## How It Works

| Hook           | When it fires                          | What it logs                                                      |
| -------------- | -------------------------------------- | ----------------------------------------------------------------- |
| `SessionStart` | You open Claude Code in a project      | Project name, git branch, timestamp                               |
| `Stop`         | After each Claude assistant turn       | First line of Claude's last reply, plus files edited in that turn |
| `SessionEnd`   | The session closes                     | Uncommitted changes (`git diff --shortstat HEAD`) and your commits today |

Each entry is filed under `### <project-name>` inside today's `## YYYY-MM-DD` section. Project name = the name of the session's working directory. The `Stop` summary is read from the session transcript Claude Code passes to the hook. Turns that only ran `/work-logger:log-entry` or `/work-logger:daily-log` are skipped, so the log doesn't fill up with notes about itself.

Concurrent writes from sibling sessions are serialized via a portable `mkdir`-based lock next to the log file (works on macOS and Linux without extra dependencies). The lock records its owner's PID, so a lock left by a killed run is cleared automatically. Each write replaces the log file atomically, so an interrupted hook can't leave it half-written.

The uncommitted-changes figure is the state of the working tree when the session ends, not only what that session changed.

---

## Configuration

Set these in the shell you launch Claude Code from (e.g. your `~/.zshrc`) so the hooks see them.

| Variable                  | Effect                                                         |
| ------------------------- | -------------------------------------------------------------- |
| `CLAUDE_DAILY_LOG`        | Log file path (default `~/.claude/daily-log.md`). Honored by the hooks and both skills. |
| `WORK_LOGGER_SKIP_STOP=1` | Don't record per-turn summaries; keep only session start/end.  |
| `WORK_LOGGER_DEBUG=1`     | Show errors instead of swallowing them (for troubleshooting).  |

### Disable or remove

Disable for now with `claude plugin disable work-logger@work-logger` (or from `/plugin` in Claude Code), and re-enable with `claude plugin enable work-logger@work-logger`. Remove it with `claude plugin uninstall work-logger@work-logger`. Your log file is left in place.

---

## Repository Layout

```
work-logger/
├── .claude-plugin/
│   ├── plugin.json            # plugin manifest
│   └── marketplace.json       # lets this repo serve as its own marketplace
├── hooks/hooks.json           # SessionStart / Stop / SessionEnd → work-logger.sh
├── scripts/work-logger.sh     # the hook script (also --manual and --show)
└── skills/
    ├── log-entry/SKILL.md     # /work-logger:log-entry
    └── daily-log/SKILL.md     # /work-logger:daily-log
```

---

## Privacy

Work Logger writes only to `~/.claude/daily-log.md` (or `$CLAUDE_DAILY_LOG`) and a transient lock directory next to it (`daily-log.md.lock`). It does not make network calls, send analytics, or share data anywhere. The log file is yours — back it up, encrypt it, or `.gitignore` it as you see fit.

The `Stop` hook records the first line of Claude's last reply (≤150 chars) and the paths of files edited in that turn. If you'd rather not record that, set `WORK_LOGGER_SKIP_STOP=1` — `SessionStart` and `SessionEnd` are enough for a basic daily timeline.

---

## Troubleshooting

**Nothing is being logged.**
Check that the plugin is installed and enabled:

```bash
claude plugin list
```

You should see `work-logger@work-logger` as enabled. In Claude Code, `/hooks` lists the three hooks it adds. Hooks load at session start, so restart any session that was open when you installed.

**Every entry appears twice.**
The old `install.sh` hooks are still in `~/.claude/settings.json`. See [Upgrading from the old `install.sh` version](#upgrading-from-the-old-installsh-version).

**Entries are interleaved or out of order.**
Concurrent sessions take turns through a lock. If a write can't get the lock within ~3 seconds, that one entry is skipped rather than risk overwriting another session's write. Stale locks from killed runs are cleared automatically; if entries keep going missing, check for a leftover `daily-log.md.lock` directory next to your log and delete it.

**Stop entries are blank.**
The hook only writes a Stop line when it can extract a meaningful summary (≥6 chars of reply text, or an edited file). Trivial turns are skipped on purpose to keep the log tidy. To see why a hook did nothing, set `WORK_LOGGER_DEBUG=1`.

**I want to wipe today's log and start clean.**
Open `~/.claude/daily-log.md` in any editor and delete today's `## YYYY-MM-DD` section. The hooks recreate it on the next event.

---

## Contributing

Issues and pull requests are welcome. Before opening a PR:

- Keep the runtime dependency surface minimal — `bash`, `python3`, `git`, and POSIX utilities only
- Run `claude plugin validate --strict .` and check the plugin loads with `claude --plugin-dir .`
- The log file format is human-readable markdown — preserve the date / project heading shape
- Test on macOS and at least one Linux distro before requesting review

---

## License

MIT. See [LICENSE](LICENSE) for details.
