现在只处理一个 bug：

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

请按最小修复流程工作：

1. 在 main 分支确认可复现。
2. 写一个最小 failing test，优先放在现有 package 的测试文件中。
3. 确认测试在修复前失败。
4. 做最小代码修改，只改与该 bug 直接相关的文件。
5. 确认新测试通过。
6. 运行相关 package 测试、go test ./...、make check。
7. 输出：
   - root cause
   - why this fix is minimal
   - behavior before/after
   - tests added
   - risk / non-goals
   - PR body draft

限制：

- 不允许顺手重构。
- 不允许修改无关格式。
- 不允许扩大 scope。
- 不允许把临时 fuzz harness / debug script 放进最终 PR，除非它是正式测试。
