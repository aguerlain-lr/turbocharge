# Turbocharge

You found a plugin that makes your coding agent actually good. One thing bothers you. So you fork it and change it.

Three weeks later, upstream ships a release with fixes you want. Now you have a choice: manually hunt through the diff to reapply your changes, or leave your fork to rot.

Turbocharge is a set of three skills that manage this cycle for you. Fork freely. Customize intentionally. Sync automatically.

---

## Install

### Claude Code

```
/plugin install turbocharge@YOUR_GITHUB_ORG
```

### Cursor

In Cursor Agent chat:

```
/add-plugin turbocharge
```

> **Note:** Exact install commands depend on marketplace registration. If the above don't work yet, clone the repo and load the skills manually.

---

## What You Get

**`setup-plugin-customization`** — Run once per plugin you want to manage. Points turbocharge at your upstream source and your fork, and writes the config file that drives the other two skills.

**`customize-plugin`** — Describe the change you want in plain language. The skill reads the upstream original and your current fork, implements the change, and records your intent in `intent.md`. That recorded intent is what lets sync work later.

**`sync-plugin-customizations`** — Checks upstream for new tagged releases. If one exists, merges it into your fork. Where it can resolve skill conflicts automatically, it does. Where it can't, it writes a detailed kickstart file and leaves you with a clean starting point for a brainstorming session.

---

## Typical Workflow

**First time:** Run `setup-plugin-customization`. It asks for your upstream URL and your fork path, clones what's missing, and writes a config file to `~/.claude/plugins/customizations/<plugin-name>/config.json`.

**Making a change:** Tell your agent: "I want `customize-plugin` to also do X." The skill reads both the upstream original and your fork's current state, makes the edit, and appends an entry to `intent.md` explaining what was changed and why. This entry becomes the memory that sync uses to understand what to preserve.

**Staying current:** Run `sync-plugin-customizations` — or set it up as a scheduled cron task. It fetches upstream tags, checks whether you're behind, and merges the new release. In the common case (upstream changed sections you didn't touch), the merge is clean and automatic. The config gets updated. Done.

---

## Conflict Resolution

When sync hits a skill conflict it can't auto-resolve — because upstream rewrote the exact section your customization targets — it doesn't leave you with a mess. It writes a kickstart file to `~/.claude/plugins/customizations/<plugin-name>/conflicts/<skill-name>-conflict.md` containing your original intent, your original diff, and the full new upstream content. It aborts the merge cleanly and marks the sync as failed in config.

When you're ready to resolve, run `/superpowers:brainstorming` and reference the kickstart file as context. Once you've redesigned the customization for the new upstream version, apply it with `customize-plugin` and clear the failed sync flag.

Skills that could be auto-resolved are handled without your involvement. You only see the ones that genuinely need judgment.

---

## How It Works

Two files live in `~/.claude/plugins/customizations/<plugin-name>/` (outside your fork repo):

**`config.json`** — workspace config. Tracks your upstream URL, local paths for both repos, the last successfully synced tag, and sync status. The sync skill reads and updates this on every run.

**`intent.md`** — the memory of your customizations. One section per modified skill, recording what was changed and why. Without this, sync can't know what to preserve when merging. `customize-plugin` writes it; `sync-plugin-customizations` reads it.

Nothing is stored in your fork repo itself. Your fork stays clean.
