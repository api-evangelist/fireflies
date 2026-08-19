---
name: Record and stream a live Fireflies meeting
description: Send the Fireflies bot into a meeting already in progress, control its recording state, and consume live transcription over WebSocket as people speak.
api: https://api.fireflies.ai/graphql
surface: graphql+websocket
operations: [addToLiveMeeting, active_meetings, updateMeetingState, createLiveActionItem, createLiveSoundbite, live_action_items]
generated: '2026-08-14'
method: generated
source: https://docs.fireflies.ai/graphql-api/mutation/add-to-live
---

# Record and stream a live Fireflies meeting

This is the real-time path: dispatch the bot, then subscribe to transcription events instead of
waiting for a finished transcript.

## 1. Dispatch the bot into a live call

```graphql
mutation AddToLiveMeeting($meetingLink: String!) {
  addToLiveMeeting(meeting_link: $meetingLink) {
    success
  }
}
```

Optional arguments, all with published bounds — respect them or you get `invalid_arguments`:

| argument | type | bounds |
|---|---|---|
| `title` | String | max 256 chars; a default is generated if omitted |
| `meeting_password` | String | max 32 chars |
| `duration` | Int | minutes, min 15, max 120, defaults to 60 |
| `language` | String | max 5 chars, defaults to English |
| `attendees` | [Attendee] | expected participants |

**Hard rate limit: 3 requests per 20 minutes**, on every plan. This is the tightest limit Fireflies
publishes, and there is no header telling you how much of it is left — the only signal is
`too_many_requests` with `extensions.metadata.retryAfter`. Do not retry blindly in a loop.

An unsupported meeting platform URL fails with `unsupported_platform`.

## 2. Find meetings already in progress

```graphql
query ActiveMeetings {
  active_meetings {
    id
    title
    organizer_email
    meeting_link
    start_time
    privacy
    state
  }
}
```

`state` is `active` or `paused`. Admins can query any team member; a regular user can only see
their own. The `id` you get here is the transcript id you will use everywhere else.

## 3. Subscribe to live transcription

The Realtime API is **Socket.IO over WebSocket** — not raw WS, not GraphQL subscriptions. It is in
beta and the provider states endpoints may change.

```ts
import { io } from 'socket.io-client';

const socket = io('wss://api.fireflies.ai', {
  path: '/ws/realtime',
  auth: {
    token: 'Bearer <YOUR_API_TOKEN>',
    transcriptId: '<TRANSCRIPT_ID>'
  }
});

socket.on('auth.success', d => console.log('authenticated', d));
socket.on('auth.failed', e => console.error('auth failed', e));   // socket disconnects after this
socket.on('connection.established', () => console.log('connected'));
socket.on('connection.error', e => console.error('connection error', e));
socket.on('transcription.broadcast', ev => handle(ev));
```

Each `transcription.broadcast` payload is:

```json
{
  "transcript_id": "abc123",
  "chunk_id": "chunk_001",
  "text": "Hello world",
  "speaker_name": "Alice",
  "start_time": 0.0,
  "end_time": 1.25
}
```

**Deduplicate on `chunk_id`.** A revised segment is re-broadcast with the *same* `chunk_id`; a new
segment gets a different one. Treat a repeated `chunk_id` as a replacement, not an append, or the
transcript you assemble will contain duplicated speech.

## 4. Act during the meeting

```graphql
mutation UpdateMeetingState($input: UpdateMeetingStateInput!) {
  updateMeetingState(input: $input) { success action }
}
```

with `{ "input": { "meeting_id": "...", "action": "pause_recording" } }` to pause or resume.

`createLiveActionItem` and `createLiveSoundbite` capture action items and clips mid-call, and
`live_action_items` reads them back.

## 5. Get notified when the bot arrives

Subscribe to the Webhooks V2 `meeting.bot_joined` event — it is the intended trigger for starting a
realtime streaming workflow, rather than polling `active_meetings`.

## Note for agent builders

None of the operations in this skill are exposed as MCP tools. The Fireflies MCP server can *read*
active meetings (`fireflies_get_active_meetings`) but cannot dispatch the bot, pause recording, or
subscribe to the stream. Live control is GraphQL-only. See `mcp/fireflies-tool-crosswalk.yml`.
