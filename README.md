**[日本語](README.ja.md)** | English

# claude-plugin-submitter-cmux

A Claude Code plugin that automates submitting your plugin to the **official Anthropic plugin marketplace** at `https://claude.ai/settings/plugins/submit`, using `cmux browser` for the form interaction.

## Why

Submitting a plugin to the official marketplace requires filling out a multi-page form on claude.ai (Introduction → Plugin information → Submission details). The form fields are partly inferable from your `plugin.json`, README, and LICENSE — so a Claude Code skill can prepare the values, drive `cmux browser` to fill the form, and pause for human confirmation before the final submit.

This plugin packages that flow as a reusable skill.

## What's Included

| Category | Description |
|----------|-------------|
| **Pre-submission self-check** | Verifies `plugin.json` required fields, presence of LICENSE / README / PRIVACY.md, repository accessibility |
| **Metadata extraction** | Reads `plugin.json` (name, description, repository, author, license) and proposes form values |
| **Form automation** | Drives `cmux browser` through Introduction → Plugin information → Submission details, fills fields, **stops before final submit** for human confirmation |
| **Form quirks handled** | Refs are re-issued on every snapshot; the Submitter email field resets across page transitions; the Introduction terms checkbox must be acknowledged before any page advance is accepted |

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [cmux](https://cmux.dev) installed, with Claude Code running inside a cmux session
- [`using-cmux`](https://github.com/hummer98/using-cmux) plugin installed (this plugin depends on its `cmux browser` operation patterns)
- A logged-in Claude.ai account in the cmux browser
- macOS or Linux

## Installation

```
/plugin marketplace add hummer98/claude-plugin-submitter-cmux
/plugin install claude-plugin-submitter-cmux
```

To update:

```
/plugin update claude-plugin-submitter-cmux
/reload-plugins
```

## Usage

From a Claude Code session inside cmux, with your plugin's repository as the current working directory:

```
/claude-plugin-submitter-cmux:check       # run pre-submission self-check only
/claude-plugin-submitter-cmux:submit      # full flow: check → fill form → pause for human confirm → submit
```

The skill will:
1. Read `plugin.json`, README, LICENSE, PRIVACY.md from the cwd
2. Run pre-submission self-check and report findings
3. Open the submission form in a new cmux browser surface
4. Fill in fields based on `plugin.json` + ask the human for any missing values
5. Pause and show the populated form for review
6. On human approval, click **レビューに提出** (Submit for review)
7. Verify the success page appeared

## Limitations

- Only the official Anthropic plugin marketplace is supported (`claude.ai/settings/plugins/submit`).
- Form structure on claude.ai may change without notice — selectors are placeholder-based to minimize breakage, but updates may be needed.
- The plugin does **not** create PRIVACY.md or LICENSE for you. The pre-submission check will flag missing files.

## Privacy

This plugin makes no network calls of its own. It drives a browser session that the user has already authenticated. See [PRIVACY.md](PRIVACY.md).

## License

[MIT](LICENSE)
