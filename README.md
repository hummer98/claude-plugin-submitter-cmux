**[日本語](README.ja.md)** | English

# claude-plugin-submitter-cmux

Automate submitting your Claude Code plugin to the **official Anthropic marketplace** via `cmux browser`, with a human-in-the-loop final confirm.

## Status

**v0.1.0 — early scaffold, not yet end-to-end verified.** The skill encodes the form structure of `claude.ai/settings/plugins/submit` as observed in early 2026 and the known quirks (terms checkbox, ref re-issuance, email reset). Treat the first run as something to babysit. Issues and PRs are very welcome — see [Contributing](#contributing).

## Why

Submitting a plugin to the official marketplace requires filling out a multi-page form on claude.ai (Introduction → Plugin information → Submission details). Most fields are inferable from your `plugin.json`, README, and LICENSE — so a Claude Code skill can prepare the values, drive `cmux browser` to fill the form, and pause for human confirmation before the final submit.

This plugin packages that flow as a reusable skill.

## What's Included

| Category | Description | Boundary |
|----------|-------------|----------|
| **Pre-submission self-check** | Verifies `plugin.json` required fields, presence of `LICENSE` / `README.md` / `PRIVACY.md`, repository URL shape | Automated |
| **Metadata extraction** | Reads `plugin.json` (name, description, repository, author, license, homepage) and proposes form values | Automated |
| **Form automation** | Drives `cmux browser` through Introduction → Plugin information → Submission details, fills fields | Automated |
| **Final review** | Shows all populated field values back to you in chat before clicking submit | Automated, human-reviewed |
| **Final submit click** | "レビューに提出" / Submit-for-review button | **Human-confirmed** |
| **Anthropic-side review** | Approval / rejection / follow-up after submission | **Out of scope** |

## What's Automated vs. Left to You

- **Automated**: reading repo metadata, opening the form, filling fields, retrying on stale `[ref=eN]` after page transitions, surfacing every value for review.
- **Human**: reviewing the Anthropic terms, deciding whether the prefilled values are correct, clicking the final submit button, and responding to anything Anthropic asks afterward.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [cmux](https://cmux.dev) installed, with Claude Code running inside a cmux session
- [`using-cmux`](https://github.com/hummer98/using-cmux) plugin installed (this plugin depends on its `cmux browser` operation patterns)
- A logged-in Claude.ai account in the cmux browser
- macOS or Linux

> **Why not Windows?** This plugin drives `cmux browser`, and cmux itself is a macOS/Linux terminal multiplexer. Windows isn't supported upstream, so it isn't supported here either.

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

From a Claude Code session inside cmux, with the **plugin you want to submit** as the current working directory:

```
/claude-plugin-submitter-cmux:check       # pre-submission self-check only (safe dry run)
/claude-plugin-submitter-cmux:submit      # full flow: check → fill form → human review → submit
```

Want to try it without risk? Run `:check` first — it never opens a browser surface and never touches the form. It just validates your `plugin.json` and surrounding files and prints findings.

The full `:submit` flow:
1. Read `plugin.json`, `README.md`, `LICENSE`, `PRIVACY.md` from the cwd
2. Run pre-submission self-check and report findings
3. Open the submission form in a new cmux browser surface
4. Fill in fields based on `plugin.json`, asking you for any missing values
5. Pause and show the populated form for review
6. On your approval, click **レビューに提出** (Submit for review)
7. Verify that the success page appeared

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Skill aborts with "cmux browser not found" | Running outside cmux, or `using-cmux` not installed | Start a cmux session, then `/plugin install using-cmux` |
| Form returns to Introduction after final submit, error "You must acknowledge the terms to continue." | The terms checkbox on the Introduction page wasn't ticked | Re-run `:submit` — the skill ticks the checkbox in Step 5; if it failed, check the Introduction page in the cmux browser surface manually |
| Browser shows the Claude.ai login page instead of the form | Browser session not authenticated | Open `https://claude.ai` in the cmux browser, sign in, then re-run `:submit` |
| Submitter email is wrong on the final review | The email field resets to your Claude.ai login on page transitions | Edit it on the final review prompt before approving submit |
| `[ref=eN]` errors / "element not found" | Stale ref after a page transition | The skill takes a fresh snapshot per page; if it still fails, re-run — claude.ai's DOM may have shifted between page renders |
| `:check` reports `PRIVACY.md missing` but you don't want to publish one | The Privacy URL field is optional on the form | Either add a `PRIVACY.md`, or accept the warning and leave the form's Privacy URL blank |

## Limitations

- Only the official Anthropic plugin marketplace is supported (`claude.ai/settings/plugins/submit`). Third-party marketplaces are explicitly out of scope.
- Form structure on claude.ai may change without notice. Selectors are placeholder-based to minimize breakage, but updates may be needed when Anthropic ships UI changes.
- The plugin does **not** create `PRIVACY.md` or `LICENSE` for you. The pre-submission check flags missing files; authoring them is on you.
- v0.1.0 has not been verified end-to-end against a successful submission. Expect rough edges.

## Roadmap

- End-to-end verification against a real submission and recording any selector drift
- Optional screencast / animated demo for the README
- Better recovery when claude.ai changes a field placeholder
- Hooks for CI: run `:check` on every push to your plugin repo

## Privacy

This plugin makes no network calls of its own. It drives a browser session that you have already authenticated. See [PRIVACY.md](PRIVACY.md).

## Contributing

Issues and pull requests welcome at [`hummer98/claude-plugin-submitter-cmux`](https://github.com/hummer98/claude-plugin-submitter-cmux/issues). If you hit a form quirk that isn't already documented in `skills/claude-plugin-submitter-cmux/SKILL.md`, a bug report with the affected page and the placeholder text you saw is the most useful thing you can send.

## License

[MIT](LICENSE)
