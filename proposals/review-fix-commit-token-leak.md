# Review: BUG-2 — `Commit` token leak (with required code adjustment)

**Branch:** `fix/commit-token-leak`
**Original commit:** `0153d449`
**Reviewer:** independent re-audit
**Date:** 2026-06-13
**Verdict:** ✅ Bug confirmed real (defense-in-depth class). ⚠️ Original fix
introduced a UX regression — **adjusted on this branch**. Now correct.

---

## 1. Severity Assessment: **LOW–MEDIUM** (defense-in-depth)

Lower than BUG-1 because `Commit` does not inject auth env vars and the
credential leak surface is narrow.

### Realistic leak paths into `git commit` output

| Vector | Likelihood | Reachable? |
|--------|------------|-----------|
| Token embedded in commit message text | Plausible (user/automation paste) | yes — `msg` is echoed by some hooks/messages |
| `pre-commit` / `commit-msg` hook prints token-bearing diagnostics | Rare | yes |
| `core.editor` invocation printing tokens | Not reachable — we always pass `-m` | no |
| Auth-related stderr (insteadOf URL rewrites) | Not reachable — `Commit` doesn't set `cmd.Env` | no |

So `Commit` is **not** a high-yield exfiltration channel like `PushRemoteWithEnv`,
but the same hardening pattern (`WrapGitError` + `sanitizeTokens`) costs almost
nothing and brings the helper in line with every other git wrapper in the
package. Worth fixing as a consistency / defense-in-depth measure.

`handler_git.go:482` writes the error verbatim to the HTTP response body —
so any token that did appear would be exposed to the API client.

---

## 2. Bug Confirmation

Pre-fix code:
```go
out, err := cmd.CombinedOutput()
if err != nil {
    return fmt.Errorf("git commit failed: %s", strings.TrimSpace(string(out)))
}
```
Same anti-pattern as BUG-1 — raw command output returned without
`sanitizeTokens` running. Confirmed by `git diff main` against pre-fix HEAD.

---

## 3. Fix Correctness — and the Regression Found in Review

### Original commit (`0153d449`) — had a regression

```go
var stderrBuf bytes.Buffer
cmd.Stderr = &stderrBuf
err := cmd.Run()
if err != nil {
    return install.WrapGitError(stderrBuf.String(), err, false)
}
```

Problem: `git commit` writes its **most common diagnostic to stdout, not
stderr**. Verified empirically:

```
$ git commit -m test (clean repo, nothing staged)
exit_code = 1
stderr    = ""
stdout    = "On branch master\nnothing to commit, working tree clean\n"
```

With stderr-only capture, the post-fix error became:
```
git failed (exit 1): command error — check network connectivity and repository URL
```

That message:
1. Loses the actionable diagnostic (`nothing to commit`).
2. **Actively misleads** — mentions "network connectivity" for a purely
   local commit failure.

`handler_git.go:482` returns this string in HTTP responses, so the regression
is observable to API clients, the web UI, and ops logs.

The original test `TestCommit_TokenNotInError` happened to pass because it
only asserted "token absent" and "no `git commit failed:` prefix" — neither
caught the diagnostic loss.

### Adjustment applied on this branch

Both stdout and stderr are now captured into one buffer:

```go
var outBuf bytes.Buffer
cmd.Stdout = &outBuf
cmd.Stderr = &outBuf
err := cmd.Run()
if err != nil {
    return install.WrapGitError(outBuf.String(), err, false)
}
```

Behavior matrix after adjustment:

| Failure mode | Stream | Pre-fix error | Post-original-fix | Post-review-adjustment |
|---|---|---|---|---|
| Nothing to commit | stdout | `git commit failed: nothing to commit, working tree clean` | `git failed (exit 1): command error — check network connectivity...` ❌ | `nothing to commit, working tree clean` ✅ |
| Not a git repo | stderr | `git commit failed: fatal: not a git repository...` | `not a git repository...` ✅ | `not a git repository...` ✅ |
| Pre-commit hook fail | stderr (usually) | raw output | sanitized via `WrapGitError` ✅ | sanitized ✅ |
| Token in commit msg/output | either | leaked verbatim ❌ | redacted (when in stderr) ⚠️ | redacted (any stream) ✅ |

Test extended with a third assertion:
```go
if !strings.Contains(err.Error(), "nothing to commit") {
    t.Errorf("expected diagnostic 'nothing to commit' in error, got: %v", err)
}
```
This fails on the original `0153d449` and passes on the adjusted code —
locking the regression closed.

### Why merging stdout+stderr is safe

- `WrapGitError`/`extractGitFatal` is line-oriented — it tolerates additional
  non-`fatal:` lines (it just returns them when no fatal line is found).
- `sanitizeTokens` runs unconditionally on the merged buffer.
- On success path, both buffers are silently dropped (same as pre-fix
  `CombinedOutput`). No behavior change on success.
- This matches what `cmd.CombinedOutput()` was effectively doing before; we
  just no longer rely on `CombinedOutput` (which behaves identically).

---

## 4. Minimal-Impact Audit (post-adjustment)

| Concern | Assessment |
|---------|------------|
| Lines changed in production code | 6 lines (replace 2 with 4 + comment) |
| New imports | None (`bytes` and `install` already imported) |
| Public API of `Commit` | Unchanged signature/return type |
| Other callers affected | `handler_git.go:401` and `:482` — both consume `err.Error()` directly; receive cleaner messages |
| Behavior on success | Unchanged (`nil`) |
| Behavior on error | Error message format changed: drops `"git commit failed:"` prefix; processes through `WrapGitError`. Diagnostics preserved. |
| Existing tests broken | None (`./internal/git`, `./internal/install`, `./internal/server` all pass) |

I searched for `"git commit failed"` callers/matchers — none outside the
modified file. No external regression risk.

The `tokenAuthAttempted` flag is hardcoded to `false` because `Commit`
never sets `cmd.Env` and therefore never uses token auth env. If a future
change adds an `extraEnv` parameter to `Commit`, this flag should be
recomputed via `install.UsedTokenAuth(extraEnv)` (mirror of `PushRemoteWithEnv`).
Documented in the new comment.

---

## 5. Test Quality (post-adjustment)

`TestCommit_TokenNotInError` now asserts three things:
1. ✅ Token value not in error message (redaction).
2. ✅ Error message does not start with `"git commit failed:"` (`WrapGitError` routing).
3. ✅ Error message contains `"nothing to commit"` (diagnostic preservation).

Sufficient regression guard. A stronger test would inject a token into the
commit message body and a hook that prints it; out of scope for a minimal fix.

---

## 6. Recommendations

### What I changed on this branch (already applied)

1. **`internal/git/info.go`** — capture stdout+stderr into one buffer instead
   of stderr only.
2. **`internal/git/info_test.go`** — add the `"nothing to commit"` assertion
   to lock in the diagnostic preservation.

### What was already fine

- The choice of `tokenAuthAttempted=false` in `WrapGitError`.
- The proposal doc captures the rationale; only minor inaccuracy is that the
  test does not assert the most realistic leak vector (commit message
  containing a token). Acceptable for a defense-in-depth fix.

### Non-goals (out of scope)

- Adding an `extraEnv` parameter to `Commit` — would require touching
  `handler_git.go` callers; not needed today.
- A shared `runGitCapture` helper to dedupe the `bytes.Buffer + cmd.Run +
  WrapGitError` pattern across the file — covered in the BUG-1 review as
  a follow-up.

---

## 7. Verification Run (post-adjustment)

```
$ go test ./internal/git/ -run TestCommit -v -count=1
=== RUN   TestCommit_TokenNotInError
--- PASS: TestCommit_TokenNotInError (0.03s)
PASS
ok      skillshare/internal/git 0.041s

$ go test ./internal/git/ ./internal/install/ ./internal/server/ -count=1
ok      skillshare/internal/git    3.992s
ok      skillshare/internal/install 1.545s
ok      skillshare/internal/server 1.105s

$ gofmt -l internal/git/info.go internal/git/info_test.go
(no output)

$ go vet ./internal/git/...
(no output)
```

✅ **Reviewed. Bug real (low-medium severity). Original fix had a UX
regression — corrected on this branch. Now correct, minimal, and safe.**
