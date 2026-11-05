# Lazybird Skills

This repository publishes official Lazybird agent skills.

## Editing Guidelines

- Keep `SKILL.md` files concise and operational.
- Put detailed endpoint or workflow reference material in `references/`.
- Keep skill names lowercase and hyphenated.
- When updating product behavior, update both the skill summary and the relevant reference file.
- Do not include secrets, private API keys, or internal-only implementation details.

## Validation

Before publishing, validate the plugin and test that it loads with its MCP server:

```bash
claude plugin validate .
claude --plugin-dir .
```

Examples must not read API keys from the user's environment or files: use the `YOUR_API_KEY` placeholder or a key the app passes in. Don't add install commands that download and run a package.

## Manifests

Keep `version`, `description` and `keywords` in sync across `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json` and `.cursor-plugin/plugin.json`. OpenAI directory listing fields (display name, short description of 30 chars or fewer, prompts, icons) live in `.codex-plugin/plugin.json` under `interface`; icons are in `assets/`.

`.mcp.json` connects the Claude Code plugin to the Lazybird MCP server (`https://api.lazybird.app/mcp`, OAuth sign-in).

The source of truth for `skills/lazybird-api` is `.agents/skills/lazybird-api` in the main Lazybird repo. Edit there, then copy here.
