# Getting started

Bring a fresh macOS host from zero to a working LLM-actuators session. Reading order assumes you have `llms.txt` open in another tab.

The goal is one command: at the end you should be able to launch Claude Code under sandbox, have every binary on `PATH`, and have a switchboard channel ready to wire teammates in.

## Prerequisites

- macOS 14+ (Sonoma or later). The sandbox profile + tmux user-options assume modern macOS.
- Xcode Command Line Tools: `xcode-select --install` (compiler + git + make).
- [Homebrew](https://brew.sh).
- [tmux](https://github.com/tmux/tmux) — `brew install tmux`. `recruit` / `dismiss` and `compact-self` all drive tmux panes.
- [Rust toolchain](https://rustup.rs) — `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`. Most binaries are Rust.
- [Go](https://go.dev/dl/) — required only for `wdb` (the browser bridge is Go).
- [Claude Code](https://docs.anthropic.com/claude/code) — the TUI host that loads the skills, drives the tool calls, and is the runtime for everything else.
- An Anthropic API key for Claude Code (or a Claude.ai login if launching from the macOS app).

## 1. Clone the workspace side-by-side

The cross-repo path-deps in `ddb` / `vdb` (consuming `semantic-schema`) assume a sibling layout. Pick one parent directory and clone there:

> Note: `substrate-distro` is the operator-workspace meta repo that hosts this `llms/` directory and ships scripts that automate the layout below. Cloning it is optional — if you do, its own README walks the rest. The hand-rolled sibling clone is the portable path.

```bash
mkdir -p ~/projects/llm-actuators && cd ~/projects/llm-actuators
git clone https://github.com/llm-actuators/ddb
git clone https://github.com/llm-actuators/idb
git clone https://github.com/llm-actuators/wdb
git clone https://github.com/llm-actuators/vdb
git clone https://github.com/llm-actuators/fdb
git clone https://github.com/llm-actuators/switchboard
git clone https://github.com/llm-actuators/recruit
git clone https://github.com/llm-actuators/tctl
git clone https://github.com/llm-actuators/token-monitor
git clone https://github.com/llm-actuators/compact-self
git clone https://github.com/llm-actuators/claude-sandbox
git clone https://github.com/llm-actuators/skill-router
git clone https://github.com/llm-actuators/substrate
git clone https://github.com/llm-actuators/semantic-schema
```

Cloning into something other than `~/projects/llm-actuators` is fine; the only hard requirement is that `ddb`, `vdb`, and `semantic-schema` are siblings.

## 2. Build + install the binaries

Each repo has its own README with build details. The aggregate one-liner:

```bash
# Rust binaries — installed into ~/.cargo/bin (already on PATH after rustup)
for repo in ddb idb vdb switchboard tctl skill-router substrate; do
  (cd "$repo" && cargo install --path . --locked)
done

# Go binary
(cd wdb && go install ./...)   # lands in $(go env GOPATH)/bin

# Bash binaries — recruit ships recruit + dismiss as one set
install -m 755 recruit/bin/recruit /usr/local/bin/recruit
install -m 755 recruit/bin/dismiss /usr/local/bin/dismiss

# token-monitor + compact-self are tiny helpers — symlink under ~/.claude/bin
mkdir -p ~/.claude/bin
ln -s "$PWD/token-monitor/bin/token-monitor" ~/.claude/bin/token-monitor
ln -s "$PWD/compact-self/bin/compact-self"  ~/.claude/bin/compact-self
```

Confirm `PATH` picks them up:

```bash
which ddb idb wdb vdb switchboard recruit dismiss tctl
```

If any is missing, fix that before continuing — the rest of the chain assumes the binaries are reachable.

## 3. Install the macOS sandbox profile

```bash
cd claude-sandbox && sudo ./install.sh
```

The installer drops the Seatbelt profile at `~/.config/claude-sandbox/claude-sandbox.sb` and the `nosandbox` allowlist at `~/.config/claude-sandbox/nosandbox-allow.conf` (root-owned — agents cannot edit it). After this you launch Claude Code via the `claude-safe` wrapper, which applies the sandbox. Read `claude-sandbox/README.md` for the allowlist semantics.

## 4. Enroll your machine with substrate

`substrate` is the skill-ecosystem CLI. It deploys global (T1) skills + hooks from this repo's store to your live `~/.claude/` paths.

```bash
substrate deploy
```

This wires:

- T1 skills (universal, in `~/.claude/skills/`) — `role-overseer`, `role-builder`, `switchboard`, `device-control-*`, `commit`, etc.
- The `PreToolUse` hook in `~/.claude/settings.json` pointing at `skill-router` with the T1 rules file.
- The `UserPromptSubmit` hook that enforces the token monitor at every prompt (see `~/.claude/CLAUDE.md` Constitution Law I).

To wire a specific company:

```bash
substrate enroll <company>   # inside the repo you want enrolled
```

This adds `.claude/settings.local.json` (PreToolUse with T1+T2 routers), symlinks the company's T2 skills into the repo's `.claude/skills/`, and excludes each symlink in `.git/info/exclude`. The repo's own T3 skills remain tracked. See `substrate/README.md`.

## 5. tmux session shape

Claude Code runs inside a tmux pane so that `recruit` can split off teammates and `compact-self` can `send-keys` `/compact`. The conventional shape:

```
┌──────────────────────────────────────┐
│ overseer (claude-safe)               │  ← your first pane
├──────────────────────────────────────┤
│ builder-XXXX (recruited)             │  ← recruit splits horizontally
└──────────────────────────────────────┘
```

A minimal `~/.tmux.conf` is fine — `recruit` doesn't depend on a custom config. The only requirement is that `$TMUX` and `$TMUX_PANE` are exported inside the pane (always true when launched under tmux).

## 6. Launch the first session

```bash
tmux new -s actuators
claude-safe                   # the sandboxed Claude Code launcher
```

In the Claude Code TUI, at the first prompt:

1. The `UserPromptSubmit` hook fires and asks you to start the token monitor. Follow the directive — copy the parameters into a `Monitor` tool call. Without the monitor, every subsequent prompt nags you.
2. Pick a handle + channel and join switchboard. The handles `operator`, `overseer`, and `everyone` are RESERVED — the binary refuses to claim them. Use a templated handle:
   ```
   export SWITCHBOARD_NAME=builder-$(printf '%04x' $((RANDOM % 65536)))
   export SWITCHBOARD_CHANNEL=global
   switchboard send "$SWITCHBOARD_NAME online"
   ```
   First send auto-emits `kind:"join"`. Read `~/.claude/skills/switchboard/SKILL.md` for the full protocol.
3. If you're working multi-seat, `recruit role-builder` splits a pane and brings up a teammate that auto-joins your channel.

## 7. You are ready when…

Run these one-shots in your Claude Code TUI's Bash tool. Every line should print without error:

```bash
switchboard --version           # wire alive
switchboard channels --all      # at least 'global' should appear
switchboard-ui --version        # TUI client alive (if installed alongside)
ddb devices status              # 0 devices is fine; the binary loading is the point
idb devices status              # same
wdb --help | head -1            # go binary alive
recruit --help | head -1        # bash binary alive
dismiss --help | head -1        # sibling
remote-control --help | head -1 # tmux injector for the Claude iOS app, if shipped
idle-scout --help    | head -1  # idle-seat surveyor invoked by the cadence cron
tctl doctor                     # toolchain self-check (command name verifies in tctl/README)
ls ~/.claude/skills/            # role-* skills present
ls ~/.config/substrate/         # _global + any enrolled companies
```

If any of those fail, jump to the relevant binary's README before continuing. The toolchain assumes the floor is solid.

## What to read next

- [llms.txt](llms.txt) — the elevator pitch + repo index.
- [use-cases.md](use-cases.md) — what to compose when you want to DO something.
- [interactions.md](interactions.md) — how the binaries talk to each other (channels, env vars, tmux user-options).
- [role-layer.md](role-layer.md) — the skill/role layer that decides WHEN to call each binary.
- [device-control.md](device-control.md) — common surface across `ddb` / `idb` / `wdb`.
- [conventions.md](conventions.md) — release discipline shared across all repos.

The toolchain is sterile primitives. The roles + skills layer (under `~/.claude/skills/`) is where intent lives. The next file to read is `role-layer.md`.
