# Review: BUG-3 — `gh:owner/repo` misparsed

**Branch:** `fix/gh-colon-prefix`
**Commit:** `e7f2337a`
**Reviewer:** independent re-audit
**Date:** 2026-06-13
**Verdict:** ✅ Bug confirmed real. Fix is correct, minimal, and safe to merge
as-is. ⚠️ Sibling bugs `gl:` / `bb:` exist with the same root cause but are
intentionally out of scope; tracked as a follow-up.

---

## 1. Severity Assessment: **LOW** (UX / surprise factor)

No correctness or security risk. The bug produces a nonsensical clone URL
that fails immediately with a clean GitHub 404. No data loss, no credential
leak, no silent-bad-state scenario.

The cost is purely user-facing confusion: someone types `gh:owner/repo`
(natural muscle memory from `gh repo clone`), gets a "repository not found"
error mentioning `gh:owner` as the owner, and has to figure out that the
prefix syntax wasn't supported in the first place.

---

## 2. Bug Confirmation (empirical)

I checked out `main`'s `source.go` and ran an internal probe test against
`ParseSource`:

```
"gh:owner/repo"               Type=github CloneURL="https://github.com/gh:owner/repo.git"  Subdir=""           Name="repo"
"gh:owner/repo/sub/dir"       Type=github CloneURL="https://github.com/gh:owner/repo.git"  Subdir="sub/dir"    Name="dir"
"gh:owner/repo.git"           Type=github CloneURL="https://github.com/gh:owner/repo.git"  Subdir=""           Name="repo"
```

The pre-fix parse path:

1. `expandGitHubShorthand("gh:owner/repo")` — no `ado:` match.
2. Skip-known-prefixes — none of `github.com/`, `http://`, `https://`,
   `ssh://`, `file://`, local-path, scp-style match.
3. `Contains(input, "/")` is true; `firstSegment = "gh:owner"` does not
   contain `.` → falls through to `return "github.com/" + input` →
   `"github.com/gh:owner/repo"`.
4. `githubPattern` `^(?:https?://)?github\.com/([^/]+)/([^/]+)(?:/(.+))?$`
   matches → owner = `"gh:owner"` (with embedded colon).

The owner `"gh:owner"` is invalid per
[GitHub username rules](https://docs.github.com/en/site-policy/other-site-policies/github-username-policy)
(alphanumeric and dashes only) — every clone fails with 404.

---

## 3. Fix Correctness

### Code change (`internal/install/source.go:212-215`)

```go
// GitHub CLI shorthand: gh:owner[/repo][/subdir]
if strings.HasPrefix(input, "gh:") {
    return "github.com/" + input[3:]
}
```

Inserted between the existing `ado:` handler and the skip-known-prefixes
block. Mirrors the `ado:` pattern in shape and placement.

### Empirical post-fix probe

```
"gh:owner/repo"               Type=github CloneURL="https://github.com/owner/repo.git"   Subdir=""           Name="repo"
"gh:owner/repo/very/deep/path" Type=github CloneURL="https://github.com/owner/repo.git"  Subdir="very/deep/path"  Name="path"
"gh:owner/repo.git"           Type=github CloneURL="https://github.com/owner/repo.git"   Subdir=""           Name="repo"
"gh:owner"                    ERR: unrecognized source format: github.com/owner       (degenerate, expected)
"gh:"                         ERR: unrecognized source format: github.com/            (degenerate, expected)
"GH:owner/repo"               Type=github CloneURL="https://github.com/GH:owner/repo.git"  (NOT FIXED — case-sensitive)
"gh:Owner/Repo"               Type=github CloneURL="https://github.com/Owner/Repo.git"  Name="Repo"   (case preserved — fine)
```

✅ The three documented cases match the test expectations exactly.
✅ Degenerate inputs (`gh:owner` with no slash) fail loudly rather than
   silently producing garbage URLs — consistent with how `owner` (no `gh:`)
   would fail today.
✅ Case sensitivity matches the existing `ado:` handler. Documented; not a
   regression.

### Why this design choice (expand vs. error) is correct

The bug-hunt-report contract was *"MUST produce an error or be parsed
correctly"*. The fix chose **parse correctly** because:

1. There is an established `gh:` convention from the GitHub CLI ecosystem
   (`gh repo clone owner/repo`); users who type `gh:owner/repo` clearly
   intend GitHub.
2. The codebase already has the `ado:` precedent — symmetric design.
3. Erroring would be safer but throws away latent useful intent; expanding
   uses an obvious, well-defined mapping with zero ambiguity.

Both options satisfy the contract; the chosen one improves UX.

---

## 4. Minimal-Impact Audit

| Concern | Assessment |
|---------|------------|
| Lines changed in production code | 4 lines added (one comment + 3 logic) |
| New imports | None |
| Public API | Unchanged |
| Existing inputs that previously worked | Unchanged — the new branch only intercepts the `gh:` prefix, which previously produced broken URLs anyway |
| Existing inputs that previously errored | Unchanged — none of the skip-known-prefixes inputs hit this branch |
| Side effects on `ado:` handler | None — `gh:` check runs after `ado:`, no prefix overlap |
| Side effects on local-path / SSH / HTTPS detection | None — all those checks run after the `gh:` branch and the branch returns before reaching them |
| Test impact | All install tests pass; new test added |

I confirmed no source files reference the literal `gh:` prefix outside the
fix and the new test. No behavior elsewhere was relying on the broken
parse.

---

## 5. Test Quality

`TestParseSource_GitHubColonPrefix` covers:

✅ Bare `gh:owner/repo` → correct clone URL, no subdir, name = repo
✅ `gh:owner/repo/skills/foo` → subdir threaded through, name = leaf
✅ `gh:owner/repo.git` → `.git` suffix tolerated

Each subtest validates `Type`, `CloneURL`, `Subdir`, `Name`. Good coverage
for the three documented happy paths.

⚠️ Not covered (acceptable):
- Degenerate inputs `gh:` / `gh:owner` that should error.
- Case sensitivity of the prefix (`GH:` falls through to broken behavior —
  not a regression, but no explicit assertion).

A hardening pass could add table entries for these, but the fix's blast
radius is small enough that the existing assertions are sufficient as a
regression guard.

---

## 6. Sibling Bugs Not Fixed (out of scope, intentional)

The original bug report (item I6) bundled three prefixes together:

> `gh:owner/repo`, `gl:owner/repo`, `bb:owner/repo` MUST produce an error or
> be parsed correctly.

I verified `gl:` and `bb:` have the **identical bug** post-fix:

```
"gl:owner/repo"   Type=github CloneURL="https://github.com/gl:owner/repo.git"   ❌ broken
"bb:owner/repo"   Type=github CloneURL="https://github.com/bb:owner/repo.git"   ❌ broken
```

This branch deliberately does **not** address them. Reasons:

1. Unlike `gh:` (which mirrors GitHub CLI), there is no widely-accepted
   `gl:` or `bb:` convention. The right action might be to **error**
   rather than expand, and that's a UX decision better made on its own
   branch with explicit user input.
2. Expanding `gl:owner/repo` to `gitlab.com/owner/repo` would assume the
   public GitLab; users with self-hosted GitLab might get surprised.
3. Same for Bitbucket (Cloud vs. Server).

**Recommendation:** Track as `BUG-3b` — error out cleanly on `gl:` / `bb:`
prefixes with a message pointing to the canonical full URL form. Not
blocking this PR.

---

## 7. Verification Run

```
$ go test ./internal/install/ -run TestParseSource_GitHubColonPrefix -v -count=1
=== RUN   TestParseSource_GitHubColonPrefix
=== RUN   TestParseSource_GitHubColonPrefix/gh:owner/repo_expands_to_GitHub
=== RUN   TestParseSource_GitHubColonPrefix/gh:owner/repo/subdir_expands_to_GitHub_with_subdir
=== RUN   TestParseSource_GitHubColonPrefix/gh:owner/repo.git_expands_to_GitHub_with_.git
--- PASS: TestParseSource_GitHubColonPrefix
--- PASS: TestParseSource_GitHubColonPrefix/gh:owner/repo_expands_to_GitHub
--- PASS: TestParseSource_GitHubColonPrefix/gh:owner/repo/subdir_expands_to_GitHub_with_subdir
--- PASS: TestParseSource_GitHubColonPrefix/gh:owner/repo.git_expands_to_GitHub_with_.git
PASS

$ go test ./internal/install/ -count=1
ok      skillshare/internal/install     0.872s
```

✅ **Reviewed. Bug real (low severity). Fix correct, minimal, safe. No code
changes required on this branch. Sibling `gl:`/`bb:` bugs flagged for a
follow-up branch.**
