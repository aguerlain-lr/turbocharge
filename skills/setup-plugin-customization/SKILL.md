---
name: setup-plugin-customization
description: "Use when setting up or reconfiguring a plugin customization workspace — cloning an upstream plugin source, linking a target repo, and writing turbocharge.json to the target repo for use by customize-plugin and sync-plugin-customizations. Triggers on requests to configure, initialize, or update settings for customize-plugin or sync-plugin-customizations."
---

# Setup Plugin Customization

## Overview

Interactive onboarding skill. Configures a target repo for use with turbocharge. Re-runnable — re-running updates `turbocharge.json` in place.

**The primary output of this skill is `turbocharge.json` committed to the target repo.** Without it, `customize-plugin` and `sync-plugin-customizations` cannot run. Do not consider this skill complete until `turbocharge.json` is committed and pushed.

## Steps

**1. Ask for the upstream plugin GitHub URL.**

> "What is the GitHub URL of the upstream plugin repo you want to customize?"

Derive `<repo-name>` from the last path segment of the URL (e.g., `my-plugin` from `https://github.com/org/my-plugin`).

**2. Ask for the target repo path.**

> "What is the local path to your target repo (the repo you want to customise)?"

This is the repo where `turbocharge.json` and `intent.md` will live.

**3. Clone upstream to `~/.turbocharge/<repo-name>`.**

Expand `~` to the actual home directory.

- If the path does not exist: run `git clone <upstream-url> ~/.turbocharge/<repo-name>`
- If the path exists and is a git repo: run `git -C ~/.turbocharge/<repo-name> pull`
- If the path exists but is not a git repo: warn the user and stop.

**4. Get the latest upstream tag.**

```bash
git -C ~/.turbocharge/<repo-name> fetch --tags
git -C ~/.turbocharge/<repo-name> tag --sort=-version:refname | head -1
```

If `fetch --tags` fails: report the error and stop. Do not write `turbocharge.json` with a stale tag.

Record the result as `lastSyncedTag`. If there are no tags, use the current commit SHA as a fallback and note this to the user.

**5. Write `turbocharge.json` to the target repo.**

Write `<targetRepoPath>/turbocharge.json`:

```json
{
  "upstreamRepo": "<upstream-github-url>",
  "lastSyncedTag": "<latest-tag>"
}
```

**6. Commit and push.**

```bash
git -C <targetRepoPath> add turbocharge.json
git -C <targetRepoPath> commit -m "turbocharge: initialise config"
git -C <targetRepoPath> push
```

**7. Confirm.**

> "`turbocharge.json` written and pushed to `<targetRepoPath>`. You're ready to run `customize-plugin`."

## Notes

- If `turbocharge.json` already exists in the target repo, read it and display the current values. Ask: "Which fields do you want to update?" Then only re-run the relevant steps.
- Never modify any files inside the upstream clone at `~/.turbocharge/<repo-name>` other than via `git pull`.

## Red Flags — Stop if You Notice These

- You have cloned upstream but not yet written `turbocharge.json` — the git setup is not the goal. `turbocharge.json` is the goal.
- You skipped fetching tags — `lastSyncedTag` must reflect a real tag from the upstream repo, not an invented value.
- You wrote `turbocharge.json` but have not committed and pushed — the skill is not complete until both are done.
- `fetch --tags` exited with an error — do not write `turbocharge.json` with a stale or missing tag.
