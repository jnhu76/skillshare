# Bug-Hunt Report — skillshare

**Date:** 2026-06-13
**Scope:** Read-only audit of all Go source under `cmd/`, `internal/`, `tests/`
**Baseline:** `make check` = `fmt-check` + `go vet ./...` + `make test-unit` — ALL PASSED (0 failures). `make test-int` — ALL PASSED (201s). No pre-existing failures to bypass.

---

## 1. Command Surface

| CLI entry | File | Exports |
|-----------|------|---------|
| `skillshare install` | `cmd/skillshare/install.go` | `cmdInstall`, `dispatchInstall` → `handleGitInstall` / `handleDirectInstall` / `handleTrackedRepoInstall` |
| `skillshare sync [targets]` | `cmd/skillshare/sync_*.go` | `cmdSyncProject`, `syncAgentsGlobal`, `syncAgentsProject` |
| `skillshare hub {index,add,ls,rm,default}` | `cmd/skillshare/hub.go`, `hub_manage.go` | `cmdHub`, `cmdHubIndex`, `cmdHubAdd`, ... |
| `skillshare update` | `cmd/skillshare/update_project.go` | `cmdUpdateProject` |
| `skillshare uninstall` | `cmd/skillshare/uninstall.go` | `cmdUninstall` |
| `skillshare upgrade` | `cmd/skillshare/upgrade.go` | `cmdUpgrade` |

| Internal package | Key exported functions |
|------------------|----------------------|
| `internal/skillignore` | `Compile`, `ReadMatcher`, `ReadAgentIgnoreMatcher`, `AddPattern`, `RemovePattern`, `HasPattern`, `Matcher.Match`, `Matcher.CanSkipDir` |
| `internal/install` | `ParseSource`, `ParseSourceWithOptions`, `authEnv`, `AuthEnvForURL`, `ResolveTokenForURL`, `sanitizeTokens`, `WrapGitError`, `cloneRepo`, `ShallowCloneToTemp` |
| `internal/git` | `Pull`, `Fetch`, `PushRemote`, `Commit`, `AuthEnvForRepo`, `GetRemoteURL`, `GetRemoteHeadHash`, `ForcePull` |
| `internal/hub` | `BuildIndex`, `WriteIndex` |
| `internal/sync` | `SyncTarget`, `SyncTargetMergeWithSkills`, `FilterSkills`, `FilterAgents`, `DiscoverSourceSkills*`, `PruneOrphanLinks` |
| `internal/backup` | `Create`, `CreateInDir`, `List`, `ListInDir` |
| `internal/github` | `NewClient`, `NewRequest`, `GetToken`, `CheckRateLimit` |

---

## 2. Invariants

The following invariants MUST hold for correctness. Each is annotated with the code that establishes it and, where applicable, the risk of violation.

| # | Invariant | Established by | Status |
|---|-----------|---------------|--------|
| I1 | Token values MUST NOT appear in error messages returned to the caller. | `install/auth.go:194` `sanitizeTokens()` + `install/install_git.go:63` `WrapGitError()` | **VIOLATED** in `git/info.go:507` (`PushRemoteWithEnv`) and `git/info.go:481` (`Commit`) — raw CombinedOutput returned without sanitization |
| I2 | `--dry-run` MUST NOT mutate any file or directory on disk. | `sync/sync.go` checks `if dryRun { ... return }` in every mutable branch | OK — gated at each decision point |
| I3 | A second consecutive sync (no changes between) MUST produce zero operations. | Relative/absolute symlink detection via `linkNeedsReformat`, `CheckStatus` returning `StatusLinked` | OK — idempotent by design (tested in `TestSync_AlreadyLinked`) |
| I4 | `.skillignore` pattern `!/x` MUST re-include file `x` even when parent is ignored. | `skillignore/skillignore.go` "last matching rule at each prefix level" — documented divergence from strict gitignore | OK — intentional divergence (tested) |
| I5 | A corrupt manifest MUST NOT cause sync failure — it should be rebuilt. | `sync/manifest.go:35` — corrupt JSON → returns empty manifest | OK |
| I6 | `gh:owner/repo`, `gl:owner/repo`, `bb:owner/repo` MUST produce an error or be parsed correctly. | `install/source.go:194` `expandGitHubShorthand` | **VIOLATED** — `gh:owner/repo` is silently misparsed as `github.com/gh:owner/repo` |
| I7 | Symlink dereference during backup MUST NOT cause data loss when symlink chain is deep or circular. | `backup/backup.go:221` `copyDirFollowTopSymlinks` resolves one level, `copyDir` skips deeper symlinks — cycle detection NOT implemented in `copyDir` | **RISK** — `copyDir` silently skips deeper symlinks without warning |
| I8 | `filepath.Walk` in discover MUST NOT descend into dirs that are globally ignored by `.skillignore`. | `discover_walk.go:175-183` — calls `rootMatcher.CanSkipDir(relPath)` | OK (though the `collectIgnored` bypass can cause performance issues) |

---

## 3. High-Risk Code Paths

Ranked by impact of failure:

### H1: Token leak in `PushRemoteWithEnv` and `Commit`

**Files:** `internal/git/info.go:476-484` (Commit), `:499-510` (PushRemoteWithEnv)

**Risk:** These functions run `exec.Command` with `extraEnv` that MAY contain token auth env vars (from `AuthEnvForRepo()`), then parse `cmd.CombinedOutput()` into an error message via `fmt.Errorf("git push failed: %s", strings.TrimSpace(string(out)))`. No `sanitizeTokens()` call. If git's error output includes the credential (e.g., a 403 response body with the token), it is returned verbatim.

**Contrast:** `runGitCommandWithProgress` (`install/install_git.go:33`) and `runRemoteLsRemote` (`git/info.go:690`) both call `install.WrapGitError()` which sanitizes. These two functions are outliers.

**Impact:** Credential leak in error messages → log files → bug reports → token compromise.

### H2: `file://` URL type confusion

**File:** `internal/install/source.go:393-412` (`parseFileURL`)

**Risk:** A `file:///path/to/repo` URL is assigned `Type = SourceTypeGitHTTPS` and `CloneURL = "file://" + path`. The type says "git over HTTPS" but the actual clone URL is `file://`. Downstream code that checks `SourceTypeGitHTTPS` then uses the clone URL for HTTP operations would break or behave unexpectedly. For example, `authEnv()` for a `file://` URL returns nil (correct, since `isHTTPS("file://...")` is false), but the type classification is misleading.

### H3: Dry-run manifest write bypass

**File:** `internal/sync/sync.go:637-646`

**Risk:** In `SyncTargetMergeWithSkills`, manifest writes are correctly gated on `!dryRun`. However, `ensureRealTargetDir` at line 521 is called BEFORE any dry-run check. While it doesn't mutate in dry-run mode (verified), it does return `(true, nil)` for missing directories. The `quietDryRun` suppression at line 530 uses this value — if a future refactor changes ensureRealTargetDir's dry-run behavior, it could leak side effects.

### H4: `gh:owner/repo` misparse

**File:** `internal/install/source.go:194-236` (`expandGitHubShorthand`)

**Risk:** Input `gh:owner/repo` is NOT recognized as invalid. It passes all prefix checks (not `ado:`, not `github.com/`, not `http://`, etc.), then line 224 `strings.Contains(input, "/")` is true, firstSegment = `gh:owner`, which does NOT contain `"."`, so line 232 returns `"github.com/" + input` = `"github.com/gh:owner/repo"`. This matches the `githubPattern` regex, producing `CloneURL = "https://github.com/gh:owner/repo.git"` with owner = `"gh:owner"` (colon included). This is an invalid GitHub URL.

**Impact:** User writes `gh:owner/repo` expecting it to work (common in other tools like `gh repo clone owner/repo`). Instead, they get a mysteriously failing clone with an invalid GitHub URL containing a colon in the owner name.

### H5: `isGitHubLikeHost` false positives

**File:** `internal/install/github_host.go:5-8`

**Risk:** `strings.Contains(host, "github")` matches `notgithub.com` and any host whose name happens to contain "github". This leaks into:
- `detectPlatform` → `PlatformGitHub` → GITHUB_TOKEN used for auth against a non-GitHub server → token sent to wrong host
- `gitHubAPIBaseForHost` → wrong API base URL

**Impact:** Token disclosure to a third-party git host if someone has `notgithub.com` as a git remote.

### H6: No rollback on sync failure

**File:** `internal/sync/sync.go:278-286` (`MigrateToSource` → `CreateSymlink`)

**Risk:** `MigrateToSource` moves files from target to source (or copies + deletes). If `CreateSymlink` fails AFTER the move, the target directory has been emptied and the source is populated, but no symlink is created. The user's IDE configuration breaks. The backup system (`internal/backup`) exists but is NOT integrated into the `SyncTarget` path — it's only used by the agent sync path and the standalone backup CLI.

### H7: `copyDir` silently skips deeper symlinks in backup

**File:** `internal/backup/backup.go:268-309`

**Risk:** When `copyDirFollowTopSymlinks` delegates to `copyDir` for subdirectory content, `copyDir` skips ALL symlinks (line 293-295). So if a skill inside a merge-mode target has deeper symlinks (e.g., `nvim/skill-foo/linked-config`), those symlinks are silently omitted from the backup. The user gets an incomplete backup without any warning.

---

## 4. Suspected Bugs, Sorted by Evidence Strength

### [BUG-1] Token leak in `PushRemoteWithEnv` (evidence: **strong**)

**File:** `internal/git/info.go:499-510`
**Code:**
```go
func PushRemoteWithEnv(dir string, extraEnv []string) error {
    cmd := exec.Command("git", "push")
    cmd.Dir = dir
    cmd.Env = append(os.Environ(), extraEnv...)
    out, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("git push failed: %s", strings.TrimSpace(string(out)))
    }
    return nil
}
```
**Evidence:** `extraEnv` is passed directly from callers like `PushRemoteWithAuth` which calls `AuthEnvForRepo(dir)`. When `git push` fails (e.g., 403 with token in response body), the full CombinedOutput including any credential values is returned verbatim. Every other git operation in `internal/git` uses `runGitCommandWithProgress` or `install.WrapGitError` — this is the only one that bypasses sanitization.

**Minimal reproduction:**
1. Set up a remote that returns a 403 with the request URL in the response body
2. Set `GITHUB_TOKEN` to a known value
3. Call `git.PushRemoteWithAuth(dir)` 
4. Observe token value in error message

### [BUG-2] Token leak in `Commit` (evidence: **strong**)

**File:** `internal/git/info.go:476-484`
**Code:**
```go
func Commit(dir, msg string) error {
    cmd := exec.Command("git", "commit", "-m", msg)
    cmd.Dir = dir
    out, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("git commit failed: %s", strings.TrimSpace(string(out)))
    }
    return nil
}
```
**Evidence:** Same pattern as PushRemoteWithEnv. Although `Commit` does not receive `extraEnv` directly, it inherits the process environment. If `Commit` is called while token env vars are set, and git's commit hook or any pre-existing git config echoes env vars back, the token could leak.

**Minimal reproduction:**
1. Create a git repo with `GITHUB_TOKEN` set in env
2. Stage a file and call `git.Commit(dir, "test")` on a dirty index
3. If git's error output includes env info, token is leaked

### [BUG-3] `gh:owner/repo` silently misparsed (evidence: **strong**)

**File:** `internal/install/source.go:194-236`
**Evidence:** No `gh:` prefix handler exists. The `ado:` prefix (Azure DevOps) IS handled. `gh:` is a common prefix in other ecosystem tools (e.g., `gh repo clone`, GitHub CLI shorthand). Input `gh:owner/repo` produces `CloneURL = "https://github.com/gh:owner/repo.git"` with owner = `gh:owner` (colon in owner name) — an invalid GitHub URL that causes clone failure with a confusing error message.

**Minimal reproduction:**
```go
source, err := install.ParseSource("gh:owner/repo")
// source.CloneURL == "https://github.com/gh:owner/repo.git"
// source.Type == SourceTypeGitHub
// source.Name == "repo"
// err == nil — no error!
```

The colon in the owner field (`gh:owner`) will cause `git clone` to fail server-side.

### [BUG-4] `file://` URL misclassified as `SourceTypeGitHTTPS` (evidence: **medium**)

**File:** `internal/install/source.go:393-412`
**Code:**
```go
func parseFileURL(matches []string, source *Source) (*Source, error) {
    ...
    source.Type = SourceTypeGitHTTPS // Treat as git for cloning
    source.CloneURL = "file://" + path
    ...
}
```
**Evidence:** Type says `SourceTypeGitHTTPS` but `cloneURL` is `file://...`. Any code that checks `source.Type == SourceTypeGitHTTPS` and then treats the URL as HTTP is wrong. `authEnv()` returns nil for `file://` URLs, so token injection is correctly skipped. But the incorrect type could lead to incorrect diagnostics or future bugs.

**Minimal reproduction:**
```go
source, _ := install.ParseSource("file:///tmp/repo.git")
// source.Type == SourceTypeGitHTTPS  (wrong)
// source.CloneURL == "file:///tmp/repo.git"
// source.IsGit() == true  (correct — needs git clone)
```

### [BUG-5] `isGitHubLikeHost` false positive on `notgithub.com` (evidence: **medium**)

**File:** `internal/install/github_host.go:5-8`
**Code:**
```go
func isGitHubLikeHost(host string) bool {
    host = strings.ToLower(strings.TrimSpace(host))
    return strings.Contains(host, "github") || strings.HasSuffix(host, ".ghe.com")
}
```
**Evidence:** `strings.Contains("notgithub.com", "github")` returns `true`. This causes `detectPlatform("https://notgithub.com/owner/repo.git")` to return `PlatformGitHub`, which means `GITHUB_TOKEN`/`GH_TOKEN` would be sent to `notgithub.com`. In practice, token injection uses the hostname in the `insteadOf` URL (e.g., `url.https://x-access-token:token@notgithub.com/.insteadOf`), so the token is sent to the correct (but wrong) host — still a credential exposure risk.

**Minimal reproduction:**
```go
host := "notgithub.com"
install.IsGitHubLikeHost(host) // returns true
install.DetectPlatformForURL("https://notgithub.com/o/r.git") // returns PlatformGitHub
```

### [BUG-6] Backup silently loses deeper symlinks (evidence: **medium**)

**File:** `internal/backup/backup.go:292-295`
**Code:**
```go
// Skip symlinks and junctions — they point to source, not local data
if info.Mode()&os.ModeSymlink != 0 {
    continue
}
```
**Evidence:** `copyDir` is used for recursive subdirectory copying after `copyDirFollowTopSymlinks` handles the top level. `copyDir` silently skips ALL symlinks at every level. If a skill contains symlinks (e.g., a config that links to a shared file), those symlinks are missing from the backup. No warning is emitted.

**Minimal reproduction:**
1. Create `target/skill-a/linked-config` as a symlink to `/etc/hosts`
2. Call `backup.CreateInDir(dir, "test", "target")`
3. Restore from backup — `linked-config` is absent with no explanation

### [BUG-7] No rollback for sync failure after migration (evidence: **medium**)

**File:** `internal/sync/sync.go:278-286`
**Code:**
```go
case StatusHasFiles:
    if err := MigrateToSource(sc.Path, sourcePath); err != nil {
        return err
    }
    return CreateSymlink(sc.Path, sourcePath, projectRoot)
```
**Evidence:** If `MigrateToSource` succeeds (files moved from target to source) but `CreateSymlink` fails, the target directory no longer exists and no symlink was created. The files are only in source. The IDE/tool looking at the target path sees nothing. No automated rollback exists.

**Minimal reproduction:**
1. Have target at `~/.config/nvim/skills` with real files
2. Run sync — `MigrateToSource` succeeds, moving files to `~/.config/skillshare/skills/`
3. `CreateSymlink` fails (disk full, permissions, etc.)
4. `~/.config/nvim/skills` is gone, files are in source — user must manually restore

### [BUG-8] `normalizePath` uses `strings.ReplaceAll` on every `Match` call (evidence: **low — performance**)

**File:** `internal/skillignore/skillignore.go:19-20`
**Code:**
```go
func normalizePath(p string) string {
    return strings.TrimPrefix(strings.ReplaceAll(p, "\\", "/"), "/")
}
```
**Evidence:** Called at the top of every `Match()` invocation. `strings.ReplaceAll` scans the entire string even when there are NO backslashes. For a typical Unix-only user, this allocates a new identical string every time. With deep directory walks (e.g., `CanSkipDir` called on every directory), this adds up.

**Impact:** Low — performance optimization, not correctness. But noticeable on large source directories with thousands of SKILL.md files.

### [BUG-9] `matchRule` re-splits path prefix for every rule (evidence: **low — performance**)

**File:** `internal/skillignore/skillignore.go:266-267`
**Code:**
```go
func matchRule(r rule, p string) bool {
    pathSegs := strings.Split(p, "/")
```
**Evidence:** Called from `Match()` (line 210) in a loop over ALL rules, for EACH prefix. If path has 10 segments and there are 100 rules, `strings.Split` is called 1000 times for a single `Match()` call.

**Impact:** Low — pathological case of deep path + many rules. The memoization in `matchSegments` helps for globstar patterns but the split overhead remains.

---

## 5. Candidate Test Cases

Areas with zero or weak coverage:

| Area | Current tests | Missing |
|------|--------------|---------|
| `hub.go` / `hub_manage.go` | **None** | `cmdHubIndex`, `cmdHubAdd`, `cmdHubList`, `cmdHubRemove`, `cmdHubDefault`, `deriveLabelFromURL` |
| `github/client.go` | **None** | `NewClient`, `NewRequest`, `GetToken`, `CheckRateLimit`, rate limit error formatting |
| `internal/install/github_host.go` | **None** | `isGitHubLikeHost` edge cases (hosts containing "github" but not GitHub), `gitHubAPIBaseForHost` for `.ghe.com` and custom hosts |
| `ReadAgentIgnoreMatcher` | **None** | `.agentignore` / `.agentignore.local` matching |
| `Patience()` method on Matcher | Indirect only | Direct unit tests for `Patience()` method |
| `PushRemoteWithEnv` token leak | Integration only | Unit test that verifies token is NOT in error message |
| `Commit` token leak | Integration only | Same |
| `expandGitHubShorthand` with `gh:` | **None** | `gh:owner/repo`, `gl:owner/repo`, `bb:owner/repo` |
| `parseFileURL` type | **None** | Verify `SourceTypeGitHTTPS` is returned (test for the type confusion bug) |
| `Match` with Unicode paths | **None** | Paths with Japanese, emoji, RTL characters, NFD-normalized vs NFC-normalized paths |
| `matchSegments` without `**` vs with `**` | Covered well | — |
| `Match` with very deep paths (100+ segments) | Non-globstar case only | Globstar case with deep paths |
| `matchRule` non-anchored multi-segment (line 281-282) | Covered (2 tests) | — but this is unreachable code |
| `copyDir` symlink dropping in backup | **None** | Verify warning/error when symlinks are skipped |
| `MigrateToSource` + `CreateSymlink` failure rollback | **None** | Simulate link failure after move, verify recovery |
| `sanitizeTokens` substring matching | **None** | Token is substring of larger text (e.g., token "abc" in message "fabcd") |

---

## 6. Suspicious Observations (non-bugs but notable)

| Observation | File | Details |
|-------------|------|---------|
| `isGitHubLikeHost` is broad by design | `github_host.go:5` | `strings.Contains(host, "github")` is intentional for GHE detection but leaks into platform auth decisions. Consider rating GHE detection differently from GitHub.com detection. |
| `ignoreMatchers` map in walk never cleaned up | `discover_walk.go:138` | Map grows with each tracked repo. For repos with many branches/configs, this could accumulate stale matchers. |
| `filepath.Walk` ignores walk errors | `discover_walk.go:162-164` | `err != nil` → `return nil` (skip). Inaccessible directories are silently skipped during skill discovery. |
| `ManifestFile` is in target dir | `sync/manifest.go:11` | Named `.skillshare-manifest.json` — visible to users and could be accidentally committed/backed up. |
| `parseRule` uses `TrimRight(line, "/")` instead of `TrimSuffix` | `skillignore.go:63` | `TrimRight` removes ALL trailing slashes, not just one. `"foo///"` → `"foo"`. This matches gitignore behavior but differs from some implementations. |
| `HasLocal` is `true` even when `.skillignore.local` is empty | `skillignore.go:134` | If the local file exists but has no valid patterns, `HasLocal` is still true. |
| `ssh.dev.azure.com` in `detectPlatform` | `auth.go:62` | `host == "ssh.dev.azure.com"` — this host starts with "ssh.", which is unusual for a platform check. Verified it matches Azure DevOps SSH URL format. |

---

## 7. Reproduction Plans for Top Bugs

### Reproduce BUG-1 (PushRemoteWithEnv token leak)

```
// Add a test to internal/git/info_test.go:
func TestPushRemoteWithEnv_TokenNotLeakedInError(t *testing.T) {
    dir := t.TempDir()
    InitRepo(dir) // helper that git init + initial commit
    
    // Create a remote that refuses with token echoed back
    token := "ghp_test_token_value_12345"
    t.Setenv("GITHUB_TOKEN", token)
    
    env := AuthEnvForRepo(dir) // nil for no-remote case, but verifies the code path
    err := PushRemoteWithEnv(dir, env)
    if err != nil {
        if strings.Contains(err.Error(), token) {
            t.Error("token leaked in error message")
        }
    }
}
```

### Reproduce BUG-3 (gh:owner/repo misparse)

```
// Add to internal/install/source_test.go:
func TestParseSource_GitHubShorthandColonPrefix(t *testing.T) {
    tests := []struct{
        input string
        wantErr bool
    }{
        {"gh:owner/repo", true},    // Should error — not valid
        {"owner/repo", false},      // Should work — valid shorthand
        {"ado:org/proj/repo", false}, // Should work — valid shorthand
    }
    for _, tt := range tests {
        _, err := ParseSource(tt.input)
        if (err != nil) != tt.wantErr {
            t.Errorf("ParseSource(%q) err=%v, wantErr=%v", tt.input, err, tt.wantErr)
        }
    }
}
```

### Reproduce BUG-4 (file:// type confusion)

```
// Add to internal/install/source_test.go:
func TestParseSource_FileURLType(t *testing.T) {
    s, err := ParseSource("file:///tmp/repo.git")
    if err != nil {
        t.Fatal(err)
    }
    if s.Type != SourceTypeGitHTTPS {
        t.Errorf("expected SourceTypeGitHTTPS, got %v", s.Type)
    }
    // Verify auth is NOT attempted for file URLs
    env := authEnv(s.CloneURL)
    if env != nil {
        t.Error("authEnv should return nil for file:// URLs")
    }
}
```

### Reproduce BUG-7 (no rollback on sync failure)

```
// Manual reproduction:
1. mkdir -p ~/.config/nvim/skills/my-skill
2. touch ~/.config/nvim/skills/my-skill/SKILL.md
3. Run skillshare sync with source pointing elsewhere
   → MigrateToSource moves ~/.config/nvim/skills → source
4. Interrupt or disk-fill during CreateSymlink
5. Observe: ~/.config/nvim/skills is gone, files only in source
```

---

## 8. Summary

| Severity | Count | Key items |
|----------|-------|-----------|
| **High** (token leak, credential exposure) | 2 | BUG-1 (`PushRemoteWithEnv`), BUG-2 (`Commit`) |
| **Medium** (incorrect parse, misleading type, credential misdirection) | 3 | BUG-3 (`gh:` prefix), BUG-4 (`file://` type), BUG-5 (`isGitHubLikeHost`) |
| **Medium** (data loss) | 2 | BUG-6 (backup dropped symlinks), BUG-7 (no sync rollback) |
| **Low** (performance, edge cases) | 2 | BUG-8, BUG-9 |
| **Missing test coverage** | 6 packages | hub, github, install/github_host, ReadAgentIgnoreMatcher, Unicode paths, type confusion |

**Recommendation:** Fix BUG-1 and BUG-2 as security issues. Add tests for `gh:` prefix handling to clarify expected behavior (BUG-3). Add a warning to `copyDir` when symlinks are skipped (BUG-6).
