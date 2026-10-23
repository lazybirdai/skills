# Lazybird API Reference

The Lazybird Speech API turns text or SSML into MP3 speech. It also clones voices from audio samples, designs voices from text descriptions, and manages and publishes your voices.

- Base URL: `https://api.lazybird.app/v1`
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
- [Errors](#errors)
- [Deprecated: /personal-voices](#deprecated-personal-voices)

## Authentication

Send the API key on every request except `GET /status`, `GET /voices` (the public list) and `/auth/*`:

```http
X-API-Key: YOUR_API_KEY
```

Fallback, only for clients that cannot set headers: `?api_key=YOUR_API_KEY`.

Users create keys in Lazybird Studio settings, or get one through the email flow below. Keep keys on the server. Never put them in browser code, mobile apps, public repositories or logs.

## Get an API key by email

Use this when the user has no key. It needs only an email address and no web login. Ask the user which email to use.

```bash
curl -sS -X POST "https://api.lazybird.app/v1/auth/request-code" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com"}'
```

- Returns `202 { "sent": true, "expiresInSeconds": 600 }`, whether or not an account exists.
- Disposable email domains get `400`.
- One code per email per minute and five per hour. Extra requests get `429` with `Retry-After`.

The user reads the 6-digit code from their inbox. You can read it yourself if you have inbox access. Then:

```bash
curl -sS -X POST "https://api.lazybird.app/v1/auth/verify-code" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","code":"123456","keyName":"my agent"}'
```

- Returns `{ "apiKey", "userId", "email", "isNewAccount" }`. A new account starts on the free plan.
- Codes expire after 10 minutes and work once. Five wrong attempts cancel the code.
- Each verification creates a new key, and existing keys keep working.
- `keyName` is optional, up to 100 characters.

## Account and upgrade

```bash
curl -sS "https://api.lazybird.app/v1/me" -H "X-API-Key: $LAZYBIRD_API_KEY"
```

```json
{
  "userId": "abc",
  "email": "you@example.com",
  "plan": { "tier": "free", "name": "Free", "isPaid": false },
  "credits": { "used": 1200, "limit": 3000, "extraCredits": 0, "remaining": 1800, "resetsAt": "2026-11-01T00:00:00.000Z" }
}
```

- `tier` is `free`, `creator`, `pro`, `max`, or `legacy` for older plans.
- The free plan has 3,000 credits a month and no commercial usage rights. Paid plans include commercial rights.

Upgrade:

```bash
curl -sS -X POST "https://api.lazybird.app/v1/checkout" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" -H "Content-Type: application/json" \
  -d '{"plan":"pro","interval":"monthly"}'
```

- Returns `{ "checkoutUrl": "https://..." }`. `plan` is `creator`, `pro` or `max`, and `interval` is `monthly` or `yearly`.
- Give the URL to the user to pay in a browser. The plan applies as soon as payment completes, and the same key keeps working.
- Pricing: https://www.lazybird.app/pricing

## Voices

### List voices

```http
GET /voices
GET /voices?owner=me
```

- `GET /voices` is public and returns the full library as a JSON array: about 530 voices, about 470 KB.
- `?owner=me` needs a key and returns only the caller's own voices (private clones and voices they published).
- `owner` accepts only `me`.
- There is no server-side search. Download the list once, cache it, and filter it locally:

```bash
curl -sS https://api.lazybird.app/v1/voices -o voices.json
jq -r '.[] | select(.language=="en-US" and .gender=="female") | [.id,.displayName,(.styleList//[]|join(","))] | @tsv' voices.json
jq -r '.[] | select(.language=="es-MX" or ((.secondaryLocaleList//[])|index("es-MX"))) | .id' voices.json
jq -r '.[] | select(.labels|index("Narrator")) | [.id,.displayName,.description] | @tsv' voices.json
```

### Voice object

```json
{
  "id": "tgJ2Hes6DauwTg8kFrRPLN",
  "displayName": "Elyssia",
  "description": "YouTube Host",
  "language": "en-US",
  "gender": "female",
  "labels": ["News", "Narrator", "Female", "Young Adult"],
  "secondaryLocaleList": ["de-DE", "es-ES", "fr-FR", "ja-JP"],
  "styleList": [],
  "previewText": "Hey everyone, I'm Elyssia. Today we're looking at a story you've probably heard wrong, so let's get into it.",
  "previewUrl": "https://cdn.lazybird.app/.../preview.mp3",
  "avatar": "https://cdn.lazybird.app/.../avatar.jpg",
  "scope": "global"
}
```

| Field | Notes |
|---|---|
| `id` | Pass as `voiceId`. An opaque string: copy it, never build or guess one. |
| `displayName`, `description` | Name, plus the voice's role or use case |
| `language` | Primary BCP-47 locale, e.g. `en-US`, `ja-JP` (150+ locales) |
| `gender` | `female`, `male` or `unknown` |
| `labels` | Tags such as `Narrator`, `Podcast`, `Female`, `Young Adult`. Ignore entries of the form `key=value`, which are internal metadata. |
| `languageGroup` | `multilingual` for voices that speak many languages |
| `secondaryLocaleList` | Other locales the voice can speak through `settings.locale` |
| `styleList` | Values accepted by `settings.style`. It's empty for most voices. |
| `previewUrl` | MP3 sample to audition the voice |
| `avatar` | Image URL. Relative paths are relative to `https://www.lazybird.app`. |
| `scope` | `global` (built-in), `community` (published by a user) or `user` (your private voice) |
| `published`, `mine` | Public-library flag, and whether the caller owns the voice |

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

Send exactly one of `text` or `ssml`, and make it non-empty. The API does not reject empty input.

Settings (all optional):

| Field | Type | Notes |
|---|---|---|
| `speed` | number | `1` is normal. `0.8` is slower, `1.2` faster. |
| `pitch` | number | `1` is the default |
| `volume` | number | `1` is the default |
| `style` | string | A value from the voice's `styleList`, e.g. `cheerful`, `sad`, `whispering`. Other values are silently ignored. |
| `styleIntensity` | number | Strength of `style` |
| `locale` | string | BCP-47 locale override for multilingual and cloned voices, e.g. `es-ES`, `ja-JP` |

Success: `200`, `Content-Type: audio/mpeg`, body = MP3 bytes. Failure: a JSON error (see [Errors](#errors)). Always check the status before writing the body to a `.mp3` file.

```bash
curl -sS -X POST "https://api.lazybird.app/v1/generate-speech" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" -H "Content-Type: application/json" \
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

`<prosody>`, `<emphasis>`, `<say-as>`, `<voice>`, `<audio>`, `<sub>`, `<p>`, `<s>` and `<lang>` are **not** supported. For rate, pitch and volume, use `settings`.

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
import json, os, urllib.request

def lazybird_tts(text: str, voice_id: str, out_path: str, settings: dict | None = None) -> None:
    body = {"voiceId": voice_id, "text": text}
    if settings:
        body["settings"] = settings
    req = urllib.request.Request(
        "https://api.lazybird.app/v1/generate-speech",
        data=json.dumps(body).encode(),
        headers={"X-API-Key": os.environ["LAZYBIRD_API_KEY"], "Content-Type": "application/json"},
        method="POST",
    )
    # urlopen raises HTTPError on 4xx/5xx; e.read() holds the JSON error
    with urllib.request.urlopen(req, timeout=300) as resp, open(out_path, "wb") as f:
        f.write(resp.read())

lazybird_tts("Hello from Lazybird.", "tgJ2Hes6DauwTg8kFrRPLN", "hello.mp3")
```

Node.js 18+:

```js
import { writeFile } from "node:fs/promises";

async function lazybirdTts(text, voiceId, outPath, settings) {
  const res = await fetch("https://api.lazybird.app/v1/generate-speech", {
    method: "POST",
    headers: { "X-API-Key": process.env.LAZYBIRD_API_KEY, "Content-Type": "application/json" },
    body: JSON.stringify({ voiceId, text, settings }),
    signal: AbortSignal.timeout(300_000),
  });
  if (!res.ok) throw new Error(`Lazybird ${res.status}: ${(await res.json()).error}`);
  await writeFile(outPath, Buffer.from(await res.arrayBuffer()));
}

await lazybirdTts("Hello from Lazybird.", "tgJ2Hes6DauwTg8kFrRPLN", "hello.mp3");
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
curl -sS -X POST "https://api.lazybird.app/v1/voices" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -F "name=Narrator" -F "rightsConfirmed=true" -F "sample=@sample.wav"
```

Or send JSON with a publicly reachable `sampleUrl`:

```bash
curl -sS -X POST "https://api.lazybird.app/v1/voices" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" -H "Content-Type: application/json" \
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
curl -sS -X POST "https://api.lazybird.app/v1/voices/design" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" -H "Content-Type: application/json" \
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
curl -sS -X PATCH "https://api.lazybird.app/v1/voices/abc123" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"Narrator","description":"Deep, calm narrator","labels":["Narrator","Male"],"language":"en-US","avatarUrl":"https://example.com/avatar.png"}'
```

| Field | Notes |
|---|---|
| `name` | Display name |
| `description` | Short text on the voice card, such as the voice's role or use case |
| `labels` | Replaces all labels. Allowed: `Male`, `Female`, `Child`, `Young Adult`, `Adult`, `Elder`, `Narrator`, `Audiobook`, `Podcast`, `Character`, `Celebrity`, `Streamer` |
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

## Errors

Errors are JSON:

```json
{ "error": "Invalid Input: Voice not found", "name": "InputError", "details": { "voiceId": "nope" } }
```

| Status | Meaning | What to do |
|---|---|---|
| `400` | Invalid input: unknown `voiceId`, unsupported SSML, missing or unknown field, wrong or expired login code | Fix the request using `error` and `details`. Don't retry unchanged. |
| `401` | Missing or invalid API key | Check `X-API-Key`, or get a key with the email flow |
| `422` | Plan limit: out of credits, no free voice slot, or feature not on the plan | `GET /me` to confirm, then tell the user. Offer `POST /checkout` to upgrade. |
| `429` | Too many requests | Wait `Retry-After` seconds, then retry |
| `5xx` | Server error | Retry with exponential backoff, up to 2–3 times |

`GET /status` (no key) returns `{ "status": "ok" }` when the service is up.

## Deprecated: /personal-voices

`/personal-voices` still works but is deprecated. Its responses carry `Deprecation` and `Link: </v1/voices>; rel="successor-version"` headers. Don't use it in new code:

| Deprecated | Use instead |
|---|---|
| `GET /personal-voices` | `GET /voices?owner=me` |
| `POST /personal-voices` | `POST /voices` |
| `DELETE /personal-voices/{voiceId}` | `DELETE /voices/{voiceId}` |
| `POST /personal-voices/{voiceId}/generate-speech` | `POST /generate-speech` with `voiceId` |
