# Agent Note: hooks-claude-code translates a Claude Code tool's own name

Status: implemented

## Problem

`hooks-claude-code` exists to run an unmodified Claude Code hook config (`bridge.spec.ts`'s own description: "unmodified Claude Code command hooks"). `matchesMatcher`'s claude-code literal-pattern branch ([matcher.ts](../../../../packages/experimental/hook-protocol/src/matcher.ts)) is an exact string compare against the pattern's pipe-separated alternatives. The bridge passed `exec.name` — this harness's own registered tool name — as the matcher subject and as the `tool_name` field in the JSON payload a hook script reads from stdin. This harness registers its shell tool as `bash` (lowercase); Claude Code's own convention, and the name a config copied from a real `.claude/settings.json` names, is `Bash` (PascalCase). A matcher written `"matcher": "Bash"` therefore never selected that config group, and a hook script branching on `tool_name === "Bash"` would misread every invocation, both silently: neither path throws or logs on a mismatch, since a matcher that selects nothing is indistinguishable from one deliberately scoped elsewhere.

## Decision

Translate the harness's tool name to Claude Code's own name at the two Claude-Code-facing surfaces (`runPoint`'s matcher subject and `preToolPayload`/`postToolPayload`'s `tool_name` field) through a small explicit map, `claudeCodeToolName`. Only `bash` → `Bash` is mapped, the one tool this session verified has a 1:1 interface equivalent in both products; every other harness tool name passes through unchanged.

## Alternatives considered

**Make `matchesMatcher`'s claude-code literal comparison case-insensitive.** Rejected: that changes shared matcher semantics for every caller (including `hooks-codex`, a different dialect) and every tool, not only the one verified mismatch, and does not fix the `tool_name` payload field a hook script reads independently of the matcher.

**Map every harness tool to a guessed Claude Code equivalent.** Rejected: this harness's tool catalog is not a verified 1:1 match to Claude Code's own (its edit tool, for one, is `str_replace_editor`, a different call interface than Claude Code's `Edit`/`Read` split), so a guessed mapping could hand a hook script a name whose payload shape does not match what that name implies under real Claude Code, which is a worse failure than an unmapped name passing through as itself.

## Consequences

A Claude Code hook config copied unmodified now matches this harness's `bash` tool under its Claude Code name `Bash`, in both the matcher and the `tool_name` payload field. [bridge.spec.ts](../../../../packages/experimental/hooks-claude-code/tests/bridge.spec.ts) adds a regression case; this package's suite requires a real POSIX shell for its hook scripts and is excluded from the default Windows test run (`vitest.config.ts`'s `windowsUnsupportedPackages`), confirmed locally to run reliably only where CI runs it — the new case was verified logically sound against the file's own established pattern and reviewed, not executed, on this platform.
