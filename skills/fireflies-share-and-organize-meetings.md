---
name: Share and organize Fireflies meetings
description: The write path an agent is actually allowed to take — share a transcript with people, revoke that access, rename it, and file it into a channel, with the ownership and rate-limit rules that govern each.
api: https://api.fireflies.ai/graphql
surface: graphql
operations: [shareMeeting, revokeSharedMeetingAccess, updateMeetingTitle, updateMeetingChannel, channels, channel, transcripts]
mcp_tools: [fireflies_share_meeting, fireflies_revoke_meeting_access, fireflies_update_meeting_title, fireflies_move_meeting, fireflies_list_channels, fireflies_get_channel]
generated: '2026-08-14'
method: generated
source: https://docs.fireflies.ai/graphql-api/mutation/share-meeting
---

# Share and organize Fireflies meetings

These four mutations are the write surface Fireflies exposes to agents through MCP. Everything
destructive (delete, privacy change, role assignment) is deliberately withheld from the tool
surface — if you need those, you are on the GraphQL API directly and you should treat them as
human-approved actions.

## Before any write: you must own it or be an admin

Every operation here requires that the authenticated user is the **meeting owner** or a **team
admin**. Otherwise you get `forbidden` (403) or `require_elevated_privilege` (403).

## Share a meeting

```graphql
mutation ShareMeeting($input: ShareMeetingInput!) {
  shareMeeting(input: $input) { success message }
}
```

```json
{ "input": { "meeting_id": "your_meeting_id", "emails": ["user@example.com"], "expiry_days": 7 } }
```

- `expiry_days` must be one of **7, 14, 30**. Omit it for no expiry.
- **Rate limit: 10 requests per hour**, and each request carries at most **50 email addresses**.
  The MCP tool description says up to 100 recipients while the rate-limit documentation caps a
  request at 50 — treat **50 as the safe ceiling** and split larger audiences across requests,
  staying inside the 10/hour budget.
- There is no idempotency key. Re-sending the same share re-sends the notification.

Read back who currently has access via the `shared_with` field on `transcript`:

```graphql
query Transcript($id: String!) {
  transcript(id: $id) { shared_with { email name expires_at } }
}
```

## Revoke access

```graphql
mutation RevokeSharedMeetingAccess($input: RevokeSharedMeetingAccessInput!) {
  revokeSharedMeetingAccess(input: $input) { success message }
}
```

One email per call.

## Rename a meeting

```graphql
mutation UpdateMeetingTitle($input: UpdateMeetingTitleInput!) {
  updateMeetingTitle(input: $input) { success title }
}
```

Title must be **5-256 characters**. Shorter or longer fails with `invalid_arguments` and
`extensions.metadata.fields` naming `title`.

## File meetings into a channel

List the channels you can write to first:

```graphql
query Channels { channels { id title is_private members { user_id email name } } }
```

Then move up to **5 transcripts** in one call:

```graphql
mutation UpdateMeetingChannel($input: UpdateMeetingChannelInput!) {
  updateMeetingChannel(input: $input) { success }
}
```

This is **all-or-nothing**: if any transcript in the batch fails validation, none are moved. That
is atomicity, not idempotency — a successful call repeated is still a second call.

Channel visibility rules: a user sees all public channels plus private channels they belong to.
Super admins see every channel in the team.

## Errors to handle

| code | status | meaning |
|---|---|---|
| `forbidden` | 403 | Not the owner. |
| `require_elevated_privilege` | 403 | Admin-only action attempted as a normal user. |
| `not_in_team` | 403 | Target user is outside your team. |
| `object_not_found` | 404 | Bad meeting or channel id; check `extensions.metadata.objectType`. |
| `invalid_arguments` | 400 | Read `extensions.metadata.fields`. |
| `too_many_requests` | 429 | Back off until `extensions.metadata.retryAfter`. |

## Agent safety note

Sharing a meeting transcript sends meeting content to third parties and cannot be undone by
revoking — revocation removes future access, it does not unsend. Treat `shareMeeting` as a
consequential, human-confirmable action. See `agentic-access/fireflies-agentic-access.yml`.
