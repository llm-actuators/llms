# Repo map

16 repos. Grouped by role.

## Binaries (13)

The actual tools an LLM session invokes.

| Repo | Lang | What it does |
|---|---|---|
| [ddb](https://github.com/llm-actuators/ddb) | Rust | Android device control. Replaces raw `adb` for tap/swipe/screenshot/install/logcat. Embeds `agent/` (Kotlin AAR — in-app introspection) and `e2e/` (Gradle demo + TC suite). |
| [idb](https://github.com/llm-actuators/idb) | Swift | iOS device control. Drives physical iPhones via WDA (submodule). Embeds `agent/` (Swift SPM library for in-app introspection). |
| [wdb](https://github.com/llm-actuators/wdb) | Go | Browser control via Chrome DevTools Protocol. Element extraction is text-cheap; screenshots are confirmation. |
| [vdb](https://github.com/llm-actuators/vdb) | Rust | Visual debug bridge. Cross-platform drift + region matrix; consumes `semantic-schema`. |
| [fdb](https://github.com/llm-actuators/fdb) | — | Figma frame extraction. Pulls design references for visual QA. |
| [switchboard](https://github.com/llm-actuators/switchboard) | Rust | File-based pub/sub for multi-LLM coordination. One channel = one append-only JSONL log. |
| [recruit](https://github.com/llm-actuators/recruit) | bash | Mid-session teammate spawner. Splits the current tmux window, launches `claude-safe`, pre-exports `SWITCHBOARD_CHANNEL` + `SWITCHBOARD_NAME` so the new pane joins the same wire. Ships `recruit`/`dismiss` (spawn / kill) as a lifecycle set. |
| [tctl](https://github.com/llm-actuators/tctl) | Rust | Toolchain control: preflight → spec → suite pipeline. Also hosts the doc authority (`docs/`). |
| [token-monitor](https://github.com/llm-actuators/token-monitor) | — | Session token usage monitor. Fires milestone alerts before auto-compact wipes the transcript. |
| [compact-self](https://github.com/llm-actuators/compact-self) | — | TUI helper — injects `/compact` into the Claude Code pane via tmux so the agent can compact itself. |
| [claude-sandbox](https://github.com/llm-actuators/claude-sandbox) | — | macOS Seatbelt profile + `nosandbox` escape hatch (root-owned allowlist). |
| [skill-router](https://github.com/llm-actuators/skill-router) | Rust | PreToolUse hook rules engine. Loads TOML rules, blocks/redirects tool calls. |
| [substrate](https://github.com/llm-actuators/substrate) | Rust + bash | Skill ecosystem CLI: `enroll <company>`, `deploy`, `validate`. Fans out skills + hooks to enrolled repos. |

## Libraries (3)

Consumed by the binaries, not invoked directly.

| Repo | Lang | Role |
|---|---|---|
| [semantic-schema](https://github.com/llm-actuators/semantic-schema) | Rust | Data types for the on-device agent payload. Consumed by ddb + vdb. |
| [semantic-agent-flutter](https://github.com/llm-actuators/semantic-agent-flutter) | Dart | In-app debug HTTP agent for Flutter apps. iOS + Android siblings live as subdirs inside `idb/agent/` and `ddb/agent/` respectively. |
| [WebDriverAgent](https://github.com/llm-actuators/WebDriverAgent) | ObjC | Fork of Facebook WDA. Pinned by `idb` as a submodule. |

## Cross-repo dependencies

```
idb  ── submodule ──> WebDriverAgent
ddb  ── path-dep ──> semantic-schema (sibling checkout)
vdb  ── path-dep ──> semantic-schema (sibling checkout)
```

The path-deps assume a `substrate-distro/`-style side-by-side workspace layout where ddb, vdb, and semantic-schema are siblings. Cloning ddb or vdb in isolation will not build until semantic-schema is cloned next to it.

## Archive status (off-org)

Two older repos were absorbed into `ddb` and `idb`:

- `semantic-agent-android` → now lives at `ddb/agent/` (former repo archived)
- `semantic-agent-ios` → now lives at `idb/agent/` (former repo archived)
