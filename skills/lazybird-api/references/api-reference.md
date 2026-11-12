# Lazybird API Reference

The Lazybird API turns text or SSML into MP3 speech, and builds voiceover and video projects that it renders to video, audio and subtitle files. It also clones voices from audio samples, designs voices from text descriptions, and manages and publishes your voices.

- Base URL: `https://api.lazybird.app/v2` (`/v1` is frozen for existing integrations: it keeps the older voice object, one unpaged list of the first 1,000 library voices and the deprecated `/personal-voices` endpoints. Use `/v2` in new code.)
- OpenAPI 3.1 spec: https://www.lazybird.app/openapi.json
- Human docs: https://www.lazybird.app/docs/api-reference
- Usage draws from the same monthly credits as Lazybird Studio: one credit per character of input text.

## Contents

- [Authentication](#authentication)
- [Get an API key by email](#get-an-api-key-by-email)
- [Account and upgrade](#account-and-upgrade)
- [Voices](#voices)
- [Generate speech](#generate-speech)
- [SSML](#ssml)
- [Code examples](#code-examples)
- [Long scripts](#long-scripts)
- [Clone a voice](#clone-a-voice)
- [Design a voice](#design-a-voice)
- [Edit, publish, preview and delete a voice](#edit-publish-preview-and-delete-a-voice)
- [Projects and video](#projects-and-video)
- [Errors](#errors)

## Authentication

Send the API key on every request except `GET /status`, `GET /voices` (the public list) and `/auth/*`:

```http
X-API-Key: YOUR_API_KEY
```

Fallback, only for clients that cannot set headers: `?api_key=YOUR_API_KEY`.

Users create keys at https://www.lazybird.app/platform (API Keys), or get one through the email flow below. Apps load the key from their own config or secret store. Keep keys on the server. Never put them in browser code, mobile apps, public repositories or logs.

## Get an API key by email

Use this when the user has no key. It needs only an email address and no web login. Ask the user which email to use.

```bash
curl -sS -X POST "https://api.lazybird.app/v2/auth/request-code" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com"}'
```

- Returns `202 { "sent": true, "expiresInSeconds": 600 }`, whether or not an account exists.
- Disposable email domains get `400`.
- One code per email per minute and five per hour. Extra requests get `429` with `Retry-After`.

The user reads the 6-digit code from their inbox. You can read it yourself if you have inbox access. Then:

```bash
curl -sS -X POST "https://api.lazybird.app/v2/auth/verify-code" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","code":"123456","keyName":"my agent"}'
```

- Returns `{ "apiKey", "userId", "email", "isNewAccount" }`. A new account starts on the free plan.
- Codes expire after 10 minutes and work once. Five wrong attempts cancel the code.
- Each verification creates a new key, and existing keys keep working.
- Give the key to the user to store in their app's config or secret store. Don't write it into code.
- `keyName` is optional, up to 100 characters.

## Account and upgrade

```bash
curl -sS "https://api.lazybird.app/v2/me" -H "X-API-Key: YOUR_API_KEY"
```

```json
{
  "userId": "abc",
  "email": "you@example.com",
  "plan": { "tier": "free", "name": "Free", "isPaid": false },
  "credits": { "used": 1200, "limit": 10000, "extraCredits": 0, "remaining": 8800, "resetsAt": "2026-11-01T00:00:00.000Z" }
}
```

- `tier` is `free`, `creator`, `pro`, `max`, or `legacy` for older plans.
- The free plan has 10,000 credits a month and no commercial usage rights. Paid plans include commercial rights.
- 1 credit = 1 character. Monthly plan credits are used first, then purchased `extraCredits`, which never expire.

Upgrade:

```bash
curl -sS -X POST "https://api.lazybird.app/v2/checkout" \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"plan":"pro","interval":"monthly"}'
```

- Returns `{ "checkoutUrl": "https://..." }`. `plan` is `creator`, `pro` or `max`, and `interval` is `monthly` or `yearly`.

Buy extra credits once, with or without a plan:

```bash
curl -sS -X POST "https://api.lazybird.app/v2/checkout" \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"creditsUsd":10}'
```

- Price per 1M credits: $30 with no plan, $20 on Creator, $5 on Pro or Max. Pay any whole-dollar amount from $10 to $1,000.
- Pay-as-you-go credits without a plan are for personal use; plans include commercial rights.
- Give the URL to the user to pay in a browser. The plan applies as soon as payment completes, and the same key keeps working.
- Pricing: https://www.lazybird.app/pricing

## Voices

### List voices

```http
GET /voices
```

- Public: no key needed. With a key, your own voices come back too.
- Returns `{ "myVoices": [...], "library": [...], "totals": { "myVoices": 0, "library": 580 } }`: your voices (private clones, voices you published and the clones teammates shared with your team; your own carry `"mine": true`) and the public library (your published voices included), each up to `limit` voices. `totals` counts the matching voices in each section.
- Filter and search on the server instead of downloading the library:

| Parameter | Notes |
|---|---|
| `q` | Search words; a voice matches when each word appears in its name, description or labels |
| `language` | A locale such as `en-US`, or a language such as `en` for all of its locales. Matches the primary locale only. |
| `gender` | `female` or `male` |
| `labels` | Comma-separated; a voice must carry all of them |
| `sort` | `recommended` (default, also when searching), `trending` (most uses in the last 7 days), `usage` (most uses of all time), `recent` (newest first) or `name` (A to Z) |
| `limit` | Voices per section, 1 to 100. Default 100. |
| `section` | `myVoices` or `library`: return only that section, one page at a time |
| `cursor` | The previous page's `nextCursor`, together with `section` |

```bash
API=https://api.lazybird.app/v2
curl -sS "$API/voices?language=en-US&gender=female" | jq -r '.library[] | [.id,.displayName,(.styleList//[]|join(","))] | @tsv'
curl -sS "$API/voices?labels=Character" | jq -r '.library[] | [.id,.displayName,.description] | @tsv'
curl -sS "$API/voices?q=narrator" | jq -r '.library[] | [.id,.displayName,.description] | @tsv'
curl -sS "$API/voices?section=myVoices" -H "X-API-Key: YOUR_API_KEY" | jq -r '.myVoices[] | [.id,.displayName] | @tsv'
```

Order: both sections follow `sort` exactly, and voices that tie are ordered by `id`, so the order stays the same from page to page. Voices the caller starred in the Studio carry `"favorite": true` but keep their place.

Paging: add `section`. While more voices match, the response has `nextCursor`; send it back as `cursor` with the same filters and sort. The last page has none. A malformed cursor, one from another sort, or one whose voice was removed returns `400`; start again from the first page.

Every voice, cloned voices included, can speak every supported language through `settings.locale`, so for a language without many native voices, any voice you like works.

### Voice object

```json
{
  "id": "tgJ2Hes6DauwTg8kFrRPLN",
  "displayName": "Elyssia",
  "description": "YouTube Host",
  "language": "en-US",
  "gender": "female",
  "labels": ["News", "Narrator", "Female", "Young Adult"],
  "styleList": [],
  "previewText": "Hey everyone, I'm Elyssia.",
  "previewUrl": "https://cdn.lazybird.app/.../preview.mp3",
  "avatar": "https://cdn.lazybird.app/.../avatar.jpg",
  "scope": "global",
  "published": false
}
```

| Field | Notes |
|---|---|
| `id` | Pass as `voiceId`. Treat it as opaque: copy it from `GET /voices`, never build it from a name. Voices cloned before the unified voice list also accept their legacy `pv:` id. |
| `displayName`, `description` | Name, plus the character, show or use case for community voices |
| `language` | Primary BCP-47 locale, e.g. `en-US`, `ja-JP` |
| `detectedLanguage` | Language heard in the voice's sample, as a bare code such as `vi`. `language` wins when both are set. |
| `detectedLocale` | Accent heard in the voice's sample, such as `en-GB`, when it is clear |
| `gender` | `female`, `male` or `unknown` |
| `labels` | Tags such as `Character`, `Celebrity`, `Narrator`, `Female`, `Child` |
| `styleList` | Values accepted by `settings.style`. It's empty for most voices. |
| `previewText`, `previewUrl` | The line the preview speaks, and the MP3 to audition the voice |
| `avatar` | Absolute image URL |
| `scope` | `global` (built-in), `community` (published by a user) or `user` (a private clone) |
| `published`, `mine` | Whether a user published the voice to the library, and whether the caller owns it |
| `favorite` | `true` when the caller starred the voice in the Studio |
| `official`, `featured` | Official Lazybird voices, and voices featured in the voice library |
| `popular`, `trending` | The 10 most used voices of all time, and of the last 7 days |
| `createdAt` | When the voice was added (ISO 8601) |
| `ownerNickname`, `ownerUserId` | The creator's name and user ID, when the creator has a public profile |
| `stats` | `usageCount` (all time) and `usageCount7d` (last 7 days) on library voices; `popularRank` and `trendingRank` for the top 10 |

Fields without a value are left out. Every voice endpoint returns this shape.

A published voice also has a public page: `https://www.lazybird.app/voices/{id}`.

### Get one voice

```http
GET /voices/{voiceId}
```

Returns `{ "voice": { ... } }` for any public voice or one of your own private voices.

## Generate speech

```http
POST /generate-speech
Content-Type: application/json
```

| Field | Type | Required | Notes |
|---|---|---:|---|
| `voiceId` | string | Yes | Voice `id` from `/voices` |
| `text` | string | One of | Plain text. Special characters are escaped for you. |
| `ssml` | string | One of | SSML document, see [SSML](#ssml) |
| `settings` | object | No | See below |
| `phone` | string | No | `pcm` or `mulaw`: an 8 kHz mono WAV for phone systems (voicemail, phone menus) instead of an MP3. `mulaw` is G.711 µ-law. |

Send exactly one of `text` and `ssml`, and not an empty one; anything else is a `400`. So is any field not listed here, in the body or in `settings`.

Settings (all optional):

| Field | Type | Notes |
|---|---|---|
| `speed` | number | `1` is normal. `0.8` is slower, `1.2` faster. Must be greater than `0`. |
| `pitch` | number | `1` is the default. Must be greater than `0`. |
| `volume` | number | `1` is the default. `0` or greater (`0` is silent). |
| `style` | string | A value from the voice's `styleList`, e.g. `cheerful`, `sad`, `whispering`. Other values are ignored, together with `styleIntensity`. |
| `styleIntensity` | number | Strength of `style`. `1` is the default; the closest level the voice has is used. |
| `locale` | string | Language and accent to speak in, for any voice, as a locale with a region, e.g. `es-ES`, `ja-JP`. A bare `es` is a `400`. |
| `instruction` | string | How the speech is acted, in a few words, e.g. `whispering, nervous`. Never spoken. Up to 200 characters; brackets are dropped, and one without words is a `400`. |
| `effect` | string | A sound effect on the finished speech: `robot`, `deep`, `monster`, `echo` (a large hall), `radio` (a walkie-talkie) or `phone` (a phone call). Leave it out for plain speech. |

Success: `200`, `Content-Type: audio/mpeg`, body = MP3 bytes; with `phone`, `Content-Type: audio/wav` and WAV bytes. Failure: a JSON error (see [Errors](#errors)). Always check the status before writing the body to a `.mp3` file.

```bash
curl -sS -X POST "https://api.lazybird.app/v2/generate-speech" \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  --max-time 300 --fail-with-body -o speech.mp3 \
  --data '{
    "voiceId": "tgJ2Hes6DauwTg8kFrRPLN",
    "text": "Hola, este audio usa una configuración regional en español.",
    "settings": { "locale": "es-ES", "speed": 0.95 }
  }'
```

## SSML

Only a small, strict subset is accepted. Anything else returns `400 Invalid SSML: ...`.

| Element | Rules |
|---|---|
| `<speak>` | Required root element |
| `<break time="..."/>` | Must be empty. `time` is a number followed by `ms` or `s`, greater than 0 and at most `60s`. Without `time` it defaults to 1s. No other attributes. |
| `<phoneme alphabet="ipa" ph="...">word</phoneme>` | `alphabet` must be `ipa`, and `ph` must be non-empty. The content must be plain text and non-empty. |
| `<sub alias="...">word</sub>` | Says `alias` in place of the word. `alias` must be non-empty, and no other attributes. The content must be plain text and non-empty. |

`<prosody>`, `<emphasis>`, `<say-as>`, `<voice>`, `<audio>`, `<p>`, `<s>` and `<lang>` are **not** supported. For rate, pitch and volume, use `settings`.

The document must be valid XML: escape `&` as `&amp;`, and `<` and `>` in text as `&lt;` and `&gt;`. Inside JSON, escape the double quotes around attributes:

```json
{
  "voiceId": "tgJ2Hes6DauwTg8kFrRPLN",
  "ssml": "<speak>Welcome back. <break time=\"700ms\"/> Today we talk about <phoneme alphabet=\"ipa\" ph=\"ˈkæʃ\">cache</phoneme> design.</speak>"
}
```

Building the JSON with a library (`json.dumps`, `JSON.stringify`) avoids quoting mistakes.

## Code examples

Python (standard library only):

```python
import json, urllib.request

# api_key: load it from your app's config or secret store; never hard-code it.
def lazybird_tts(api_key: str, text: str, voice_id: str, out_path: str, settings: dict | None = None) -> None:
    body = {"voiceId": voice_id, "text": text}
    if settings:
        body["settings"] = settings
    req = urllib.request.Request(
        "https://api.lazybird.app/v2/generate-speech",
        data=json.dumps(body).encode(),
        headers={"X-API-Key": api_key, "Content-Type": "application/json"},
        method="POST",
    )
    # urlopen raises HTTPError on 4xx/5xx; e.read() holds the JSON error
    with urllib.request.urlopen(req, timeout=300) as resp, open(out_path, "wb") as f:
        f.write(resp.read())

lazybird_tts(api_key, "Hello from Lazybird.", "tgJ2Hes6DauwTg8kFrRPLN", "hello.mp3")
```

Node.js 18+:

```js
import { writeFile } from "node:fs/promises";

// apiKey: load it from your app's config or secret store; never hard-code it.
async function lazybirdTts(apiKey, text, voiceId, outPath, settings) {
  const res = await fetch("https://api.lazybird.app/v2/generate-speech", {
    method: "POST",
    headers: { "X-API-Key": apiKey, "Content-Type": "application/json" },
    body: JSON.stringify({ voiceId, text, settings }),
    signal: AbortSignal.timeout(300_000),
  });
  if (!res.ok) throw new Error(`Lazybird ${res.status}: ${(await res.json()).error}`);
  await writeFile(outPath, Buffer.from(await res.arrayBuffer()));
}

await lazybirdTts(apiKey, "Hello from Lazybird.", "tgJ2Hes6DauwTg8kFrRPLN", "hello.mp3");
```

For browser apps, call your own backend endpoint, which calls Lazybird with the key. Never call Lazybird from the browser with the key.

## Long scripts

- There is no hard length cap, but generation time grows with length. In one test, 3,000 characters took ~45 s and returned ~4 minutes of audio. Use long timeouts (120–300 s).
- For long narration, split the script into chunks of up to ~3,000 characters on paragraph or sentence boundaries. Generate the chunks in order, or a few in parallel, then concatenate them:

```bash
printf "file '%s'\n" part-*.mp3 > list.txt
ffmpeg -f concat -safe 0 -i list.txt -c copy narration.mp3
```

- Keep the same `voiceId` and `settings` for every chunk so they sound consistent.
- Cost is one credit per character, so check `GET /me` → `credits.remaining` first.

## Clone a voice

```http
POST /voices
```

Creates a private cloned voice from a clean 10–30 second recording of one speaker, with no music or background noise.

Only clone a voice that is the user's own, or whose speaker agreed to it (see the [Acceptable Use Policy](https://www.lazybird.app/acceptable-use)). Ask the user to confirm this before cloning a real person's voice, then send `rightsConfirmed: true`. It is recorded on the voice. Never clone a voice to impersonate someone or mislead listeners.

Upload a file as multipart (`name` and `sample`, MP3, WAV or OGG, up to 20 MB):

```bash
curl -sS -X POST "https://api.lazybird.app/v2/voices" \
  -H "X-API-Key: YOUR_API_KEY" \
  -F "name=Narrator" -F "rightsConfirmed=true" -F "sample=@sample.wav"
```

Or send JSON with a publicly reachable `sampleUrl`:

```bash
curl -sS -X POST "https://api.lazybird.app/v2/voices" \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"Narrator","sampleUrl":"https://example.com/sample.wav","rightsConfirmed":true}'
```

- Send `sample` or `sampleUrl`, not both.
- `rightsConfirmed` is optional, but send it once the user has confirmed they may clone the voice. A voice the user designed with `POST /voices/design` is theirs to clone.
- Returns `201 { "voice": { ... } }`, with `scope: "user"` and `published: false`.
- Each cloned voice uses a voice slot: 3 on free, 10 on creator, unlimited on pro and max. There is also a monthly creation cap (3 on free, 50 on creator). Hitting either returns `422`. Deleting a voice frees its slot but not the monthly creation.
- Generate with it like any other voice: `"voiceId": voice.id`. To speak another language, set `settings.locale`.

## Design a voice

```http
POST /voices/design
```

Generates a brand-new voice from a description of who is speaking, and returns a sample of it speaking `text`. The description itself is never spoken.

| Field | Required | Notes |
|---|---:|---|
| `description` | Yes | Up to 500 characters: age, gender, pitch, energy, accent, emotion |
| `text` | Yes | Up to 1000 characters, the line the sample says |
| `language` | No | Locale of `text`, e.g. `en-US`. It's detected when omitted. |
| `seed` | No | 0–2147483647. Omit it for a new random voice. |

```bash
curl -sS -X POST "https://api.lazybird.app/v2/voices/design" \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"description":"A calm, deep-voiced middle-aged British man, warm documentary narrator","text":"Deep in the forest, the old oak has stood for three hundred years."}'
```

```json
{ "sampleUrl": "https://cdn.lazybird.app/users/.../voice-design-....mp3", "seed": 1257365475, "duration": 4.2 }
```

- Each design costs credits equal to the length of `text`, with a minimum of 50.
- Designing does **not** create a voice or use a slot. Let the user audition `sampleUrl`, and try again without `seed` for a different voice.
- The same `seed`, description and text reproduce the same voice.
- To keep a design, clone it: `POST /voices {"name":"Narrator","sampleUrl":"<sampleUrl>"}`.

## Edit, publish, preview and delete a voice

These work on your own voices only.

Edit: send only the fields to change.

```bash
curl -sS -X PATCH "https://api.lazybird.app/v2/voices/abc123" \
  -H "X-API-Key: YOUR_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"Narrator","description":"Deep, calm narrator","labels":["Narrator","Male"],"language":"en-US","avatarUrl":"https://example.com/avatar.png"}'
```

| Field | Notes |
|---|---|
| `name` | Display name |
| `description` | Short text on the voice card, such as the character or show |
| `labels` | Replaces all labels. Allowed: `Male`, `Female`, `Child`, `Young Adult`, `Adult`, `Elder`, `Narrator`, `Audiobook`, `Podcast`, `Character`, `Celebrity`, `Streamer`, `Advertising`, `Anime`, `ASMR`, `Customer Service`, `Documentary`, `Education`, `Horror`, `Kids`, `Meditation`, `News`, `Sleep`, `Storytelling`, `Trailer`, `Calm`, `Deep`, `Energetic` |
| `language` | Locale such as `en-US` or `es-MX`. `""` clears it. Changing it clears the preview clip. |
| `avatarUrl` | Public PNG, JPEG or WebP URL |

- To upload a local avatar, send multipart with an `avatar` file (up to 5 MB). Text fields can go in the same form, with `labels` comma-separated: `-F "avatar=@avatar.png" -F "labels=Narrator,Male"`.
- Avatars are cropped to 200×200. Send a file or `avatarUrl`, not both.
- Unknown fields are rejected with `400`.
- Returns `{ "voice": { ... } }`. Edits to a published voice show up publicly right away.

Publish or unpublish:

```http
POST /voices/{voiceId}/publish
POST /voices/{voiceId}/unpublish
```

- Publishing adds the voice to the public library (`scope: "community"`), where anyone can find and use it, and gives it a page at `https://www.lazybird.app/voices/{voiceId}`.
- Set the name, description, labels, language and avatar first, because that is what people see.
- Unpublishing makes the voice private again.
- Both return `{ "voice": { ... } }`.

Preview clip:

```http
POST /voices/{voiceId}/preview
```

- Returns `{ "previewUrl": "https://...mp3", "duration": 4.2 }`, the voice speaking its `previewText`.
- The first call renders the clip, charged to the voice owner's credits. Later calls return the stored clip, without `duration`.

Delete:

```http
DELETE /voices/{voiceId}
```

Returns `{ "deleted": true }` and frees the voice slot. The voice's processed sample, preview clip and avatar are deleted too.

## Projects and video

A project is a voiceover script, optionally on a video canvas with pictures, video clips, text, shapes, music and animations. It is what the Lazybird Studio edits, so a project made through the API opens there too (`projectUrl`). Speech uses the same credits as `/generate-speech`.

The flow: `POST /projects` → `PATCH /projects/{projectId}` with changes → `POST /projects/{projectId}/speech` → `POST /projects/{projectId}/exports`, poll the export → `POST /projects/{projectId}/share`.

### Create, list, read, duplicate, delete

```http
POST /projects                          {"name": "Product launch ad"}   → 201 { project }
GET /projects?q=&limit=&cursor=          → { projects, total, nextCursor? }
GET /projects/{projectId}                → the whole project
POST /projects/{projectId}/duplicate     {"name"?}                       → 201 { project }
DELETE /projects/{projectId}
```

- A project summary has `id`, `name`, `kind` (`audio`, or `video` once it has a picture), `voiceIds`, `firstLine`, `version`, `thumbnailUrl`, `projectUrl`, `createdAt`, `updatedAt`. `limit` is 1–100 (default 20); pass `nextCursor` back as `cursor`.
- A new project has one empty voiceover block.
- `GET /projects/{projectId}` returns the voiceover in order (each block's `id`, `start`, `duration`, voice, settings, pauses, `text` and `speech` state), the voices it uses, the canvas, tracks, timeline blocks (elements, video, images, audio), animations, styles, variables and uploads. `speech` is `has speech`, `speech being made`, `needs speech`, `speech failed` or `empty`. When speech stopped because the account ran out of credits, the read has `outOfCredits` with a `message` and an `upgradeUrl`.

### Change a project

```http
GET /projects/change-schema     (no key needed)
PATCH /projects/{projectId}     {"changes": [ ... ]}
```

`GET /projects/change-schema` returns `{ description, changes, guide }`: the JSON Schema of every change and a guide to building the picture and sound (the shape of every block, animation, canvas, style, font and variable, with examples and timing craft). Read the guide before the first `add_block`.

`changes` is a list of up to 200, applied in order, all or none:

| Actions | What they change |
|---|---|
| `update_project` | `project`: `name`, details, master volume, `autoGenerate` (make speech after each edit) |
| `insert_voiceover`, `update_voiceover`, `move_voiceover`, `delete_voiceover` | Voiceover blocks (`text` up to 5,000 characters, or `ssml`; `voiceId`; `voiceSettings` `speed`/`pitch` 0.5–1.5, `volume`, `style`; pauses before and after) and pauses (`pause` in seconds). Place with `after` or `before`, else at the end. |
| `add_block`, `update_block`, `delete_block` | Text, shapes, SVG, images, video clips and audio on the timeline |
| `add_animation`, `update_animation`, `delete_animation` | Animations and transitions |
| `set_canvas` | `width`, `height` (even, 16–4096), `fps` (24, 25, 30, 60), `background` |
| `set_style`, `delete_style`, `set_font`, `delete_font`, `set_variable`, `delete_variable` | Style classes, uploaded fonts and editable variables |

Give a new item a `ref`; later changes in the same request use the ref as its id. A two-voice dialogue:

```json
{"changes": [
  {"action": "insert_voiceover", "ref": "q", "text": "Ever wished your videos could talk?", "voiceId": "VOICE_A"},
  {"action": "insert_voiceover", "ref": "a", "after": "q", "text": "Now they can, in any voice you like.", "voiceId": "VOICE_B"},
  {"action": "insert_voiceover", "after": "a", "pause": 0.4}
]}
```

The response has `changed`, a `summary`, `ids` (each ref's new id), `speechBlockIds` (speech already being made, with `autoGenerate`) and `needSpeechBlockIds`. Blocks whose text, voice or settings change need speech again.

### Make speech

```http
POST /projects/{projectId}/speech   {"blockIds"?: [...], "regenerate"?: true}   → 202 { requested }
```

Makes speech for every block that needs it, or only `blockIds`; `regenerate: true` also redoes blocks that have speech. Poll `GET /projects/{projectId}` every few seconds until no block is `speech being made`. Then each block has its `start` and `duration`: time the picture to them.

### Render (export)

```http
POST /projects/{projectId}/exports   {"type": "video", "format": "mp4"}   → 202 { export }
GET /projects/{projectId}/exports                                         → { exports }
GET /projects/{projectId}/exports/{exportId}                              → { export }
```

| `type` | `format` | Options |
|---|---|---|
| `video` | `mp4`, `mov` | `width` and `height` together (even, 16–4096) to render at another size; the canvas's by default |
| `audio` | `mp3`, `wav`, `ogg` | `phone` (`pcm` or `mulaw`) with `wav`: an 8 kHz mono WAV for phone systems |
| `transcript` | `txt`, `docx`, `srt`, `vtt` | `srt` and `vtt` are timed subtitles |

- `allowMissingSpeech: true` exports even though some blocks have no speech (they are silent). Without it, every block needs speech, and a video needs a picture: otherwise the export is refused with a message saying what's missing.
- An export has `id`, `projectId`, `type`, `format`, `resolution`, `phone`, `status`, `url`, `projectVersion`, `createdAt`, `updatedAt`. `status` is `exporting`, then `completed` (with `url`) or `failed`. Files are kept for 30 days, then `expired` without a `url`. `projectVersion` is the project `version` it rendered.
- Each account renders a few exports at a time and a limited length of output a day, counted in seconds of the rendered file, with video counted apart from audio; a transcript counts only toward exports at a time. Past either: `429` with `Retry-After` (also `retryAfterSeconds` in the body). Wait; don't retry in a loop.

### Upload media

```http
POST /projects/{projectId}/uploads/from-url        {"url", "name"}                            → 201 { upload }
POST /projects/{projectId}/uploads                 {"fileName", "type", "size"}               → 201 { upload, uploadUrl, uploadFields }
POST /projects/{projectId}/uploads/{uploadId}/finalize                                        → { upload }
GET /projects/{projectId}/uploads                                                             → { uploads }
```

- `from-url` copies a stock result or another Lazybird file (`cdn.lazybird.app`, Pexels or Pixabay URLs). Other sites are refused (`400 Unsupported asset URL`): download the file and upload it.
- A local file: `POST .../uploads` with `type` `image`, `video` or `audio` and `size` in bytes. Then POST a multipart form to `uploadUrl`, without the API key: each of `uploadFields` as a field, in order, then the file as `file`. The form works for an hour. Then `finalize`.

```bash
# uploadFields → one -F per field, in order; the file goes last
curl -X POST "$UPLOAD_URL" -F "key=..." -F "Content-Type=video/mp4" -F "Policy=..." \
  -F "X-Amz-Signature=..." ... -F "file=@clip.mp4"
```

- A file is up to 100 MB, and uploads share the plan's storage; on the free plan they are kept for 30 days.
- An upload has `id`, `name`, `type`, `url`, `status`, `thumbnailUrl`, `duration` (video, audio), `size`. Use its `url` in `add_block`.

### Stock media and music

```http
GET /stock/images?q=&page=      → { results }
GET /stock/videos?q=&page=      → { results }
GET /music?q=&instrumental=&minDuration=&maxDuration=&page=&limit=   → { tracks }
```

- Each stock result has its preview and a `block` ready for `add_block`: give it a `startTime` (and a `duration`). Its URL can be used directly, or copied into the project with `from-url`.
- Music `limit` is 1–25 (default 10); durations are in seconds. Each track has `title`, `artist`, `duration` and `cdnUrl`: use it as the audio block's URL, with `intrinsicDuration` set to the track's length. Keep music well under the voice.

### Share

```http
POST /projects/{projectId}/share   → { url }
```

A link anyone can open to watch or listen, the same each time. It plays the latest MP4 (or MP3 for a voiceover) while that export matches the project, otherwise a live preview: export after the last change to share the finished file.

### Team API keys

A team API key calls only what its scopes allow: `projects:read`, `projects:write` and `exports` for projects, next to `speech`, `voices:read` and `voices:write`. Personal keys can call everything.

## Errors

Errors are JSON:

```json
{ "error": "Invalid Input: Voice not found", "name": "InputError", "details": { "voiceId": "nope" } }
```

| Status | Meaning | What to do |
|---|---|---|
| `400` | Invalid input: unknown `voiceId`, unsupported SSML, missing or unknown field, invalid `cursor`, wrong or expired login code | Fix the request using `error` and `details`. Don't retry unchanged. |
| `401` | Missing or invalid API key | Check the key the app sends in `X-API-Key`, or create a new one |
| `422` | Plan limit: out of credits, no free voice slot, storage full, or feature not on the plan | `GET /me` to confirm, then tell the user. Give them `upgradeUrl` when the body has one, or offer `POST /checkout`. |
| `429` | Too many requests, too many exports at once, or today's render length used up | Wait `Retry-After` seconds, then retry |
| `5xx` | Server error | Retry with exponential backoff, up to 2–3 times |

`GET /status` (no key) returns `{ "status": "ok" }` when the service is up.
