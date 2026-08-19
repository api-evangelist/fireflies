---
name: Firefliesai
description: Use when building integrations with meeting transcription and analysis, querying meeting data via GraphQL, setting up AI-powered meeting analysis with AskFred, configuring MCP servers for AI tools, managing meeting transcripts, or building real-time transcription features.
metadata:
    mintlify-proj: firefliesai
    version: "1.0"
---

# Fireflies.ai API Skill

## Product Summary

Fireflies.ai is a GraphQL-based API for accessing and managing meeting transcripts, summaries, and insights. The API runs on `https://api.fireflies.ai/graphql` and requires bearer token authentication. Key capabilities include querying transcripts and metadata, uploading audio/video, managing meetings, using AskFred (AI-powered Q&A), accessing real-time transcription via WebSocket, and integrating with AI tools via MCP (Model Context Protocol). The primary documentation is at https://docs.fireflies.ai with comprehensive page navigation at https://docs.fireflies.ai/llms.txt.

## When to Use

Reach for this skill when:
- Building integrations that need to fetch meeting transcripts, summaries, or action items
- Querying meeting data with filters (date range, participants, organizers)
- Uploading audio or video files for transcription
- Implementing AI-powered meeting analysis using AskFred (requires AI credits)
- Setting up MCP servers to connect AI tools (Claude, Cursor, Devin) to meeting data
- Creating soundbites (clips) from meetings
- Sharing meetings with external users or managing access
- Building real-time transcription features with WebSocket connections
- Retrieving team analytics or user information
- Automating meeting organization (moving to channels, updating titles)

## Quick Reference

### Authentication
All requests require `Authorization: Bearer your_api_key` header. Get your API key from app.fireflies.ai → Integrations → Fireflies API.

### API Endpoint
```
https://api.fireflies.ai/graphql
```

### Rate Limits
| Plan | Limit |
|------|-------|
| Free | 50 requests/day |
| Pro | 500 requests/day |
| Business/Enterprise | 60 requests/min |

Special limits: Add to Live (3 req/20 min), Share Meeting (10 req/hour, max 50 emails/request)

### Upload Limits
| Type | Free | Pro/Business/Enterprise |
|------|------|------------------------|
| Audio | 200MB | 200MB |
| Video | 100MB | 1.5GB |

### Core GraphQL Operations
- **Queries**: users, transcripts, transcript, channels, active_meetings, askfred_threads, analytics, bites, contacts
- **Mutations**: uploadAudio, createAskFredThread, continueAskFredThread, createBite, shareMeeting, updateMeetingTitle, updateMeetingChannel, setUserRole, deleteTranscript

### MCP Tools (for AI integration)
- Search/retrieve: `fireflies_search`, `fireflies_get_transcripts`, `fireflies_get_transcript`, `fireflies_fetch`, `fireflies_get_summary`
- Management: `fireflies_share_meeting`, `fireflies_revoke_meeting_access`, `fireflies_update_meeting_title`, `fireflies_move_meeting`
- Metadata: `fireflies_get_active_meetings`, `fireflies_get_analytics`, `fireflies_list_channels`, `fireflies_get_user`
- Soundbites: `fireflies_create_soundbite`, `fireflies_get_soundbites`

## Decision Guidance

### When to Use GraphQL API vs MCP
| Scenario | Use GraphQL | Use MCP |
|----------|------------|--------|
| Custom backend integration | ✓ | |
| Direct AI tool access (Claude, Cursor) | | ✓ |
| Programmatic automation | ✓ | |
| AI-assisted meeting analysis | ✓ | ✓ |
| Building web/mobile apps | ✓ | |

### When to Query vs Fetch vs Search
| Tool | Best For | Returns |
|------|----------|---------|
| `fireflies_search` | Complex filtering with grammar syntax | Multiple meetings with summaries |
| `fireflies_get_transcripts` | Structured queries with clear filters | Multiple meetings with summaries |
| `fireflies_get_transcript` | Full conversation with timestamps | Single meeting, no summary |
| `fireflies_fetch` | Complete data in one call | Single meeting with transcript + summary |
| `fireflies_get_summary` | Quick insights only | Single meeting summary only |

### When to Use AskFred vs Direct Queries
| Use Case | AskFred | Direct Query |
|----------|---------|--------------|
| Natural language questions | ✓ | |
| Specific field extraction | | ✓ |
| Multi-meeting analysis | ✓ | ✓ |
| Requires AI credits | ✓ | |
| Structured data retrieval | | ✓ |

## Workflow

### 1. Fetch Meeting Transcripts
1. Determine if you need single meeting or multiple meetings
2. For single meeting: use `fireflies_get_transcript` (conversation only) or `fireflies_fetch` (complete data)
3. For multiple meetings: use `fireflies_get_transcripts` with filters (keyword, date range, participants)
4. Extract meeting IDs and transcript content from response
5. Parse sentences array for speaker attribution and timestamps

### 2. Query with AskFred (AI Analysis)
1. Verify account has AI credits (check Upgrade page if error: `require_ai_credits`)
2. Create thread: `createAskFredThread` with query, transcript_id (or filters for multi-meeting)
3. Extract `thread_id` and `answer` from response
4. For follow-ups: `continueAskFredThread` with thread_id and new query
5. Review `suggested_queries` for next questions

### 3. Upload Audio/Video
1. Prepare file (max 200MB audio, 1.5GB video for paid plans)
2. Use `uploadAudio` mutation with file URL, title, optional webhook
3. Optionally include `clientReferenceId` for tracking
4. Receive webhook notification when transcription completes
5. Query transcript using returned meeting ID

### 4. Set Up MCP Server for AI Tools
1. Get API key from app.fireflies.ai → Settings → Developer Settings
2. Add Fireflies MCP server to AI tool config: `https://api.fireflies.ai/mcp`
3. For Claude Desktop: add to `claude_desktop_config.json` with Authorization header
4. Restart AI tool
5. Invoke MCP tools directly from AI tool (e.g., "Search for meetings about Q4 roadmap")

### 5. Share Meeting with External Users
1. Get meeting ID from transcript query
2. Use `shareMeeting` mutation with meeting_id, email array (max 50), optional expiry_days (7/14/30)
3. Verify user is meeting owner or team admin
4. To revoke: use `revokeMeetingAccess` with meeting_id and email

### 6. Create Soundbite (Meeting Clip)
1. Fetch transcript to identify key moments by timestamp
2. Use `fireflies_create_soundbite` with transcript_id, startTime (seconds), endTime (seconds)
3. Optionally set name, privacy settings, summary
4. Receive soundbite ID and share URL in response

## Common Gotchas

- **Missing AI credits**: AskFred mutations fail with `require_ai_credits` error. Upgrade account first.
- **Auth header format**: Must be `Authorization: Bearer your_api_key` (not `Basic` or missing `Bearer`)
- **Webhook only for owners**: Webhooks only fire for meeting organizer. Set up for correct user.
- **Rate limit headers**: Check response headers for remaining quota; implement exponential backoff on 429 errors.
- **Transcript not ready**: Newly uploaded audio takes time to process. Poll with `activeTranscripts` query or wait for webhook.
- **MCP vs GraphQL confusion**: MCP tools are wrappers around GraphQL; use MCP for AI tools, GraphQL for backend code.
- **Meeting ID vs Transcript ID**: These are interchangeable in the API; use either term.
- **Expired API keys**: Keys don't auto-rotate; regenerate manually if compromised.
- **Webhook signature verification**: Always verify `x-hub-signature` header using shared secret to prevent spoofing.
- **Soundbite time boundaries**: startTime must be >= 0, endTime must be > startTime; use milliseconds or seconds consistently.
- **Share meeting email limit**: Max 50 emails per request; batch larger lists.
- **Realtime API beta**: WebSocket API is in beta; features may change; use with caution in production.

## Verification Checklist

Before submitting work:
- [ ] API key is stored securely (environment variable, not hardcoded)
- [ ] Authorization header includes `Bearer` prefix
- [ ] Requests target correct endpoint: `https://api.fireflies.ai/graphql`
- [ ] GraphQL query/mutation syntax is valid (test in GraphQL Playground)
- [ ] Rate limits are respected (implement backoff for 429 responses)
- [ ] Error responses are handled (check `errors` array in response)
- [ ] Webhook URLs are HTTPS and publicly accessible
- [ ] Webhook signatures are verified if secret is configured
- [ ] AskFred queries have AI credits available
- [ ] Meeting IDs are valid before querying (test with `transcripts` query first)
- [ ] MCP server configuration matches AI tool requirements
- [ ] Pagination is implemented for large result sets (use `skip`/`limit`)
- [ ] Sensitive data (transcripts, emails) is not logged in plain text

## Resources

**Comprehensive Navigation**: https://docs.fireflies.ai/llms.txt

**Critical Documentation Pages**:
1. [Quickstart](https://docs.fireflies.ai/getting-started/quickstart) — Make your first API request in 5 minutes
2. [Authorization](https://docs.fireflies.ai/fundamentals/authorization) — API key management and security
3. [MCP Tools Reference](https://docs.fireflies.ai/mcp-tools/overview) — Complete tool reference for AI integration

---

> For additional documentation and navigation, see: https://docs.fireflies.ai/llms.txt