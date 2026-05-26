# skillshare v0.19.22 Release Notes

Release date: 2026-05-26

## TL;DR

1. **Web UI install now handles mixed-track repos** — repositories that ship both skills and agents (for example `github/awesome-copilot`) used to fail with a confusing error when installed with **Track**. The dashboard now asks you to pick which kind to track.
2. **Faster recovery on the mixed-track path** — the install API reports skill and agent counts directly in the ambiguity response, so the dashboard skips an extra repository clone before showing the picker.

This is a patch release — bug fix only, no new commands or flags.

---

## Mixed-Track Install Now Has a Recovery Path

If you opened the dashboard's Install page, pasted a repository that contains both skills and agents (for example `github/awesome-copilot`), checked **Track**, and hit Install, the request used to fail with the message:

> tracked install is ambiguous for mixed repositories; pass --kind skill or --kind agent

…and the dashboard had no way to follow that advice — there was no Skills/Agents toggle to pick.

In v0.19.22 the dashboard catches this case automatically. A dialog pops up showing how many skills and how many agents the repository contains:

```
Mixed Repository
This repository contains both skills and agents. Choose what to install:

  [📦] Skills    335 items  ›
  [🤖] Agents    215 items  ›
```

Click either option and the install continues with the chosen kind. The CLI is unaffected — `skillshare install <repo> --track --kind skill` (or `--kind agent`) already worked and still works exactly the same way.

## One Fewer Clone During Recovery

The previous flow needed three sequential `git clone` operations on the recovery path: one to discover that the repo was mixed, one for the dashboard to fetch the skill / agent counts, and one for the final install. The install API now embeds those counts in the ambiguity error itself, so the dashboard reuses them directly and only two clones happen end to end. On large repositories this noticeably shortens the time between clicking Install and seeing the kind picker.
