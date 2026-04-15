---
name: sync-plugin-customizations
description: Use when running a scheduled check for upstream plugin release updates — fetches the latest tagged release, merges into the fork, validates the intent artifact, and injects warnings on unresolvable conflicts.
---

# Sync Plugin Customizations

## Overview

Automated sync skill. Checks upstream for a new tagged release and merges it into the target fork. Designed to run as a scheduled cron task. Never pushes — push is a human-triggered step.

## Prerequisites

`~/.claude/plugins/customizations/<plugin-name>/config.json` must exist with valid `localUpstreamPath`, `targetForkPath`, `upstreamRepo`, and `lastSyncedTag`.

## Red Flags — Stop if You Notice These

- You are about to merge a branch tip or commit SHA instead of a tag — STOP. Only merge tagged releases.
- Merge succeeded but you have not dispatched the validation subagent — do not commit yet. The subagent run is required before the commit.
- Merge failed and you are about to leave the repo in a mid-merge state without injecting warnings — STOP. Always: inject warnings first, then abort, then commit, then update config.
- You are about to run `git push` — STOP. Never push. Push is a human-triggered step.
- Config has `syncStatus: "failed"` and you are about to proceed with a new merge — STOP. The prior failure must be resolved first.
- `git fetch --tags` exited with an error — STOP. Log the error and exit. Do not proceed with a stale tag list.

## Steps

**0. Check for prior failed sync.**

Read `~/.claude/plugins/customizations/<plugin-name>/config.json`.

If `"syncStatus": "failed"` is present:

> "⚠️ A previous sync failed for tag `<failedTag>`. Resolve that before syncing to a newer tag. Run `customize-plugin` on any skill with a sync-failed warning, then remove `syncStatus` and `failedTag` from config.json."

Stop. Do not attempt a new merge.

**1. Fetch upstream tags.**

```bash
git -C <localUpstreamPath> fetch --tags
```

If the fetch command fails (non-zero exit): log the error and stop. Do not proceed with a stale local tag list.

**2. Find latest tag.**

```bash
git -C <localUpstreamPath> tag --sort=-version:refname | head -1
```

Store as `latestTag`.

If `latestTag` is empty (no tags found in upstream): log 'No tags found in upstream. Exiting.' and stop.

If `latestTag` looks like a pre-release (e.g., contains `-beta`, `-rc`, `-alpha`), skip it and use the last non-pre-release tag instead. Pre-release tags are not considered stable sync targets.

**3. Compare to lastSyncedTag.**

If `latestTag == lastSyncedTag`: log "No new upstream release. Current: `<lastSyncedTag>`. Exiting." and stop.

**4. Ensure upstream remote exists in fork.**

```bash
git -C <targetForkPath> remote get-url upstream
```

If this fails (remote not set): run:
```bash
git -C <targetForkPath> remote add upstream <upstreamRepo>
git -C <targetForkPath> fetch upstream --tags
```

If it succeeds: run:
```bash
git -C <targetForkPath> fetch upstream --tags
```

**5. Attempt merge.**

```bash
git -C <targetForkPath> merge <latestTag> --no-edit
```

---

### On Merge Success

**6. Dispatch validation subagent.**

This step is required. Do not skip it and do not do it yourself — dispatch a fresh subagent with these exact instructions:

Substitute `<plugin-name>` and `<latestTag>` with their actual values before dispatching these instructions.

> "If `intent.md` does not exist or is empty, report 'intent.md not found or empty — no validation needed' and stop without making changes.
> Read `~/.claude/plugins/customizations/<plugin-name>/intent.md`.
> List all directories under `<localUpstreamPath>/skills/`.
> For each `## <section-name>` heading in intent.md, check whether a directory named `<section-name>` exists in the upstream skills list.
> If a skill directory is missing from upstream (could be removed or renamed): remove that section from intent.md and append a note: `<!-- <skill-name> removed or renamed in upstream <latestTag> -->`.
> Save the updated intent.md.
> Report what changes (if any) were made by printing a summary to stdout."

**7. Update config.**

In `~/.claude/plugins/customizations/<plugin-name>/config.json`:
- Set `lastSyncedTag` to `<latestTag>`
- Remove `syncStatus` and `failedTag` keys if present

**8. Commit.**

```bash
git -C <targetForkPath> add -A
git -C <targetForkPath> commit -m "sync: merge upstream <latestTag>"
```

Log: "Sync complete. Now at upstream `<latestTag>`."

---

### On Merge Failure

**9. Identify conflicted files.**

```bash
git -C <targetForkPath> diff --name-only --diff-filter=U
```

**10. Inject warning into each conflicted SKILL.md.**

For each conflicted file whose path ends in `SKILL.md`:
- Read the file (it will contain git conflict markers)
- Prepend the following block as the very first content in the file:

```
> ⚠️ SYNC FAILED — This skill has unresolved merge conflicts with upstream <latestTag>
> and needs human review before running. Run `customize-plugin` to resolve,
> then remove `syncStatus` and `failedTag` from config.json.

---

```

- Write the file back with the warning prepended. Do not remove or alter the conflict markers — leave them intact so a human can see the full conflict when they open the file.

**11. Abort the merge.**

```bash
git -C <targetForkPath> merge --abort
```

Note: The working-tree changes made in Step 10 (warning injections) are preserved by the abort — git does not overwrite modified files on abort.

**12. Commit warning-injected files.**

If no conflicted files ended in SKILL.md (no warnings were injected), skip this commit. The abort (Step 11) and config update (Step 13) still proceed.

If there are staged changes:
```bash
git -C <targetForkPath> add skills/
git -C <targetForkPath> commit -m "sync: FAILED merge with upstream <latestTag> — warnings injected into conflicted skills"
```

**13. Update config.**

In `~/.claude/plugins/customizations/<plugin-name>/config.json`, add:
```json
"syncStatus": "failed",
"failedTag": "<latestTag>"
```

Save config.json.

**14. Log and stop.**

> "Sync failed for upstream `<latestTag>`. Warnings injected into conflicted skills. Resolve manually with `customize-plugin`."
