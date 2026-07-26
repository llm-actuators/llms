# Mock corpus shape

Cross-cutting conventions for project-side HTTP mocks consumed by `idb`/`ddb`'s in-app `MockRegistry`. Tests run with the backing service offline; the corpus owns the request/response shapes the app sees.

Authoritative source: `idb/agent/Sources/SemanticAgent/MockRegistry.swift` and the same on the `ddb` side.

## Sterility line

- **Mock corpus content** (JSON envelopes with project nouns, IDs, body strings) lives **project-side** at `<project>/catalogue/fixtures/api/`. Never crosses into `substrate-distro/`.
- **Mock corpus primitives** (registry, request matchers, mutation engine, overlay loader) live in `idb/agent/`. App-noun-free — only generic JSON shapes and operations.

This split is non-negotiable. Adding `siteId == 12345` to substrate-distro is an ADR-008 violation.

## Envelope shape

Each mock is a single JSON file describing one URL pattern × HTTP method:

```json
{
  "url_pattern": "/v3.1/sites/{site_id}/relationships/questions",
  "method": "GET",
  "response": {
    "status": 200,
    "body": { "data": [ ... ] }
  }
}
```

Files are loaded from a directory walk; the loader registers each envelope keyed by `(url_pattern, method)`.

## Per-TC variant selection (overlay/merge)

Pattern: TC needs a different mock state than the corpus default (empty list vs populated, foreign-author vs own-author, etc.). Earlier convention was to duplicate the entire corpus; the loader now supports **overlay merge** instead.

Convention: variants live at `<mock_dir>/.variants/<variant_name>/`. Each variant contains only the **envelopes that differ** from the base — typically 1-2 JSON files.

Invocation (per-TC):
```bash
IDB_MOCK_OVERLAY=catalogue/fixtures/api/mocks/populated/.variants/empty-questions \
  SINGLE_TC=<slug> run-suite.sh single
```

Loader behavior: walks the base `<mock_dir>/` first, then walks `<mock_dir>/.variants/<overlay>/` and **replaces** any entries with matching `(url_pattern, method)`. Variant wins. Backward compatible — empty/unset env is no-op.

Loader endpoint: `POST /mock/overlay` (idb agent). Runner forwards via `IDB_MOCK_OVERLAY=<dir>` env.

## Stateful side-effects (mutation)

Pattern: TC mutates state (DELETE, POST) and then re-fetches, expecting the GET response to reflect the mutation. Without stateful mocks, every URL returns its static response regardless of prior method calls, breaking mutation-then-refetch flows.

Convention: a DELETE or POST envelope can declare **mutations** that fire on serve, modifying the registered GET response for another URL.

```json
{
  "url_pattern": "/v3.1/questions/{id}",
  "method": "DELETE",
  "response": { "status": 200, "body": { ... } },
  "mutates": [
    {
      "target_url_pattern": "/v3.1/sites/12345/relationships/questions",
      "target_method": "GET",
      "op": "remove_by_id",
      "json_path": "data",
      "id": "42"
    }
  ]
}
```

Op vocabulary:

| Op | Effect |
|---|---|
| `remove_by_id` | Remove the array element with matching `id` from the target body at `json_path`. |
| `add_to_array` | Append an inline `item: { ... }` object to the target body's array at `json_path`. Used for POST→GET flows. |

Behavior:
1. The DELETE/POST envelope serves its static response unchanged.
2. After the response is sent, each declared mutation locks the registry, finds the target entry by `(url_pattern, method)`, decodes its body, applies the op, re-encodes, replaces the entry.
3. Next GET to the target URL returns the mutated body.

Thread-safety: mutations acquire the registry's lock for the read+modify+write cycle.

Backward compatible: envelopes without `mutates` behave identically to today.

## Choosing between overlay and mutation

| Need | Use |
|---|---|
| TC starts in a non-default state (no setup actions needed) | Overlay variant |
| TC mutates state mid-run and re-reads | Mutation |
| TC mutates AND starts non-default | Both (overlay sets the start; mutation handles the deltas) |

Overlay is declarative ("here's the world before the test"); mutation is reactive ("here's how the world changes during the test"). They compose.

## Anti-patterns

- **Duplicating the entire mock corpus for a single envelope change** — that's what overlay solves. Resist the temptation; a 60-file copy-paste is technical debt.
- **Mock content drift** — keep envelopes minimal and pinned to canonical IDs from `fixtures.yaml`. Drift in mock JSON manifests as flaky TCs no one can root-cause.
- **Mutation without lock** — concurrent mutations + serve will produce undefined behavior. Always use the registered API path; never edit response bodies directly.

## Cross-references

- [agent-contract.md](agent-contract.md) — HTTP contract every in-app agent implements
- [conventions.md](conventions.md) — release discipline (tags, CHANGELOGs, ADRs)
- `idb/agent/Sources/SemanticAgent/MockRegistry.swift` — reference Swift implementation
- `ddb/agent/.../MockRegistry.kt` — Android sibling
- [tctl/docs/tech-debt.md](https://github.com/llm-actuators/tctl/blob/main/docs/tech-debt.md) — TD ledger including TD-207 (stateful mutation), TD-208 (overlay/merge)
