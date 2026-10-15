# Lazybird Skills

Official agent skills for [Lazybird](https://www.lazybird.app): text-to-speech for AI agents.

Install the skill and your coding agent can turn text or SSML into MP3 voiceovers with 500+ voices in 150+ locales, clone a voice from a short sample, or design a new voice from a description, all through the [Lazybird Speech API](https://www.lazybird.app/docs/api-reference). It works with Claude Code, Codex, Cursor and any agent that reads `SKILL.md` files.

## Install

Any agent (via [skills.sh](https://skills.sh)):

```bash
npx skills add lazybirdai/skills
```

Claude Code plugin:

```text
/plugin marketplace add lazybirdai/skills
/plugin install lazybird@lazybird
```

Then set an API key, or let the agent get one for you by email code:

```bash
export LAZYBIRD_API_KEY=your-key-here
```

New accounts start on the free plan (3,000 characters a month).

## Try it

- "Turn this paragraph into an MP3 voiceover with a warm US English female voice."
- "Find a deep British male narrator voice and generate a 20-second sample."
- "Narrate chapter1.md as an audiobook, split into chunks and joined into one MP3."
- "Design a calm, raspy old wizard voice and save it to my voices."
- "Add Lazybird text-to-speech to my Node.js app."

## Available skills

| Skill | What it covers |
|---|---|
| [`lazybird-api`](./skills/lazybird-api) | API keys by email code, voice search, text and SSML speech generation, speed/pitch/volume/style settings, long scripts, voice cloning, voice design, plans and error handling. |

## Documentation

- API reference: https://www.lazybird.app/docs/api-reference
- OpenAPI 3.1 spec: https://www.lazybird.app/openapi.json
- Agent docs (llms.txt): https://api.lazybird.app/llms.txt

## License

MIT
