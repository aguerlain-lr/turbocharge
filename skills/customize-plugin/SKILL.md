---
name: customize-plugin
description: Use when you want to implement a customization in your plugin fork — describe the desired change and have it applied to the target fork with the intent recorded.
---

# Customize Plugin

## Overview

Implements a described customization in the target fork and records the reasoning in `intent.md`. Always forward-direction: describe what you want, the skill implements it. Never reverse-engineers changes from manual edits.

**This skill always ends with a commit and a written intent entry.** Do not consider the task complete until both are done.

## Prerequisites

`~/.claude/plugins/customizations/<plugin-name>/config.json` must exist. Run `setup-plugin-customization` first if it does not.

## Steps

**1. Read config.**

Read `~/.claude/plugins/customizations/<plugin-name>/config.json` to get `localUpstreamPath`, `targetForkPath`, `pluginName`, and `lastSyncedTag`.

Do this even if the user told you the path — the config is the authoritative source and may contain paths the user didn't mention.

If multiple `config.json` files exist under `~/.claude/plugins/customizations/`, ask the user which plugin they want to customize.

**2. List available skills.**

List skill directories in `<targetForkPath>/skills/` and show the user.

**3. Ask which skill(s) to customize.**

> "Which skill(s) do you want to customize?"

**4. Ask what change to make and why.**

> "Describe the change you want to make and why you want it."

**5. Read both versions for context.**

Read these two files before touching anything:
- `<localUpstreamPath>/skills/<skill-name>/SKILL.md` — the upstream original
- `<targetForkPath>/skills/<skill-name>/SKILL.md` — the current fork state

If either file does not exist, stop and tell the user before proceeding.

Understanding both is required. The upstream read is not optional — it ensures you understand what you are diverging from and prevents accidental reversion of prior customizations.

**6. Implement the change in the fork.**

Edit `<targetForkPath>/skills/<skill-name>/SKILL.md` only. Never touch the upstream clone.

**7. Update intent.md.**

File: `~/.claude/plugins/customizations/<plugin-name>/intent.md`

Create the file if it does not exist. Then:
- If a `## <skill-name>` section already exists: update it to reflect the new/revised intent.
- If not: append a new section:

```markdown
## <skill-name>
**Changed:** <what was changed, concisely>
**Why:** <why the user wanted this>
**Upstream ref:** <pluginName> <lastSyncedTag>
```

**8. Commit.**

```bash
git -C <targetForkPath> add skills/<skill-name>/SKILL.md
git -C <targetForkPath> commit -m "customize: <skill-name> — <one-line description>"
```

The commit is not optional — it creates the record that the sync skill relies on to determine what changed.

**9. Confirm.**

> "Done. `<skill-name>` updated in `<targetForkPath>` and intent recorded in `~/.claude/plugins/customizations/<plugin-name>/intent.md`."

## Red Flags — Stop if You Notice These

- The user told you the file path and you are about to skip reading config.json — STOP. Always read config first. The user may not know all the paths that matter.
- You are about to edit a file inside `localUpstreamPath` — STOP. The upstream clone is read-only.
- You read the fork version but skipped the upstream version — STOP. Both reads are required before editing.
- You have made the edit but haven't written `intent.md` — do not commit yet.
- You are about to skip the commit — STOP. The commit is required, not optional.
