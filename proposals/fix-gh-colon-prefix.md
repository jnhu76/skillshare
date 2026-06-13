# Fix: `gh:owner/repo` silently misparsed

**Bug:** BUG-3 from bug-hunt-report.md
**File:** `internal/install/source.go:194-236`
**Date:** 2026-06-13

## Root Cause

`expandGitHubShorthand` has a handler for `ado:` (Azure DevOps) but none for
`gh:` (GitHub CLI shorthand). When input `gh:owner/repo` reaches the owner/repo
detection logic at line 224, `strings.Contains("gh:owner/repo", "/")` is true,
`firstSegment = "gh:owner"` does not contain `"."`, so the function returns
`"github.com/" + "gh:owner/repo"` = `"github.com/gh:owner/repo"`. The colon
ends up in the owner field, producing an invalid GitHub URL.

## Why the Fix Is Minimal

Added a 3-line `gh:` handler in `expandGitHubShorthand`, immediately after the
existing `ado:` handler and before the skip-prefixes section. The handler strips
the `gh:` prefix and delegates the rest to the `github.com/<rest>` expansion
that the fallthrough code would have performed anyway — but without the `gh:`
corrupting the owner segment.

## Behavior Before/After

| Input | Before | After |
|-------|--------|-------|
| `gh:owner/repo` | `https://github.com/gh:owner/repo.git` (invalid) | `https://github.com/owner/repo.git` (valid) |
| `gh:owner/repo/subdir` | `https://github.com/gh:owner/repo.git` (invalid, owner has colon) | `https://github.com/owner/repo.git` with `subdir` |
| `gh:owner/repo.git` | `https://github.com/gh:owner/repo.git` (invalid) | `https://github.com/owner/repo.git` (valid) |

## Tests Added

`TestParseSource_GitHubColonPrefix` in `internal/install/source_test.go`:

- `gh:owner/repo` → CloneURL `https://github.com/owner/repo.git`, Name `repo`
- `gh:owner/repo/subdir` → CloneURL with subdir `skills/foo`, Name `foo`
- `gh:owner/repo.git` → CloneURL with `.git` suffix preserved

## Risk / Non-Goals

- **Non-goal:** Adding `gl:`, `bb:`, or other colon-prefix shorthands. Each
  shorthand needs its own handler (different domain mapping). Scope expansion.
- **Non-goal:** Changing `gh:` to produce an error instead of expanding it.
  Treating `gh:` as a valid GitHub shorthand is consistent with the existing
  `ado:` pattern and is more user-friendly.
- **Risk:** Very low. The handler only activates on explicit `gh:` prefix.
  Non-`gh:` paths are unaffected. No new imports or API changes.
- **No format changes, no refactoring, no scope expansion.**
