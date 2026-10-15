# Lazybird Skills

This repository publishes official Lazybird agent skills.

## Editing Guidelines

- Keep `SKILL.md` files concise and operational.
- Put detailed endpoint or workflow reference material in `references/`.
- Keep skill names lowercase and hyphenated.
- When updating product behavior, update both the skill summary and the relevant reference file.
- Do not include secrets, private API keys, or internal-only implementation details.

## Validation

Before publishing, test installation locally and from GitHub:

```bash
npx skills add lazybirdai/skills --skill lazybird-api
```

## Manifests

Keep `version`, `description` and `keywords` in sync across `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json` and `.cursor-plugin/plugin.json`. OpenAI directory listing fields (display name, short description of 30 chars or fewer, prompts, icons) live in `.codex-plugin/plugin.json` under `interface`; icons are in `assets/`.

The source of truth for `skills/lazybird-api` is `.agents/skills/lazybird-api` in the main Lazybird repo. Edit there, then copy here.

```bash
claude plugin validate .
```
