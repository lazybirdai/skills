# Lazybird API Reference

## Overview

The Lazybird Speech API lets you list available voices and generate MP3 speech from text or SSML.

Base URL:

```text
https://api.lazybird.app/v1
```

## Authentication

Create an API key from Lazybird Studio settings, then send it with each request.

Preferred header:

```http
X-API-Key: YOUR_API_KEY
```

Fallback query parameter:

```text
https://api.lazybird.app/v1/voices?api_key=YOUR_API_KEY
```

Keep API keys private. Do not expose them in browser code, mobile apps, public repositories, or client-side logs.

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

Use `id` as `voiceId` when generating speech.

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
    "styleIntensity": 1
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
