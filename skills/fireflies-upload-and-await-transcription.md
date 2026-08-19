---
name: Upload audio to Fireflies and await transcription
description: Submit an audio or video file for transcription and receive the completion event, including webhook signature verification and the correlation id that ties an event back to your upload.
api: https://api.fireflies.ai/graphql
surface: graphql
operations: [uploadAudio, transcript]
generated: '2026-08-14'
method: generated
source: https://docs.fireflies.ai/graphql-api/mutation/upload-audio
---

# Upload audio to Fireflies and await transcription

Transcription is asynchronous. You submit a URL, Fireflies fetches and processes the media, then
notifies your endpoint. There is no polling contract published, so wire the webhook.

## 1. Check the plan gate and size limits before you call

- `uploadAudio` requires **Pro or higher**. On Free it fails with `paid_required` and
  `extensions.metadata.tier: pro_or_higher`.
- Audio: up to 200MB on every plan.
- Video: 100MB on Free, up to 1.5GB on Pro / Business / Enterprise.
- Anything under 50kb fails with `payload_too_small`.

## 2. Submit the upload

```graphql
mutation UploadAudio($input: AudioUploadInput!) {
  uploadAudio(input: $input) {
    success
    title
    message
  }
}
```

```json
{
  "input": {
    "url": "https://example.com/path/to/media.mp4",
    "title": "Q3 customer interview",
    "webhook": "https://your-app.example.com/hooks/fireflies",
    "client_reference_id": "your-own-correlation-id",
    "custom_language": "en",
    "save_video": true,
    "attendees": [
      { "displayName": "Alice Example", "email": "alice@example.com" }
    ]
  }
}
```

Key fields:

- **`webhook`** — a per-upload webhook. It overrides nothing and applies only to this one upload,
  unlike the account-level webhook configured in the dashboard.
- **`client_reference_id`** — your own id, echoed back on the completion event. This is the only
  way to match an event to the upload that produced it.
- **`download_auth`** — supply bearer-token or HTTP basic credentials when the source URL is behind
  authentication (private S3/GCS/Azure, protected file servers).

## No idempotency — retry carefully

Fireflies publishes **no** idempotency-key contract. Calling `uploadAudio` twice ingests the file
twice and bills two transcriptions. `client_reference_id` correlates, it does **not** deduplicate.
Before retrying a call whose response you did not see, query `transcripts` for the title first.

## 3. Receive the completion event

Prefer **Webhooks V2** (configured at app.fireflies.ai/integrations/api/webhook) and subscribe to
`meeting.transcribed`:

```json
{
  "event": "meeting.transcribed",
  "timestamp": 1710876543210,
  "meeting_id": "ASxwZxCstx",
  "client_reference_id": "your-own-correlation-id"
}
```

Subscribe to `meeting.summarized` as well if you need the AI summary — the transcript is ready
before the summary is.

Your endpoint must return **2xx within 10 seconds** or the delivery is marked failed. No retry
behaviour is documented, so acknowledge immediately and process out of band.

## 4. Verify the signature

Every delivery carries `X-Hub-Signature: sha256=<hex HMAC-SHA256 of the raw request body>`,
computed with the signing secret you configured. Compute your own over the **raw** body (not the
re-serialized JSON) and compare with a timing-safe equality check. Reject on mismatch.

## 5. Fetch the result

Use the `meeting_id` from the event as the transcript id:

```graphql
query Transcript($id: String!) {
  transcript(id: $id) {
    id
    title
    duration
    summary { overview action_items keywords }
    sentences { speaker_name text start_time }
  }
}
```

## Errors to handle

| code | status | meaning |
|---|---|---|
| `paid_required` | 403 | Upload requires Pro or higher. |
| `payload_too_small` | 400 | File under 50kb. |
| `invalid_language_code` | 400 | `custom_language` not supported. |
| `too_many_requests` | 429 | Back off until `extensions.metadata.retryAfter`. |

## Gotcha: who receives webhooks

Webhooks fire only for meetings **you own** (you are the `organizer_email`). Team-wide webhooks
require the Enterprise tier with the Super Admin role.
