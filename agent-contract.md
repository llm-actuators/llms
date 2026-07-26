# In-app agent contract

Every platform-specific semantic agent — Android (`ddb/agent/`), iOS (`idb/agent/`), Flutter (`semantic-agent-flutter`) — implements the same HTTP contract for in-app UI introspection. The app embeds the agent as a debug-only dependency; the agent runs an HTTP server in-process; `ddb` / `idb` consume it.

## Authoritative source

The contract is defined in [tctl/docs/agent-porting-guide.md](https://github.com/llm-actuators/tctl/blob/main/docs/agent-porting-guide.md). That document is the single source of truth — endpoint surface, `ElementRecord` schema, idle-resource registry, G1–G5 genericness invariants.

Companion: [tctl/docs/semantic-agent-spec.md](https://github.com/llm-actuators/tctl/blob/main/docs/semantic-agent-spec.md).

## Reference implementations

| Platform | Source | Consumed by |
|---|---|---|
| Android | [`ddb/agent/`](https://github.com/llm-actuators/ddb/tree/main/agent) | `ddb` |
| iOS | [`idb/agent/`](https://github.com/llm-actuators/idb/tree/main/agent) | `idb` |
| Flutter | [`semantic-agent-flutter`](https://github.com/llm-actuators/semantic-agent-flutter) | (standalone) |

## Payload shape

The data types emitted by the agents and consumed by `ddb` / `vdb` live in [`semantic-schema`](https://github.com/llm-actuators/semantic-schema). See [schema.md](schema.md) for an overview.

## Porting a new platform

Read the porting guide end-to-end before implementing. The Flutter reference implementation is the cleanest standalone example (it doesn't have an absorption history complicating its repo structure).
