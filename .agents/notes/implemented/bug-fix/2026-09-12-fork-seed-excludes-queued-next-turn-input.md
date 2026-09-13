# Agent Note: session.fork excludes queued next-turn input from its seed

Status: implemented

## Problem

`SessionCommandController.fork` ([commands.ts](../../../../packages/api/session-controller/src/commands.ts)) cuts its seed by scanning forward from the anchored `turn/end` to the next `turn/start`, so that trailing housekeeping events between turns (`request/header` carrying the latest model selection, `session/title`, and similar) still land in the child's seed. That scan had no exception for `agent/inbox/spliced`: when the parent had already queued its next submission before the fork, agent-loop's inbox projection records that submission as a spliced insert into the pending `next-turn` list before the next turn opens. The scan swept that event into the seed along with everything else, so a fork taken while a submission was queued (or in flight) started the child with the parent's next task pre-loaded as pending input, and the child's first turn re-ran work the parent had not executed yet on its own.

## Decision

Stop the forward scan at an `agent/inbox/spliced` event whose `inserted` array is non-empty, instead of only at the next `turn/start`. Such a splice describes a message queued for a turn that has not started — the parent's future work, not history of the completed turn — so the child must never inherit it. Every other event type between the boundary and the next `turn/start` keeps carrying forward unchanged, preserving the model-selection inheritance the existing `installs the latest logged model selection before the child can run` test asserts.

## Alternatives considered

**Cut exactly at `boundary.seq + 1`, dropping the forward scan entirely.** Matches the function's own doc comment ("one completed-turn prefix") and is the simplest fix, but it also drops the trailing `request/header` a session logs after `turn/end` to record a model-selection change, which an existing test depends on the child inheriting. Rejected: fixes the reported bug by breaking a documented, tested case the scan exists to serve.

**Filter `agent/inbox/spliced` out of the sliced range instead of stopping the scan there.** Would keep scanning past a spliced insert to still pick up any further housekeeping events after it, at the cost of a `seq` gap inside the seed array. No consumer of a forked seed currently tolerates non-contiguous `seq` values, and no report needs a case where a housekeeping event follows a queued-input splice. Rejected as unproven complexity for an unobserved shape.

## Consequences

A fork taken while the parent has a submission queued (from the in-conversation branch action or the sidebar's fork action) produces a child whose seed ends at the completed turn's `turn/end`, with no pending input carried over; the child's first turn is its own. [session-fork.host.spec.ts](../../../../packages/api/session-controller/tests/session-fork.host.spec.ts) adds a regression case reproducing the exact log shape reported (`turn/end` → `agent/inbox/spliced` with a queued prompt → `turn/start`) and asserts the splice is absent from the child's seed, alongside the pre-existing case that still exercises the trailing-event carry-forward this fix preserves.
