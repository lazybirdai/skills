---
name: lazybird-api
description: Use when building with the Lazybird Speech API, generating MP3 speech from text or SSML, listing Lazybird voices, authenticating API requests, choosing voice settings, debugging Lazybird API integrations, or adding Lazybird text-to-speech to an application.
license: MIT
metadata:
  author: Lazybird
  version: "1.0.0"
  homepage: https://www.lazybird.app
  source: https://github.com/lazybirdai/skills
  documentation: https://www.lazybird.app/docs/api-reference
---

# Lazybird API

Use the Lazybird Speech API to list voices and generate MP3 speech from text or SSML.

## Critical Rules

- Never put Lazybird API keys in browser code, mobile clients, public repos, or client-side logs.
- Prefer the `X-API-Key` header. Use `api_key` query parameter only when a tool cannot set headers.
- `POST /generate-speech` returns an MP3 file, not JSON. Save the response body to a file or binary buffer.
- Call `GET /voices` first when you need a valid `voiceId`.
- Provide either `text` or `ssml` for speech generation.
- Keep SSML valid XML and escape quotation marks correctly inside JSON.

## Base URL

```text
https://api.lazybird.app/v1
```

## Authentication

```http
X-API-Key: YOUR_API_KEY
```

## Quick Example

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

## Endpoints

| Endpoint | Purpose |
|---|---|
| `GET /status` | Check API health |
| `GET /voices` | List voices and voice IDs |
| `POST /generate-speech` | Generate MP3 speech from text or SSML |

## When You Need Details

Read [references/api-reference.md](references/api-reference.md) for request bodies, response shapes, settings, and examples.

Official docs: https://www.lazybird.app/docs/api-reference
