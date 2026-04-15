---
name: setup-plugin-customization
description: Use when setting up or reconfiguring a local plugin customization workspace — cloning an upstream plugin source, linking a target fork, and writing config needed by customize-plugin and sync-plugin-customizations.
---

# Setup Plugin Customization

## Overview

Interactive onboarding skill. Configures the local workspace for managing a customized plugin fork. Re-runnable — re-running updates existing config fields.

**The primary output of this skill is `config.json`.** Without it, `customize-plugin` and `sync-plugin-customizations` cannot run. Do not consider this skill complete until config.json is written.

## Steps

**1. Ask for the upstream plugin GitHub URL.**

> "What is the GitHub URL of the upstream plugin repo you want to customize?"

**2. Determine where to clone upstream locally.**

> "Where should I clone it? (Suggested: `~/plugins/upstream/<plugin-name>`)"

- If path exists and is a git repo: run `git -C <path> pull`
- If path does not exist: run `git clone <url> <path>`

**3. Ask about the target fork.**

> "Do you have an existing target fork repo? (y/n)"

- **Yes:** Ask for its local path (you already have it — confirm the path).
- **No:**
  - Tell the user: "Please create a GitHub fork of `<upstream-url>` and paste the fork URL here."
  - Wait for the fork URL, then ask where to clone it locally.
  - Run: `git clone <fork-url> <local-path>`

**4. Get the latest upstream tag.**

```bash
git -C <localUpstreamPath> fetch --tags
git -C <localUpstreamPath> tag --sort=-version:refname | head -1
```

Record the result as `lastSyncedTag`. If there are no tags, use the current commit SHA as a fallback and note this.

**5. Write config.**

Create `~/.claude/plugins/customizations/<plugin-name>/` if it does not exist.

Write `~/.claude/plugins/customizations/<plugin-name>/config.json`:

```json
{
  "pluginName": "<plugin-name>",
  "upstreamRepo": "<upstream-github-url>",
  "localUpstreamPath": "<absolute-expanded-path>",
  "targetForkPath": "<absolute-expanded-path>",
  "lastSyncedTag": "<latest-tag>"
}
```

Expand `~` to the actual home directory in all paths. Do not store `~` literally.

**6. Confirm.**

> "Config written to `<absolute-expanded-path-to-config.json>`. You're ready to run `customize-plugin`."

## Notes

- If `config.json` already exists, read it first and show the user current values. Only update fields they explicitly want to change.
- Never modify any files inside the upstream clone or the target fork.

## Red Flags — Stop if You Notice These

- You have set up git remotes but not yet written config.json — the git setup is not the goal. Config.json is the goal.
- You are storing `~` literally in config.json paths — always expand to the absolute path (e.g., `/Users/username/...`).
- You skipped fetching tags — `lastSyncedTag` must reflect a real tag from the upstream repo, not an invented value.
- The user provided their fork path and you feel the setup is complete — it is NOT complete until config.json has been written to `~/.claude/plugins/customizations/<plugin-name>/config.json`.
