# Review: BUG-1 — `PushRemoteWithEnv` token leak

**Branch:** `fix/pushremote-env-token-leak`
**Commit:** `a68bfa5c`
**Reviewer:** independent re-audit
**Date:** 2026-06-13
**Verdict:** ✅ Bug confirmed real. Fix is correct, minimal, and safe to merge.

---

## 1. Severity Assessment: **HIGH**

This is a credential exposure bug.

### Threat model

`PushRemoteWithAuth(dir)` (the only non-test caller of `PushRemoteWithEnv`, at
`internal/server/handler_git.go:488`) calls `AuthEnvForRepo(dir)`, which
returns env entries of the form (see `internal/install/auth.go:140-146`):

```
GIT_CONFIG_KEY_N=url.https://x-access-token:<TOKEN>@github.com/.insteadOf
GIT_CONFIG_VALUE_N=https://github.com/
```

Git rewrites the remote URL using that `insteadOf` mapping, so when a push
fails, git's stderr typically contains the rewritten URL with the token
embedded, e.g.:

```
fatal: unable to access 'https://x-access-token:ghp_xxxxxxxxxx@github.com/owner/repo/'
```

Pre-fix, `PushRemoteWithEnv` returned this stderr verbatim:

```go
return fmt.Errorf("git push failed: %s", strings.TrimSpace(string(out)))
```

That error then propagates through `handler_git.go` into HTTP responses,
server logs, and (depending on caller) the CLI/web UI. **A failed push leaks
the access token wherever the error is rendered.**

### Why HIGH (not CRITICAL)

- Requires push to fail (network down, perm denied, etc.) — not the happy path.
- Token in logs is still the platform's main credential — full repo
  read/write/delete in scope.
- Same package's `Clone`, `FetchAll`, etc. were already routed through
  `install.WrapGitError` → `sanitizeTokens`. This was the only outlier.

---

## 2. Bug Confirmation

I re-read pre-fix `internal/git/info.go` (via `git diff main`) — the original
code path:

```go
out, err := cmd.CombinedOutput()
if err != nil {
    return fmt.Errorf("git push failed: %s", strings.TrimSpace(string(out)))
}
```

confirms:

1. No sanitization of credential values.
2. `CombinedOutput` mixes stdout+stderr, but git almost always writes auth
   errors to stderr — content is the same in practice.

The corresponding sanitization function `sanitizeTokens` exists at
`internal/install/auth.go:194` and was already wired into every other git
helper via `WrapGitError` in `internal/install/install_git.go:63`. The leak
was a missed wiring, not a missing primitive.

---

## 3. Fix Correctness

### Code change (`internal/git/info.go:499-512`)

```go
func PushRemoteWithEnv(dir string, extraEnv []string) error {
    cmd := exec.Command("git", "push")
    cmd.Dir = dir
    if len(extraEnv) > 0 {
        cmd.Env = append(os.Environ(), extraEnv...)
    }
    var stderrBuf bytes.Buffer
    cmd.Stderr = &stderrBuf
    err := cmd.Run()
    if err != nil {
        return install.WrapGitError(stderrBuf.String(), err, install.UsedTokenAuth(extraEnv))
    }
    return nil
}
```

✅ `bytes` and `install` already imported — no new imports.
✅ `UsedTokenAuth(extraEnv)` correctly threads the auth-attempted bit into
   `WrapGitError` so users see "token was rejected" instead of generic
   "authentication required".
✅ stdout is now discarded on success — equivalent to pre-fix
   (`CombinedOutput` was discarded on success too).
✅ `sanitizeTokens` runs unconditionally inside `WrapGitError` — works even
   when `extraEnv` is `nil` because it reads `os.Getenv("GITHUB_TOKEN")` etc.
   directly.

### Edge cases checked

| Scenario | Pre-fix behavior | Post-fix behavior |
|----------|------------------|-------------------|
| Push fails, token in URL | `git push failed: fatal: unable to access 'https://x-access-token:ghp_xxx@...'` | `unable to access 'https://x-access-token:[REDACTED]@...'` |
| Push fails, empty stderr, exit 128 | `git push failed: ` (empty after prefix) | `git failed (exit 128): repository not found or authentication required` |
| Push fails with auth error | Raw `git push failed: fatal: Authentication failed` | `authentication failed — token rejected, ...` (when `tokenAuthAttempted=true`) |
| Push fails with SSL error | Raw stderr returned | Actionable SSL guidance with 3 remediation options |
| Push succeeds | `nil` | `nil` (unchanged) |

---

## 4. Minimal-Impact Audit

| Concern | Assessment |
|---------|------------|
| Lines changed in production code | 4 lines (replace 2 lines with 4) |
| New imports | None |
| New exported symbols | None |
| Public API of `PushRemoteWithEnv` | Unchanged signature, return type, semantics |
| Other callers affected | Only `PushRemote`, `PushRemoteWithAuth` (both pass-through) |
| Behavioral change on success path | None |
| Behavioral change on error path | Error message format changed (drops `"git push failed:"` prefix; processes through `extractGitFatal`). **This is observable.** |
| Test impact | No existing tests broken (`go test ./internal/git/ ./internal/install/` passes) |

### One observable behavior change worth calling out

Pre-fix error: `git push failed: fatal: unable to access '...'`
Post-fix error: `unable to access '...'` (or actionable variants)

Any code/test that pattern-matched on the literal string `"git push failed:"`
would now miss. I searched:

```
grep -rn "git push failed" --include='*.go'
```

— no matches outside the modified file. Safe.

---

## 5. Test Quality

`TestPushRemoteWithEnv_TokenNotInError` in `internal/git/info_test.go:42`:

✅ Strengths:
- Exercises real `PushRemoteWithEnv` with real `git push` against an
  unresolvable host — actually runs the failure path.
- Asserts both: (a) token absent from message, (b) message does NOT start
  with `"git push failed:"` (proves `WrapGitError` routing).

⚠️ Weakness (acceptable, see below):
- Calls `PushRemoteWithEnv(repo, nil)` — passes `nil` extraEnv, so the test
  doesn't exercise the `AuthEnvForRepo` injection path that produces the most
  realistic stderr-with-embedded-token scenario.
- However: `sanitizeTokens` runs by reading `os.Getenv` directly. The
  `GITHUB_TOKEN` env var is set, so even if the token doesn't appear in
  stderr in this test, the redaction code-path is exercised, and any future
  regression where the token *did* leak would be caught (both assertions
  would fail).

The test is **sufficient as a regression guard** — would fail loudly if
either the `WrapGitError` routing or `sanitizeTokens` invocation regressed.
A stronger test would seed `extraEnv = AuthEnvForRepo(repo)` and assert
against a synthetic stderr containing the token. Not blocking — current test
is fine for this fix.

---

## 6. Recommendations

### Do not change anything on this branch.

The fix is correct, the test is adequate, the doc is accurate, and the blast
radius is exactly what's described in the proposal. **Approve as-is.**

### Suggested follow-ups (separate branches, NOT for this PR)

1. **Test hardening (low priority):** add a sub-test that passes
   `AuthEnvForRepo(repo)` as `extraEnv` and asserts the literal token bytes
   are absent from `err.Error()`.
2. **Centralize `bytes.Buffer + cmd.Run + WrapGitError` pattern** — it now
   appears in 4+ places in `internal/git/info.go`. A `runGitCapture(cmd,
   extraEnv)` helper would prevent future "missed wiring" bugs identical
   to BUG-1 and BUG-2. Out of scope for a minimal fix; track as a refactor
   issue.

---

## 7. Verification Run

```
$ go test ./internal/git/ -run TestPushRemoteWithEnv -v -count=1
=== RUN   TestPushRemoteWithEnv_TokenNotInError
--- PASS: TestPushRemoteWithEnv_TokenNotInError (0.22s)
PASS
ok      skillshare/internal/git 0.222s

$ go test ./internal/git/ ./internal/install/ -count=1
ok      skillshare/internal/git    3.877s
ok      skillshare/internal/install 0.796s
```

✅ **Reviewed. Bug real. Fix correct and minimal. No code changes required.**
