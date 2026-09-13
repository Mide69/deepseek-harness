# Agent Note: resource protocol key no longer depends on WHATWG URL host parsing

Status: implemented

## Problem

`protocolOf` ([resources.ts](../../../../packages/client/resources/src/client/resources.ts)) derived a resource address's protocol key from `new URL(address).hostname`. Whether the WHATWG URL parser assigns a host to a non-special scheme's authority (`dsh-resource://<host>/...`) is itself version-dependent: the 2024 URL Standard change allowing non-special schemes to have hosts (whatwg/url#731) shipped in Chrome/Edge 126. On an older Chromium, `new URL('dsh-resource://file/session/x/C:/a.txt').hostname` reads as `''` instead of `'file'`, `protocolOf` returns `undefined`, the `ResourceRegistry` records the address as unresolvable, and every file preview surface (the sidebar Files tab, chat file links, deliverables) fails with no request ever reaching the Host — deterministic, global, and silent, since the failure never leaves the client.

## Decision

Read the protocol key directly from the address string instead of through `new URL(...).hostname`: match the `dsh-resource://` prefix case-insensitively (mirroring the URL parser's own scheme lower-casing), then take the authority substring up to the first `/`, `?`, or `#`. This has no engine-version dependency, since it never asks a URL parser to decide whether a non-special scheme's authority names a host.

## Alternatives considered

**Special-case the affected Chromium range.** Would require detecting the engine/version client-side and branching, adding a maintenance burden tied to a browser release rather than fixing the actual dependency on unspecified parser behavior.

**Keep `new URL(...)` for scheme/format validation, only hand-parse the host.** Rejected: the same parser call that reads `.protocol` is unaffected (scheme lower-casing is unconditional), so the only reason to keep it was habit; hand-parsing both halves is one function, not two.

## Consequences

`protocolOf` no longer depends on `new URL(address).hostname` at all, so its behavior is identical across every engine and version, including the ones this bug was reported against (Chrome ≤ 125). [resources.client.spec.ts](../../../../packages/client/resources/tests/resources.client.spec.ts) adds a case reproducing the exact reported address shape (a Windows drive-letter path under a `dsh-resource://file/...` address) alongside the existing case-insensitive-scheme and empty-host assertions, which still pass unchanged.
