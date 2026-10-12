# Agent Note: Snapshot git commands do not run repository hooks or fsmonitor

Status: implemented

## Problem

`workspace-changes` snapshots the working tree at each turn start with `add --all --ignore-errors` into a private index and `write-tree`, through `GitRunner.run` in [`packages/deliverables/workspace-changes/src/git.ts`](../../../../packages/deliverables/workspace-changes/src/git.ts). The runner scrubbed the environment (`GIT_CONFIG_COUNT=0` and the credential scrub), but that only cancels configuration passed through the environment. Git still reads the repository's own `.git/config`, and two of its settings start programs: `core.hooksPath` (the `post-index-change` hook runs after the index is written) and `core.fsmonitor`. A project opened in the Web or Desktop host, for example an extracted archive that carries its `.git` directory, therefore ran repository-chosen programs with the user's privileges when a turn started, without any tool call ([deepseek-harness#9382](https://github.com/deepseek-ai/deepseek-harness/discussions/9382)).

Measured with git 2.56 on Windows: an unguarded `git add --all` fired a hook, an fsmonitor program, and a clean filter configured in the repository; `git write-tree` fired the hook and the fsmonitor program.

## Decision

`GitRunner.run` prefixes every command with `-c core.hooksPath=/dev/null -c core.fsmonitor=false`, the same pair [`packages/experimental/worktree/src/index.ts`](../../../../packages/experimental/worktree/src/index.ts) already puts on its git calls. The scope is the runner, not the working directory: it covers the commands the harness starts for its own bookkeeping, while a `git commit` that a user asks for through the shell tool keeps the hooks they installed.

With the pair in place the same fixture ran no hook and no fsmonitor program. A clean filter still ran.

## Alternatives considered

**Also neutralize clean filters with `--attr-source`.** Pointing attributes at the empty tree stops an in-tree `.gitattributes` filter, but a filter selected by `.git/info/attributes` still ran in the same measurement, and a repository that ships its `.git` directory controls that file. It would also change what a snapshot records for repositories that rely on attribute-driven conversion. The residual filter risk is recorded in the package README instead of claimed as fixed.

**Harden the plugin-manager `git ls-remote` call.** The report listed it as a second caller of the same channel. `ls-remote` writes no index, and the same fixture ran no hook and no fsmonitor program for it, so it is not part of this channel.

**Run git in a directory outside the repository.** Snapshots need the work tree, so the working directory cannot move.

## Testing

| Evidence | Behaviour |
|---|---|
| [git.spec.ts](../../../../packages/deliverables/workspace-changes/tests/git.spec.ts) `does not run repository-configured hooks or fsmonitor programs` | A repository configured with `core.hooksPath` and `core.fsmonitor` writing marker files. An unguarded `git add` first produces both markers, which proves the fixture fires; after clearing them, `locateGitWorkspace` and `snapshotTree` through `GitRunner` produce none. The test fails without the change (both markers present) and passes with it. |
| `npx vitest run packages/deliverables/workspace-changes` | The new test passes. Five other tests in the package fail on the machine used because its home directory is itself a git repository and they expect no repository; they fail identically without this change. |

## Consequences

- Opening a project no longer lets its repository configuration run a hook or an fsmonitor program through the turn-start snapshot.
- A repository-configured `filter.<name>.clean` command still runs during the snapshot, and `core.splitIndex` and git-lfs behavior is unchanged; the README lists the filter case under known limitations.
- Repositories whose workflow depends on a `post-index-change` hook or fsmonitor firing during harness snapshots lose that effect, which is the intended isolation.
