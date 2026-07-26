# Use cases

Canonical compositions. binaries.md tells you what each tool IS. This file tells you which tools to compose when you want to DO something.

Each scenario: goal, tools, env / channel topology, sequence.

## 1. Test an Android UI feature end-to-end

**Goal:** verify a change against a real Android device + capture visual evidence.

**Tools:** `ddb` (device), `vdb` (visual matrix), `tctl` (suite orchestrator).

**Sequence:**
1. `ddb devices` — confirm the device is reachable.
2. `tctl preflight` — toolchain health.
3. Install the build (`ddb install <apk>`) and launch (`ddb launch <package>`).
4. Drive UI via `ddb tap` / `swipe` / `type` / `scroll`. Use `ddb elements --label <X>` for semantic targeting (cheap), `ddb screencap` for visual confirmation (expensive — only when needed).
5. Capture region matrix via `vdb` to confirm no drift.
6. Run a TC suite via `tctl` if multiple scenarios are in flight.

**Pitfalls:** never call raw `adb` — always `ddb`. Skill router will block it. On sandboxed macOS sessions, wrap `./gradlew` and `xcodebuild` with `nosandbox` (allowlist at `~/.config/claude-sandbox/nosandbox-allow.conf`). Cold sessions hit this before they learn it exists.

## 2. Coordinate two seats (one iOS, one Android)

**Goal:** verify cross-platform parity with two LLM sessions, one driving each platform, plus a coordinator overseeing both.

**Tools:** `switchboard` (wire), `recruit` (spawn), `dismiss` (teardown).

**Topology:**
- Overseer pane: subscribed to `global` + a working channel (`app-parity`).
- iOS builder pane: subscribed to `app-parity`; drives via `idb`.
- Android builder pane: subscribed to `app-parity`; drives via `ddb`.

**Sequence:**
1. Overseer: `recruit role-builder --channel app-parity --cwd <ios-repo> --split h` and `recruit role-builder --channel app-parity --cwd <android-repo> --split v`. Each new pane joins the wire with `SWITCHBOARD_CHANNEL=app-parity` and a handle like `builder-XXXX`. `--cwd` is load-bearing when the builder works a different repo than the recruiter (router and tool defaults misread the project without it).
2. Overseer assigns scope on the wire via `switchboard send --to ios-builder` / `--to android-builder`.
3. Builders report PASS/FAIL with evidence (SHA, screenshot path) directly to the overseer's handle. Acks close loops.
4. When a builder is mid-investigation and hits 85%, overseer dismisses + recruits-fresh from the wire checkpoint (per role-overseer doctrine §41).
5. On completion: `dismiss <handle>` for each builder. Channel stays for the audit log.

## 3. Survive a compact

**Goal:** keep working past the model's context-window limit without losing state.

**Tools:** `token-monitor` (alarm), `compact-self` (action).

**Sequence:**
1. At session start, arm `token-monitor` via `Monitor` (persistent). It fires JSONL events at 60 / 70 / 75 / 80 / 85 / 90% milestones.
2. 60–85% milestones: checkpoint to wire/tasks (file paths, in-progress work, solutions received). Do NOT compact yet.
3. 90% milestone: run `~/.claude/bin/compact-self` immediately. It injects `/compact` into your TUI pane via `tmux send-keys` — requires the session is running inside tmux (`$TMUX` + `$TMUX_PANE` auto-set; binary exits non-zero otherwise). Auto-compact is imminent and lossier than self-triggered.
4. After resume: announce `<handle> back on after compact` on every channel you're subscribed to. Re-invoke any skills that grant tool bypass (per Constitution Law III).

## 4. Figma-to-built drift check

**Goal:** compare a design source-of-truth to what's running on the device.

**Tools:** `fdb` (Figma extraction), `vdb` (visual matrix + drift), `ddb`/`idb` (live screen capture).

**Sequence:**
1. `fdb frames --node <node-id>` — extract design frames as image refs.
2. `ddb screencap` or `idb screencap` — capture the equivalent built screen.
3. `vdb compare --design <fdb-output> --built <screencap>` — drift report with region-level diffs.
4. Apply fixes app-side or flag genuine platform-convention differences (some visual differences are correct per-platform).

## 5. Drive a browser

**Goal:** automate a web flow — navigate, fill, assert, capture state.

**Tools:** `wdb` (CDP-backed browser control).

**Sequence:**
1. `wdb launch <url>` — Chrome under CDP.
2. `wdb elements --type input` — DOM extraction (cheap text feedback).
3. `wdb type --selector '#email' <value>` / `wdb click --selector 'button[type=submit]'`.
4. `wdb assert --selector '.success'` to gate.
5. `wdb screencap` only for visual confirmation; prefer `wdb elements` for state checks.

## 6. Multi-brand pipeline coordination

**Goal:** trigger N CI pipelines (one per brand / variant), monitor them in parallel, surface failed jobs without polling.

**Tools:** the company's CI CLI (T2 skills), `switchboard` (status wire), `Monitor` (pipeline tail).

**Sequence:**
1. From the consumer repo (substrate-enrolled), invoke the company's CI skills for each brand pipeline.
2. Arm a `Monitor` task per pipeline tailing pipeline status; each terminal status emits one stdout line that becomes a notification.
3. On a failure that maps to a known issue category, the company's ticket-opening skill files it directly into the backlog.
4. Operator stays asleep — pipelines complete, notifications fire, only emergencies require keyboard.

**Why this composition:** CI is the trigger surface; switchboard is the cohort visibility; Monitor is the interrupt model that lets agents react when something happens, not every 30 seconds.

## 7. Cross-repo library fix

**Goal:** ship a fix that spans a library repo and a consumer repo without losing track of either commit.

**Tools:** `commit` (T1/T2 skill), local artifact publish (`publishToMavenLocal`, `pod install --local`, etc.), `ddb` / `idb` (device verify).

**Sequence:**
1. Patch the library repo. Publish locally so the consumer can resolve the new version.
2. Bump the library version in the consumer's dependency manifest.
3. Build + install on the verification device via `ddb install` / `idb install`.
4. Verify the fix on device.
5. `commit` skill on the library repo. `commit` skill on the consumer repo. Two SHAs, both referenced in the operator's MR description.

**Pitfall:** the local artifact cache desynchronizes if you forget to refresh dependencies on the consumer after re-publishing the library. If the consumer picks up a stale artifact, the device install will pass but the runtime won't show the fix.

## 8. Cold session lands on a company-enrolled repo

**Goal:** orient yourself when you open a Claude Code session inside a repo that has `.claude/settings.local.json` and a skill-router hook configured.

**Tools / state to inspect:** `~/.config/substrate/<company>/`, `.claude/settings.local.json`, skill router rules.

**Sequence:**
1. Check the enrollment: `cat .claude/settings.local.json` — the PreToolUse hook field references `skill-router` with paths to T1 (global) and T2 (company) rules files.
2. Find the company configs: `ls ~/.config/substrate/<company>/` — `secrets.properties`, `router_rules.toml`, `repos.toml`/`.properties`, plus optional `jira.toml` / `linear.properties` / etc.
3. List T2 skills available in this repo: `ls .claude/skills/` — these are symlinks into the company's T2 skill store, individually excluded in `.git/info/exclude` so they stay un-tracked but operational.
4. When the router blocks a raw command (e.g. `git commit` → redirects to `/commit`), read the hint message; it names the skill to load. `Skill(<name>)` then retry.
5. Find the role file for your seat (overseer / builder / janitor / etc.) under `~/.claude/skills/role-*/SKILL.md`. The role pins the cron arms, monitors, and doctrine to load at activation.

**Why this matters:** without orientation, a cold session discovers the enrollment by getting blocked and reading error messages — wastes turns. The two-minute survey above gets you operational immediately.
