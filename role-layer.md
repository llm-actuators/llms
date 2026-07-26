# Role layer

The llm-actuators binaries are sterile primitives. They don't decide WHEN to tap a button, WHEN to recruit a builder, or WHEN to wire a checkpoint. That decision belongs in the **skill / role layer** — operator-side files under `~/.claude/skills/` that pin behavior onto a Claude Code session.

This file is the bridge: it tells a cold LLM session opening the toolchain that there is a second layer above the binaries, where it lives, and how to reach it.

## Why roles aren't in this repo

The binaries are public, sterile, and shared. The roles are private, mutable, and machine-local. Mixing them would leak operator-specific patterns (cron cadence, doctrine §-numbers, company configs) into a tool repo where they don't belong, and make the binaries less portable.

The split:

| Layer | Location | What it holds |
|---|---|---|
| Binaries | `llm-actuators/*` (this org) | Sterile primitives. No domain verbs. |
| Skills + roles | `~/.claude/skills/` (yadm-tracked) | Behavioral pins — when to call which primitive. |
| Company configs | `~/.config/substrate/<company>/` | Secrets, router rules, todo files. |
| Project enrollment | `<repo>/.claude/settings.local.json` | Per-repo hook + T2 skill symlinks. |

A binary that knew about role doctrine would be unshippable to a different operator. The split keeps every layer self-contained.

## The canonical role catalogue

These are the roles the operator currently maintains under `~/.claude/skills/role-*/`. Each lives in its own directory with a `SKILL.md` (activation prompt + arming sequence) and a sibling `doctrine.md` (universal behaviors).

| Role | When to load | Doctrine length |
|---|---|---|
| `role-overseer` | Multi-seat session — coordinator + adversarial reviewer in one. Owns scope enforcement, dispatch, verify-before-trust. Arms token monitor + global switchboard subscription + cadence cron + doctrine-reload cron + idle-seat cron. | 41+ numbered sections |
| `role-builder` | Implementation seat — writes production code, runs tests, ships SHAs. Reports up via switchboard to the overseer. Compactable + dismissable (per overseer doctrine §41). |
| `role-enforcer` | Standalone adversarial review seat — used when an independent verifier is wanted on top of an overseer. Tests, reviews, signs off — never builds. (Subsumed into `role-overseer` for 2-seat arrangements.) |
| `role-janitor` | Infrastructure housekeeping — storage, sims/emulators/devices, AVDs, repos, worktrees. Cross-cutting; works the `janitor` switchboard channel. |
| `role-custodian` | Cross-platform parity caretaker — keeps iOS/Android implementations aligned, surfaces drift. Manual oversight seat for parity-sensitive work. |
| `role-substrate-dev` | Substrate development seat — auto-loaded when cwd is the substrate repo itself. Knows the deploy/enroll internals; safe to mutate the skill store. |
| `role-tdd-zealot` | Auto-activates on testing keywords. TDD spec authorship + adversarial test design. Enforces failing-fixture-first discipline. |

To enumerate what's actually on this machine: `ls -d ~/.claude/skills/role-*/`. Roles like `toolmaker`, `recruiter`, `operator` show up as switchboard *handles* but are not separate skills — they are recruited builders with project context, or the human operator themselves.

Project-specific doctrine lives in sibling ref files (`doctrine-<company>.md`), one per enrolled company. Each role's `SKILL.md` instructs the cron to re-read `doctrine.md` + the active project doctrine on a twice-hourly cadence (`17,47 * * * *`). Doctrine grows; the refresh keeps behavior anchored.

## Skill / hook tiers

Substrate manages three tiers of skills (see `substrate/README.md`):

| Tier | Location | Scope | Examples |
|---|---|---|---|
| T1 | `~/.claude/skills/` (deployed by `substrate deploy` from this repo's store) | Universal — every machine, every repo | `switchboard`, `device-control-*`, `role-*`, `commit` |
| T2 | `~/.config/substrate/<company>/.claude/skills/` (symlinked into enrolled repos) | Per-company — CI/CD, Jira/Linear, Firebase, brand setup | `gl-pipeline-status`, `jira-rft-skill`, `<company>-brands` |
| T3 | `<repo>/.claude/skills/` (tracked in the repo) | Per-repo — build, logcat, brand setup | `<app>-build`, `<app>-fixtures` |

A repo gets enrolled with `substrate enroll <company>` — that writes `.claude/settings.local.json` with the skill-router PreToolUse hook pointing at T1 + T2 rules.toml, individually symlinks the T2 skills into `.claude/skills/`, and excludes each symlink in `.git/info/exclude` so they stay un-tracked but operational. The repo's own T3 skills remain tracked. Never exclude the whole `.claude/skills/` directory — T3 must survive.

## The skill router

`skill-router` is a `PreToolUse` hook (Rust). On every tool call:

1. Loads T1 + T2 `rules.toml` files (paths configured in `settings.local.json`).
2. Matches the tool call against rule patterns (command regex, MCP server, etc.).
3. Decides: `allow`, `block`, or `redirect`. Block returns a hint message naming the skill to load.

Rule format is documented in `skill-router/README.md`. The contract for an LLM is simpler: when you see `BLOCKED. Invoke the Skill tool with skill="<name>" BEFORE retrying`, you call `Skill(skill="<name>")` and retry. The skill loads context and grants bypass for the duration. No workarounds — see Constitution Law II.

## The sandbox + nosandbox

`claude-sandbox` runs Claude Code under a macOS Seatbelt profile. Most tools work fine inside the sandbox. The ones that don't (anything that internally calls `sandbox-exec` — `xcodebuild`, `swift`, `idb wda build`, `cargo`, `gradlew`) need the `nosandbox` escape hatch.

`nosandbox` dispatches the command via `tmux run-shell -b` (the tmux server predates the sandbox, so its children inherit none). It enforces an **allowlist** at `~/.config/claude-sandbox/nosandbox-allow.conf` — root-owned, agents cannot edit. `bash` is explicitly excluded; never wrap commands as `nosandbox bash -c ...`. See `claude-sandbox/README.md`.

## Mental model

```
┌───────────────────────────────────────────────────────────────────┐
│                      Claude Code TUI session                      │
│             (the LLM is the runtime — tools are effectors)        │
└───┬────────────────────┬─────────────────────┬────────────────────┘
    │                    │                     │
    ▼                    ▼                     ▼
┌───────────┐    ┌─────────────────┐   ┌──────────────────┐
│  Skills   │    │  Hooks (router) │   │ Tool calls       │
│ ~/.claude │    │ skill-router    │   │ Bash, Read, etc. │
│ /skills/  │    │ PreToolUse      │   │                  │
└─────┬─────┘    └────────┬────────┘   └─────────┬────────┘
      │                   │                      │
      │ activation        │ rule match           │ direct invocation
      ▼                   ▼                      ▼
┌──────────────────────────────────────────────────────────────┐
│                  llm-actuators binaries                      │
│   ddb │ idb │ wdb │ vdb │ fdb │ switchboard │ recruit │ ...  │
└──────────────────────────────────────────────────────────────┘
      ▲                                          ▲
      │                                          │
      │ company config                           │ project-side
┌─────┴────────────────────────┐    ┌────────────┴────────────┐
│ ~/.config/substrate/<co>/   │    │ <repo>/.claude/         │
│  rules.toml │ secrets │ ... │    │  settings.local.json    │
└──────────────────────────────┘    └─────────────────────────┘
```

The LLM is the runtime. Skills tell it what role it's playing. The router enforces that contract on every tool call. The binaries are the hands.

## Cold-session orientation

If you opened a Claude Code session and have no idea which role you are:

1. Check `~/.claude/skills/` — what `role-*/` directories exist on this machine? The operator's intent for this session is one of those.
2. Look at the system reminder injected at the top of the conversation — Claude Code lists the active skills there. The role skill (if loaded) will appear.
3. If no role is loaded, you are running unscoped. The Constitution + the user's CLAUDE.md are still binding; nothing in the role layer pins additional cron arms, monitors, or doctrine.
4. To adopt a role mid-session, the operator types `/role-overseer` (or whichever role) — Claude Code invokes the corresponding `Skill` tool. The skill's `SKILL.md` is the activation prompt — including the arming sequence (token monitor, switchboard streams, crons).

The doctrine is dense. Trust the cron's hourly re-read more than your memory of section numbers. When you drift, re-read the doctrine file in the active role.

## What to read next

- `~/.claude/skills/role-overseer/SKILL.md` + `doctrine.md` — the most-developed role.
- `substrate/README.md` — the enrollment CLI in detail.
- `skill-router/README.md` — rule format + decision semantics.
- [use-cases.md](use-cases.md) → scenario 8 — "Cold session lands on a company-enrolled repo".
