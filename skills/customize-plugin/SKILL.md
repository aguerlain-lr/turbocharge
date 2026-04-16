---
name: customize-plugin
description: Use when you want to implement a customization in your target repo — describe the desired change and have it applied with the intent recorded.
---

# Customize Plugin

## Overview

Implements a described customization in the target repo and records the reasoning in `intent.md`. Always forward-direction: describe what you want, the skill implements it. Never reverse-engineers changes from manual edits.

**This skill always ends with a commit, a written intent entry, a README update, and a push.** Do not consider the task complete until all four are done.

## Prerequisites

`.turbocharge/settings.json` must exist in the target repo. Run `setup-plugin-customization` first if it does not.

## Steps

**1. Get the target repo path and read config.**

Ask for the target repo path if the user has not provided it:
> "What is the local path to your target repo?"

Read `<targetRepoPath>/.turbocharge/settings.json` to get `upstreamRepo` and `lastSyncedTag`.

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

File: `<targetRepoPath>/.turbocharge/intent.md`

Create the file if it does not exist. Then:
- If a `## <skill-name>` section already exists: update it to reflect the new/revised intent.
- If not: append a new section:

```markdown
## <skill-name>
**Changed:** <what was changed, concisely>
**Why:** <why the user wanted this>
**Upstream ref:** <repo-name> <lastSyncedTag>
```

**8. Update README.md.**

Check `<targetRepoPath>/README.md` for the line `<!-- turbocharge-customized -->`.

**If the marker is absent (first run or README has not yet been customized):**

Run `git -C <targetRepoPath> remote get-url origin` to get the fork's origin URL. Derive `<fork-name>` from its last path segment. If this command fails (no `origin` remote), use `<repo-name>` as `<fork-name>` and skip the title-change check.

Compare `<fork-name>` to `<repo-name>` (upstream). If they differ, the README title will change to `<fork-name>`. If they are the same, the title stays as-is.

Read:
- `<targetRepoPath>/README.md` — current README content to use as source material
- `<targetRepoPath>/.turbocharge/intent.md` — customization entries
- Each `<targetRepoPath>/skills/<skill-name>/SKILL.md` listed in `intent.md`

Write a new `README.md` with this structure:
1. `# <fork-name>` (or upstream name if unchanged)
2. `_A customized fork of [<repo-name>](<upstreamRepo>)._`
3. Rewritten body — based on upstream README content, updated to reflect customized skills. Preserve relevant upstream content; drop content that no longer applies.
4. `## Customizations` — one prose entry per skill in `intent.md`, drawing from its **Changed** and **Why** fields.
5. If the upstream README had an installation section, replace its content with this literal line:
   `<!-- TODO: update installation instructions for your deployment -->`
   If the upstream README had no installation section, do not add one.
6. Final line (must be last): `<!-- turbocharge-customized -->`

**If the marker is present (subsequent runs):**

Read `<targetRepoPath>/README.md`. If `<targetRepoPath>/.turbocharge/intent.md` does not exist, remove the `## Customizations` section from the README if present, and proceed to the commit step. Otherwise, read `intent.md` and continue below.

Locate `## Customizations`:
- If it exists: replace it with regenerated content.
- If it does not exist: insert it immediately before `<!-- turbocharge-customized -->`.

Regenerated `## Customizations` content: one prose entry per skill currently in `intent.md`, drawing from its **Changed** and **Why** fields. Skills no longer in `intent.md` are dropped. Leave everything else in the README untouched.

**9. Commit and push.**

```bash
git -C <targetRepoPath> add skills/<skill-name>/SKILL.md .turbocharge/intent.md README.md
git -C <targetRepoPath> commit -m "customize: <skill-name> — <one-line description>"
git -C <targetRepoPath> push
```

The commit and push are not optional — the commit creates the record that the sync skill relies on, and the push makes the intent available to collaborators.

**10. Confirm.**

> "Done. `<skill-name>` updated, intent recorded in `<targetRepoPath>/.turbocharge/intent.md`, and README updated."

## Red Flags — Stop if You Notice These

- You have not read `.turbocharge/settings.json` yet and are about to edit files — STOP. Always read config first.
- You are about to edit a file inside `~/.turbocharge/<repo-name>` — STOP. The upstream clone is read-only.
- You read the target repo version but skipped the upstream version — STOP. Both reads are required before editing.
- You have made the edit but haven't written `intent.md` — do not commit yet.
- You are about to skip the commit or the push — STOP. Both are required.
- You are about to write README.md but haven't read the current README first — STOP. The rewrite must be informed by existing content.
- The README contains `<!-- turbocharge-customized -->` and you are about to rewrite the entire file — STOP. Only regenerate `## Customizations`.
- You removed `<!-- turbocharge-customized -->` from the README output — STOP. It must always be the last line.
