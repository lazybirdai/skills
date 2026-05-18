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
