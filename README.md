# Lazybird Skills

Official agent skills for [Lazybird](https://www.lazybird.app): speech, voiceovers and video for AI agents.

Install the skill and your agent can turn text or SSML into natural speech, build multi-voice voiceovers and videos with stock footage, music and on-screen text, export MP4 video, MP3 audio and SRT subtitles, share a project by link, clone a voice from a short sample, or design a new voice from a description, all through the [Lazybird API](https://www.lazybird.app/docs/api-reference). It works with Claude Code, Codex, Cursor and any agent that reads `SKILL.md` files.

## Install

Claude Code plugin:

```text
/plugin marketplace add lazybirdai/skills
/plugin install lazybird@lazybird
```

The plugin adds the skill and connects the Lazybird MCP server. Run `/mcp` and sign in to your Lazybird account; no API key needed. New accounts start on the free plan (10,000 credits a month).

Other agents (Codex, Cursor and any agent that reads `SKILL.md` files): copy [`skills/lazybird-api`](./skills/lazybird-api) into your agent's skills folder and add the remote MCP server `https://api.lazybird.app/mcp`.

To add speech or video rendering to your own app, create an API key at https://www.lazybird.app/platform. The skill shows your agent how to call the REST API with it, with the key loaded from your app's config or secret store.

## Try it

- "Make a 15-second vertical video ad for my coffee shop with music and subtitles."
- "Turn this script into a two-voice podcast intro and export it as an MP3."
- "Turn this paragraph into an MP3 voiceover with a warm US English female voice."
- "Find a deep British male narrator voice and generate a 20-second sample."
- "Narrate chapter1.md as an audiobook, split into chunks and joined into one MP3."
- "Design a calm, raspy old wizard voice and save it to my voices."
- "Add Lazybird text-to-speech to my Node.js app."

## Available skills

| Skill | What it covers |
|---|---|
| [`lazybird-api`](./skills/lazybird-api) | Speech, voiceovers and video through the Lazybird MCP tools, and the REST API for your own code: voice search, text and SSML speech, multi-voice voiceover and video projects, stock photos, clips and music, MP4/MOV, MP3/WAV/OGG and TXT/DOCX/SRT/VTT exports, share links, voice cloning, voice design, API keys, team key scopes, plans, render limits and error handling. |

## Data and privacy

The skill and plugin store nothing themselves. The plugin's MCP server is `https://api.lazybird.app/mcp`: you sign in to your Lazybird account with OAuth, and your agent keeps the sign-in token. Code the agent writes for your app calls `https://api.lazybird.app` with the API key your app loads. Either way, requests go directly to Lazybird over HTTPS and carry:

- the text or SSML you ask it to speak, plus voice and audio settings
- the projects you ask it to build: scripts, media you add, and the stock photos, clips and music it picks
- audio samples you provide for voice cloning, and descriptions you give for voice design
- your email address, only if you ask the agent to get an API key for you by email code

Speech made through the MCP server is saved as an MP3 in your Lazybird account and shared as a `https://cdn.lazybird.app` link; REST calls return the audio directly. Projects stay in your account and open in the Lazybird Studio; exported files download from `https://cdn.lazybird.app` for 30 days. Voice previews and design samples also download from `https://cdn.lazybird.app`. The skill contacts no other service. A plan upgrade returns a checkout link that you open yourself.

On the Lazybird side:

- Records of each speech generation, including its text, and temporary copies of the audio are kept for 30 days.
- Cloned and designed voices stay in your account until you delete them. Deleting a voice removes its processed sample, preview and avatar right away. The original recording you uploaded, and voice-design samples you didn't save, are kept until you delete your account (Settings → Account → Delete account).
- Lazybird does not sell your data and does not use your text, uploads or voice recordings to train AI models.

Only clone a voice that is yours, or whose speaker agreed to it. The skill asks you to confirm before cloning a real person's voice. Details: [Privacy Policy](https://www.lazybird.app/privacy), [Terms of Service](https://www.lazybird.app/terms) and [Acceptable Use Policy](https://www.lazybird.app/acceptable-use). Questions: support@lazybird.app.

## Documentation

- API reference: https://www.lazybird.app/docs/api-reference
- OpenAPI 3.1 spec: https://www.lazybird.app/openapi.json
- Agent docs (llms.txt): https://api.lazybird.app/llms.txt

## License

MIT
