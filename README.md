# Turbocharge

You found a plugin that makes your coding agent actually good. One thing bothers you. So you fork it and change it.

Three weeks later, upstream ships a release with fixes you want. Now you have a choice: manually hunt through the diff to reapply your changes, or leave your fork to rot.

Turbocharge is a set of three skills that manage this cycle for you. Fork freely. Customize intentionally. Sync automatically.

Agent-agnostic. The skills are plain Markdown — anything that can read and execute a SKILL.md (Claude Code, Cursor, or any other harness that runs skills) can drive them.

---

## Install

Clone the repo somewhere stable:

```bash
git clone git@github.com:aguerlain-lr/turbocharge.git ~/dev/turbocharge
```

Then point your agent at the `skills/` directory. Examples below — adapt for whichever harness you use.

### Claude Code

```bash
claude plugin marketplace add ~/dev/turbocharge
claude plugin install turbocharge@turbocharge-dev
```

To pick up updates:

```bash
cd ~/dev/turbocharge && git pull
claude plugin update turbocharge@turbocharge-dev
```

### Cursor

```bash
git clone git@github.com:aguerlain-lr/turbocharge.git ~/.cursor/plugins/local/turbocharge
```

Restart Cursor. To update: `cd ~/.cursor/plugins/local/turbocharge && git pull` and restart.

### Other agents

Any agent that loads skills from a directory works. Point it at `~/dev/turbocharge/skills/` (or wherever you cloned it) and the three skills become available by name.

---

## What You Get

**`setup-plugin-customization`** — Run once per target repo. Asks for the upstream plugin URL and the local path to your fork, clones upstream to a local cache, and writes `.turbocharge/settings.json` into the fork. Commits and pushes.

**`customize-plugin`** — Describe a change in plain language. The skill reads the upstream original and your fork's current state, applies the change, records your intent in `.turbocharge/intent.md`, regenerates the fork's `README.md` to reflect customizations, then commits and pushes. The recorded intent is what makes sync work later.

**`sync-plugin-customizations`** — Checks upstream for new tagged releases. If one exists, merges it into your fork. Skill conflicts get dispatched to subagents — each either resolves cleanly or writes a kickstart file describing the conflict for human review. Commits the result either way.

---

## How It Works

Two files live inside your fork at `<fork>/.turbocharge/`:

- **`settings.json`** — tracks the upstream URL and the last successfully synced tag. Updated by sync. Committed to the fork.
- **`intent.md`** — one section per customized skill describing what the fork does differently from upstream and why. Written by `customize-plugin`. Committed to the fork.

The upstream source is cloned locally to `~/.turbocharge/<repo-name>/` for reference. This clone is read-only — skills never modify it, only `git pull` it. Kickstart files for unresolved conflicts also land under `~/.turbocharge/<repo-name>/conflicts/` and are local-only (not tracked by git).

Because the config lives inside the fork, anyone who clones your fork can run sync without first running setup. The intent travels with the code.

---

## Typical Workflow

**First time.** Run `setup-plugin-customization`. It asks for the upstream URL and your fork path, clones upstream to `~/.turbocharge/<repo-name>/`, writes `<fork>/.turbocharge/settings.json`, then commits and pushes.

**Making a change.** Tell your agent: "use `customize-plugin` to make skill X do Y." The skill reads the upstream version and your fork's current state, applies the edit, appends a new section to `.turbocharge/intent.md`, regenerates the fork's `README.md` so the `## Customizations` block reflects current intent, then commits and pushes.

**Staying current.** Run `sync-plugin-customizations` — manually, or on a scheduled task. It fetches upstream tags, compares to `lastSyncedTag`, and if there's a newer release, merges it. Clean merges auto-commit. Conflicts get the resolution flow below.

---

## Conflict Resolution

When sync hits a merge conflict in a customized skill, it dispatches one subagent per conflicted `SKILL.md`. Each subagent reads the intent entry, the original customization diff, and the new upstream content, then decides:

- **Can resolve cleanly** — writes the merged file, sync continues, single commit lands.
- **Cannot resolve** — leaves the conflict markers in place, writes a kickstart file to `~/.turbocharge/<repo-name>/conflicts/<skill-name>-conflict.md` containing the original intent, the original diff, and the full new upstream content. Injects a warning at the top of the conflicted file pointing to the kickstart.

If any kickstart was generated, sync aborts the merge, commits the warning-injected files, and sets `syncStatus: "failed"` in `settings.json`.

To resolve, run a planning/brainstorming skill, such as `/superpowers:brainstorming` (from the [Superpowers plugin](https://github.com/obra/superpowers)), with the kickstart file(s) as context to design the new customization, then apply it with `customize-plugin`. Delete the kickstart, remove `syncStatus` and `failedTag` from `settings.json`, and you're unblocked.

---

## Privacy

See [PRIVACY.md](PRIVACY.md). Short version: turbocharge runs locally, talks to git remotes you already use, and stores nothing beyond what's in your fork and the local cache.

---

## License

MIT — see [LICENSE](LICENSE).
