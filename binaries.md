# Binaries

One paragraph each. For invocation details, see each tool's README (linked).

The org holds **34 public repos** = 32 toolchain/content repos + 2 org-infra. This file covers the invokable binaries. Some repos ship more than one binary (`switchboard` → `switchboard-ui`; `recruit` → `dismiss` + `idle-scout`; `workflows` → `gate`); those are grouped under their parent repo below.

## Device control

### `ddb` — Android Device Debug Bridge
Replaces raw `adb` for everything an LLM session does to an Android device: tap, swipe, scroll, type, press back/home, launch/kill apps, screenshot, clear data, install APKs, push/pull files, logcat. Output is parseable. Doctrine: never let the LLM call `adb shell input` directly — always go through `ddb`. Source: [ddb/README.md](https://github.com/llm-actuators/ddb#readme).

### `idb` — iOS Device Debug Bridge
Same surface as `ddb` for physical iPhones. Drives WebDriverAgent under the hood. Includes `idb wda build`/`start` for managing the WDA lifecycle. Source: [idb/README.md](https://github.com/llm-actuators/idb#readme).

### `wdb` — Web Debug Bridge
Browser control via Chrome DevTools Protocol. 33 commands covering click, type, fill, navigate, frames, cookies, storage, drag, assert, log. DOM element extraction is the primary feedback channel (cheap); screenshots are for visual confirmation only. Source: [wdb/README.md](https://github.com/llm-actuators/wdb#readme).

The three follow a [common surface](device-control.md) — same verbs, same flag style, same exit code semantics. Any LLM session that knows one knows the others.

### `device-claim` — atomic per-device mutex
A file-based lease that lets multiple autonomous agent sessions share a pool of physical phones, simulators, and emulators without two of them driving the same device at once. Wraps `O_CREAT|O_EXCL` for the atomic acquire, then layers TTL expiry, heartbeat, and PID-liveness so a claim is released promptly whether its owner exits cleanly, crashes, or wedges. Wired transparently into `ddb`/`idb` on mutating subcommands. Source: [device-claim/README.md](https://github.com/llm-actuators/device-claim#readme).

### `resources` — device/sim/emulator inventory
A read-only enumerator that snapshots every enrolled and live iOS/Android device, simulator, and emulator on a host, plus any in-flight claim events from the coordination bus. Answers "what devices exist, which are booted, and who's holding what" in one command, so multiple agents don't collide on the same physical device. Source: [resources/README.md](https://github.com/llm-actuators/resources#readme).

## Visual + design

### `vdb` — Visual Debug Bridge
Cross-platform visual regression and drift detection. Consumes the `semantic-schema` payload from on-device agents. Produces region matrices for visual QA. Source: [vdb/README.md](https://github.com/llm-actuators/vdb#readme).

### `fdb` — Figma Debug Bridge
Extracts frames from Figma via the REST API (or an offline JSON dump) and emits YAML in the same schema that `ddb` / `idb` produce from live devices — that's what makes the cross-platform drift pipeline work. Pairs with `vdb` for design-vs-built comparisons. Source: [fdb/README.md](https://github.com/llm-actuators/fdb#readme).

## Coordination

### `switchboard` — multi-LLM wire
File-based pub/sub. Channels are append-only JSONL logs under `~/Library/Caches/switchboard/<channel>/`. One cursor per subscriber. Flock-based presence; reserved sentinels `everyone` (broadcast) and empty `--to` (journal write, log-only, never matches `--mine`).

4-layer state model per directed message: `delivered` (auto-emitted by recipient stream on metadata-surface) → `read` (auto-emitted by `switchboard read --id <X>` when the agent fetches the body) → `ack` (manual, with subject — `👀 working`, `thinking`, or bare `closed loop`) → `message` (the final reply, `--reply-to <id>`). Stream output is metadata-only by default — agents must `read --id` to fetch the body. Use `stream --full` for local TUIs.

Subjects with shell-active punctuation (`; & | ` ` `$()`) trip the harness Bash matcher; sidestep via `--subject-file <path>` (write the subject to a file first). Same flag exists on `ack` and `brick`. Prefix-id lookup ≥6 chars works everywhere an id is accepted. Source: [switchboard/README.md](https://github.com/llm-actuators/switchboard#readme), [switchboard/WIRE.md](https://github.com/llm-actuators/switchboard/blob/main/WIRE.md).

### `switchboard-ui` — ratatui chat client over switchboard
Three-pane TUI (channels / chat / input) that the operator and overseers use to read the wire interactively. Multi-channel persistent flocks (one stream subprocess per channel ever visited; toggling channels never releases a flock). Ctrl+S toggles the channel sidebar; Ctrl+E sticks `@everyone` on every send; Ctrl+V pastes an image from the clipboard as `[[img:/tmp/sb-images/...]]`; Tab cycles @-autocomplete, Enter accepts. Renders `delivered` → `read` → `ack` receipts under each message in chronological order. Lives in the switchboard repo as a second binary target. Source: [switchboard/src/bin/chat.rs](https://github.com/llm-actuators/switchboard/blob/main/src/bin/chat.rs).

### `recruit` — mid-session teammate spawner
Splits the current tmux window with a new pane, launches `claude-safe --model <model>`, and pre-exports `SWITCHBOARD_CHANNEL` + `SWITCHBOARD_NAME` so the spawned agent joins the recruiter's wire immediately. Counterpart to `switchboard` — switchboard carries the messages, recruit calls in the seats. Tags the new pane with tmux user-options (`@recruit-handle`, `@recruit-channel`, `@recruit-role`) so its lifecycle partner `dismiss` can find it later. Use mid-session when an overseer decides they need a builder/enforcer/QA without tearing down the existing tmux layout. Source: [recruit/README.md](https://github.com/llm-actuators/recruit#readme).

### `dismiss` — terminate a recruited seat
Counterpart to `recruit`, lives in the same repo. Finds the tmux pane tagged with the given handle, kills it (hard by default, `--soft` opt-in sends `/quit` first with a grace window), and emits `kind:leave` on the dismissed handle's behalf so peers see a clean exit instead of a stale-peer timeout. Source: [recruit/README.md](https://github.com/llm-actuators/recruit#readme).

### `idle-scout` — cull-candidate scanner (cron-fired)
Per-overseer cron (`3-58/5 * * * *` — off-minute so the cohort doesn't all collide on :00/:05/:10). Walks `recruit`-tagged tmux panes, queries switchboard for each handle's last `kind:message`, and posts a directed cull-candidate list to `--report-channel` (default `idle-scout`), `--to` the overseer that armed it. Silent when no candidate crosses the threshold (default 10 min idle). Pairs with `dismiss` — the overseer reads the list, decides to dismiss or assign work. Lives in the recruit repo.

## Session management

### `token-monitor` — context-loss alarm
Watches the session's token usage and fires milestone alerts (60/70/75/80/85/90% of the model's context window). Not a token-cost cop — a brain-save alarm. The 90% alert is critical: compact deliberately before auto-compact wipes the transcript at an arbitrary moment. Source: [token-monitor/README.md](https://github.com/llm-actuators/token-monitor#readme).

### `compact-self` — self-triggered compaction
A small helper that injects `/compact` into the Claude Code TUI via tmux. Lets the agent compact itself when `token-monitor` fires the 90% alert, instead of asking the human to type the command. Source: [compact-self/README.md](https://github.com/llm-actuators/compact-self#readme).

### `remote-control` — expose this session to the Claude iOS app
A small helper that injects `/remote-control` into the current tmux pane so the operator can drive the session from the Claude iOS app over their VPN. Mirrors `compact-self`'s pattern (tmux send-keys + delay). Source: [remote-control/README.md](https://github.com/llm-actuators/remote-control#readme).

## Fleet-ops

### `burn` — fleet-wide token-burn aggregator
Reads the per-session transcripts of every agent seat under a Claude Code projects root, sums each session's context tokens, and emits a single GREEN/AMBER/RED level as JSON. Where `token-monitor` watches one seat, `burn` answers "is any seat close to the edge?" across the whole fleet. It surfaces data and never blocks — the reporter tells you the level; you decide. Source: [burn/README.md](https://github.com/llm-actuators/burn#readme).

### `actuators-doctor` — fleet health check
A fleet-wide health check for the org's binaries: tiered checks for existence, version, and PATH-shadowing, with a pretty TTY mode, `--json` (JSONL, one finding per line), and `--strict` (exit 1 on any WARN/FAIL). Answers "is my toolchain installed and consistent?" in one command. Source: [actuators-doctor/README.md](https://github.com/llm-actuators/actuators-doctor#readme).

### `fleet-tui` — read-only fleet dashboard
A read-only terminal dashboard for a multi-agent fleet. Shows a sidebar of every project bucket, which seats are live and what they're doing (from `switchboard status`), and each project's tracked todos with their verification state. It only reads — it never posts to the wire or edits the ledger. Source: [fleet-tui/README.md](https://github.com/llm-actuators/fleet-tui#readme).

### `todo` — priority-aware task ledger
A single-binary CLI that stores a shared, per-project todo list as human-readable Markdown, with priority buckets, a parked backlog tier, salience scoring, and an append-only audit log. Built so a cohort of autonomous agents (and a human operator) can coordinate cross-session work over a file that stays greppable and hand-editable. The binary is a sterile primitive — it records facts, it does not decide policy. Source: [todo/README.md](https://github.com/llm-actuators/todo#readme).

## Workflow enforcement

### `gate` — workflow state-machine enforcer (in `workflows`)
A single Rust binary that tracks workflow phase, validates required artifacts at each transition, and blocks invalid tool invocations — driven by declarative YAML and wired into Claude Code's `PreToolUse`/`PostToolUse`/`Stop` hooks. It makes behavioral rules mechanical: instead of trusting an agent to "design before coding" or "test before committing," the rule is a state machine the agent cannot silently skip. `gate` is a binary inside the `workflows` repo, not a standalone repo. Source: [workflows/README.md](https://github.com/llm-actuators/workflows#readme).

## Toolchain control

### `tctl` — preflight → spec → suite
The orchestrator. Runs preflight checks, validates test specifications, executes suites. Also the doc authority for the whole toolchain — `tctl/docs/` holds ADRs, the tech-debt ledger (180+ entries), epics, gaps, getting-started. Source: [tctl/README.md](https://github.com/llm-actuators/tctl#readme), [tctl/docs/llms.txt](https://github.com/llm-actuators/tctl/blob/main/docs/llms.txt).

## Infrastructure

### `skill-router` — PreToolUse hook engine
Loads TOML rule files and decides whether to block, allow, or redirect a tool call. Used to enforce "always go through `/commit` skill, never raw `git commit`" style invariants. Source: [skill-router/README.md](https://github.com/llm-actuators/skill-router#readme).

### `substrate` — skill ecosystem CLI
The harness manager. `substrate enroll` wires a repo (settings.local.json + symlinks + git excludes). `substrate deploy` fans skills + hooks out to all enrolled repos. `substrate validate` runs the invariant checks. Source: [substrate/README.md](https://github.com/llm-actuators/substrate#readme), [substrate/llms.txt](https://github.com/llm-actuators/substrate/blob/main/llms.txt).

### `claude-sandbox` — macOS Seatbelt + escape hatch
Runs Claude Code under a macOS Seatbelt sandbox profile. Provides `nosandbox <cmd>` for tools that fail under the sandbox (xcodebuild, swift, idb, cargo, gradlew). The allowlist is root-owned — agents cannot modify it. Source: [claude-sandbox/README.md](https://github.com/llm-actuators/claude-sandbox#readme).
