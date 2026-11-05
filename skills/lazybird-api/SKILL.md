---
name: lazybird-api
description: Generate MP3 speech, voiceovers and narration from text or SSML with Lazybird text-to-speech (MCP tools and the api.lazybird.app REST API), pick from 500+ voices in 150+ locales, clone a voice from an audio sample, or design a new voice from a text description. Use when the user wants to turn text into audio, add TTS or AI voices to an app, script, or agent, clone or design a voice, get a Lazybird API key, or debug a Lazybird API integration.
license: MIT
metadata:
  author: Lazybird
  version: "1.4.0"
  homepage: https://www.lazybird.app
  source: https://github.com/lazybirdai/skills
  documentation: https://www.lazybird.app/docs/api-reference
---

# Lazybird text-to-speech

Turn text into MP3 speech with 500+ voices, clone a voice from a short sample, or design one from a description. One credit = one character of input text.

Pick the way in by what the user wants:

- **Audio now** (a voiceover, narration, a sample to listen to): use the Lazybird MCP tools. They act on the user's Lazybird account through an OAuth sign-in, so no API key is involved.
- **Text-to-speech in their code** (an app, script or backend): write calls to the REST API with an API key that the app loads from its own config or secret store.

Never read a Lazybird API key from the user's environment or files to send requests yourself.

## Audio with the MCP tools

The Lazybird plugin connects the MCP server. The user signs in to Lazybird once through `/mcp`; new accounts start on the free plan. Without the plugin, connect the remote (streamable HTTP) server `https://api.lazybird.app/mcp`, e.g. in Claude Code `claude mcp add --transport http lazybird https://api.lazybird.app/mcp`, then sign in. If the tools are missing, tell the user how to connect them.

| Tool | What it does |
|---|---|
| `search_voices` | Finds voices by search words (`q`), `language`, `gender` and `labels`, with the user's own and team voices. Returns ids, preview audio links and avatars. Page with `section` and the returned cursor. |
| `generate_speech` | `{ voiceId, text, speed?, locale? }` → a link to the MP3 (`audioUrl`) and its duration. Up to 5,000 characters per call. |
| `design_voice` | `{ description, text }` → a sample of a new voice (`sampleUrl`). Costs at least 50 characters. The voice isn't saved. |
| `save_voice` | `{ name, sampleUrl, rightsConfirmed: true }` → saves a designed sample, or clones a public link to a clear 10–60 s recording of one speaker. Returns the voice id. Takes a voice slot. |
| `get_account` | Plan, characters used and left, and when the allowance resets. |

1. Help write or polish the script. If the user hasn't picked a voice, `search_voices` and suggest two or three that fit, with their preview links.
2. `generate_speech`, then share the link. To save a file, download it: `curl -sSL -o voiceover.mp3 "<audioUrl>"`.
3. Before long jobs or batches, check `get_account`. When the user runs out, say so and link https://www.lazybird.app/pricing.

- Every voice, cloned ones included, speaks every supported language. For text in another language than the voice's, pass `locale` with a region (`es-ES`).
- `speed` runs from 0.5 to 2; 1 is normal.
- Treat voice ids as opaque: copy them from `search_voices`, never build them from a name.
- The tools don't cover SSML, pitch, volume, speaking styles, or editing, publishing and deleting voices. For those, the user can use the Lazybird app, or you write REST calls for them to run with their own key.

## Text-to-speech in code (REST API)

- Base URL: `https://api.lazybird.app/v2`
- Auth header: `X-API-Key: YOUR_API_KEY` (query `?api_key=` only when headers are impossible)
- OpenAPI 3.1: https://www.lazybird.app/openapi.json · Agent docs: https://api.lazybird.app/llms.txt

1. **Key.** The user creates one at https://www.lazybird.app/platform (API Keys) or gets one by email code (`POST /auth/request-code` → `POST /auth/verify-code`, see the reference). Write the code to load it from the app's config or secret store and ask the user to put it there. Never write a key into code.
2. **Voice.** `GET /voices` is public: look up voice ids without a key (see "Finding a voice").
3. **Generate.** `POST /generate-speech {"voiceId","text"}` returns raw `audio/mpeg` bytes. Write them to a file. Long text takes time: allow a 120+ second timeout.
4. **Errors.** Read the JSON `error` message and follow the error table below.

```bash
curl -sS -X POST "https://api.lazybird.app/v2/generate-speech" \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  --max-time 300 --fail-with-body -o speech.mp3 \
  --data '{"voiceId":"tgJ2Hes6DauwTg8kFrRPLN","text":"Hello from Lazybird."}'
```

### Critical rules

- Keep API keys server-side. Never put them in browser or mobile code, public repos or logs.
- `/generate-speech` returns MP3 bytes, not JSON and not a URL. Check the status code before saving: errors come back as JSON.
- Send exactly one of `text` and `ssml`, and not an empty one. Anything else is a `400`, and so is any field the API doesn't document, in the body or in `settings`.
- SSML supports only `<speak>`, `<break time="500ms|2s"/>` (max 60s) and `<phoneme alphabet="ipa" ph="...">word</phoneme>`. Any other tag (`<prosody>`, `<emphasis>`, `<say-as>`, `<voice>`...) is a `400`. Use `settings` for speed, pitch and volume instead. Escape `&`, `<`, `>` in text, and escape quotes inside JSON.
- `settings.style` only works with a value from the voice's `styleList` (most voices have none). Other values are ignored, together with `styleIntensity`.
- For text in another language than the voice's `language`, pass `settings.locale` as a locale with a region (`es-ES`; a bare `es` is a `400`).
- Before spending a lot of credits (long scripts, batches), check `GET /me` → `credits.remaining`.

### Finding a voice

`GET /voices` is public; with a key it also returns the key owner's voices. It returns `{ myVoices, library, totals }`: those voices (plus the clones teammates shared with their team) and the public library without them, each up to `limit` (default and max 100). Both lists follow `sort` exactly, ties by `id`; voices starred in the Studio carry `"favorite": true` but keep their place.

```bash
API=https://api.lazybird.app/v2
# English (UK) male voices
curl -sS "$API/voices?language=en-GB&gender=male" | jq -r '.library[] | [.id,.displayName] | @tsv'
# Every English locale, female
curl -sS "$API/voices?language=en&gender=female" | jq -r '.library[] | [.id,.displayName,.language] | @tsv'
# Character / celebrity voices
curl -sS "$API/voices?labels=Character" | jq -r '.library[] | [.id,.displayName,.description] | @tsv'
# Search names, descriptions and labels
curl -sS "$API/voices?q=narrator" | jq -r '.library[] | [.id,.displayName,.description] | @tsv'
# Voices with speaking styles (on one page)
curl -sS "$API/voices?language=en-US" | jq -r '.library[] | select((.styleList//[])|length>0) | [.id,(.styleList|join(","))] | @tsv'
```

Filters: `q` (search words, all must match), `language` (a locale like `ja-JP`, or `ja` for all its locales), `gender`, `labels` (comma-separated, all must match), `sort` (`recommended` default, `trending` for most uses in the last 7 days, `usage`, `recent`, `name`). To page, add `section=library` (or `myVoices`) and pass each response's `nextCursor` back as `cursor` with the same filters and sort; the last page has none. A malformed cursor, one from another sort, or one whose voice was removed is a `400`: start again from the first page.

Useful fields are `id`, `displayName`, `description`, `language` (primary locale), `gender` (`female`, `male` or `unknown`), `labels`, `styleList`, `previewUrl` (an MP3 to audition), `avatar` (an absolute image URL), `scope` (`global` built-in, `community` published by users, `user` a private clone), `mine` and `stats.usageCount7d`. Fields without a value are left out; the reference lists them all.

### Speech settings

`settings` is optional: `{ "speed": 1, "pitch": 1, "volume": 1, "style": "cheerful", "styleIntensity": 1, "locale": "es-ES" }`. `1` is the default for speed, pitch and volume (e.g. `0.8` is slower, `1.2` faster).

### Cloned and designed voices

- **Clone:** `POST /voices` with multipart `name` + `sample` (MP3/WAV/OGG, ≤20 MB, a clean 10–30 s recording of one speaker), or JSON `{"name","sampleUrl"}`. Returns `201 { voice }`. Use `voice.id` as `voiceId`. It uses a voice slot (3 on free, 10 on creator, unlimited on pro and max).
- **Design:** `POST /voices/design {"description":"calm, deep British male narrator","text":"Line to speak"}` returns `{ sampleUrl, seed, duration }` (it costs max(50, text length) credits). It does not create a voice. To keep it, `POST /voices {"name","sampleUrl"}`. Reuse `seed` with the same description and text to reproduce the voice.
- **Manage:** `GET|PATCH|DELETE /voices/{id}`, `POST /voices/{id}/publish|unpublish|preview`. PATCH accepts `name`, `description`, `labels` (from the allowed list), `language`, `avatarUrl` or a multipart `avatar` file.

### Account and billing

- `GET /me` → `{ plan: { tier, isPaid }, credits: { used, limit, remaining, resetsAt } }`. The free plan has 10,000 credits a month and no commercial rights. Paid plans: `creator`, `pro`, `max`.
- `POST /checkout {"plan":"creator|pro|max","interval":"monthly|yearly"}` → `{ checkoutUrl }`. Give the URL to the user to pay in a browser. The same key gets the new plan once payment completes.
- `POST /checkout {"creditsUsd":10}` → one-time extra credits that never expire, bought in whole dollars ($10–$1,000): $30 per 1M with no plan, $20 on Creator, $5 on Pro/Max.

### Errors

All errors are JSON: `{ "error": "message", "name": "InputError", "details": {...} }`.

| Status | Meaning | What to do |
|---|---|---|
| `400` | Invalid input (unknown voice, bad SSML, unknown or bad field, invalid cursor, wrong login code) | Read `error` and `details`, fix the request. Don't retry unchanged. |
| `401` | Missing or invalid API key | Check the key the app loads, or have the user create a new one |
| `422` | Plan limit: out of credits, no free voice slot, or feature not on plan | `GET /me`, then tell the user and offer `POST /checkout` |
| `429` | Rate limited | Wait `Retry-After` seconds, then retry |
| `5xx` | Server error | Retry with backoff a couple of times |

## Long scripts

Time grows with length: about 3,000 characters took ~45 s and produced ~4 minutes of audio. For audiobooks or long narration, split on paragraph or sentence boundaries into parts of up to ~3,000 characters (`generate_speech` takes 5,000 at most; the REST API has no hard cap), generate them in order (or a few in parallel), then join the MP3s, e.g. `ffmpeg -f concat -safe 0 -i list.txt -c copy out.mp3`. In SSML, `<break time="1s"/>` adds pauses between sections.

## Consent for cloned voices

Only clone a voice that is the user's own, or whose speaker agreed to it ([Acceptable Use Policy](https://www.lazybird.app/acceptable-use)). Ask the user before cloning a real person's voice, and send `rightsConfirmed: true` (multipart: `rightsConfirmed=true`) only after they confirm; a `design_voice` sample needs no question. Never clone a voice to impersonate someone or mislead listeners.

## Reference

Read [references/api-reference.md](references/api-reference.md) for every REST endpoint's request and response shape, Python and Node.js examples, the email login rules, and voice editing details.
