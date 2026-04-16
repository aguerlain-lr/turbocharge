---
name: customize-plugin
description: Use when you want to implement a customization in your target repo — describe the desired change and have it applied with the intent recorded.
---

# Customize Plugin

## Overview

Implements a described customization in the target repo and records the reasoning in `intent.md`. Always forward-direction: describe what you want, the skill implements it. Never reverse-engineers changes from manual edits.

**This skill always ends with a commit and a written intent entry.** Do not consider the task complete until both are done.

## Prerequisites

`turbocharge.json` must exist in the target repo. Run `setup-plugin-customization` first if it does not.

## Steps

**1. Get the target repo path and read config.**

Ask for the target repo path if the user has not provided it:
> "What is the local path to your target repo?"

Read `<targetRepoPath>/turbocharge.json` to get `upstreamRepo` and `lastSyncedTag`.

Derive `<repo-name>` from the last path segment of `upstreamRepo` (e.g., `my-plugin` from `https://github.com/org/my-plugin`).

Ensure `~/.turbocharge/<repo-name>` exists and is up to date:

- If the path does not exist: run `git clone <upstreamRepo> ~/.turbocharge/<repo-name>`
- If the path exists and is a git repo: run `git -C ~/.turbocharge/<repo-name> pull`
- If the path exists but is not a git repo: warn the user and stop.

**2. List available skills.**

List skill directories in `<targetRepoPath>/skills/` and show the user.

**3. Ask which skill(s) to customize.**

> "Which skill(s) do you want to customize?"

**4. Ask what change to make and why.**

> "Describe the change you want to make and why you want it."

**5. Read both versions for context.**

Read these two files before touching anything:
- `~/.turbocharge/<repo-name>/skills/<skill-name>/SKILL.md` — the upstream original
- `<targetRepoPath>/skills/<skill-name>/SKILL.md` — the current fork state

If either file does not exist, stop and tell the user before proceeding.

Understanding both is required. The upstream read is not optional — it ensures you understand what you are diverging from and prevents accidental reversion of prior customizations.

**6. Implement the change in the fork.**

Edit `<targetRepoPath>/skills/<skill-name>/SKILL.md` only. Never touch the upstream clone.

**7. Update intent.md.**

File: `<targetRepoPath>/intent.md`

Create the file if it does not exist. Then:
- If a `## <skill-name>` section already exists: update it to reflect the new/revised intent.
- If not: append a new section:

```markdown
## <skill-name>
**Changed:** <what was changed, concisely>
**Why:** <why the user wanted this>
**Upstream ref:** <repo-name> <lastSyncedTag>
```

**8. Commit and push.**

```bash
git -C <targetRepoPath> add skills/<skill-name>/SKILL.md intent.md
git -C <targetRepoPath> commit -m "customize: <skill-name> — <one-line description>"
git -C <targetRepoPath> push
```

The commit and push are not optional — the commit creates the record that the sync skill relies on, and the push makes the intent available to collaborators.

**9. Confirm.**

> "Done. `<skill-name>` updated and intent recorded in `<targetRepoPath>/intent.md`."

## Red Flags — Stop if You Notice These

- You have not read `turbocharge.json` yet and are about to edit files — STOP. Always read config first.
- You are about to edit a file inside `~/.turbocharge/<repo-name>` — STOP. The upstream clone is read-only.
- You read the target repo version but skipped the upstream version — STOP. Both reads are required before editing.
- You have made the edit but haven't written `intent.md` — do not commit yet.
- You are about to skip the commit or the push — STOP. Both are required.
