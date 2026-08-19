---
name: Fetch Fireflies meeting intelligence
description: Find the right meetings in Fireflies and pull their transcripts, summaries and action items — the read path an agent needs before it can reason about what was said.
api: https://api.fireflies.ai/graphql
surface: graphql
operations: [transcripts, transcript, contacts, user]
mcp_tools: [fireflies_get_transcripts, fireflies_get_transcript, fireflies_get_summary, fireflies_get_user_contacts]
generated: '2026-08-14'
method: generated
source: https://docs.fireflies.ai/graphql-api/query/transcripts
---

# Fetch Fireflies meeting intelligence

Fireflies is a single-endpoint GraphQL API. Every call is `POST https://api.fireflies.ai/graphql`
with a JSON body `{ "query": "...", "variables": { ... } }`.

## Authenticate

Send `Authorization: Bearer <api_key>` on every request. The key comes from
app.fireflies.ai → Integrations → Fireflies API. There is no OAuth on the GraphQL API — OAuth
exists only on the MCP server at `https://api.fireflies.ai/mcp`.

If the header is missing or wrong you get error code `auth_failed`. Note that this arrives with
HTTP **500**, not 401 — do not treat the transport status as the error.

## 1. Narrow to the right meetings first

Use the `transcripts` query. It is offset-paginated with `limit` (max 50) and `skip`, and there is
no total or has-more field in the response, so stop when a page comes back short.

```graphql
query Transcripts(
  $keyword: String
  $scope: String
  $fromDate: DateTime
  $toDate: DateTime
  $participants: [String]
  $organizers: [String]
  $mine: Boolean
  $channel_id: String
  $limit: Int
  $skip: Int
) {
  transcripts(
    keyword: $keyword
    scope: $scope
    fromDate: $fromDate
    toDate: $toDate
    participants: $participants
    organizers: $organizers
    mine: $mine
    channel_id: $channel_id
    limit: $limit
    skip: $skip
  ) {
    id
    title
    date
    duration
    host_email
    organizer_email
    participants
    is_live
  }
}
```

Filter rules that will bite you:

- Use `keyword` + `scope` (`title` | `sentences` | `all`). The `title` argument is deprecated.
- Use the array forms `organizers` / `participants`. The singular `organizer_email` /
  `participant_email` are deprecated, and passing an old field **together with** its replacement
  is a validation error (`invalid_arguments`).
- `limit` is capped at 50.

## 2. Pull the content you actually need

`transcript(id:)` is the same field whether you want conversation or summary — ask for only the
sub-selection you need, because Fireflies' own guidance is to keep selection sets minimal.

```graphql
query Transcript($id: String!) {
  transcript(id: $id) {
    id
    title
    date
    duration
    is_live
    transcript_url
    audio_url
    video_url
    summary {
      overview
      short_summary
      gist
      action_items
      keywords
      topics_discussed
      meeting_type
      outline
    }
    sentences {
      index
      speaker_name
      text
      start_time
      end_time
    }
  }
}
```

Two behaviours to handle:

- **`is_live: true`** means the meeting is still running and `sentences` returns a point-in-time
  snapshot of live captions, not a finished transcript. Re-fetch, or subscribe to the Realtime API.
- Meeting id and transcript id are the same value. MCP calls it `meetingId`, GraphQL calls it
  `transcript_id`.

## 3. Find meetings by who was in them

```graphql
query Contacts { contacts { email name last_meeting_date } }
```

Then feed the chosen email into `transcripts(participants: [...])`.

## Errors to handle

| code | status | what to do |
|---|---|---|
| `auth_failed` | 500 | Fix the `Authorization: Bearer` header. |
| `object_not_found` | 404 | Check the id; `extensions.metadata.objectType` names what was missing. |
| `invalid_arguments` | 400 | Read `extensions.metadata.fields` for the offending argument. |
| `paid_required` | 403 | `extensions.metadata.tier` names the plan required. |
| `too_many_requests` | 429 | Back off until `extensions.metadata.retryAfter` (epoch ms). |

Errors arrive in the `errors[]` array with `message`, `code`, `friendly` and `extensions`. The real
status is `extensions.status`, not the HTTP status. See `errors/fireflies-error-codes.yml`.

## Rate limits

Free 50 requests/day, Pro 500 requests/day, Business and Enterprise 60 requests/minute. There are
**no** `X-RateLimit-*` or `Retry-After` response headers — the only backoff signal is
`extensions.metadata.retryAfter` inside the `too_many_requests` error. Budget your calls up front;
you cannot discover remaining quota at runtime. See `rate-limits/fireflies-rate-limits.yml`.
