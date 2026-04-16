# Privacy Policy

**Last updated:** April 16, 2026

## Overview

Turbocharge is a plugin for AI coding agents (Claude Code, Cursor) that helps developers manage customized forks of other plugins. This policy describes what data Turbocharge handles and how.

## What Data Turbocharge Handles

Turbocharge reads and writes **plugin configuration files** that you create as part of normal use:

- The URL of an upstream plugin repository (e.g., a public GitHub URL)
- The local file path to your plugin fork
- The last-synced upstream version tag
- A log of customizations you have applied (`intent.md`)

These values are provided by you explicitly when you run the `setup-plugin-customization` skill.

## Where Data Is Stored

All configuration and customization data is written to files **inside your local repositories**:

- `~/.claude/plugins/customizations/<plugin-name>/config.json` — stores upstream URL, fork path, and last-synced tag
- `<your-fork-repo>/.turbocharge/intent.md` — stores your customization log
- `<your-fork-repo>/.turbocharge/settings.json` — stores sync settings

No data is sent to any server operated by this plugin. No data is written outside of your local file system and your own repositories.

## Third-Party Services

Turbocharge does not connect to any external services on its own. During normal use, your AI agent may:

- Clone or fetch from Git repositories you have specified (using your existing Git credentials)
- Read files from those repositories

All such network activity uses credentials and permissions you have already configured on your machine, and is directed only at repositories you specify.

## Data Sharing

Turbocharge does not collect, transmit, aggregate, or sell any user data to any third party. No analytics, telemetry, or usage data is gathered.

## Data Retention and Deletion

All data Turbocharge writes is stored as plain files in locations you control. To delete it:

- Remove `~/.claude/plugins/customizations/<plugin-name>/` to delete the configuration
- Remove or edit `.turbocharge/` inside your fork repository to delete customization records

## Contact

For questions about this policy, open an issue at [https://github.com/aguerlain-lr/turbocharge](https://github.com/aguerlain-lr/turbocharge).
