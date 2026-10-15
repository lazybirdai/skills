# Lazybird API Reference

## Overview

The Lazybird Speech API lets you list available voices, generate MP3 speech from text or SSML, and create, edit and publish cloned voices.

OpenAPI 3.1 spec: https://www.lazybird.app/openapi.json

Base URL:

```text
https://api.lazybird.app/v1
```

## Authentication

Create an API key from Lazybird Studio settings, or get one by email without the UI (below), then send it with each request.

Preferred header:

```http
X-API-Key: YOUR_API_KEY
```

Fallback query parameter:

```text
https://api.lazybird.app/v1/voices?api_key=YOUR_API_KEY
```

Keep API keys private. Do not expose them in browser code, mobile apps, public repositories, or client-side logs.

## Get an API Key by Email

For agents: sign up or log in with only an email address. No key is needed for these two calls.

```bash
curl -X POST "https://api.lazybird.app/v1/auth/request-code" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com"}'
```

Returns `202` with `{ "sent": true, "expiresInSeconds": 600 }`, whether or not an account exists. Disposable email domains get `400`. One code per email per minute, five per hour; extra requests get `429` with `Retry-After`.

Read the 6-digit code from the email (or ask the user for it), then:

```bash
curl -X POST "https://api.lazybird.app/v1/auth/verify-code" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","code":"123456","keyName":"my agent"}'
```

Returns `{ "apiKey", "userId", "email", "isNewAccount" }`. A new account starts on the free plan. Codes expire after 10 minutes and work once; five wrong attempts cancel the code. Each verification creates a new key, and existing keys keep working.

## Account and Upgrade

`GET /me` returns the key's account: `{ userId, email, plan: { tier, name, isPaid }, credits: { used, limit, extraCredits, remaining, resetsAt } }`. `tier` is `free`, `creator`, `pro`, `max` or `legacy`.

`POST /checkout` with `{ "plan": "creator" | "pro" | "max", "interval": "monthly" | "yearly" }` returns `{ "checkoutUrl" }`. Open it in a browser to pay; the plan applies to the same key as soon as payment completes.

## Check Service Status

```http
GET /status
```

Example:

```bash
curl "https://api.lazybird.app/v1/status"
```

Response:

```json
{
  "status": "ok"
}
```

## List Voices

```http
GET /voices
```

Example:

```bash
curl "https://api.lazybird.app/v1/voices" \
  -H "X-API-Key: $LAZYBIRD_API_KEY"
```

Response:

```json
[
  {
    "id": "msa.en-US.Jenny",
    "displayName": "Jenny",
    "language": "English (United States)",
    "gender": "Female",
    "previewText": "Hi, I am Jenny.",
    "avatar": "https://example.com/avatar.jpg"
  }
]
```

Use `id` as `voiceId` when generating speech. Add `?owner=me` to list only your own voices.

## Generate Speech

```http
POST /generate-speech
```

Headers:

| Header | Required | Value |
|---|---:|---|
| `X-API-Key` | Yes | Lazybird API key |
| `Content-Type` | Yes | `application/json` |

Body:

| Field | Type | Required | Description |
|---|---|---:|---|
| `voiceId` | string | Yes | Voice ID from `/voices` |
| `text` | string | No | Plain text to convert to speech |
| `ssml` | string | No | SSML content. Use instead of `text` for pauses or speech markup. |
| `settings` | object | No | Optional voice settings |

Provide either `text` or `ssml`.

Plain text example:

```bash
curl -X POST "https://api.lazybird.app/v1/generate-speech" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -H "Content-Type: application/json" \
  --output speech.mp3 \
  --data '{
    "voiceId": "msa.en-US.SerenaMultilingual",
    "text": "My name is Serena and I am a voice from Lazybird."
  }'
```

SSML example:

```bash
curl -X POST "https://api.lazybird.app/v1/generate-speech" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -H "Content-Type: application/json" \
  --output speech.mp3 \
  --data '{
    "voiceId": "msa.en-US.SerenaMultilingual",
    "ssml": "<speak>My name is Serena <break time=\"1s\"/> and I am a voice from Lazybird.</speak>"
  }'
```

Settings example:

```json
{
  "voiceId": "msa.en-US.SerenaMultilingual",
  "text": "This line is slower and slightly higher pitched.",
  "settings": {
    "style": "sad",
    "speed": 0.7,
    "pitch": 1.2,
    "volume": 1,
    "styleIntensity": 1,
    "locale": "es-ES"
  }
}
```

Settings:

| Field | Type | Description |
|---|---|---|
| `style` | string | Speaking style or tone, when supported by the selected voice |
| `speed` | number | Speech speed. `1` is normal speed. |
| `pitch` | number | Voice pitch. `1` is the default pitch. |
| `volume` | number | Output volume. `1` is the default volume. |
| `styleIntensity` | number | Strength of the selected speaking style |
| `locale` | string | BCP-47 locale override, such as `en-US`, `es-ES`, `fr-FR`, `de-DE`, or `ja-JP`, when supported by the selected voice |

Locale override example:

```bash
curl -X POST "https://api.lazybird.app/v1/generate-speech" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -H "Content-Type: application/json" \
  --output spanish.mp3 \
  --data '{
    "voiceId": "msa.en-US.SerenaMultilingual",
    "text": "Hola, este audio usa una configuración regional en español.",
    "settings": {
      "locale": "es-ES"
    }
  }'
```

## List Your Voices

```http
GET /voices?owner=me
```

Same list and voice shape as `GET /voices`, filtered to the caller's own voices (private clones and voices they published).

```bash
curl "https://api.lazybird.app/v1/voices?owner=me" \
  -H "X-API-Key: $LAZYBIRD_API_KEY"
```

Response (abbreviated):

```json
[
  {
    "id": "abc123",
    "displayName": "Narrator",
    "scope": "user",
    "published": false,
    "mine": true
  }
]
```

Treat `id` as opaque. Voices created before the unified voice list also carry a `slug` starting with `pv:`; that legacy id still works anywhere a voice ID is accepted.

## Create a Voice

```http
POST /voices
```

Creates a private cloned voice. Upload the sample as `multipart/form-data` with fields `name` (text) and `sample` (MP3, WAV, or OGG file, up to 20 MB; a clean 10–30 second recording of one speaker):

```bash
curl -X POST "https://api.lazybird.app/v1/voices" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -F "name=Narrator" \
  -F "sample=@sample.wav"
```

Or send JSON with a publicly reachable `sampleUrl`:

```bash
curl -X POST "https://api.lazybird.app/v1/voices" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "Narrator",
    "sampleUrl": "https://example.com/sample.wav"
  }'
```

Send either `sample` or `sampleUrl`, not both. Returns `201 Created` with `{ "voice": { ... } }` in the same shape as `GET /voices`. Requires a plan with voice cloning access and a free voice slot.

Generate speech with a cloned voice through the normal endpoint, passing its `id` as `voiceId`:

```bash
curl -X POST "https://api.lazybird.app/v1/generate-speech" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -H "Content-Type: application/json" \
  --output cloned.mp3 \
  --data '{
    "voiceId": "abc123",
    "text": "Este audio usa mi voz clonada en español.",
    "settings": {
      "locale": "es-ES"
    }
  }'
```

Get one voice (any public voice, or one of your own):

```http
GET /voices/{voiceId}
```

Returns `{ "voice": { ... } }`, including `description`, `labels`, `language`, `avatar`, `previewText` and `previewUrl` when set.

## Edit, Publish and Preview a Voice

Edit one of your voices. Send only the fields to change:

```bash
curl -X PATCH "https://api.lazybird.app/v1/voices/abc123" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "Narrator",
    "description": "Deep, calm narrator",
    "labels": ["Narrator", "Male"],
    "language": "en-US",
    "avatarUrl": "https://example.com/avatar.png"
  }'
```

| Field | Notes |
|---|---|
| `name` | Display name |
| `description` | Short text on the voice card, such as the character or show |
| `labels` | Replaces the labels. Allowed: `Male`, `Female`, `Child`, `Young Adult`, `Adult`, `Elder`, `Narrator`, `Audiobook`, `Podcast`, `Character`, `Celebrity`, `Streamer` |
| `language` | Locale such as `en-US`, `en-GB`, `es-MX`, `ja-JP`; `""` clears it. Changing it clears the preview clip. |
| `avatarUrl` | Public PNG, JPEG or WebP URL |

To upload a local avatar image, send multipart with the file as `avatar` (up to 5 MB). Text fields can go in the same form, with `labels` comma-separated:

```bash
curl -X PATCH "https://api.lazybird.app/v1/voices/abc123" \
  -H "X-API-Key: $LAZYBIRD_API_KEY" \
  -F "avatar=@avatar.png" \
  -F "labels=Narrator,Male"
```

Avatars are cropped to a 200×200 square. Send a file or `avatarUrl`, not both. Unknown fields (for example `avatar` as a JSON string) are rejected with `400`. Returns `{ "voice": { ... } }`.

Publish to the public library, or make it private again:

```http
POST /voices/{voiceId}/publish
POST /voices/{voiceId}/unpublish
```

Both return `{ "voice": { ... } }`. A published voice has `scope: "community"` and a public page at `https://www.lazybird.app/voices/{voiceId}`. Set the name, description, labels, language and avatar before publishing.

Get the preview clip (the voice speaking its `previewText`):

```http
POST /voices/{voiceId}/preview
```

Returns `{ "previewUrl": "https://...mp3", "duration": 4.2 }`. The first call renders the clip, charged to the voice owner's credits. Later calls return the stored clip without `duration`.

Delete one of your voices:

```http
DELETE /voices/{voiceId}
```

Returns `{ "deleted": true }`.

## Deprecated: /personal-voices

`/personal-voices` still works but is deprecated. Responses carry `Deprecation` and `Link: </v1/voices>; rel="successor-version"` headers. Do not use it in new code:

| Deprecated | Use instead |
|---|---|
| `GET /personal-voices` | `GET /voices?owner=me` |
| `POST /personal-voices` | `POST /voices` |
| `DELETE /personal-voices/{voiceId}` | `DELETE /voices/{voiceId}` |
| `POST /personal-voices/{voiceId}/generate-speech` | `POST /generate-speech` with `voiceId` |

Successful speech generation returns an MP3 file:

```http
HTTP/1.1 200 OK
Content-Type: audio/mpeg
```

Testing tips:

- Make API requests from a server environment, not browser code.
- Start by calling `/voices` and copying a voice ID from the response.
- Save `generate-speech` responses with `--output speech.mp3`.
- Keep SSML valid XML.
