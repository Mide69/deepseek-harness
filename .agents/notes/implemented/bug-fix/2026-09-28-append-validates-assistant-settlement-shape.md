# Agent Note: Validating Assistant settlement fields on append, not only on seed/load

Status: implemented

## Problem

`assistant/attempt` and `assistant/message` events carry `turn`, `step`, and `stream` fields that restored Session lifecycle logic reads directly, without replaying the embedded stream (see [v2 embedded Assistant streams](../architecture/2026-09-01-v2-embedded-assistant-streams.md)). `assertAssistantSettlementShape` in [`packages/core/session/src/index.ts`](../../../../packages/core/session/src/index.ts) enforces that `turn` and `step` are non-negative safe integers and `stream` is an array, but it was wired only into `assertCurrentLlmShape`, which `assertSessionEventEnvelope` calls at the seed/load boundary (`Session.fromRestore`, `Session.create` with a seed). `Session.append`, the path a running agent loop writes through, called only `validateSessionEventData` and `surfaceManager.validateNext`, neither of which checks these fields.

A plugin, provider adapter, or any other append() caller could therefore write an `assistant/message` with a missing or malformed `turn`, `step`, or `stream` and the write would succeed silently. The corruption surfaced only the next time that session was restored from storage, as `stored session "<id>" is corrupt: seed assistant/message at index <n> has invalid settlement fields`, with the whole session then unreadable and no indication of which write produced it. A user hit exactly this ([deepseek-harness#8084](https://github.com/deepseek-ai/deepseek-harness/discussions/8084)) on an assistant message thousands of events into their session, discovered only when the session next failed to load.

This is the same class of gap as the sibling `assertMessageEventShape` check (message identity: `id`/`role`/`source`), which is wired only into `adoptSessionEvent`, the seed/load and storage-read path, and not into `append` either: a validator that exists but is reachable only from the read/restore path, so a bad write is accepted and only ever rejected later, at a point disconnected from its cause.

## Decision

- `assertAssistantSettlementShape` now takes a `subject` string (matching `assertMessageEventShape`'s signature) instead of a `type` and numeric `index`, so the same function serves both the seed/load subject format (`seed ${type} at index ${index}`) and `append`'s own (`session event "${type}" at seq ${event.seq}`). This is a pure signature refactor; the seed/load call sites build the identical string they built before, so `Session.fromRestore` and `Session.create` reject the same inputs with the same messages.
- `Session.append` now calls `assertAssistantSettlementShape` for `assistant/attempt` and `assistant/message`, right after `validateSessionEventData` and before `surfaceManager.validateNext`, mirroring where `assertMessageEventShape` was added for message identity. A malformed settlement write now throws at the call site, naming the event and its seq, instead of being accepted and corrupting the log for the next restore.
- The check runs against `dataSnapshot`, the already-JSON-round-tripped value `append` computes for its own non-serializable-data guard. That guard runs first and rejects `-0` before this check ever sees it (`JSON.stringify(-0) === "0"`, which the package's lossless-JSON snapshot treats as unrepresentable) — one fewer case for this check to special-case, and a clearer error (`carries non-JSON-serializable data`) for that specific input.

## Alternatives considered

**Validate at the surface layer (`surface.ts`) instead of in `append`.** `validateSessionEventData` already runs for every append and covers `request/header` and `tool/result` shape checks. Settlement-field validation is restore-lifecycle logic specific to `assistant/attempt` and `assistant/message`, colocated with `assertCurrentLlmShape` and the Session class that owns `firstLifecycleSeq`/`inheritedEventCount`, not a generic surface-metadata concern; moving it would separate the check from the invariant it protects.

**Loosen the seed/load check instead, so a malformed record does not fail the whole session.** The class of consumer this protects (durable lifecycle bookkeeping keyed on `turn`/`step`, and stream replay) has no defined behavior for a missing or malformed value; a loosened read-side check would trade an honest "session is corrupt" refusal for one plugin's bad event silently desyncing lifecycle state for the rest of the session. The gap was never that the check was too strict — the same fixtures that produced `seed ... has invalid settlement fields` at load time are still rejected identically here.

## Testing

| Evidence | Behaviour |
|---|---|
| [session.spec.ts](../../../../packages/core/session/tests/session.spec.ts) | `rejects the same malformed Assistant settlement fields on append, not only on seed/load` appends the same nine malformed `{turn, step, stream}` shapes the existing seed/load test rejects (string/negative/non-integer `turn` or `step`, non-array `stream`) as both `assistant/attempt` and `assistant/message`, asserts each throws `has invalid settlement fields`, and asserts none of the rejected appends entered the log. The existing `validates Assistant settlement fields without replaying embedded streams` seed/load test is unchanged and still passes, confirming the refactored `subject` parameter preserves its exact error messages. |
| [transport.host.spec.ts](../../../../packages/api/session-controller/tests/transport.host.spec.ts) | Two pre-existing fixtures appended `assistant/message` events with `{ turn, step, message }` and no `stream`, which this change now rejects; both were fixed to add `stream: []`, which does not affect what the test exercises (message-aligned pagination). |
| `packages/core/session`, `packages/core/agent-loop`, `packages/core/agent`, `packages/api/session-controller`, `packages/subagent` (`npx vitest run`) | Full suites pass; the small number of unrelated failures on this run are pre-existing timeouts under parallel load (real-product subprocess tests needing external CLIs, resource-contention timeouts), confirmed by grepping the run's output for `invalid settlement` (zero matches outside the new test). |
| `npm run typecheck`, `npm run lint` (`packages/core/session/src/index.ts`, its tests, `transport.host.spec.ts`) | Both pass cleanly. |

## Consequences

- A plugin or adapter that appends a malformed `assistant/attempt` or `assistant/message` now gets an immediate, specific error at the append call site instead of a session that fails to load later with no link back to the write that caused it.
- `assertAssistantSettlementShape`'s error message no longer says `seed` for an append-time rejection; it says `session event "<type>" at seq <n>`, matching every other append-time validation error in this file.
- `-0` in `turn` or `step` is now rejected at append time by the earlier non-JSON-serializable guard rather than by this check; the seed/load path, which accepts hand-built JS objects that never passed through a JSON round trip, still exercises this check directly for `-0`.
