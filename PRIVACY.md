# Privacy Policy

_Last updated: 2026-04-28_

## What this plugin does

`claude-plugin-submitter-cmux` is a Claude Code plugin consisting of skill markdown files that instruct the agent to drive `cmux browser` against the official Claude Code plugin submission form on `claude.ai`.

## What this plugin does NOT do

- It performs no network access on its own. It does not contact any server other than via the user's already-authenticated browser session that the user controls.
- It does not collect, store, or transmit any user data, telemetry, or analytics.
- It does not include or load any third-party services.
- It does not read or modify files outside the working directory the user invokes it in (which is the plugin repository being submitted).

## Data flow during use

When you invoke this plugin's skills, the following happens locally on your machine:

1. The agent reads files from the working directory: `plugin.json`, `README.md`, `LICENSE`, `PRIVACY.md`.
2. The agent uses `cmux browser` (provided by `using-cmux`) to open `claude.ai/settings/plugins/submit` in a browser surface authenticated as you.
3. The agent fills the form fields with values inferred from your `plugin.json` and the answers you give in the chat.
4. You review the populated form and confirm submission. The submission itself is delivered by your browser to Anthropic, governed by Anthropic's privacy policy — not by this plugin.

Your prompts and the agent's tool results flow through Claude Code, governed by [Anthropic's privacy policy](https://www.anthropic.com/legal/privacy), not by this plugin.

## Contact

For privacy inquiries: yuji.yamamoto@tayorie.jp

## License

This plugin is released under the [MIT License](LICENSE).
