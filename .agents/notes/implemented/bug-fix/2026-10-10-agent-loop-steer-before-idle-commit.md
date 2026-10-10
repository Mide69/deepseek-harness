# Agent Note: A waking send between the last inbox check and idle

Status: implemented

## Problem

`Agent.steer` and `Agent.followup` insert into the inbox and call `wakeDriver`. While a driver is running, `wakeDriver` returns without recording the wake; the live driver is expected to claim the queued item itself ([deepseek-harness#9370](https://github.com/deepseek-ai/deepseek-harness/discussions/9370)). That expectation fails in one window of [`packages/core/agent-loop/src/agent.ts`](../../../../packages/core/agent-loop/src/agent.ts):

1. `turn()` appends `turn/end` in its `finally`, then reads `inbox.hasPending` and returns `false`.
2. `kick` resumes from `await this.turn()` one microtask later and commits the idle phase.
3. A `session/event` observer of `turn/end` can queue a microtask (`queueMicrotask`, `Promise.resolve().then`) that runs between steps 1 and 2. Its `steer` splices an item, `wakeDriver` sees a running phase and records nothing, and the agent settles idle with the item still pending until some later wake.

Callers that deliver through promise reactions hit this; callers that deliver from a macrotask cannot interleave there.

## Decision

The running phase carries a `settling` flag that `turn()` sets at the moment it finds the inbox empty and is about to return. `wakeDriver` latches `wakeRequested` for a waking send that lands while `settling` is set, the same latch it already uses for maintenance and aborted activities. `kick` already replays that latch after committing idle when the inbox still holds input, so the stranded item starts the next turn at once.

The flag is set only on the normal completion path. A turn that ends by error or abort never reaches that return, so a send after it is not latched.

## Alternatives considered

**Latch every waking send while the driver runs.** Rejected after trying it: seven existing tests failed. A turn that fails or is cancelled intentionally retains later steering until the next explicit wake, and an unconditional latch wakes the agent for that retained input as soon as the driver exits.

**Have `kick` replay whenever the inbox has pending input at idle.** Rejected: `inject` queues next-step input without waking, and pending injected context must wait for a waking message ([architecture](../../../../docs/architecture.md)).

## Testing

| Evidence | Behaviour |
|---|---|
| [contract-regressions.spec.ts](../../../../packages/core/agent-loop/tests/contract-regressions.spec.ts) `steer() queued as a microtask by the final turn/end runs in a second turn` | A `session/event` observer queues `steer` as a microtask on the first `turn/end`. The agent runs a second turn that contains the message and ends with an empty `nextStep`. The test fails without the change (one model request) and passes with it (two). |
| `npx vitest run packages/core/agent-loop --coverage` | All 450 existing and new tests pass; `agent.ts` stays at 100% statements, branches, functions, and lines. |

## Consequences

- A waking message delivered from a promise reaction at the end of a turn is no longer stranded.
- Retention after an errored or cancelled turn, `inject` without a wake, and maintenance and abort replay behave as before.
