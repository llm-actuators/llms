# Repo map

**34 public repos** = 32 toolchain/content repos + 2 org-infra (`.github` = org profile, `llm-actuators.github.io` = Pages landing site). Grouped by role below; the groups sum to 34.

## Binaries (21)

The actual tools an LLM session invokes. Some repos ship more than one binary (noted inline).

| Repo | Lang | What it does |
|---|---|---|
| [ddb](https://github.com/llm-actuators/ddb) | Rust | Android device control. Replaces raw `adb` for tap/swipe/screenshot/install/logcat. Embeds `agent/` (Kotlin AAR — in-app introspection) and `e2e/` (Gradle demo + TC suite). |
| [idb](https://github.com/llm-actuators/idb) | Swift | iOS device control. Drives physical iPhones via WDA. Embeds `agent/` (Swift SPM library for in-app introspection). |
| [wdb](https://github.com/llm-actuators/wdb) | Go | Browser control via Chrome DevTools Protocol. Element extraction is text-cheap; screenshots are confirmation. |
| [vdb](https://github.com/llm-actuators/vdb) | Rust | Visual debug bridge. Cross-platform drift + region matrix; consumes `semantic-schema`. |
| [fdb](https://github.com/llm-actuators/fdb) | Rust | Figma frame extraction into the shared semantic schema. Pairs with `vdb` for design-vs-built comparison. |
| [switchboard](https://github.com/llm-actuators/switchboard) | Rust | File-based pub/sub for multi-LLM coordination. One channel = one append-only JSONL log. Also ships `switchboard-ui` (ratatui chat client) as a second binary target. |
| [recruit](https://github.com/llm-actuators/recruit) | Shell | Mid-session teammate spawner. Splits the current tmux window, launches `claude-safe`, and joins the recruiter's switchboard wire. Ships `dismiss` (kill a seat) and `idle-scout` (cron cull-candidate scanner) as companion binaries. |
| [tctl](https://github.com/llm-actuators/tctl) | Rust | Toolchain control: preflight → spec → suite pipeline. Also hosts the doc authority (`docs/`). |
| [token-monitor](https://github.com/llm-actuators/token-monitor) | Shell | Session token usage monitor. Fires milestone alerts before auto-compact wipes the transcript. |
| [compact-self](https://github.com/llm-actuators/compact-self) | Shell | Injects `/compact` into the Claude Code pane via tmux so the agent can compact itself. |
| [remote-control](https://github.com/llm-actuators/remote-control) | Shell | Injects `/remote-control` into the pane so the operator can drive the session from the Claude iOS app over VPN. |
| [burn](https://github.com/llm-actuators/burn) | Rust | Fleet-wide token-burn aggregator. Sums every seat's context tokens and emits one GREEN/AMBER/RED level as JSON. |
| [todo](https://github.com/llm-actuators/todo) | Rust | Priority-aware Markdown task ledger for agent cohorts. The Markdown is the database; the binary parses, mutates, re-renders. |
| [device-claim](https://github.com/llm-actuators/device-claim) | Rust | Atomic per-device mutex. Atomic acquire + TTL + heartbeat + PID-liveness so concurrent sessions never drive the same device. |
| [resources](https://github.com/llm-actuators/resources) | Rust | Read-only device/sim/emulator inventory across the fleet, plus in-flight claim events. |
| [actuators-doctor](https://github.com/llm-actuators/actuators-doctor) | Rust | Fleet-wide health check for the org's binaries: existence, version, PATH-shadowing, with tiered/JSON/strict modes. |
| [fleet-tui](https://github.com/llm-actuators/fleet-tui) | Rust | Read-only terminal dashboard for a multi-agent fleet — companies, in-flight seats, and per-project todos. Never posts or edits. |
| [workflows](https://github.com/llm-actuators/workflows) | Shell | Ships the `gate` binary: a workflow state-machine enforcer wired into Claude Code `PreToolUse`/`PostToolUse`/`Stop` hooks, driven by declarative YAML. |
| [skill-router](https://github.com/llm-actuators/skill-router) | Rust | PreToolUse hook rules engine. Loads TOML rules, blocks/redirects tool calls. |
| [substrate](https://github.com/llm-actuators/substrate) | Rust | Skill ecosystem CLI: `enroll`, `deploy`, `validate`. Fans out skills + hooks to enrolled repos. |
| [claude-sandbox](https://github.com/llm-actuators/claude-sandbox) | Shell | macOS Seatbelt profile + `nosandbox` escape hatch (root-owned allowlist). |

## Libraries (6)

Consumed by the binaries or wired into the harness, not invoked as standalone device/session tools.

| Repo | Lang | Role |
|---|---|---|
| [semantic-schema](https://github.com/llm-actuators/semantic-schema) | Rust | The canonical semantic-UI wire format that ties `ddb`, `idb`, `fdb`, and `vdb` together. Pure library crate. |
| [semantic-agent-flutter](https://github.com/llm-actuators/semantic-agent-flutter) | Dart | Debug-only in-process HTTP agent for Flutter apps. Sibling of the Android (Kotlin) and iOS (Swift) agents inside `ddb/agent/` and `idb/agent/`. |
| [idle-work](https://github.com/llm-actuators/idle-work) | Rust | Checkpoint + interrupt-safety primitives for long background jobs (plus a thin `idle-checkpoint` CLI). Atomic state persistence, SIGTERM-safe boundaries, chunk journaling. |
| [fleet-tooling](https://github.com/llm-actuators/fleet-tooling) | Python | Read-only fleet observability & background-work helpers: a JSON aggregator + digest renderer, a git-worktree lifecycle helper, and a review-report generator. Reads its project topology from config. |
| [hooks](https://github.com/llm-actuators/hooks) | Shell | Enforcement-architecture showcase: two benign runnable lifecycle-hook templates (session-start, post-compact) plus an ARCHITECTURE.md documenting the enforcement-hook design, abstracted. |
| [specs](https://github.com/llm-actuators/specs) | — | Example TOML config schemas for the toolchain's soft-gates. Each `*.toml.example` is a self-documenting, versioned template. |

## iOS components (3)

iOS-specific support repos for the device-control and coordination surfaces.

| Repo | Lang | Role |
|---|---|---|
| [WebDriverAgent](https://github.com/llm-actuators/WebDriverAgent) | ObjC | Fork of the Facebook WDA — the on-device XCUITest automation server iOS control drives. |
| [device-control-ios](https://github.com/llm-actuators/device-control-ios) | — | Support repo that pins and vendors the exact WDA revision `idb` builds against, so device automation is reproducible across machines. |
| [switchboard-ios](https://github.com/llm-actuators/switchboard-ios) | Swift | Native SwiftUI iOS client for the switchboard wire; talks to a Mac running the `switchboard` CLI over SSH. |

## Docs + specs (2)

| Repo | Lang | Role |
|---|---|---|
| [llms](https://github.com/llm-actuators/llms) | Mermaid | This repo — the LLM-first documentation set and architecture diagrams for the whole org. |
| [visual-qa](https://github.com/llm-actuators/visual-qa) | — | Cross-platform device-automation framework overview: semantic agents extract the full UI tree, YAML specs drive devices, results aggregate into a matrix — properties, not screenshots, are the source of truth. |

## Org-infra (2)

Not toolchain content — GitHub org plumbing.

| Repo | Lang | Role |
|---|---|---|
| [.github](https://github.com/llm-actuators/.github) | HTML | Org profile (the landing README shown on the org page). |
| [llm-actuators.github.io](https://github.com/llm-actuators/llm-actuators.github.io) | HTML | GitHub Pages landing site for the org. |

## Cross-repo dependencies

```
idb  ── pins ──────> device-control-ios ── submodule ──> WebDriverAgent
ddb  ── path-dep ──> semantic-schema (sibling checkout)
vdb  ── path-dep ──> semantic-schema (sibling checkout)
fdb  ── consumes ──> semantic-schema (shared wire format)
```

The path-deps assume a `substrate-distro/`-style side-by-side workspace layout where `ddb`, `vdb`, and `semantic-schema` are siblings. Cloning `ddb` or `vdb` in isolation will not build until `semantic-schema` is cloned next to it.

## Archive status (off-org)

Two older repos were absorbed into `ddb` and `idb`:

- `semantic-agent-android` → now lives at `ddb/agent/` (former repo archived)
- `semantic-agent-ios` → now lives at `idb/agent/` (former repo archived)
