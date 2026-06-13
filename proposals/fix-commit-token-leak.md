# Fix: Token leak in `Commit`

**Bug:** BUG-2 from bug-hunt-report.md
**File:** `internal/git/info.go:476-484`
**Date:** 2026-06-13

## Root Cause

`Commit` used `cmd.CombinedOutput()` + raw
`fmt.Errorf("git commit failed: %s", strings.TrimSpace(string(out)))` when
`git commit` failed. The combined output was returned verbatim without
sanitization. Although `Commit` does not accept `extraEnv` (unlike
`PushRemoteWithEnv`), it inherits the process environment. If any token env
var (`GITHUB_TOKEN`, `GH_TOKEN`, `GITLAB_TOKEN`, etc.) is set in the server
process — or if any caller sets such vars before calling `Commit` — then an
error from `git commit` (e.g., from a hook that echoes env) could leak the
token value.

## Why the Fix Is Minimal

3-line behavioral change in `Commit`:

1. Replaced `cmd.CombinedOutput()` + raw error formatting with `cmd.Stderr`
   capture + `install.WrapGitError(stderr, err, false)`.
2. Passing `false` for `tokenAuthAttempted` because `Commit` never explicitly
   injects token auth env vars — the sanitization still runs (via
   `sanitizeTokens` which reads `os.Getenv`), but auth errors are attributed
   to missing setup rather than a rejected token.
3. No new imports needed (`bytes` already imported, `install` already
   imported).
4. Dropped the `"git commit failed:"` prefix — `WrapGitError`/`extractGitFatal`
   already produce a more informative error.

## Behavior Before/After

| Aspect | Before | After |
|--------|--------|-------|
| stderr capture | `cmd.CombinedOutput()` (mixed stdout/stderr) | `cmd.Stderr = &stderrBuf`, `cmd.Run()` |
| Error formatting | `fmt.Errorf("git commit failed: %s", string(out))` | `install.WrapGitError(stderrBuf.String(), err, false)` |
| Token leak | Raw output returned, env var values visible if set | `sanitizeTokens` redacts known token env var values |
| stdout on success | Discarded by `CombinedOutput` (return was `nil`) | Discarded (stdout not captured at all) |
| Auth/SSL error classification | None | `WrapGitError` provides actionable messages |
| Edge case (empty stderr, non-zero exit) | Returns "git commit failed: <empty>" | Returns exit-code-based message |

## Tests Added

`TestCommit_TokenNotInError` in `internal/git/info_test.go`:

1. Creates a test repo (clean, has initial commit).
2. Sets `GITHUB_TOKEN` env var.
3. Calls `Commit(repo, "test")` — git commit fails because nothing is staged
   ("nothing to commit, working tree clean").
4. Asserts token value is NOT present in error message (structural coverage).
5. Asserts error does NOT start with `"git commit failed:"` (verifies
   `WrapGitError` path was taken, not raw output).
6. Before fix: assertion #5 fails (raw `"git commit failed: ..."` returned).
   After fix: both assertions pass.

## Risk / Non-Goals

- **Non-goal:** Accepting `extraEnv` parameter in `Commit`. The current
  callers (`handler_git.go`) don't need it — tokens are set by
  `PushRemoteWithAuth`'s subprocess, not in the server process environment.
  Adding `extraEnv` would be scope expansion and would require changing
  callers.
- **Risk:** Very low. The change is purely in the error-handling path.
  `WrapGitError` is the same function used by every other git operation in the
  same package and `internal/install`. stdout is now discarded on success
  (previously `CombinedOutput` discarded it too — the return was `nil` on
  success in both cases).
- **No format changes, no refactoring, no scope expansion.**
