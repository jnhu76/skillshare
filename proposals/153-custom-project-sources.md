# Feature Proposal: Custom Project Source Directories

## Status

**Shipped in v0.19.15** (2026-05-20). Issue [#153](https://github.com/runkids/skillshare/issues/153), PR [#162](https://github.com/runkids/skillshare/pull/162).

What actually shipped:

- `sources` map only — the `source:` shorthand from the original proposal was dropped to keep one canonical form. Use `sources.skills:` for the same effect.
- All three keys (`sources.skills`, `sources.agents`, `sources.extras`) supported. Each is optional; omitted keys fall back to `.skillshare/<type>/`.
- Defense-in-depth guards added during implementation: sync rejects source/target path overlap, gitignore management skips external paths, destructive project commands fail closed on malformed `config.yaml`.

See [CHANGELOG.md](../CHANGELOG.md#01915---2026-05-20) and [project-skills docs](../website/docs/understand/project-skills.md#custom-source-directories).

## Problem

Project mode currently treats `.skillshare/skills/` as the fixed source directory for project-scoped skills.

That default works for tool-managed project state, but some repositories want project skills to live in a more discoverable or documentation-oriented location, such as `docs/skills/`, `ai/skills/`, or `.github/skills/`. In those repositories, keeping skills under `.skillshare/skills/` makes them feel hidden, less reviewable, and less aligned with normal contributor documentation workflows.

The existing `targets[].skills.path` option controls where skills are synced to. It does not configure where project skills are authored from. Users currently need symlink workarounds to make another directory behave like the project source.

## Proposed Solution

Add explicit project source configuration to `.skillshare/config.yaml` while keeping existing defaults unchanged.

Support a simple skills-source shorthand:

```yaml
source: ./docs/skills

targets:
  - claude
  - cursor
```

Also support structured resource sources:

```yaml
sources:
  skills: ./docs/skills
  agents: ./docs/agents
  extras: ./docs/extras

targets:
  - claude
  - cursor
```

Relative paths should resolve from the project root, not from `.skillshare/`.

Implementation shape:

- Add `source` and `sources` fields to project config parsing and schema.
- Treat `source` as shorthand for `sources.skills`.
- Keep defaults unchanged when no custom source is configured:
  - skills: `.skillshare/skills/`
  - agents: `.skillshare/agents/`
  - extras: `.skillshare/extras/`
- Add shared project source resolvers so CLI commands and the UI server use the same paths.
- Update project-mode commands that currently hardcode `.skillshare/skills/`, especially `sync`, `install`, `new`, `check`, `status`, `target`, `extras`, and server hub/search handlers.
- Keep `targets[].skills.path` semantics unchanged: it remains the target directory, not the source directory.
- Update docs and tests for both default behavior and custom source behavior.

No new runtime dependencies are required.

## Alternatives Considered

Use symlinks from `.skillshare/skills/` to `docs/skills/`.

This works today, but it is less explicit in config, less discoverable for contributors, and awkward across platforms or environments where symlink support differs.

Only support `source`.

This is simpler, but it only covers skills. The requested structured form is useful for repositories that also want agents and extras near contributor docs, so the proposal supports both `source` and `sources`.

Only support `sources`.

This is more uniform, but the issue explicitly proposes a concise `source: ./docs/skills` form. Supporting it as shorthand keeps the common case easy while still allowing structured resource sources.

Allow multiple skills sources.

This is out of scope. One project skills source keeps discovery, metadata, sync, and update behavior predictable.

## Scope

Estimate the scope of changes:

- [ ] Small (1-3 files, < 200 lines)
- [ ] Medium (3-10 files, 200-500 lines)
- [x] Large (10+ files, 500+ lines)

Expected areas:

- Project config structs, validation, schema, and tests
- Project runtime source resolution
- Project-mode CLI commands that read source paths
- Project-mode UI/server handlers
- Project docs and integration tests

## Open Questions

- **Resolved** — If a tracked skill repo is installed into a custom source outside `.skillshare/`, should skillshare update the project root `.gitignore`?
  Decision: skillshare leaves ignore rules entirely to the repository. `ProjectGitignoreTarget` returns empty values for external paths and all callers skip gitignore management.
- **Resolved** — Should `skillshare init -p` gain a `--source` flag now?
  Decision: deferred. Custom source setup remains an explicit `config.yaml` edit. `init -p` still seeds the default `.skillshare/skills/` and `.skillshare/agents/` directories.
