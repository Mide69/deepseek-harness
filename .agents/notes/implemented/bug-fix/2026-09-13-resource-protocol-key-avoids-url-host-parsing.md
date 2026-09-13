# Agent Note: dsh-resource address parsing no longer depends on WHATWG URL host parsing

Status: implemented

## Problem

Two client functions derived structure from a `dsh-resource://<host>/<path>` address through `new URL(...)`: `protocolOf` ([resources.ts](../../../../packages/client/resources/src/client/resources.ts)) read `.hostname` for the protocol key, and `pathOf` ([tab-registry.ts](../../../../packages/client/ui-sidebar-right/src/client/tab-registry.ts)) read `.pathname` for glob matching. Whether the WHATWG URL parser assigns a host to a non-special scheme's authority — and so excludes it from `.pathname` — is itself engine- and version-dependent (the URL Standard changed to allow non-special schemes to have hosts, and Chromium's rollout of that behavior spans multiple release lines). On an engine that has not shipped it, `new URL('dsh-resource://file/session/x/C:/a.txt').hostname` reads as `''` instead of `'file'`, and `.pathname` includes the host in the opaque path instead of excluding it. `protocolOf` then returns `undefined`, so the `ResourceRegistry` records the address as unresolvable and every file preview surface (the sidebar Files tab, chat file links, deliverables) fails with no request ever reaching the Host; `pathOf` returns the wrong string, so sidebar-right tab types registered against a path-only glob (`*.png`, `*.md`) never match a `dsh-resource://` address on the same engines. Both failures are deterministic, global to the affected engine, and silent, since neither ever reaches the Host to log anything.

## Decision

Read both structures directly from the address string instead of through the URL parser's authority/path split. `protocolOf` matches the `dsh-resource://` prefix case-insensitively (mirroring the parser's own unconditional scheme lower-casing), then takes the authority substring up to the first `/`, `?`, or `#`. `pathOf` matches a generic `scheme://authority` prefix (RFC 3986 scheme grammar, authority up to the same three delimiters) and returns everything after it. Neither asks a URL parser to decide whether a non-special scheme's authority names a host, so neither has an engine-version dependency.

## Alternatives considered

**Special-case the affected Chromium range.** Would require detecting the engine/version client-side and branching, adding a maintenance burden tied to a browser release rather than fixing the actual dependency on unspecified parser behavior.

**Keep `new URL(...)` for scheme/format validation, only hand-parse the host/path.** Rejected for `protocolOf`: the same parser call that reads `.protocol` is unaffected (scheme lower-casing is unconditional), so the only reason to keep it was habit. Not applicable to `pathOf`, which has no other reason to construct a `URL` at all once the authority is stripped by hand.

## Consequences

Neither function depends on the URL parser's non-special-scheme host/path behavior, so both are identical across every engine and version, including the ones this was reported against. [resources.client.spec.ts](../../../../packages/client/resources/tests/resources.client.spec.ts) and [tab-registry.client.spec.ts](../../../../packages/client/ui-sidebar-right/tests/tab-registry.client.spec.ts) each add a case reproducing the exact reported address shape (a Windows drive-letter path under a `dsh-resource://file/...` address) alongside their existing suites, which still pass unchanged.
