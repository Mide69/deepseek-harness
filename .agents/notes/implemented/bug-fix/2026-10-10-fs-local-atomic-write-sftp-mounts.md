# Agent Note: Atomic writes on SFTP-backed mounts

Status: implemented

## Problem

`writeFileAtomic` in [`packages/fs/fs-local/src/fsio.ts`](../../../../packages/fs/fs-local/src/fsio.ts) called `mkdir(directory, { recursive: true })` on the target's parent and then three mode changes on the staging directory and file. Both steps failed on a Windows drive mounted from an SFTP server with sshfs-win ([deepseek-harness#9376](https://github.com/deepseek-ai/deepseek-harness/discussions/9376)), so the model `write` and `edit` tools and the sidebar save aborted before any content was written.

- Node's recursive `mkdir` treats an existing directory as success only when the filesystem answers `EEXIST`. SFTP version 3 has no status for an existing path, so the server answers a generic failure that sshfs maps to `EPERM`. Every write into an existing directory on such a mount therefore failed at the first step.
- `chmod` on the staging directory and `fchmod` on the staged file answered `EINVAL` on the same mount. The code already documents these modes as inert on Windows, where access comes from the DACL, but the failure was fatal.

## Decision

- `ensureDirectory` runs the recursive `mkdir` and accepts `EPERM` or `EACCES` only after `stat` shows the path is a directory. Any other error, a path that is not a directory, and a path that cannot be observed rethrow the original `mkdir` error. The check is by outcome, not platform, because the status mapping belongs to the mount.
- `applyStagingMode` wraps the three staging mode changes and ignores `EINVAL` when the target platform is `win32`. On POSIX every failure stays fatal, because the `0o700` and `0o600` modes are the staging privacy guarantee there. `EINVAL` is the only code reported, so it is the only one ignored.

## Alternatives considered

**Ignore every `mkdir` failure.** A real permission failure on a missing directory would surface later as a staging error that names the wrong path.

**Skip `mkdir` when `stat` says the directory exists.** This adds a `stat` to every write and still races a directory removed between the check and staging. Calling `mkdir` first and consulting `stat` only after a refusal costs nothing on filesystems that behave.

**Tolerate more codes in `applyStagingMode`.** Without a reported mount that returns them, a wider list hides real failures; a later report can extend it.

## Testing

| Evidence | Behaviour |
|---|---|
| [fsio.spec.ts](../../../../packages/fs/fs-local/tests/fsio.spec.ts) `ensureDirectory` | Creates missing parents and accepts an existing directory; accepts injected `EPERM` and `EACCES` for a directory; rethrows them for a file and for an unobservable path; rethrows other codes. |
| [fsio.spec.ts](../../../../packages/fs/fs-local/tests/fsio.spec.ts) `applyStagingMode` | Runs the change; ignores `EINVAL` on `win32`; keeps `EINVAL` fatal on `linux`; keeps other codes fatal on `win32`. |
| `npx vitest run packages/fs/fs-local --coverage` | Suite passes and `fsio.ts` stays at 100% statements, branches, functions, and lines. |

The SFTP mount itself was not available here; the mount behavior comes from the report, and the tests inject the same errno values.

## Consequences

- Writes and edits work on directories of SFTP-backed mounts without changing behavior on filesystems that report `EEXIST`.
- A directory that is genuinely not creatable still fails with the original `mkdir` error.
- Mount settings such as `VolumePrefix` and `FileSecurity` that the report also needed for `realpath` and directory listing stay outside this package.
