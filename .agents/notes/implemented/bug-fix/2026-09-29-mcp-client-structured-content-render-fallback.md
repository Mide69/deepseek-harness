# Agent Note: Projecting structuredContent when an MCP result has no content blocks

Status: implemented

## Problem

`createOutput`'s `render` in [`packages/mcp/mcp-client/src/tools.ts`](../../../../packages/mcp/mcp-client/src/tools.ts) is the synchronous projection that decides what the model sees for a tool call: it calls `extractText(result.content, rawName)`, which walks `result.content` and, when nothing in it yields a text or image block, falls back to the placeholder `(${toolName} returned no model-visible content)`. This fallback never looked at `result.structuredContent`.

Some MCP servers answer entirely through `structuredContent` and return an empty `content` array — the MCP spec treats `structuredContent` as a first-class, independently valid result channel, not something every server is required to mirror into `content`. Against such a server, the tool call succeeds server-side, but the model receives the "no model-visible content" placeholder, which reads exactly like a timeout or a silent failure. The agent then retries a call that already worked ([deepseek-harness#8240](https://github.com/deepseek-ai/deepseek-harness/discussions/8240), reproduced against `mcp-stata` and `paper-search-mcp`).

## Decision

`render` now checks, before calling `extractText`: if `result.content` is empty and `result.structuredContent` is present, project `JSON.stringify(result.structuredContent, null, 2)` as the one model-visible text block instead of the placeholder. Any non-empty `content` — even content that itself yields no text or image blocks — is left exactly as before, still walking `extractText`'s per-block-type fallback and, if nothing there projects either, the existing placeholder. This keeps the fix scoped to the one case in the report: a server that used `structuredContent` as its only channel, not a broader "prefer structured content" policy.

`result.structuredContent` is already the validated `JsonValue` produced by `specTypeSchemas.CallToolResult['~standard'].validate` earlier in `createExecutor`; `JSON.stringify` on a `JsonValue` cannot fail or return `undefined` (no cycles, no non-JSON members), so this needed no defensive try/catch, unlike the community patch the report proposed.

## Alternatives considered

**Always prefer `structuredContent` over `content` when both are present.** Rejected: a server that returns human-readable `content` alongside a structured payload intends the text for the model and the structured payload for programmatic/PTC callers (both already flow through unchanged in the canonical `McpResult` value); overriding that would discard an intentional summary in favor of raw JSON. The added `prefers content over structuredContent when both are present` test pins this.

**Treat any content that projects to nothing (not just an empty array) as eligible for the structuredContent fallback.** Rejected as broader than the report: `content` arrays with unsupported/refused blocks already have a defined text diagnostic for each block type in `projectContent`, and conflating "diagnosed a refusal" with "server used a different channel" would make refusal diagnostics silently disappear whenever `structuredContent` happens to also be present.

## Testing

| Evidence | Behaviour |
|---|---|
| [mcp-client.spec.ts](../../../../packages/mcp/mcp-client/tests/mcp-client.spec.ts) | `projects structuredContent as model-visible text when content is empty` — an empty-`content` / present-`structuredContent` result projects the pretty-printed JSON. `prefers content over structuredContent when both are present` — a non-empty `content` result keeps projecting its text, ignoring `structuredContent`. The pre-existing `handles empty content array` test (empty `content`, no `structuredContent`) is unchanged and still asserts the placeholder, confirming the new branch only fires when `structuredContent` is actually present. |
| `packages/mcp/mcp-client` (`npx vitest run`) | Full suite passes (130 tests). |
| `npm run typecheck`, `npm run lint` (`packages/mcp/mcp-client/src/tools.ts`, its tests) | Both pass cleanly. |
| `pnpm run verify-translation-pairing packages/mcp/mcp-client/README.md` | Passes after regenerating `README.i18n.yaml` for the updated `README.md`/`README.zh.md` pair. |

## Consequences

- A tool call to a `structuredContent`-only MCP server is now visible to the model as its structured payload, instead of a placeholder indistinguishable from a real failure; the agent no longer blind-retries a call that already succeeded.
- The projected text is the raw pretty-printed JSON, not a server-authored summary — a server that wants a specific model-facing description of its structured payload should still put that in `content` alongside it.
