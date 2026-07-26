# Conventions

Release discipline shared across all repos in this org.

## Tags

Every shipped binary tags its release. Tag format is `<tool>-vX.Y.Z` (e.g. `ddb-v0.4.0`, `idb-v0.4.0`). Tags are annotated, not lightweight.

Library repos (semantic-schema, semantic-agent-flutter, WebDriverAgent) tag as `vX.Y.Z` without the tool prefix — the repo name is the prefix.

The toolchain occasionally publishes a `stable-YYYY-MM-DD` rollup tag (see `tctl/docs/stable-*.md`) pinning a known-good combination of tool versions.

## CHANGELOGs

Every repo has a `CHANGELOG.md` at root. Entries are added at release time, not opportunistically. Use the [Keep a Changelog](https://keepachangelog.com) shape:

```
## [vX.Y.Z] — YYYY-MM-DD
### Added
### Changed
### Fixed
### Removed
```

## ADRs (Architectural Decision Records)

Long-lived structural decisions get an ADR. Per-repo ADRs live at `<repo>/docs/adr/NNN-slug.md` with sequential numbering. Cross-repo / org-level decisions (when there are any) would live here in this repo.

The most ADR-heavy repos are `tctl` (16 ADRs) and `substrate` (62 ADRs). Both maintain their own indexes.

Status values: `Proposed`, `Accepted`, `Superseded by ADR-NNN`, `Retired`. Superseded ADRs stay in the tree as historical record.

## Tech-debt ledger

The toolchain-wide TD ledger lives at [tctl/docs/tech-debt.md](https://github.com/llm-actuators/tctl/blob/main/docs/tech-debt.md). Format: `TD-N` numbered entries with a fixed shape (problem, shape, fix sketch, status). Search this before adding any new mechanism — most "why doesn't X work" questions are already answered there.

## Commits

Commit messages are single-line, conventional-commit style (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`), max 72 chars. No multi-paragraph commit bodies unless the change really needs it. No Claude attribution.

## Submodules

Two patterns in use:

- **Fork pinning** (`idb` → `WebDriverAgent`): correct pattern for forks of external projects. Pin the fork commit, update intentionally.
- **Shared library** (`ddb` → `semantic-schema` was previously a submodule, now a sibling path-dep): submodules for first-party shared crates create drift and double-checkout pain. The current convention is path-dep on a workspace-sibling checkout.

## Repository visibility

All repos in this org are public unless they contain operator-specific configuration (`substrate` is currently private because it touches enrollment + machine-specific paths).
