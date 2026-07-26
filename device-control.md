# Device control — common surface

`ddb` (Android), `idb` (iOS physical devices), and `wdb` (browser via CDP) are siblings, not clones. They share vocabulary and conventions where the underlying systems agree, and diverge where they don't.

This document maps the actual CLI surface as it exists in source (not aspirational parity). If you know one tool, this page tells you what carries over to the others and what doesn't.

## What's the same

### Conventions

- **Long flags `--kebab-case`.** Short flags are sparse and inconsistent — don't assume `-o` exists on a given command.
- **Exit code `0` success, `1` failure.** No semantic codes except `ddb test` which uses `2` for regression-gate failures.
- **`stderr` for errors, `stdout` for output.** Don't parse stdout for success/failure.
- **Screenshot output is a positional**, not `-o`. Defaults: `ddb` → `/tmp/screen.png`, `idb` → `screenshot.png`, `wdb` → `/tmp/web_screen.png`.
- **`--catalogue <path>`** is a shared flag on artifact-producing commands (`ui`, `screenshot`) in ddb + idb — feeds the toolchain's catalogue system.
- **`type <text>`** for text entry, identical in all three.
- **`scroll`** with direction (`up`/`down`/`left`/`right`) in ddb + idb; `wdb scroll` adds `top`/`bottom` and an optional pixel count.

### Verbs that map cleanly

| Verb | ddb | idb | wdb |
|---|---|---|---|
| Type text | `type <text>` | `type <text>` | `type <text...>` |
| Scroll direction | `scroll <dir>` | `scroll <dir>` | `scroll [dir] [px]` |
| Screenshot | `screenshot [path]` | `screenshot [path]` | `screenshot [path]` |
| UI introspection | `ui` | `ui` | `ui` |
| Page/element source | (in `ui --raw`) | (in `ui --raw`) | `source [selector]` |
| Run YAML test specs | `test <specs...>` | `test <specs...>` | — |
| Crawl/explore an app | `crawl --package` | `crawl -b/--bundle` | — |
| Calibrate test specs | `calibrate` | `calibrate` | — |

### Doctor / config / completions

All three have `doctor` (health check) and `config` subcommand groups for managing local state. Shell completions: ddb has `completions <shell>`; idb and wdb don't ship completions.

## What diverges

### Coordinate input vs selector input

`ddb` and `idb` are device drivers — they speak in **screen coordinates**. `wdb` speaks the **DOM** — selectors and text content.

| Concept | ddb / idb | wdb |
|---|---|---|
| Click/tap a point | `tap <x> <y>` | `click --xy <x> <y>` (selector is the primary path) |
| Click an element | (not a thing — there are no elements) | `click <selector>` / `click --text <s>` |
| Drag between two points | `swipe <x1> <y1> <x2> <y2> [--duration]` | (does not exist) |
| Drag between two elements | (does not exist) | `drag <from-sel> <to-sel>` |
| Hover | (does not exist) | `hover <selector>` |

This is not an inconsistency to fix — it's the underlying reality. Devices don't have selectors; browsers don't have screen coordinates as a primary input model.

### App lifecycle

| Action | ddb | idb | wdb |
|---|---|---|---|
| Launch app | `app launch <package>` | `app launch <bundleId>` | — |
| Open URL | — | — | `navigate <url>` |
| Launch browser | — | — | `launch [url]` |
| Kill app | `app kill <package>` | `app kill <bundleId>` | `stop` (closes browser) |
| Install build | `app install <path>` | `app install <path>` | — |
| Clear app data | `app clear <package>` | — | `storage --delete <key>` / `cookie --delete <name>` |
| Connect to running instance | (always on) | (always on) | `connect <port-or-ws-url>` |

`ddb deploy <path> [package]` is a `ddb`-only conveniences (install + auto-detect package).

### Device targeting (the messy part)

| Tool | Pattern |
|---|---|
| `ddb` | One global flag: `-d/--device <name>` applies to every subcommand. Clean. |
| `idb` | **Three** styles depending on subcommand: shared `DeviceOption` (`-d/--device`, optional, auto-resolves single device), required positional `<device>` (most `wda` subcommands and `syslog`), or optional positional (`mirror`, `devices status`). |
| `wdb` | No concept — one Chrome instance, tracked via `cdpFile`. Multi-target needs `connect <port-or-ws>`. |

If you're scripting against idb, check each subcommand's signature — there's no single pattern.

### Buttons / hardware keys

| Tool | Surface | Available |
|---|---|---|
| `ddb` | `button <name>` + dedicated `home` / `back` commands (duplicate surface) | home, back, power, enter, menu, recents, volume_up/volup, volume_down/voldown, delete/del, raw keycode |
| `idb` | `button <name>` + dedicated `home` / `back` | home, volumeUp, volumeDown (only) |
| `wdb` | `press <key>` | Enter, Tab, Esc, F1–F12, arrows, modifiers, single chars |

### UI inspection flags

| Flag | ddb | idb | wdb |
|---|---|---|---|
| `--raw` | ✓ | ✓ | — |
| `--semantic` | ✓ | ✓ | — |
| `--json` | ✓ | (use raw + parse) | ✓ |
| `--catalogue <path>` | ✓ | ✓ | — |
| `--source-root <path>` | ✓ | ✓ | — |
| `--scroll` | (use `scroll-capture`) | ✓ | — |
| `--filter <s>` | — | (use `elements --predicate`) | ✓ |
| `--limit <n>` | — | — | ✓ |
| `--all` | — | — | ✓ |

### Streaming logs

| Tool | Command |
|---|---|
| `ddb` | (use `adb` passthrough: `ddb adb logcat`) |
| `idb` | `syslog <device> [-p/--process]` |
| `wdb` | `log [console\|network\|errors]` |

### Mirroring / screencast

All three have `mirror` for live device viewing:
- `ddb mirror [extra args→scrcpy]`
- `idb mirror [device] [--scale] [--mjpeg-port] [--touch-port]`
- `wdb` does not (CDP-attached Chrome is already visible)

### Passthrough escape hatch

- `ddb adb <args...>` — forward to underlying `adb`, auto-inject `-s <transport>`. Forwards adb's exit code.
- `idb` has no equivalent (no `xcrun devicectl` passthrough at CLI level — internal use only).
- `wdb eval <js...>` — execute arbitrary JavaScript in the current frame.

## Environment variables

Each tool reads its own `$TOOL_*` env vars for defaults the CLI doesn't surface. Examples:

- `DDB_AGENT_PORT`, `DDB_RESULTS_DIR`, `DDB_TESTS_DIR`, `DDB_EXPECTED_HASH` (required at test runtime), `DDB_PROJECT_DIR`, `DDB_TC_MAP`, `DDB_RUN_ID`, plus ~10 more around build/mock/retention.
- `IDB_AGENT_HOST`, `IDB_AGENT_PORT` (default 9877), `IDB_TEST_BUNDLE_ID`, `IDB_BUNDLE_ID`, `IDB_MOCK_DIR`, `IDB_STRICT_COORDS`, `IDB_RUN_ID`.
- `wdb` reads no env vars (state in sidecar files: `cdpFile`, `frameFile`, `pageFile`).

The full list per tool is in each repo's source — these are not stable contracts.

## Doctrine

These binaries provide **primitives only**. `ddb tap 540 1200` is a primitive; `ddb verify-checkout-flow` would not be. Domain verbs belong in project-level recipes that compose primitives — see [tctl/docs/adr/ADR-005-recipe-infrastructure.md](https://github.com/llm-actuators/tctl/blob/main/docs/adr/ADR-005-recipe-infrastructure.md). If you find yourself wanting to teach `ddb` about your app's login flow, that's a recipe, not a feature.

## Known inconsistencies (acknowledged, not fixed)

These are real and won't change without coordinated work across all three:

- **Output as positional vs flag.** `screenshot` takes path positionally; `ui` takes `-o/--output`; `scroll-capture` takes `--output`. No clean rule.
- **`home`/`back` exist twice in ddb and idb** (top-level commands AND values for `button`). Cosmetic, but present.
- **idb device-targeting has three styles** (see above). Worth knowing before scripting.
- **`type` text joining**: wdb joins trailing args with spaces (`type hello world` → `"hello world"`); ddb and idb take a single quoted string. If you script `wdb type "hello world"` it works either way; quoting habits differ.
- **`wdb` usage strings have `wdbtype` / `wdbeval` / `wdbwait` glitches** (missing space). Cosmetic.
- **`wdb ws-capture --output` defaults to `/tmp/figma_kiwi`** — a project-specific leak in a generic tool.

The "if X then also Y" parity work would be:
1. Standardize device-targeting in idb (collapse the three styles to one).
2. Drop the duplicate `home`/`back` top-level commands in favor of `button home`/`button back`.
3. Either add `--json` to `idb ui` or document that `--raw` is the JSON path.
