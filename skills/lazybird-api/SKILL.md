---
name: lazybird-api
description: Generate MP3 speech, voiceovers and narration from text or SSML with the Lazybird text-to-speech API (api.lazybird.app), pick from 500+ voices in 150+ locales, clone a voice from an audio sample, or design a new voice from a text description. Use when the user wants to turn text into audio, add TTS or AI voices to an app, script, or agent, clone or design a voice, get a Lazybird API key, or debug a Lazybird API integration.
license: MIT
metadata:
  author: Lazybird
  version: "1.2.1"
  homepage: https://www.lazybird.app
  source: https://github.com/lazybirdai/skills
  documentation: https://www.lazybird.app/docs/api-reference
---

# Lazybird Speech API

Turn text into MP3 speech with 500+ voices, clone a voice from a short sample, or design one from a description. One credit = one character of input text.

- Base URL: `https://api.lazybird.app/v1`
- Auth header: `X-API-Key: $LAZYBIRD_API_KEY` (query `?api_key=` only when headers are impossible)
- OpenAPI 3.1: https://www.lazybird.app/openapi.json · Agent docs: https://api.lazybird.app/llms.txt

## Workflow

1. **Get a key.** Use `$LAZYBIRD_API_KEY` if it is set. If not, ask the user for their email, then:
   `POST /auth/request-code {"email"}` → user reads the 6-digit code from their inbox → `POST /auth/verify-code {"email","code","keyName"}` returns `{ apiKey, isNewAccount }`. New accounts start on the free plan. Store the key in an env var or secret store, never in code.
2. **Pick a voice.** `GET /voices` is public and returns the whole library (~530 voices, ~470 KB). Save it to a file and filter it; never print the raw list into context. See "Finding a voice" below.
3. **Generate.** `POST /generate-speech {"voiceId","text"}` returns raw `audio/mpeg` bytes. Write them to a file. Long text takes time: allow a 120+ second timeout.
4. **On errors,** read the JSON `error` message and follow the error table below.

```bash
curl -sS -X POST "https://api.lazybird.app/v1/generate-speech" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" -H "Content-Type: application/json" \
  --max-time 300 --fail-with-body -o speech.mp3 \
  --data '{"voiceId":"msa.en-US.SerenaMultilingual","text":"Hello from Lazybird."}'
```

## Critical rules

- Keep API keys server-side. Never put them in browser or mobile code, public repos or logs.
- `/generate-speech` returns MP3 bytes, not JSON and not a URL. Check the status code before saving: errors come back as JSON.
- Always send a non-empty `text` or `ssml` (not both). The API does not reject empty input.
- SSML supports only `<speak>`, `<break time="500ms|2s"/>` (max 60s) and `<phoneme alphabet="ipa" ph="...">word</phoneme>`. Any other tag (`<prosody>`, `<emphasis>`, `<say-as>`, `<voice>`...) is a `400`. Use `settings` for speed, pitch and volume instead. Escape `&`, `<`, `>` in text, and escape quotes inside JSON.
- `settings.style` only works with a value from the voice's `styleList` (most voices have none). Other values are silently ignored.
- For another language, use a voice whose `language` matches, or a multilingual or cloned voice with `settings.locale` (e.g. `es-ES`).
- Treat voice `id`s as opaque. Built-in ids look like `msa.en-US.Jenny`, while community and cloned voices have random ids.
- `/personal-voices` is deprecated. Use the `/voices` endpoints.
- Before spending a lot of credits (long scripts, batches), check `GET /me` → `credits.remaining`.

## Finding a voice

```bash
curl -sS https://api.lazybird.app/v1/voices -o voices.json
# English (UK) male voices
jq -r '.[] | select(.language=="en-GB" and .gender=="male") | [.id,.displayName] | @tsv' voices.json
# Voices that can speak Japanese (native or via settings.locale)
jq -r '.[] | select(.language=="ja-JP" or ((.secondaryLocaleList//[])|index("ja-JP"))) | .id' voices.json
# Character / celebrity voices, by name or description
jq -r '.[] | select(.labels|index("Character")) | [.id,.displayName,.description] | @tsv' voices.json
jq -r '.[] | select(.displayName|test("serena";"i")) | .id' voices.json
# Voices with speaking styles
jq -r '.[] | select((.styleList//[])|length>0) | [.id,(.styleList|join(","))] | @tsv' voices.json
```

Useful fields are `id`, `displayName`, `description`, `language` (primary locale), `gender` (`female`, `male` or `unknown`), `labels`, `styleList`, `secondaryLocaleList`, `previewUrl` (an MP3 to audition), and `scope` (`global` built-in, `community` published by users, `user` your own). Ignore labels of the form `key=value`, which are internal metadata. Multilingual voices have `languageGroup: "multilingual"`. `GET /voices?owner=me` lists only the caller's own voices.

## Speech settings

`settings` is optional: `{ "speed": 1, "pitch": 1, "volume": 1, "style": "cheerful", "styleIntensity": 1, "locale": "es-ES" }`. `1` is the default for speed, pitch and volume (e.g. `0.8` is slower, `1.2` faster).

## Long scripts

There is no hard length cap, but time grows with length: about 3,000 characters took ~45 s and produced ~4 minutes of audio. For audiobooks or long narration, split on paragraph or sentence boundaries into chunks of up to ~3,000 characters, generate them in order (or a few in parallel), then join them, e.g. `ffmpeg -f concat -safe 0 -i list.txt -c copy out.mp3`. Use `<break time="1s"/>` for pauses between sections.

## Cloned and designed voices

- **Clone:** `POST /voices` with multipart `name` + `sample` (MP3/WAV/OGG, ≤20 MB, a clean 10–30 s recording of one speaker), or JSON `{"name","sampleUrl"}`. Returns `201 { voice }`. Use `voice.id` as `voiceId`. It uses a voice slot (3 on free, 10 on creator, unlimited on pro and max).
- **Consent:** only clone a voice that is the user's own, or whose speaker agreed to it ([Acceptable Use Policy](https://www.lazybird.app/acceptable-use)). Ask the user before cloning a real person's voice, and send `rightsConfirmed: true` (multipart: `rightsConfirmed=true`) only after they confirm. Never clone a voice to impersonate someone or mislead listeners.
- **Design:** `POST /voices/design {"description":"calm, deep British male narrator","text":"Line to speak"}` returns `{ sampleUrl, seed, duration }` (it costs max(50, text length) credits). It does not create a voice. To keep it, `POST /voices {"name","sampleUrl"}`. Reuse `seed` with the same description and text to reproduce the voice.
- **Manage:** `GET|PATCH|DELETE /voices/{id}`, `POST /voices/{id}/publish|unpublish|preview`. PATCH accepts `name`, `description`, `labels` (from the allowed list), `language`, `avatarUrl` or a multipart `avatar` file.

## Account and billing

- `GET /me` → `{ plan: { tier, isPaid }, credits: { used, limit, remaining, resetsAt } }`. The free plan has 3,000 credits a month and no commercial rights. Paid plans: `creator`, `pro`, `max`.
- `POST /checkout {"plan":"creator|pro|max","interval":"monthly|yearly"}` → `{ checkoutUrl }`. Give the URL to the user to pay in a browser. The same key gets the new plan once payment completes.

## Errors

All errors are JSON: `{ "error": "message", "name": "InputError", "details": {...} }`.

| Status | Meaning | What to do |
|---|---|---|
| `400` | Invalid input (unknown voice, bad SSML, bad field, wrong login code) | Read `error` and `details`, fix the request. Don't retry unchanged. |
| `401` | Missing or invalid API key | Check the key, or get a new one with the email flow |
| `422` | Plan limit: out of credits, no free voice slot, or feature not on plan | `GET /me`, then tell the user and offer `POST /checkout` |
| `429` | Rate limited | Wait `Retry-After` seconds, then retry |
| `5xx` | Server error | Retry with backoff a couple of times |

## Reference

Read [references/api-reference.md](references/api-reference.md) for every endpoint's request and response shape, Python and Node.js examples, the email login rules, and voice editing details.
