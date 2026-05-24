# skillshare v0.19.21 Release Notes

Release date: 2026-05-24

## TL;DR

1. **Update checks now finish for nested GitHub-installed skills** — the dashboard no longer leaves items stuck on `Checking` when the backend reports them by relative path
2. **Update Selected handles nested installs correctly** — selecting a skill installed under a subdirectory now updates the right item
3. **Remote checks fail fast instead of hanging** — slow, private, or credential-gated remotes now surface an error instead of leaving the spinner running indefinitely

This is a patch release — bug fixes only, no new commands or flags.

---

## Update Page No Longer Gets Stuck on Nested Skills

The dashboard Update page now handles skills installed into subdirectories consistently. If a skill is shown as `agent-browser` but stored under a path like `tools/agent-browser`, **Check All** correctly applies the backend result to the visible row instead of leaving it on `Checking`.

This also applies to **Update Selected**: when you select a nested GitHub-installed skill, the dashboard now targets the actual installed path, so the update reaches the right metadata entry.

## Remote Checks Now Time Out Cleanly

Remote update checks use lightweight Git probes. Those probes now disable interactive credential prompts and time out after 15 seconds. If a remote is private, unreachable, or waiting for credentials, Skillshare now reports an error in the UI or CLI instead of leaving the update check running forever.
