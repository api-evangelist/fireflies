# Fireflies.ai GraphQL API

The Fireflies.ai GraphQL API provides a single endpoint at `https://api.fireflies.ai/graphql` for querying and mutating all meeting intelligence data. The API exposes rich access to transcripts (with sentence-level detail and speaker attribution), AI-generated summaries, action items, sentiment analytics, bite clips, AskFred AI conversation threads, channels, contacts, user management, and team-level conversation analytics. All meeting data recorded or uploaded through Fireflies is accessible via strongly-typed GraphQL operations, making it straightforward to integrate meeting context into CRM, project management, and workflow automation tools.

Authentication uses API key bearer tokens passed in the `Authorization` header as `Bearer <api_key>`. Keys are obtained from the Fireflies account settings. The API supports both queries for reading data and mutations for creating, updating, and deleting resources such as transcripts, bites, AskFred threads, and live meeting sessions. Rate limits apply per account plan tier.

Key object types in the schema include `Transcript` (the primary resource, with nested `Sentence`, `Summary`, `Analytics`, `Speaker`, `MeetingAttendee`, and `Channel` objects), `Bite` (a shareable clip extracted from a transcript), `AskFredThread` and `AskFredMessage` (AI conversation threads over meeting content), `Channel` (workspace organizational groups), `Contact`, `User`, `UserGroup`, and comprehensive analytics types covering both per-transcript and team-level conversation metrics. Mutations support uploading audio files, managing live meeting sessions, sharing transcripts, updating privacy, and administering user roles.

**Endpoint:** https://api.fireflies.ai/graphql

**Documentation:** https://docs.fireflies.ai/getting-started/introduction

**References:**

- Documentation: https://docs.fireflies.ai/getting-started/introduction
- GettingStarted: https://docs.fireflies.ai/getting-started/authentication
- Reference: https://docs.fireflies.ai/graphql-api/queries
- SDKs: https://github.com/firefliesai/fireflies-node-sdk
- n8n Integration: https://github.com/firefliesai/n8n-nodes-fireflies
