# Feature 13 -- Chat & AI

**Dependencies:** 01-auth-users, 02-clinics-orgs-pods
**Source:** Dashboard prod (Hope AI + Slack integration)
**Users:** Admin, Growth, AM, CC (all internal users)

---

## Hope AI

Claude-powered chat assistant with access to live Meta data.

**Model:** Haiku 4.5 default, Sonnet toggle in toolbar
**Streaming:** Server-Sent Events (SSE) for real-time responses
**Tool calling:** Claude can call `get_insights(accountId, datePreset, level)` to pull live Meta data into conversations
**System prompt:** Injects Meta account context based on selected subaccount in clinic switcher
**Max tool rounds:** 4 per conversation turn

### API
```
POST /api/v1/chat          -- SSE streaming (request body: messages array, model preference)
```

## Slack Integration

HTTP-based via Slack Web API. Two channels per clinic.

- **Psycle channel** (`settings.externalSlackId`): Client-facing communication
- **Staff channel** (`settings.internalSlackId`): Internal team discussion

**Methods:** `conversations.history` (load messages), `chat.postMessage` (post)
**Auth:** Single agency-level `SLACK_BOT_TOKEN` (xoxb-)
**Scopes:** channels:history, channels:read, chat:write

### API
```
POST /api/v1/chat/slack/context     -- Set Slack session context (channel selection)
GET  /api/v1/chat/slack/history     -- Load message history
POST /api/v1/chat/slack/send        -- Post message
```

**Future:** Upgrade to Socket Mode (`SLACK_APP_TOKEN` xapp-) for real-time event listening.

## Psycle Analyst (Future -- from Psycle-Data)

Natural language -> SQL query engine. Currently a separate Cloud Run service in Sebastian's stack (`psycle-analyst`). Pilot limited to Sebastian + account managers.

- Vetted SQL templates only (no arbitrary queries)
- Scoped by department access
- Will port to Flywheel when the analytics layer is mature enough
