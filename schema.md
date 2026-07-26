# semantic-schema

The shared Rust crate that defines the data types every on-device agent emits and every consumer (ddb, vdb) parses.

## Source

[github.com/llm-actuators/semantic-schema](https://github.com/llm-actuators/semantic-schema)

## Why a shared crate

The on-device agent (Kotlin / Swift / Dart) and the controller (Rust) must agree on the payload shape. A single Rust crate is the canonical schema; the agent implementations re-derive equivalent types in their own language. Drift here breaks introspection silently — keep the schema as the source of truth and treat each platform agent's local definitions as derivations.

## How consumers use it

- **`ddb`** depends on it via a sibling-checkout path-dep (`semantic-schema = { path = "../semantic-schema" }`). Cloning ddb in isolation will not build until the schema crate is cloned next to it.
- **`vdb`** uses the same pattern.

Both consumers assume a side-by-side workspace layout (the kind `substrate-distro/` provides). No `crates.io` publication.

## What's in the payload

For the field-by-field schema, read the crate's `src/` directly. For the contract that produces this payload, see [agent-contract.md](agent-contract.md) and the authoritative [agent-porting-guide.md](https://github.com/llm-actuators/tctl/blob/main/docs/agent-porting-guide.md).
