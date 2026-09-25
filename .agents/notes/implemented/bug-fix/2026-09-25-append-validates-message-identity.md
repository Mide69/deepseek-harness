# Agent Note: Session.append validates message identity, not only on adopt

Status: implemented

## Problem

`assertMessageEventShape` ([index.ts](../../../../packages/core/session/src/index.ts)) rejects a `user/message`, `assistant/message`, `system/message`, `developer/message`, or `tool/result` event whose message lacks an `id`, carries the wrong `role`, or has an invalid `source`/`content` shape. `adoptSessionEvent` — the path a seed, a fork, or a storage read goes through — already calls it. `Session.append` — the path every live write goes through, including a plugin injecting a message into the inbox at runtime — did not, even though its own doc comment states the intended contract: "The event log is the durable source of truth, so a bad event fails at the append site rather than later during a backend flush." A plugin appending a malformed message (missing `id`, most commonly) succeeded silently at write time and only surfaced as `lacks an identified message` the next time the session was read back, by which point the durable log already held the bad event and the whole session became unreadable.

## Decision

Call `assertMessageEventShape(event, subject)` inside `append`, in the same position `adoptSessionEvent` calls it (after `validateSessionEventData`, before the event is published). A malformed message now fails at the append site, matching the function's documented contract, instead of surfacing later as an unreadable session.

## Alternatives considered

**Validate only at the boundary a plugin's injection API exposes.** Rejected: `append` is the one seam every writer, core and plugin alike, already passes through; validating there covers every current and future caller without each one repeating the check, and matches how every other `append`-time invariant (JSON-serializability, surface contract, request-header/tool-result shape) is already enforced.

**Auto-repair a missing `id` instead of rejecting.** Rejected: silently inventing an identity for a message the caller did not identify hides a caller bug instead of surfacing it, and the frozen read-side migration path this durable log eventually reaches is deliberately strict about accepting only complete, caller-supplied identities.

## Consequences

A plugin that appends a `user/message` (or other message-typed event) without a valid `id`/`role`/`source`/`content` shape now gets an immediate, attributable exception at the call site instead of a session that becomes unreadable only when it is next opened. [session.spec.ts](../../../../packages/core/session/tests/session.spec.ts) adds a regression test reusing the existing seed/load malformed-shape fixtures against `append` directly, confirming the same rejections apply on both paths. Two existing test fixtures across [session.spec.ts](../../../../packages/core/session/tests/session.spec.ts) and [transport.host.spec.ts](../../../../packages/api/session-controller/tests/transport.host.spec.ts) constructed minimal `user/message`/`assistant/message` payloads without an identity for unrelated assertions (dense-array serialization, surfaceOp validation, pagination); both now carry a minimal valid identity so those assertions still exercise their own behavior.
