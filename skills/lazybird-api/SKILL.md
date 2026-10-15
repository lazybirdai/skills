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

Use the Lazybird Speech API to list voices, generate MP3 speech from text or SSML, and create, edit and publish cloned voices.

## Critical Rules

- Never put Lazybird API keys in browser code, mobile clients, public repos, or client-side logs.
- Prefer the `X-API-Key` header. Use `api_key` query parameter only when a tool cannot set headers.
- `POST /generate-speech` returns an MP3 file, not JSON. Save the response body to a file or binary buffer.
- Call `GET /voices` first when you need a valid `voiceId`.
- Cloned voices are regular voices: list your own with `GET /voices?owner=me`, create one with `POST /voices` (multipart `sample` file upload or JSON `sampleUrl`), and use its opaque `id` as `voiceId`.
- No API key yet? Get one by email with `POST /auth/request-code` then `POST /auth/verify-code`; no studio login needed.
- Edit your voice's name, description, labels, language or avatar with `PATCH /voices/:voiceId` (JSON `avatarUrl`, or multipart `avatar` file). Unknown fields are rejected, and labels must come from the allowed list.
- `/personal-voices` is deprecated. Never use it in new code.
- Override spoken language or accent with `settings.locale` when using multilingual or cloned voices.
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
| `POST /auth/request-code` | Email a 6-digit login code (no key needed) |
| `POST /auth/verify-code` | Trade the code for a new API key; creates a free account if needed |
| `GET /me` | The key's account, plan and credit usage |
| `POST /checkout` | Get a checkout link to upgrade the plan |
| `GET /voices` | List voices and voice IDs |
| `POST /generate-speech` | Generate MP3 speech from text or SSML |
| `GET /voices?owner=me` | List your own (cloned) voices |
| `POST /voices` | Create a cloned voice from an uploaded sample file or a sample URL |
| `GET /voices/:voiceId` | Get one voice |
| `PATCH /voices/:voiceId` | Edit one of your voices: name, description, labels, language, avatar |
| `POST /voices/:voiceId/publish` | Add one of your voices to the public library |
| `POST /voices/:voiceId/unpublish` | Make a published voice private again |
| `POST /voices/:voiceId/preview` | Get the voice's preview clip URL, rendering it if needed |
| `DELETE /voices/:voiceId` | Delete one of your voices |
| `/personal-voices` (all) | Deprecated, replaced by the `/voices` endpoints above |

## When You Need Details

Read [references/api-reference.md](references/api-reference.md) for request bodies, response shapes, settings, and examples.

Official docs: https://www.lazybird.app/docs/api-reference

OpenAPI 3.1 spec: https://www.lazybird.app/openapi.json
