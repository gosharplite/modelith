# Azure DevOps transport via az rest

Vendoring models from Azure DevOps (`dev.azure.com`) delegates to the `az` CLI
via `az rest` with the canonical Azure DevOps AAD Resource ID, writes content
via `--output-file` for byte fidelity, and enforces clean process termination
via process groups and `WaitDelay = 5s`. Formally supersedes ADR-0015's
"`gh` is the only transport" clause; the rest of ADR-0015 stands.

## Context

ADR-0015 established vendoring as a whole-file copy and designated `gh` as the
sole transport, noting that non-GitHub origins would be addressed when a
concrete requirement arose. Enterprise environments frequently host canonical
domain models in private Azure DevOps Git repositories (`dev.azure.com`).

Adding Azure DevOps support required solving authentication against Azure
Active Directory (Microsoft Entra ID), preserving the zero-network /
zero-credential boundary established in ADR-0011, maintaining byte-level
fidelity for SHA-256 digest verification, and preventing hung subprocesses.

## Decision

**1. Transport delegation via `az rest`.**
`modelith` delegates all Azure DevOps API interactions to the external `az` CLI
(`az rest`), executed directly as an argv array with no shell interpolation.
Like the `gh` transport, `modelith` holds no HTTP client, TLS configuration,
or credentials in binary (ADR-0011). Authentication is handled entirely by
the user's existing `az login` session.

**2. Canonical Azure DevOps AAD Resource ID.**
`az rest` requests pass `--resource 499b84ac-1321-427f-aa17-267ca6975798`.
This well-known Microsoft first-party application ID ensures `az` requests an
OAuth token with the correct Azure DevOps audience, regardless of tenant
configuration or custom domain mappings.

**3. Content fetch byte fidelity via `--output-file`.**
When `az rest` prints response payloads to stdout, it appends a trailing
newline. For YAML models that do not end with a trailing newline (or carry
exact line endings), piping stdout drifts the file contents and breaks
offline SHA-256 digest verification (`# modelith-digest:`). To ensure byte
fidelity with upstream, `fetchContentADO` specifies `--output-file <tempfile>`
and reads the raw bytes back directly.

**4. Commit SHA extraction.**
The commit SHA is extracted via `az rest` querying `value[0].commitId` formatted
as TSV. Empty outputs or `"null"` strings (indicating no commit touching the
specified path at the given ref) are defensively rejected with an actionable
error message.

**5. Process lifecycle, group cancellation, and `WaitDelay`.**
`az` commands frequently spawn background helper processes (token refresh
daemons, credential helpers) that inherit stdout/stderr pipes. If `modelith`
only terminates the direct child on timeout or cancellation, open pipe handles
can hold `cmd.Wait()` indefinitely.
- On Unix (`!windows`), `ExecRunner` sets `Setpgid: true` and sends `SIGKILL`
  to the entire process group (`-pid`).
- On Windows, where POSIX process groups do not exist, `cmd.WaitDelay = 5s`
  bounds the pipe drain wait after child termination.
- Top-level signal trapping in `cmd/modelith/main.go` traps `SIGINT`/`SIGTERM`
  via `signal.NotifyContext`, terminates child processes, emits `"interrupted"`,
  and exits with code `130`.

**6. Legacy domain interception.**
Legacy `*.visualstudio.com` URLs are intercepted during URL parsing with an
actionable error directing the user to copy the URL from `dev.azure.com`.

**7. Supersession.**
This decision formally supersedes the "`gh` is the only transport" clause in
ADR-0015. `dev.azure.com` is a first-class supported origin for `deps import`.

## Consequences

- `modelith deps import` accepts both `github.com` and `dev.azure.com` URLs.
- Provenance headers generated for Azure DevOps imports use `fetch: git` and
  `origin: https://dev.azure.com/<org>/<project>/_git/<repo>`.
- Offline `modelith lint` verifies Azure DevOps vendored files against their
  content digests identically to GitHub vendored files.
- Azure DevOps fetching requires the `az` CLI installed and authenticated via
  `az login`.
