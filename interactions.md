# Interactions

Cross-binary contracts. Each tool listed in [binaries.md](binaries.md) is sterile on its own — the value emerges from how they compose. This file is the edge list. Read [use-cases.md](use-cases.md) for goal-driven workflows; this file is the underlying graph.

## At a glance

```
                       ┌──────────────────┐
                       │  token-monitor   │  watches context window
                       └────────┬─────────┘
                                │ 90% alert
                                ▼
                       ┌──────────────────┐
                       │   compact-self   │  /compact via tmux send-keys
                       └──────────────────┘

  ┌────────────────────────── switchboard ──────────────────────────┐
  │ append-only JSONL per channel; flock-based presence; metadata    │
  │ stream by default                                                │
  │                                                                  │
  │   send → delivered (auto) → read (auto on `switchboard read`)    │
  │       → ack (manual, with subject) → reply (--reply-to)          │
  └──────────────┬──────────────────────────────┬────────────────────┘
                 │                              │
        env-injects channel/handle      ┌───────┴───────┐
                 ▼                      ▼               ▼
       ┌──────────────────┐    ┌────────────────┐   ┌──────────────┐
       │     recruit      │───▶│ switchboard-ui │   │   dismiss    │
       │ splits tmux pane │    │ ratatui chat   │   │ kills pane,  │
       │ + claude-safe    │    │ over the wire  │   │ emits leave  │
       │ --model X        │    └────────────────┘   └──────┬───────┘
       └────────┬─────────┘                                 │
                │ tags pane with @recruit-handle            │
                │                                           │
                └─────────────────┬─────────────────────────┘
                                  │
                                  │
                                  ▼
                       ┌──────────────────┐
                       │   idle-scout     │
                       │ cron walker;     │
                       │ posts candidates │
                       │ via switchboard  │
                       └──────────────────┘

  ┌──── device control (each emits the same schema) ─────┐
  │   ddb (Android)        idb (iOS)        wdb (browser)│
  │       │                  │                  │        │
  │       └──────────────────┴──────────────────┘        │
  │                          │                            │
  │                          ▼                            │
  │                       ┌──────┐                        │
  │                       │  vdb │  drift / region matrix │
  │                       └───┬──┘                        │
  │                           │                            │
  │                           ▼                            │
  │                       ┌──────┐                        │
  │                       │  fdb │  Figma frame extraction│
  │                       └──────┘                        │
  └───────▲───────────────────────────────────────────────┘
          │
   preflight + suite
          │
       ┌──┴──┐
       │tctl │  orchestrator; reports state via switchboard
       └─────┘

  ┌──── infra (always in the call chain, rarely named) ──┐
  │   skill-router (PreToolUse hook)                      │
  │     ▲                                                 │
  │     │  enforces format contracts, redirects raw calls │
  │     │  to canonical skills, denies dangerous patterns │
  │     │                                                 │
  │   substrate (deploy / enroll / validate)              │
  │   claude-sandbox (Seatbelt + `nosandbox` escape)      │
  └───────────────────────────────────────────────────────┘
```

## Edges

### `recruit` ⇄ `dismiss` ⇄ `idle-scout`
The seat-lifecycle trio lives in one repo and shares one tag scheme: tmux user-options `@recruit-handle`, `@recruit-channel`, `@recruit-role` set by `recruit` on the new pane. `dismiss` and `idle-scout` both locate panes via those tags. Killing a pane out-of-band (manual `tmux kill-pane`) breaks `dismiss`'s clean-leave emission; always go through the three binaries.

### `recruit` → `switchboard`
`recruit` exports `SWITCHBOARD_CHANNEL` + `SWITCHBOARD_NAME` into the new pane's environment **before** `claude-safe` execs, so the child agent's first send/ack lands on the recruiter's wire automatically. The pre-loaded skill (`role-builder`, `role-enforcer`, …) runs `switchboard` connect on first prompt and announces the join.

### `idle-scout` → `switchboard`
Default `--report-channel=idle-scout` and `--to=$SWITCHBOARD_NAME` (the calling overseer). The dedicated channel is journal-write-safe — operators and the parent overseer subscribe; nobody else is taxed. Pairs with `dismiss` downstream.

### `switchboard-ui` → `switchboard`
Reads via `switchboard log --last 100` on channel open, then tails via `switchboard stream --channel <X> --from-cursor --full` (full bodies — TUIs aren't subject to harness truncation, unlike agent Monitor tasks). Posts via `switchboard send` / `ack` / `read`. Pre-acquires the flock on `global` at startup to close the consumer-dead race.

### `token-monitor` → `compact-self`
`token-monitor` emits milestone JSONL events on stdout. When the agent sees the `90` (critical) line, it shells out to `compact-self`, which tmux-injects `/compact` into the agent's own pane. The whole chain runs without the human touching the TUI.

### `remote-control` ⇄ Claude iOS app
`remote-control` injects `/remote-control` into the local pane; the Claude iOS app then connects via the operator's VPN. Independent of `switchboard` — the wire keeps working without it, but for mobile control of any individual pane (not the wire), this is the path.

### `ddb` / `idb` / `wdb` → `vdb`
All three device bridges emit the same YAML semantic schema (defined in `semantic-schema`). `vdb` consumes that schema across devices, builds region matrices, and reports drift. The schema contract is what makes "Android passes, iOS fails" comparisons cheap. See [device-control.md](device-control.md) for the surface and [schema.md](schema.md) for the wire format.

### `tctl` → `ddb` / `idb` / `wdb` + `switchboard`
The orchestrator. `tctl preflight` checks that each device bridge can reach its target (device connected, browser CDP responsive, etc.) before any suite runs. `tctl suite` walks specs, invokes the right device bridge per step, captures YAML, runs assertions. State pings flow over `switchboard` so an overseer watching the wire sees suite progress without tailing tctl's stdout. `tctl/docs/` is also the deepest toolchain doc authority (ADRs, tech-debt ledger, getting-started) — when these llms/* files aren't enough, point at `tctl/docs/llms.txt`.

### `fdb` / `figma-kiwi-protocol` → `vdb`
`fdb` produces the same schema from Figma frames (REST or offline dump) so the drift pipeline becomes design-vs-built. `figma-kiwi-protocol` is the lower-level Kiwi-WebSocket access — use it when REST hits rate limits or when an agent needs to PUSH mutations back into Figma. They share the schema contract but live at different abstraction layers.

### `skill-router` (under everything)
PreToolUse hook. Every `Bash` invocation goes through router rules in `~/.config/substrate/router_rules.toml` (T1) + tier-specific files. Rules DENY dangerous patterns (e.g. `rm -rf` against the switchboard cache), REDIRECT raw binary calls to their canonical skill (`switchboard` → `Skill(switchboard)`), or ALLOW. Agents rarely name skill-router — they hit it as a wall and read the hint.

### `substrate` → `skill-router` + `claude-sandbox`
`substrate enroll <co>` writes `.claude/settings.local.json` with the skill-router PreToolUse hook entry, symlinks T2 company skills into `.claude/skills/`, and registers git excludes. `substrate deploy` fans out T1 + T2 skills across enrolled repos. `claude-sandbox` is configured once per machine — agents call it indirectly every time they run anything (the Seatbelt profile is loaded by `claude-safe`).

## What's NOT an edge

- `switchboard` does **not** call `recruit`. The agent calling switchboard is who decides whether to spawn a peer.
- `vdb` does **not** call `ddb`/`idb`/`wdb` — it consumes their emitted YAML, never invokes them. That's why each device bridge can ship independently.
- `compact-self` does **not** call `token-monitor`. The monitor surfaces the alert; the agent decides to invoke `compact-self`.
- `dismiss` does **not** call `switchboard send` to announce — it emits `kind:leave` directly on the dismissed handle's behalf (uses switchboard's library, not the binary). The recruiter is expected to also post a one-liner from their own handle ("dismissed peer-X — work redistributed").
