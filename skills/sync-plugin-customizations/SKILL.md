---
name: sync-plugin-customizations
description: Use when running a scheduled check for upstream plugin release updates — fetches the latest tagged release, merges into the fork, uses agent-assisted conflict resolution (with kickstart files for unresolvable conflicts), and validates the intent artifact.
---

# Sync Plugin Customizations

## Overview

Automated sync skill. Checks upstream for a new tagged release and merges it into the target fork. Designed to run as a scheduled cron task. Never pushes — push is a human-triggered step.

## Prerequisites

`~/.claude/plugins/customizations/<plugin-name>/config.json` must exist with valid `localUpstreamPath`, `targetForkPath`, `upstreamRepo`, and `lastSyncedTag`.

## Red Flags — Stop if You Notice These

- You are about to merge a branch tip or commit SHA instead of a tag — STOP. Only merge tagged releases.
- Merge succeeded but you have not dispatched the validation subagent — do not commit yet. The subagent run is required before the commit.
- Merge failed and you are about to leave the repo in a mid-merge state without dispatching resolution subagents — STOP. Always: dispatch subagents first, then either continue the merge (all resolved) or inject warnings + write kickstart files + abort + commit + update config.
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

If the merge fails with the message "refusing to merge unrelated histories", retry with:
```bash
git -C <targetForkPath> merge <latestTag> --no-edit --allow-unrelated-histories
```
This handles forks that were initialized without shared git history (e.g., detached from a GitHub fork network). If this retry also fails, proceed to the On Merge Failure path below.

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

Store the list as `conflictedFiles`. Filter to only files ending in `SKILL.md` — these are the ones that need agent-assisted resolution.

**10. Dispatch one conflict-resolution subagent per conflicted SKILL.md.**

For each file in `conflictedFiles` that ends in `SKILL.md`, extract the skill name from the path (e.g., `skills/brainstorming/SKILL.md` → `brainstorming`). Ensure `~/.claude/plugins/customizations/<plugin-name>/conflicts/` exists (create if absent). Then dispatch a fresh subagent with the following exact instructions (substitute all `<placeholders>` with their actual values before dispatching):

> "You are resolving a merge conflict in a plugin customization skill.
>
> **Conflicted file:** Read `<targetForkPath>/<skill-file-path>` — it contains git conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`).
>
> **Original customization intent:** Read `~/.claude/plugins/customizations/<plugin-name>/intent.md` and extract the `## <skill-name>` section.
>
> **Original customization diff:** Run:
> ```bash
> git -C <targetForkPath> diff <lastSyncedTag> HEAD -- <skill-file-path>
> ```
> This shows exactly what was changed from the upstream base.
>
> **Upstream content at `<latestTag>` (clean, no conflict markers):** Read `<localUpstreamPath>/skills/<skill-name>/SKILL.md`.
>
> **Task:** Using the intent and original diff as context, attempt to produce a clean merged version of the conflicted file that (1) incorporates the upstream changes from `<latestTag>` and (2) preserves the customization described in the intent.
>
> **Decision:**
>
> If you can produce a clean merge with confidence (the upstream change and the customized section are clearly compatible, or the intent maps cleanly onto the new upstream structure): write the resolved content to `<targetForkPath>/<skill-file-path>` with all conflict markers removed, then report exactly: `RESOLVED: <skill-name>`
>
> If the upstream rewrote or removed the section the customization targeted, making the original intent ambiguous or inapplicable: do NOT modify the conflicted file. Write a kickstart file to `~/.claude/plugins/customizations/<plugin-name>/conflicts/<skill-name>-conflict.md` with the content below, then report exactly: `KICKSTART_GENERATED: <skill-name>`
>
> Kickstart file content (substitute actual values):
> ```
> # Conflict: <skill-name> — <plugin-name> v<lastSyncedTag> → v<latestTag>
>
> ## What happened
> The upstream skill changed in a way that conflicts with the existing customization.
> This file must be resolved via `/superpowers:brainstorming` before sync can complete.
>
> ## Original intent
> <copied verbatim from intent.md for this skill>
>
> ## Original customization diff
> <git diff output from above>
>
> ## New upstream content (v<latestTag>)
> <full content of <localUpstreamPath>/skills/<skill-name>/SKILL.md>
>
> ## Resolution instructions
> 1. Run `/superpowers:brainstorming` and reference this file as context.
> 2. Design the updated customization for the new upstream version.
> 3. Apply the change via `turbocharge:customize-plugin`.
> 4. **Delete this file** — it is stale once resolved.
> 5. **Update `intent.md`** for `<skill-name>` — the old intent is no longer valid.
> 6. Remove `syncStatus` and `failedTag` from `config.json`.
> ```"

**11. Assess subagent results and branch.**

Collect outcomes from all dispatched subagents. Each returned either `RESOLVED: <skill-name>` or `KICKSTART_GENERATED: <skill-name>`.

**If all subagents returned RESOLVED:**

Stage the resolved files and continue the merge:

```bash
git -C <targetForkPath> add <space-separated list of resolved skill file paths>
git -C <targetForkPath> merge --continue --no-edit
```

Then proceed to **Step 6** (dispatch validation subagent) as if the merge had succeeded. Do NOT proceed to Steps 12–14.

**If any subagent returned KICKSTART_GENERATED:**

For each skill that returned `KICKSTART_GENERATED`, prepend the following warning to its conflicted SKILL.md (leave conflict markers intact — do not remove them):

```
> ⚠️ SYNC CONFLICT — See `~/.claude/plugins/customizations/<plugin-name>/conflicts/<skill-name>-conflict.md` to resolve.
> Run `/superpowers:brainstorming` with that file as context, then `turbocharge:customize-plugin`.

---

```

For skills that returned `RESOLVED`, their files are already clean — no warning needed.

Abort the merge:

```bash
git -C <targetForkPath> merge --abort
```

Note: warning-injected files written before abort are preserved (git does not overwrite modified files on abort).

Proceed to Step 12.

**12. Commit conflict artifacts.**

Stage and commit all modified files: warning-injected SKILL.md files and any kickstart files:

```bash
git -C <targetForkPath> add skills/
git -C <targetForkPath> commit -m "sync: FAILED merge with upstream <latestTag> — <N> skill(s) need review"
```

Where `<N>` is the count of skills that returned `KICKSTART_GENERATED`.

**13. Update config.**

In `~/.claude/plugins/customizations/<plugin-name>/config.json`, add:

```json
"syncStatus": "failed",
"failedTag": "<latestTag>"
```

Save config.json.

**14. Log and stop.**

> "Sync failed for upstream `<latestTag>`. <RESOLVED_COUNT> skill(s) resolved automatically. <KICKSTART_COUNT> skill(s) need human review — see `~/.claude/plugins/customizations/<plugin-name>/conflicts/`."
