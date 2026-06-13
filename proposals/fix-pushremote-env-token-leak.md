# Fix: Token leak in `PushRemoteWithEnv`

**Bug:** BUG-1 from bug-hunt-report.md
**File:** `internal/git/info.go:499`
**Date:** 2026-06-13

## Root Cause

`PushRemoteWithEnv` used `cmd.CombinedOutput()` + raw
`fmt.Errorf("git push failed: %s", string(out))` when git push failed. The
combined output could contain credential values (via `GIT_CONFIG_KEY_0` env
vars injected by `AuthEnvForRepo`) — those values were returned verbatim
without sanitization. Every other git operation in the same file was already
routed through `install.WrapGitError` → `sanitizeTokens` → `extractGitFatal`;
this one was missed.

## Why the Fix Is Minimal

2-line behavioral change in `PushRemoteWithEnv`:

1. Replaced `cmd.CombinedOutput()` + raw error formatting with `cmd.Stderr`
   capture + `install.WrapGitError(stderr, err, usedToken)`.
2. No new imports needed (`bytes` already imported, `install` already imported,
   `UsedTokenAuth` already exported from `install` package).
3. Dropped the `"git push failed:"` prefix — `WrapGitError`/`extractGitFatal`
   already produce a more informative error.

## Behavior Before/After

| Aspect | Before | After |
|--------|--------|-------|
| stderr capture | `cmd.CombinedOutput()` (mixed stdout/stderr) | `cmd.Stderr = &stderrBuf`, `cmd.Run()` |
| Error formatting | `fmt.Errorf("git push failed: %s", strings.TrimSpace(string(out)))` | `install.WrapGitError(stderrBuf.String(), err, install.UsedTokenAuth(extraEnv))` |
| Token leak | Raw output returned, credential values visible if present | `sanitizeTokens` redacts known token env var values |
| "fatal:" prefix | Retained in error message | Stripped by `extractGitFatal` |
| Auth/SSL error classification | None | `WrapGitError` provides actionable messages for auth failures and SSL errors |
| Edge case (empty stderr, non-zero exit) | Returns "git push failed: <empty>" | Returns exit-code-based message (e.g. "authentication token was rejected") |

## Tests Added

`TestPushRemoteWithEnv_TokenNotInError` in `internal/git/info_test.go`:

1. Creates a test repo, sets `GITHUB_TOKEN` env var.
2. Adds remote on `host.invalid` (RFC 2606 reserved, fails fast).
3. Calls `PushRemoteWithEnv(repo, nil)` — git push fails with connection error.
4. Asserts token value is NOT present in error message.
5. Asserts error does NOT start with `"git push failed:"` (verifies
   `WrapGitError` path was taken, not raw output).
6. Before fix: assertion #5 fails (raw `"git push failed: ..."` returned).
   After fix: both assertions pass.

## Risk / Non-Goals

- **Non-goal:** Fixing `Commit` (BUG-2 from report). Same pattern exists there
  but `Commit` doesn't accept `extraEnv` — token leakage via commit output is
  far less likely since no credentials are injected into commit's environment.
  Separate issue.
- **Risk:** Very low. The change is purely in the error-handling path.
  `WrapGitError` is the same function used by every other git operation in the
  same package and `internal/install`. stdout is now discarded on success
  (previously `CombinedOutput` discarded it too — the return was `nil` on
  success in both cases).
- **No format changes, no refactoring, no scope expansion.**
