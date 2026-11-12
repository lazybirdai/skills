---
name: lazybird-api
description: Make speech, voiceovers and videos with Lazybird (MCP tools and the api.lazybird.app REST API). Turn text or SSML into MP3 speech; build voiceover projects with several voices (podcasts, dialogue, audiobooks); make video ads, explainers and social videos with stock footage, music and on-screen text; export MP4/MOV video, MP3/WAV/OGG audio and TXT/DOCX/SRT/VTT subtitles and transcripts; share a project by link; clone a voice from a sample or design one from a description. Use when the user wants a voiceover, narration, a multi-voice podcast or dialogue, a video with voiceover, subtitles, an export or share link, to add TTS or video rendering to an app or agent, to clone or design a voice, to get a Lazybird API key, or to debug a Lazybird API integration.
license: MIT
metadata:
  author: Lazybird
  version: "2.0.0"
  homepage: https://www.lazybird.app
  source: https://github.com/lazybirdai/skills
  documentation: https://www.lazybird.app/docs/api-reference
---

# Lazybird speech, voiceovers and video

Turn text into natural speech, build voiceover and video projects and export them, clone a voice from a short sample, or design one from a description. One credit = one character of spoken text.

Pick the way in by what the user wants:

- **Audio or video now** (a voiceover, a sample to listen to, a video to post): use the Lazybird MCP tools. They act on the user's Lazybird account through an OAuth sign-in, so no API key is involved.
- **Speech or video in their code** (an app, script or backend): write calls to the REST API with an API key that the app loads from its own config or secret store.
- **One voice, one short script** → `generate_speech` / `POST /generate-speech`. **Several voices, a long script, a video, subtitles or a share link** → a project (`create_project` / `POST /projects`).

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
| `list_projects` | The user's projects, latest edited first: id, name, kind (`audio`/`video`), first line. Filter with `q`, page with `cursor`. |
| `create_project` | `{ name? }` → an empty project (one empty voiceover block). |
| `read_project` | The whole project as JSON: voiceover blocks in order (id, voice, text, `speech` state, start, duration), timeline blocks, animations, canvas, uploads. Has `outOfCredits` when speech stopped for lack of credits. |
| `read_video_guide` | How to build the picture and sound: every block, animation, canvas and style shape, with examples. Read it before the first `add_block`. |
| `edit_project` | `{ projectId, changes: [...] }`: up to 200 changes, applied in order, all or none. Voiceover, blocks, animations, canvas, styles (see below). |
| `generate_project_speech` | `{ projectId, blockIds?, regenerate? }` → makes speech for blocks that need it, in the background. |
| `export_project` | `{ projectId, type, format, width?, height?, allowMissingSpeech? }` → starts a render (see formats below). |
| `get_exports` | A project's exports, latest first: status, and the file's URL once `completed`. Files are kept 30 days. |
| `share_project` | A link anyone can open to watch or listen. Plays the latest export while it matches the project, otherwise a live preview. |
| `add_project_file` | `{ projectId, url, name }` copies a `search_stock_media` or `search_music` result (or another Lazybird link) into the project. Other web links are refused. |
| `search_stock_media` | `{ kind: images|videos, q, page? }` → free stock photos or clips, each with a ready `block` for `add_block` (give it a `startTime`). |
| `search_music` | `{ q?, instrumental?, minDuration?, maxDuration?, page?, limit? }` → music tracks with a `cdnUrl` and `duration`. |

1. Help write or polish the script. If the user hasn't picked a voice, `search_voices` and suggest two or three that fit, with their preview links.
2. `generate_speech`, then share the link. To save a file, download it: `curl -sSL -o voiceover.mp3 "<audioUrl>"`.
3. Before long jobs or batches, check `get_account`. When the user runs out, say so and link https://www.lazybird.app/pricing.

- Every voice, cloned ones included, speaks every supported language. For text in another language than the voice's, pass `locale` with a region (`es-ES`).
- `speed` runs from 0.5 to 2; 1 is normal.
- Treat voice ids as opaque: copy them from `search_voices`, never build them from a name.
- `generate_speech` speaks plain text at a speed. For SSML (pauses, pronunciations), pitch, volume or a speaking style, put the script in a project: voiceover blocks take `ssml` and `voiceSettings`.
- The tools don't cover editing, publishing and deleting voices. For those, the user can use the Lazybird app, or you write REST calls for them to run with their own key.

### Voiceover and video projects

A project is a voiceover script (blocks, each with its own voice if wanted) on a timeline that can also hold video, images, text, shapes, music and animations. It opens in the Lazybird Studio too (`projectUrl`), so the user can keep editing it there.

1. `create_project`, then `edit_project` with `insert_voiceover` changes: `{"action":"insert_voiceover","ref":"a1","text":"...","voiceId":"..."}`. Give a new item a `ref`; later changes in the same call use it as its id (`"after":"a1"`). Each voice in a dialogue is its own block. A pause is `{"action":"insert_voiceover","after":"a1","pause":0.5}`.
2. `generate_project_speech`, then `read_project` until no block is `speech being made`. Every block is then `has speech` (or `empty`, `needs speech`, `speech failed`). Now each block has its `start` and `duration`: time the visuals to them.
3. For a video: `read_video_guide`, set the canvas (`set_canvas`, e.g. 1080×1920 for a vertical ad), then `add_block` the pictures (`search_stock_media` results, or `add_project_file` uploads), on-screen text and a music bed (`search_music`, kept well under the voice). Read `read_project` again to check the result.
4. `export_project`, then `get_exports` until the export is `completed` (or `failed`), and share its `url`. Export a transcript as `srt` or `vtt` for subtitles. `share_project` gives a link to the finished project.

| `type` | `format` | Notes |
|---|---|---|
| `video` | `mp4`, `mov` | Needs a picture. `width` and `height` (both, even) to render at another size. |
| `audio` | `mp3`, `wav`, `ogg` | The REST API also takes `phone` (`pcm`/`mulaw`) with `wav`: 8 kHz mono for phone systems. |
| `transcript` | `txt`, `docx`, `srt`, `vtt` | `srt`/`vtt` are timed subtitles. |

- Every block needs speech before an export, unless `allowMissingSpeech: true` (those blocks are silent).
- Each account renders a few exports at a time and a limited length of output a day (seconds of rendered file, video counted apart from audio). Past that, the tool says so and the REST API returns `429` with `Retry-After`: wait, then export again.
- Speech uses the user's characters. When they run out, `read_project` has `outOfCredits` with an `upgradeUrl`: tell the user and give them the link.
- Changes the user made in the Studio show up in `read_project`: read before editing.

## Text-to-speech in code (REST API)

- Base URL: `https://api.lazybird.app/v2`
- Auth header: `X-API-Key: YOUR_API_KEY` (query `?api_key=` only when headers are impossible)
- OpenAPI 3.1: https://www.lazybird.app/openapi.json · Agent docs: https://api.lazybird.app/llms.txt

1. **Key.** The user creates one at https://www.lazybird.app/platform (API Keys) or gets one by email code (`POST /auth/request-code` → `POST /auth/verify-code`, see the reference). Write the code to load it from the app's config or secret store and ask the user to put it there. Never write a key into code.
2. **Voice.** `GET /voices` is public: look up voice ids without a key (see "Finding a voice").
3. **Generate.** `POST /generate-speech {"voiceId","text"}` returns raw `audio/mpeg` bytes. Write them to a file. Long text takes time: allow a 120+ second timeout.
4. **Projects** for several voices, video, subtitles or a share link: see "Projects and video" below.
5. **Errors.** Read the JSON `error` message and follow the error table below.

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
- SSML supports only `<speak>`, `<break time="500ms|2s"/>` (max 60s), `<phoneme alphabet="ipa" ph="...">word</phoneme>` and `<sub alias="spoken words">word</sub>`. Any other tag (`<prosody>`, `<emphasis>`, `<say-as>`, `<voice>`...) is a `400`. Use `settings` for speed, pitch and volume instead. Escape `&`, `<`, `>` in text, and escape quotes inside JSON.
- `settings.style` only works with a value from the voice's `styleList` (most voices have none). Other values are ignored, together with `styleIntensity`.
- For text in another language than the voice's `language`, pass `settings.locale` as a locale with a region (`es-ES`; a bare `es` is a `400`).
- Before spending a lot of credits (long scripts, batches), check `GET /me` → `credits.remaining`.

### Finding a voice

`GET /voices` is public; with a key it also returns the key owner's voices. It returns `{ myVoices, library, totals }`: those voices (plus the clones teammates shared with their team) and the public library (the owner's published voices included), each up to `limit` (default and max 100). Both lists follow `sort` exactly, ties by `id`; voices starred in the Studio carry `"favorite": true` but keep their place.

```bash
API=https://api.lazybird.app/v2
# English (UK) male voices
curl -sS "$API/voices?language=en-GB&gender=male" | jq -r '.library[] | [.id,.displayName] | @tsv'
# Every English locale, female
curl -sS "$API/voices?language=en&gender=female" | jq -r '.library[] | [.id,.displayName,.language] | @tsv'
# Character voices
curl -sS "$API/voices?labels=Character" | jq -r '.library[] | [.id,.displayName,.description] | @tsv'
# Search names, descriptions and labels
curl -sS "$API/voices?q=narrator" | jq -r '.library[] | [.id,.displayName,.description] | @tsv'
# Voices with speaking styles (on one page)
curl -sS "$API/voices?language=en-US" | jq -r '.library[] | select((.styleList//[])|length>0) | [.id,(.styleList|join(","))] | @tsv'
```

Filters: `q` (search words, all must match), `language` (a locale like `ja-JP`, or `ja` for all its locales), `gender`, `labels` (comma-separated, all must match), `sort` (`recommended` default, `trending` for most uses in the last 7 days, `usage`, `recent`, `name`). To page, add `section=library` (or `myVoices`) and pass each response's `nextCursor` back as `cursor` with the same filters and sort; the last page has none. A malformed cursor, one from another sort, or one whose voice was removed is a `400`: start again from the first page.

Useful fields are `id`, `displayName`, `description`, `language` (primary locale), `gender` (`female`, `male` or `unknown`), `labels`, `styleList`, `previewUrl` (an MP3 to audition), `avatar` (an absolute image URL), `scope` (`global` built-in, `community` published by users, `user` a private clone), `mine` and `stats.usageCount7d`. Fields without a value are left out; the reference lists them all.

### Speech settings

`settings` is optional: `{ "speed": 1, "pitch": 1, "volume": 1, "style": "cheerful", "styleIntensity": 1, "locale": "es-ES", "instruction": "whispering, nervous" }`. `1` is the default for speed, pitch and volume (e.g. `0.8` is slower, `1.2` faster). `instruction` says how the speech is acted, in a few words; it is never spoken.

### Cloned and designed voices

- **Clone:** `POST /voices` with multipart `name` + `sample` (MP3/WAV/OGG, ≤20 MB, a clean 10–30 s recording of one speaker), or JSON `{"name","sampleUrl"}`. Returns `201 { voice }`. Use `voice.id` as `voiceId`. It uses a voice slot (3 on free, 10 on creator, unlimited on pro and max).
- **Design:** `POST /voices/design {"description":"calm, deep British male narrator","text":"Line to speak"}` returns `{ sampleUrl, seed, duration }` (it costs max(50, text length) credits). It does not create a voice. To keep it, `POST /voices {"name","sampleUrl"}`. Reuse `seed` with the same description and text to reproduce the voice.
- **Manage:** `GET|PATCH|DELETE /voices/{id}`, `POST /voices/{id}/publish|unpublish|preview`. PATCH accepts `name`, `description`, `labels` (from the allowed list), `language`, `avatarUrl` or a multipart `avatar` file.

### Projects and video

The same project flow as the MCP tools, as REST routes (the reference has every body and response):

1. `POST /projects {"name"}` → `201 { project: { id, projectUrl, ... } }`.
2. `GET /projects/change-schema` → the JSON Schema of every change, plus a `guide` to building the picture and sound. Then `PATCH /projects/{id} {"changes":[...]}`: the same actions as `edit_project` (`insert_voiceover`, `add_block`, `set_canvas`, `add_animation`...), up to 200, all or none. The response has `ids` for your refs and `needSpeechBlockIds`.
3. `POST /projects/{id}/speech {}` → `202`. Poll `GET /projects/{id}` until no voiceover block is `speech being made`.
4. `POST /projects/{id}/exports {"type":"video","format":"mp4"}` → `202 { export }`. Poll `GET /projects/{id}/exports/{exportId}` until `status` is `completed` (then `url` is the file) or `failed`. Same types and formats as the table above, plus `phone` for audio `wav`.
5. `POST /projects/{id}/share` → `{ url }`.

- Media: `POST /projects/{id}/uploads/from-url {"url","name"}` for a public file, or `POST /projects/{id}/uploads {"fileName","type":"image|video|audio","size"}`, then POST the file as a form to the returned `uploadUrl` (each of `uploadFields` first, the file last as `file`), then `POST /projects/{id}/uploads/{uploadId}/finalize`. Up to 100 MB a file. Use the upload's `url` in `add_block`.
- Free stock: `GET /stock/images` and `GET /stock/videos` (`q`, `page`), each result with a ready `block`; `GET /music` (`q`, `instrumental`, `minDuration`, `maxDuration`, `page`, `limit`).
- Also `GET /projects` (`q`, `limit`, `cursor`), `POST /projects/{id}/duplicate`, `DELETE /projects/{id}`, `GET /projects/{id}/exports`, `GET /projects/{id}/uploads`.
- A team API key calls only what its scopes allow: `projects:read`, `projects:write`, `exports`, `speech`, `voices:read`, `voices:write`. Personal keys can call everything.

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
| `422` | Plan limit: out of credits, no free voice slot, or feature not on plan | `GET /me`, then tell the user: give them `upgradeUrl` when the body has one, or offer `POST /checkout` |
| `429` | Rate limited, too many exports at once, or today's render length used up | Wait `Retry-After` seconds, then retry |
| `5xx` | Server error | Retry with backoff a couple of times |

## Long scripts

Time grows with length: about 3,000 characters took ~45 s and produced ~4 minutes of audio. For audiobooks or long narration, split on paragraph or sentence boundaries into parts of up to ~3,000 characters (`generate_speech` takes 5,000 at most; the REST API has no hard cap), generate them in order (or a few in parallel), then join the MP3s, e.g. `ffmpeg -f concat -safe 0 -i list.txt -c copy out.mp3`. In SSML, `<break time="1s"/>` adds pauses between sections.

## Consent for cloned voices

Only clone a voice that is the user's own, or whose speaker agreed to it ([Acceptable Use Policy](https://www.lazybird.app/acceptable-use)). Ask the user before cloning a real person's voice, and send `rightsConfirmed: true` (multipart: `rightsConfirmed=true`) only after they confirm; a `design_voice` sample needs no question. Never clone a voice to impersonate someone or mislead listeners.

## Reference

Read [references/api-reference.md](references/api-reference.md) for every REST endpoint's request and response shape (speech, voices, projects, exports, uploads, stock media and music, sharing), Python and Node.js examples, the email login rules, and voice editing details.
