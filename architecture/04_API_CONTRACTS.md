# MakunAI — API Contracts & Schema Reference

> Exact request/response shapes, error code mappings, MongoDB document structures, and SSE wire protocol.

---

## 1. Standard Response Envelope

**Every non-SSE endpoint** returns one of these shapes:

### Success (200 / 201)
```json
{
  "status": "success",
  "message": "Human-readable message",
  "data": { ... }
}
```

### Error (400 / 401 / 403 / 404 / 409 / 500)
```json
{
  "status": "error",
  "error_code": "NOT_FOUND",
  "detail": "Agent not found."
}
```

### Response Classes → HTTP Codes
| Response Class | HTTP Code | `error_code` |
|---|---|---|
| `SuccessResponse` | 200 | — |
| `ResourceCreatedResponse` | 201 | — |
| `SuccessNoContentResponse` | 204 | — |
| `AcceptedResponse` | 202 | — |
| `BadRequestResponse` | 400 | `BAD_REQUEST` |
| `ValidationErrorResponse` | 400 | `VALIDATION_ERROR` |
| `UnauthorizedResponse` | 401 | `UNAUTHORIZED` |
| `ForbiddenResponse` | 403 | `FORBIDDEN_REQUEST` |
| `NotFoundResponse` | 404 | `NOT_FOUND` |
| `ResourceConflictResponse` | 409 | `RESOURCE_CONFLICT_REQUEST` |
| `InternalServerErrorResponse` | 500 | `INTERNAL_SERVER_ERROR` |

---

## 2. Agents API (`/ai/v1/agents`)

### POST `/ai/v1/agents` — Create Agent

**Request Body** (`AgentCreate`):
```json
{
  "name": "Support Agent",                    // required, 1-255 chars
  "description": "Handles support queries",   // optional
  "provider": "openai",                       // "openai" | "anthropic", default: "openai"
  "model": "gpt-4o",                          // required
  "temperature": 0.7,                         // 0.0-1.0, default: 0.7
  "max_tokens": 1024,                         // 64-8192, default: 1024
  "system_prompt": "You are a helpful...",     // required, min 10 chars
  "first_message": "Hi! How can I help?",     // optional, voice greeting
  "context_text": "Company FAQ content...",    // optional, injected into prompt
  "voice_id": "aura-2-asteria-en",            // optional, TTS voice
  "medium": "text",                            // "text" | "voice", default: "text"
  "max_tool_calls_per_turn": 5,               // 1-20, default: 5
  "sliding_window": 15,                       // 5-50, default: 15
  "agent_config": {},                         // optional, arbitrary JSON config
  "tools": []                                 // optional, array of ToolCreate
}
```

**Response** (201):
```json
{
  "status": "success",
  "message": "Agent created successfully",
  "data": {
    "partner_id": 42,
    "agent_id": 1,
    "agent_name": "Support Agent",
    "description": "Handles support queries",
    "provider": "openai",
    "model": "gpt-4o",
    "temperature": 0.7,
    "max_tokens": 1024,
    "system_prompt": "You are a helpful...",
    "first_message": "Hi! How can I help?",
    "context_text": null,
    "voice_id": null,
    "medium": "text",
    "max_tool_calls_per_turn": 5,
    "is_active": false,
    "created_at": "2026-05-13T00:00:00",
    "tools": [],
    "sliding_window": 15
  }
}
```

### Error Mapping — Agents
| Exception | HTTP | When |
|---|---|---|
| `AgentNotFoundError` | 404 | Agent ID doesn't exist or wrong partner |
| `AgentAlreadyExistsError` | 400 | Duplicate agent name for same partner |
| `AgentPublishError` | 400 | Missing system_prompt or already published |
| `AgentConfigurationError` | 400 | Invalid config on publish |
| `AgentDeactivationError` | 400 | Agent already deactivated |

### PUT `/ai/v1/agents/{agent_id}` — Update Agent

**Request Body** (`AgentUpdate`) — all fields optional:
```json
{
  "name": "New Name",
  "description": "Updated desc",
  "provider": "anthropic",
  "model": "claude-3-5-sonnet-20241022",
  "medium": "text",
  "max_tool_calls_per_turn": 10,
  "temperature": 0.5,
  "max_tokens": 2048,
  "system_prompt": "Updated prompt...",
  "first_message": "Hello!",
  "context_text": "Updated context",
  "voice_id": "new-voice-id",
  "is_active": true,
  "sliding_window": 20,
  "agent_config": {"new": "value"}
}

> Note: Update requests support explicit `null` values to clear optional fields.
```

### PUT `/ai/v1/agents/{agent_id}/tools` - Update Agent Tools
Replaces the agent's tool mappings with the provided list of tool IDs.

**Request Body** (`AgentToolsUpdate`):
```json
{
  "tool_ids": [1, 2, 5]
}
```

**Response** (200):
```json
{
  "status": "success",
  "message": "Tools updated successfully",
  "data": {
    "tools": [ /* ToolRead objects */ ]
  }
}
```

### GET `/ai/v1/agents` — List Agents

**Query Params:** `include_inactive=false`, `skip=0`, `limit=50`

**Response** (200):
```json
{
  "status": "success",
  "message": "Agents retrieved successfully",
  "data": {
    "agents": [ /* AgentRead objects */ ],
    "total": 12,
    "skip": 0,
    "limit": 50,
    "has_more": false
  }
}
```

### POST `/ai/v1/agents/{agent_id}/publish` — Publish Agent
No request body. Returns updated `AgentRead` with `is_active=true`.

### DELETE `/ai/v1/agents/{agent_id}` — Deactivate Agent
No request body. Returns 204 No Content.

### GET `/ai/v1/agents/{agent_id}/routes` — List Sub-Agent Routes
Lists all active delegation routes for this orchestrator agent.

**Response** (200):
```json
{
  "status": "success",
  "message": "success",
  "data": {
    "routes": [
      {
        "id": 1,
        "parent_agent_id": 10,
        "child_agent_id": 20,
        "child_agent_name": "Sales Agent",
        "routing_description": "Handles sales inquiries.",
        "routing_description_override": null,
        "priority": 0,
        "is_active": true,
        "created_at": 1690000000
      }
    ]
  }
}
```

### POST `/ai/v1/agents/{agent_id}/routes` — Add Sub-Agent Route
Configures a child agent that this agent can delegate tasks to.

**Request Body**:
```json
{
  "child_agent_id": 20,
  "routing_description_override": "Custom instructions for when to route here."
}
```

**Response** (200):
Returns the created route object.

### DELETE `/ai/v1/agents/{agent_id}/routes/{route_id}` — Remove Sub-Agent Route
Soft-deletes a sub-agent route. Returns 204 No Content.

### GET `/ai/v1/agents/available-models` — List Available Models

**Query Params:**
- `provider`: Filter by provider (e.g., `openai`, `anthropic`, `gemini`).
- `search`: Case-insensitive substring search for model names.

**Response** (200):
```json
{
  "status": "success",
  "message": "Available models retrieved successfully",
  "data": {
    "models": [
      {
        "id": "gpt-4o",
        "provider": "openai",
        "context_limit": 128000
      }
    ]
  }
}
```

---

## 3. Chat API (`/ai/v1/chat`)

### POST `/ai/v1/chat/message` — Stream Chat (SSE)

**Request Body** (`ChatMessageRequest`):
```json
{
  "agent_id": 1,                   // required
  "session_id": "sess_abc",        // required, unique thread ID
  "message": "What is in this image?",  // optional, default "" — allows media-only turns
  "interface": "chat",             // "chat" | "whatsapp" | "pstn" | "telegram" | "email", default: "chat"
  "medium": "text",                // "text" | "voice", default: "text"
  "attachments": [                 // optional, pre-uploaded refs
    {
      "asset_id": 12345,
      "media_type": "image",       // "image" | "audio" | "video" | "document"
      "filename": "receipt.jpg",
      "mime_type": "image/jpeg"
    }
  ]
}
```
> Note: `partner_id` is injected from auth middleware, NOT sent by client.
> `attachments[].asset_id` must come from a prior `POST /ai/v1/chat/attachments` for the same `session_id`.
> Either `message` or at least one attachment must be provided; both empty yields a no-op stream.

**Response:** `text/event-stream` — see Section 6 for SSE protocol.

### Error Mapping — Chat
| Exception | HTTP | When |
|---|---|---|
| `AgentNotFoundError` | 404 | Agent not found or inactive |
| `SessionArchivedError` | 400 | Session is archived |
| `InvalidMessageError` | 400 | Empty or invalid message |
| `LLMStreamError` | 500 | LLM provider error during streaming |
| `ChatServiceError` | 400 | Generic chat business logic error |
| `SessionNotFoundError` | 404 | Session ID not found in MongoDB |
| `SessionClearError` | 500 | Failed to delete session |

### POST `/ai/v1/chat/attachments` — Upload File Attachment

Uploads a file (image / document / audio / video) before referencing it in a chat message. Stores the file as a digital asset, writes per-attachment metadata to Redis, and returns the `asset_id` to include in the next `ChatMessageRequest.attachments[]`.

**Request:** `multipart/form-data`

| Field | Type | Required | Description |
|---|---|---|---|
| `session_id` | string (form) | yes | The chat session this attachment belongs to |
| `file` | file (multipart) | yes | The file bytes; `Content-Type` is used to detect MIME |

**Supported MIME types and size limits** (see `src/components/chat/media/constants.py`):

| Bucket | Examples | Max size |
|---|---|---|
| image | `image/jpeg`, `image/png`, `image/webp`, `image/gif` | 5 MB |
| audio | `audio/ogg`, `audio/mpeg`, `audio/mp4`, `audio/wav`, `audio/webm` | 16 MB |
| video | `video/mp4`, `video/3gpp`, `video/webm`, `video/quicktime` | 16 MB |
| document | `application/pdf`, `application/msword`, `.docx`, `text/plain`, `application/zip` | 100 MB |

**Response** (201) — `AttachmentUploadResponse`:
```json
{
  "status": "created",
  "message": "Attachment uploaded successfully.",
  "data": {
    "asset_id": 12345,
    "filename": "receipt.jpg",
    "media_type": "image",
    "mime_type": "image/jpeg",
    "url": "https://<bucket>/<key>?<presigned 24h>"
  }
}
```

### Error Mapping — Chat Attachments
| Exception | HTTP | When |
|---|---|---|
| `UnsupportedMimeTypeError` | 400 | MIME type not in `ALLOWED_MIME_TYPES` |
| `MimeMismatchError` | 400 | Declared `Content-Type` does not match magic-byte detected type |
| `FileTooLargeError` | 400 | File exceeds per-bucket size limit |
| `AttachmentNotFoundError` | (chat 500 internally) | Referenced `asset_id` has no Redis metadata (expired / never uploaded). The chat stream substitutes a `[Attachment '…' is no longer available.]` text block. |
| `WhatsAppMediaDownloadError` | (webhook 200) | WhatsApp Graph API download failed after retries; logged and dropped. |

### GET `/ai/v1/chat/session/{session_id}/history`
*Moved to `GET /ai/v1/sessions/{session_id}/history` in the Sessions module.*

### DELETE `/ai/v1/chat/session/{session_id}/clear`
*Moved to `DELETE /ai/v1/sessions/{session_id}/clear` in the Sessions module.*

---

## 3.5. Sessions API (`/ai/v1/sessions`)

### GET `/ai/v1/sessions/`
Retrieves a paginated list of chat sessions for the partner.

**Query Params:** `skip` (default 0), `limit` (default 20, max 100)

**Response** (200):
```json
{
  "status": "success",
  "data": {
    "sessions": [
      {
        "id": 1,
        "session_id": "sess_abc",
        "agent_id": 1,
        "status": "ACTIVE",
        "turns": 5,
        "updated_at": "2026-07-15T12:00:00Z"
      }
    ],
    "total": 1
  }
}
```

### GET `/ai/v1/sessions/{session_id}/history`
Fetches the detailed transcript history from MongoDB.

**Query Params:** `skip` (default 0), `limit` (default 20, max 100)

**Response** (200):
```json
{
  "status": "success",
  "message": "Session history retrieved successfully",
  "data": {
    "session_id": "sess_abc",
    "messages": [
      {"role": "user", "content": "Hello!", "medium": "text", "interface": "chat"},
      {"role": "assistant", "content": "Hi there!", "medium": "text", "interface": "chat"}
    ],
    "total": 2
  }
}
```

For a message with attachments, `content` is an array of blocks — see Section 8 (`chat_transcripts` shape).

### DELETE `/ai/v1/sessions/{session_id}/clear`
Returns 204 No Content. Also deletes any digital assets and Redis metadata/byte-cache keys for attachments uploaded in this session.

---

## 4. Tools API (`/ai/v1/tools`)

### GET `/ai/v1/tools` - List Global Tools
Lists all tools available to the partner, excluding integration-managed tools (`target_service == INTEGRATION`).

**Query Params:** `include_inactive=false`, `skip=0`, `limit=50`

**Response** (200):
```json
{
  "status": "success",
  "message": "Tools retrieved successfully",
  "data": {
    "tools": [ /* ToolRead objects */ ],
    "total": 5,
    "skip": 0,
    "limit": 50,
    "has_more": false
  }
}
```

### POST - Create Tool

**Request Body** (`ToolCreate`):
```json
{
  "tool_name": "check_availability",           // required, regex: ^[a-zA-Z0-9_]{1,100}$
  "display_name": "Check Availability",        // optional
  "tool_description": "Checks if a time slot is available", // required, min 10 chars
  "tool_type": "LOOKUP",                       // ACTION|LOOKUP|TRANSFER|HANDOFF|TRIGGER|CUSTOM
  "delegation_mode": "SILENT",                 // SILENT|TRANSFER|INJECT|HANDOFF
  "target_agent_id": null,                     // required if delegation_mode=TRANSFER
  "handoff_message": null,                     // optional, farewell for HANDOFF
  "json_schema": {                             // required, must have type=object + properties
    "type": "object",
    "properties": {
      "date": {"type": "string", "description": "Date in YYYY-MM-DD"},
      "time": {"type": "string", "description": "Time in HH:MM"}
    },
    "required": ["date"]
  },
  "target_service": "MAGLO",                   // MAGLO|BABBLER|WEBHOOK|INTERNAL|INTEGRATION
  "api_endpoint": "/api/v1/availability",      // supports {param} templating
  "method": "POST",                            // GET|POST|PUT|PATCH|DELETE
  "auth_credential_id": 1,                     // optional, ID of associated tool credential
  "tool_config": {},                           // optional, arbitrary JSON config
  "tool_headers": {}                           // optional, custom HTTP headers
}
```

### Tool Execution Result (Internal)
```json
{
  "tool_call_id": "call_abc123",
  "tool_name": "check_availability",
  "delegation_mode": "SILENT",
  "success": true,
  "data": {"available": true, "slots": ["10:00", "14:00"]},
  "target_agent_id": null,
  "handoff_message": null,
  "error_message": null,
  "execution_time_ms": 245
}
```

### Error Mapping — Tools
| Exception | HTTP | When |
|---|---|---|
| `ToolNotFoundError` | 404 | Tool name/ID not found |
| `ToolAlreadyExistsError` | 400 | Duplicate tool name on agent |
| `ToolPublishError` | 400 | Missing description/schema/endpoint |

---

## 5. Voice API (`/ai/v1/voice`)

### POST `/ai/v1/voice/token` — Get LiveKit Token

**Request Body** (`TokenRequest`):
```json
{
  "agent_id": 1,
  "session_id": "voice_test"    // default: "voice_test"
}
```

**Response** (200):
```json
{
  "status": "success",
  "data": {
    "token": "eyJ...",              // LiveKit JWT (30 min expiry)
    "livekit_url": "wss://lk.makunai.com",
    "room_name": "makun_sess_abc_f3a1b2c3",
    "session_id": "sess_abc"
  }
}
```

### POST `/ai/v1/voice/sessions` — Create Voice Session

**Request Body** (`VoiceSessionRequest`):
```json
{
  "agent_id": 1,
  "partner_id": 42,          // overridden by auth middleware
  "session_id": "sess_xyz",  // optional, auto-generated if null
  "medium": "voice"
}
```

**Response** (201):
```json
{
  "status": "success",
  "data": {
    "session_id": "sess_xyz",
    "call_record_id": 7,
    "room_name": "makun_sess_xyz_a1b2c3d4",
    "room_url": "wss://lk.makunai.com",
    "caller_token": "eyJ...",
    "livekit_url": "wss://lk.makunai.com"
  }
}
```

### POST `/ai/v1/voice/sessions/{session_id}/end`

**Request Body** (`EndSessionRequest`):
```json
{
  "duration_sec": 180,       // optional, server calculates if null
  "status": "COMPLETED"      // "COMPLETED" | "FAILED"
}
```

### Error Mapping — Voice
| Exception | HTTP | When |
|---|---|---|
| `AgentNotReadyError` | 404 | Agent not published/active |
| `LiveKitConnectionError` | 500 | LiveKit server unreachable |
| `SessionNotFoundError` | 404 | Session ID not found |
| `VoiceException` | 400 | Generic voice config error |

### RoomMetadata (Internal — JSON in LiveKit room)
```json
{
  "session_id": "sess_xyz",
  "partner_id": 42,
  "agent_id": 1,
  "language": "en-US",
  "call_record_id": 7,
  "tts_provider": "cartesia",
  "voice_id": "aura-2-asteria-en",
  "system_prompt": "You are a helpful...",
  "greeting_message": "Hi! How can I help?"
}
```

---

## 6. SSE Wire Protocol (Chat Streaming)

**Content-Type:** `text/event-stream`
**Headers:** `Cache-Control: no-cache`, `X-Accel-Buffering: no`

### Event Format
```
data: {"type":"delta","content":"Hello"}\n\n
data: {"type":"delta","content":" there"}\n\n
data: {"type":"done","total_tokens":42,"finish_reason":"stop"}\n\n
data: [DONE]\n\n
```

### Event Types
```json
// Text chunk
{"type": "delta", "content": "Hello"}

// Tool call (LLM wants to execute a function)
{"type": "tool_call", "tool_name": "check_availability", "tool_call_id": "call_abc", "arguments": "{\"date\":\"2026-05-13\"}"}

// Tool result (Emitted when a tool has finished executing)
{"type": "tool_result", "tool_name": "check_availability", "tool_call_id": "call_abc", "result": {"available": true}}

// Agent transfer
{"type": "transfer", "from_agent_id": 1, "to_agent_id": 2, "to_agent_name": "Sales Agent", "first_message": "Hi, I can help with pricing!"}

// Human handoff
{"type": "handoff", "session_id": "sess_abc", "partner_id": 42, "agent_id": 1, "farewell": "Let me connect you with a human agent.", "context": "Customer wants refund"}

// Stream complete
{"type": "done", "total_tokens": 156, "finish_reason": "stop"}

// Error mid-stream
{"type": "error", "message": "Stream error"}
```

### Sentinel
Stream always ends with: `data: [DONE]\n\n`

### Frontend Implementation Pattern
```javascript
const eventSource = new EventSource('/ai/v1/chat/message', { method: 'POST', body: payload });
// OR use fetch + ReadableStream for POST:
const response = await fetch('/ai/v1/chat/message', { method: 'POST', body: JSON.stringify(payload) });
const reader = response.body.getReader();
// Read chunks, split on "\n\n", parse "data: " prefix, JSON.parse each event
```

---

## 7. Integrations API (`/ai/v1/integrations`)

### GET `/templates/{template_id}` — Get Integration Template Details
Returns the detailed configuration of an integration template.
**Response (`IntegrationTemplateDetailRead`):**
```json
{
  "id": 8,
  "provider": "GOOGLE_CALENDAR",
  "display_name": "Google Calendar",
  "category": "PRODUCTIVITY",
  "provider_prefix": "google_calendar",
  "adapter_key": "GOOGLE_CALENDAR",
  "credential_type": "OAUTH2",
  "scopes": ["https://www.googleapis.com/auth/calendar"],
  "oauth_config": {
    "client_id": "...",
    "auth_url": "..."
  },
  "is_active": true
}
```
*Note: `provider_prefix` may be `null` for legacy records before v14 migration.*

### POST `/discover-mcp` — Discover & Register MCP Provider
Performs Dynamic Client Registration (DCR) and discovers tool schemas for an MCP server.
**Request Body (`McpDynamicDiscoveryRequest`):**
```json
{
  "provider_name": "My Custom Internal Tool",
  "issuer_url": "https://auth.internal.corp",
  "mcp_base_url": "https://tools.internal.corp/mcp",
  "provider_prefix": "corp_tools",
  "scopes": ["read", "write"]
}
```

### POST `/templates/upsert` — Create or Update Template
**Request Body (`IntegrationTemplateUpsertRequest`):**
```json
{
  "provider": "CUSTOM_CRM",
  "provider_prefix": "custom_crm",
  "display_name": "Custom CRM",
  "category": "CRM",
  "credential_type": "API_KEY",
  "adapter_key": "HTTP_GENERIC",
  "is_active": true
}
```

### POST `/oauth/init` — Start OAuth Flow
**Request Body:**
```json
{ "provider": "GOOGLE_CALENDAR" }
```
**Response:**
```json
{
  "auth_url": "https://accounts.google.com/o/oauth2/v2/auth?...",
  "provider": "GOOGLE_CALENDAR",
  "state": "crypto-random-state-token"
}
```

### POST `/credentials/api-key` — Register API Key
```json
{ "provider": "CALENDLY", "api_key": "cal_xxxxx..." }
```

### POST `/credentials/razorpay`
```json
{ "key_id": "rzp_live_xxx", "key_secret": "secret_xxx" }
```

### POST `/{agent_id}/enable` — Enable Integration on Agent
```json
{ "provider": "GOOGLE_CALENDAR", "config": {"calendar_id": "primary"} }
```

### POST `/{agent_id}/sync` — Sync Tools from Provider
Triggers a manual sync of available tools from the integration (e.g., re-fetching MCP `tools/list`).
```json
{ "provider": "GOOGLE_CALENDAR" }
```

### GET `/{agent_id}/tools` — List Active Tools
Returns all tools currently registered for integrations on this agent.

### POST `/credentials/revoke` — Revoke Credential
Revokes an existing integration credential.
```json
{ "provider": "CALENDLY" }
```

### GET `/ai/v1/integrations/whatsapp/webhook` — Meta Webhook Verification

Called by Meta once when registering the webhook URL in the Meta for Developers dashboard.

**Query params:** `hub.mode=subscribe`, `hub.verify_token=<must match ENV.WHATSAPP_VERIFY_TOKEN>`, `hub.challenge=<random>`.

**Response:**
- `200 text/plain` with the `hub.challenge` value echoed back on success.
- `403` if `hub.verify_token` does not match `ENV.WHATSAPP_VERIFY_TOKEN`.

This endpoint is `include_in_schema=False` (not shown in Swagger).

### POST `/ai/v1/integrations/whatsapp/webhook` — Inbound WhatsApp Messages

Called by Meta whenever an end user sends a message to a connected WhatsApp number. Routes text and supported media into the chat pipeline.

**Headers:** `X-Hub-Signature-256: sha256=<hmac>` — HMAC-SHA256 of the raw request body, keyed by `ENV.WHATSAPP_APP_SECRET`. Requests with a missing or mismatched signature are silently dropped (response `200`, never `4xx` — Meta retries on non-200).

**Request Body:** Meta WhatsApp webhook envelope (`entry[].changes[].value.messages[]`).

**Handler behaviour:**
- `text` → forwarded as `ChatMessageRequest{message=<body>, interface=whatsapp, medium=text}`.
- `image` / `document` / `audio` / `video` → media downloaded via `ChatAttachmentService.upload_from_whatsapp(media_id, mime_type, …)` and attached to a `ChatMessageRequest` with the WhatsApp text (if any) as `message`.
- `sticker` / `reaction` / `unsupported` → logged and acknowledged with `200`; not sent to the LLM.
- Any other type → logged and acknowledged with `200`.

`session_id` is derived as `wa:{partner_id}:{wa_from_number}` so each user has one stable thread.

**Response:** Always `200`, regardless of internal outcome — Meta retries any non-200 response.

### Error Mapping — Integrations
| Exception | HTTP | When |
|---|---|---|
| `ProviderNotSupportedError` | 400 | Unknown provider key |
| `CredentialNotFoundError` | 404 | No active credential for provider |
| `IntegrationAlreadyEnabledError` | 409 | Integration already active on agent |
| `OAuthStateInvalidError` | 400 | State token expired or tampered |
| `OAuthTokenExchangeError` | 500 | Provider rejected code exchange |
| `IntegrationNotFoundError` | 404 | Integration not found |
| `WhatsAppMediaDownloadError` | (logged, webhook returns 200) | Graph API media download failed after retries |

---

## 8. Partner Config API (`/ai/v1/partner/config`)

Enables CRUD operations on a partner's operational constraints and overrides. Unprovisioned partners automatically fall back to system defaults.

### GET `/ai/v1/partner/config` — Get Partner Config
Retrieves the configuration for the authenticated partner. If the partner hasn't been provisioned, returns the default fallback configurations.

**Response** (200) — `PartnerConfigSuccessResponse`:
```json
{
  "status": "success",
  "message": "Partner config retrieved successfully",
  "data": {
    "id": 42,
    "partner_id": 2000,
    "chat_rate_per_min": 60,
    "daily_token_limit": 100000,
    "rate_limit_enabled": true,
    "idempotency_enabled": false,
    "meta_config": {
      "openai_api_key": "sk-proj-xxxxxxxxxxxxxxxxxxxxxxxx",
      "anthropic_api_key": "sk-ant-xxxxxxxxxxxxxxxxxxxxxxxx"
    },
    "use_custom_api_key": true,
    "created_by": 1,
    "updated_by": 1,
    "deleted_by": null,
    "description": null,
    "additional_info": null,
    "created_at": 1716700000000,
    "updated_at": 1716750000000,
    "deleted_at": null
  }
}
```

### POST `/ai/v1/partner/config` — Create Partner Config
Provisions a new configuration row for the authenticated partner.

**Request Body** (`PartnerConfigCreate`):
```json
{
  "chat_rate_per_min": 120,
  "daily_token_limit": 500000,
  "rate_limit_enabled": true,
  "idempotency_enabled": false,
  "meta_config": {
    "openai_api_key": "sk-proj-xxxxxxxxxxxxxxxxxxxxxxxx"
  },
  "use_custom_api_key": true
}
```

**Response** (200) — `PartnerConfigSuccessResponse`:
```json
{
  "status": "success",
  "message": "Partner config created successfully",
  "data": {
    "id": 43,
    "partner_id": 2000,
    "chat_rate_per_min": 120,
    "daily_token_limit": 500000,
    "rate_limit_enabled": true,
    "idempotency_enabled": false,
    "meta_config": {
      "openai_api_key": "sk-proj-xxxxxxxxxxxxxxxxxxxxxxxx"
    },
    "use_custom_api_key": true,
    "created_by": 1,
    "updated_by": 1,
    "deleted_by": null,
    "description": null,
    "additional_info": null,
    "created_at": 1716700000000,
    "updated_at": 1716700000000,
    "deleted_at": null
  }
}
```

### PATCH `/ai/v1/partner/config` — Update Partner Config
Partially updates an existing configuration row. Omitted fields retain their database values.

**Request Body** (`PartnerConfigUpdate`):
```json
{
  "chat_rate_per_min": 200,
  "meta_config": {
    "openai_api_key": "sk-proj-xxxxxxxxxxxxxxxxxxxxxxxx",
    "anthropic_api_key": "sk-ant-xxxxxxxxxxxxxxxxxxxxxxxx"
  },
  "use_custom_api_key": true
}
```
> Note: `meta_config` has full-replace semantics; you must send all keys you want to retain.

**Response** (200) — `PartnerConfigSuccessResponse` containing the updated record.

### Error Mapping — Partner Config
| Exception | HTTP | `error_code` | When |
|---|---|---|---|
| `PartnerConfigAlreadyExistsError` | 400 | `BAD_REQUEST` | Configuration row already provisioned for this partner ID |
| `PartnerConfigNotFoundError` | 404 | `NOT_FOUND` | Updating a configuration that hasn't been created yet |
| `ValidationError` | 400 | `VALIDATION_ERROR` | Request body validation failed (e.g. rate limit < 1) |

## 9. Prompt Builder API (`/ai/v1/prompts`)

### GET `/ai/v1/prompts/templates` — List Templates
Retrieves available industry-specific system prompt templates.

**Response** (200):
```json
{
  "status": "success",
  "data": [
    {
      "template_id": "ECOMMERCE",
      "name": "E-Commerce Assistant",
      "description": "Template for retail stores.",
      "schema_definition": {
        "type": "object",
        "properties": {
          "store_name": {"type": "string", "title": "Store Name"}
        }
      }
    }
  ]
}
```

### POST `/ai/v1/prompts/generate` — Generate Prompt
Generates a full system prompt from a template and user-provided details.

**Request Body** (`PromptGenerateRequest`):
```json
{
  "template_id": "ECOMMERCE",
  "business_details": {
    "store_name": "Acme Widgets",
    "tone": "friendly"
  }
}
```

**Response** (200):
```json
{
  "status": "success",
  "data": {
    "generated_prompt": "You are a helpful assistant for Acme Widgets..."
  }
}
```

---

## 10. Knowledge API (RAG / `/ai/v1/knowledge`)

### POST `/ai/v1/knowledge/upload` — Upload Document
Initiates the asynchronous document ingestion pipeline. 

**Request:** `multipart/form-data`
| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | yes | The document (PDF, DOCX, CSV, XLSX). Max 20MB. |

**Response** (201):
```json
{
  "status": "success",
  "message": "Document ingestion started.",
  "data": {
    "document_id": 1,
    "task_id": "uuid-1234"
  }
}
```

### GET `/ai/v1/knowledge/documents/{task_id}/progress` — Stream Upload Progress
Streams Server-Sent Events (SSE) detailing the Celery worker's progress.

**Response:** `text/event-stream`
```
data: {"progress": 10, "status": "PROCESSING"}

data: {"progress": 100, "status": "COMPLETED"}

```

### POST `/ai/v1/knowledge/documents/{task_id}/cancel` — Cancel Upload
Cancels an ongoing document ingestion task.

**Response** (200) `SuccessResponse`.

### GET `/ai/v1/knowledge/documents` — List Documents
**Query Params:** `search`, `agent_id`, `status`, `skip`, `limit`

**Response** (200) returns paginated list of documents with metadata and presigned digital asset URLs.

### POST `/ai/v1/knowledge/documents/{id}/retry` — Retry Upload
Retries processing a failed document using the already-uploaded digital asset.

**Response** (201):
```json
{
  "status": "success",
  "message": "Document ingestion retry started.",
  "data": {
    "document_id": 1,
    "task_id": "new-uuid-5678"
  }
}
```

---

## 10b. Skin Analysis API (`/ai/v1/skin-analysis`)

### GET `/ai/v1/skin-analysis/scans` — List Scans by Session
Returns all skin scan records associated with the given session ID, ordered newest first.

**Query Params:** `session_id` (required), `skip` (default 0), `limit` (default 50, max 100)

**Response** (200):
```json
{
  "status": "success",
  "message": "Skin scans retrieved successfully.",
  "data": {
    "scans": [
      {
        "scan_id": 13,
        "original_asset_id": 63,
        "annotated_asset_id": 64,
        "overall_score": 92.0,
        "engine": "analyze_skin",
        "model": "gpt-4o",
        "rag_documents_used": 0,
        "attribute_scores": [
          {"attribute": "acne", "score": 92.0, "band": "excellent"}
        ],
        "region_findings": [
          {
            "concern": "uneven_texture",
            "confidence": 0.7,
            "annotation_label": "Mild uneven texture",
            "bounding_box": {"x": 0.32, "y": 0.55, "width": 0.36, "height": 0.18}
          }
        ],
        "recommendations": [
          {
            "product_name": "Exfoliating Serum",
            "description": "AHAs/BHAs smooth texture...",
            "target_concern": "uneven_texture",
            "source": "general"
          }
        ],
        "improvement_trajectory": {
          "day_30": "Subtle improvements...",
          "day_60": "Minimised pores...",
          "day_90": "Noticeably smoother texture..."
        },
        "created_at": 1781850254214
      }
    ],
    "total": 1
  }
}
```

### GET `/ai/v1/skin-analysis/scans/{scan_id}` — Get Full Skin Scan Report
Returns the complete skin scan report matching the given `scan_id`.

**Response** (200):
```json
{
  "status": "success",
  "message": "Skin scan report retrieved successfully.",
  "data": {
    "scan_id": 13,
    "partner_id": 2000,
    "session_id": "wa:2000:9199999999",
    "agent_id": 1,
    "original_asset_id": 63,
    "annotated_asset_id": 64,
    "engine": "analyze_skin",
    "model": "gpt-4o",
    "overall_score": 92.0,
    "attribute_scores": [
      {"attribute": "acne", "score": 92.0, "band": "excellent"}
    ],
    "region_findings": [
      {
        "concern": "uneven_texture",
        "confidence": 0.7,
        "annotation_label": "Mild uneven texture",
        "bounding_box": {"x": 0.32, "y": 0.55, "width": 0.36, "height": 0.18}
      }
    ],
    "recommendations": [
      {
        "product_name": "Exfoliating Serum",
        "description": "AHAs/BHAs smooth texture...",
        "target_concern": "uneven_texture",
        "source": "general"
      }
    ],
    "improvement_trajectory": {
      "day_30": "Subtle improvements...",
      "day_60": "Minimised pores...",
      "day_90": "Noticeably smoother texture..."
    },
    "routine_steps": [
      "Cleanse twice daily...",
      "Exfoliate 2-3 times per week..."
    ],
    "example_image_prompts": [
      "A young man with clear skin..."
    ],
    "rag_documents_used": 0,
    "created_at": 1781850254214
  }
}
```

---

---

## 11. MongoDB Document Structures

### `chat_transcripts` Collection

**Document Shape:**
```json
{
  "_id": "ObjectId(...)",
  "session_id": "sess_abc",
  "partner_id": 42,
  "agent_id": 1,
  "interface": "chat",
  "medium": "text",
  "messages": [
    {
      "role": "user",
      "content": "Hello!",
      "tool_calls": null,
      "tool_call_id": null,
      "name": null,
      "created_at": "2026-05-13T00:00:00Z",
      "interface": "chat",
      "medium": "text"
    },
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "What is in this image?"},
        {
          "type": "attachment_ref",
          "asset_id": 12345,
          "filename": "receipt.jpg",
          "media_type": "image",
          "mime_type": "image/jpeg"
        }
      ],
      "created_at": "2026-05-13T00:00:01Z",
      "interface": "chat",
      "medium": "text"
    },
    {
      "role": "assistant",
      "content": "It looks like a coffee shop receipt for $4.50.",
      "tool_calls": null,
      "created_at": "2026-05-13T00:00:02Z",
      "interface": "chat",
      "medium": "text"
    },
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [
        {
          "id": "call_abc123",
          "type": "function",
          "function": {
            "name": "check_availability",
            "arguments": "{\"date\": \"2026-05-14\"}"
          }
        }
      ],
      "created_at": "2026-05-13T00:00:03Z",
      "interface": "chat",
      "medium": "text"
    },
    {
      "role": "tool",
      "content": "{\"available\": true}",
      "tool_call_id": "call_abc123",
      "name": "check_availability",
      "created_at": "2026-05-13T00:00:04Z",
      "interface": "chat",
      "medium": "text"
    }
  ],
  "created_at": "2026-05-13T00:00:00Z",
  "updated_at": "2026-05-13T00:00:04Z"
}
```

### Indexes
- **Unique:** `(session_id, partner_id)`
- Used for sliding window queries with `$slice: -N`

### Multimodal `content` Shape

`content` may be either a plain string (text-only messages) or a **list of content blocks** (when the user sent attachments). Each block has a `type` field:

| Block `type` | Purpose | Other fields |
|---|---|---|
| `text` | Plain text segment | `text: string` |
| `attachment_ref` | Lightweight reference to a stored attachment. Base64 image/PDF data is **never** persisted — it is rehydrated from the digital asset on the next LLM call via `LLMContentBlockBuilder._rehydrate_history_message()`. | `asset_id: int`, `filename: string`, `media_type: string`, `mime_type: string` |

### Message `interface` and `medium` Values

| Field | Enum source | Values |
|---|---|---|
| `interface` | `Interface` in `src/components/common/channel.py` | `"chat"`, `"whatsapp"`, `"pstn"`, `"telegram"`, `"email"` |
| `medium` | `Medium` in `src/components/common/channel.py` | `"text"`, `"voice"` |

Both fields are stored on the transcript document (the dominant interface/medium for the session) **and** repeated on every message so per-message routing is preserved. The authoritative `interface → {medium}` mapping lives in `INTERFACE_SUPPORTED_MEDIUMS` (same file). All session messages share one transcript document, enabling unified history across chat and voice turns.

### Sliding Window Query
When building LLM context, only the last N messages are fetched:
```python
# MongoDB projection
{"messages": {"$slice": -max_messages}}
```
The `created_at` and `None` fields are stripped before sending to the LLM.

---

## 11. Key Method Signatures (Critical Path)

### ChatService
```python
async def stream_chat_message(
    self, request: ChatMessageRequest
) -> AsyncGenerator[ChatStreamEvent, None]
```

### AgentService
```python
async def load_agent_config(partner_id: int, agent_id: int) -> AgentConfig
async def build_llm_request(config: AgentConfig, session_id: str, message: str) -> LLMRequest
async def build_llm_request_with_inject(config: AgentConfig, session_id: str, inject_data: str) -> LLMRequest
```

### LLMCore
```python
async def generate_response_stream(
    self, request: LLMRequest, voice_mode: bool = False,
    local_history: list[dict] | None = None, bypass_db: bool = False
) -> AsyncGenerator[LLMStreamEvent, None]
# LLMStreamEvent = TextChunkEvent | ToolCallEvent | EndOfTurnEvent

# Private helpers added for media support:
async def _build_user_turn(self, request: LLMRequest) -> tuple[ContentValue, ContentValue]
# Returns (user_content_for_llm, history_content_for_storage). The history
# value swaps base64 image/PDF blocks for lightweight attachment_ref blocks
# via LLMContentBlockBuilder.to_history_ref().

async def _rehydrate_history_message(
    self, message: dict, partner_id: int, provider: ProviderType, model: str
) -> dict
# Walks attachment_ref blocks in a stored message and rebuilds them into
# full multimodal blocks (image_url / image base64 / document, …) just-in-time
# before passing history to the provider.
```

### LLMRequest (new field)
```python
class LLMRequest:
    ...
    new_message: str = ""                              # now optional; allows media-only turns
    attachments: list[ChatAttachment] | None = None    # pre-uploaded refs
```

### ChatAttachmentService (`src/components/chat/media/services.py`)
```python
async def upload_from_api(
    self, file: UploadFile, partner_id: int, session_id: str, user_id: int = 0
) -> AttachmentUploadResponse

async def upload_from_file(
    self, file: UploadFile, partner_id: int, session_id: str, user_id: int = 0
) -> AttachmentUploadResponse

async def upload_from_whatsapp(
    self, media_id: str, mime_type: str, partner_id: int,
    session_id: str, wa_token: str | None = None,
) -> AttachmentUploadResponse
# Raises WhatsAppMediaDownloadError after WA_MAX_RETRIES (3) attempts.

async def resolve_attachment(
    self, asset_id: int, partner_id: int
) -> tuple[bytes, str, str, str]  # (file_bytes, media_type, mime_type, filename)
# Raises AttachmentNotFoundError if metadata missing,
# PermissionError if partner_id does not own the attachment.

async def delete_session_attachments(
    self, session_id: str, asset_ids: list[int], partner_id: int
) -> None
```

### LLMContentBlockBuilder (`src/components/chat/media/content_blocks.py`)
```python
async def build_user_content(
    self, text: str, attachments: list[ChatAttachment], partner_id: int,
    provider: ProviderType, model: str = "", is_history: bool = False,
) -> str | list[dict]
# Returns plain text when no attachments. Otherwise builds provider-specific
# blocks: OpenAI `image_url` (detail=AUTO new turn / LOW for history) or
# Anthropic `image`/`document` base64. Non-vision OpenAI models receive a
# `[IMAGE ATTACHMENT: …]` text descriptor instead of an image_url block.

def to_history_ref(
    self, content: str | list[dict], attachments: list[ChatAttachment]
) -> str | list[dict]
# Replaces each non-text block in `content` with an `attachment_ref` block so
# base64 data is never written into MongoDB.
```

### ToolExecutor
```python
async def execute_tool(
    self, partner_id: int, agent_id: int,
    request: ToolCallRequest, session_id: str | None = None
) -> ToolExecutionResult
```

### IntegrationHandler
```python
async def execute(
    self, tool_name: str, args: dict,
    agent_id: int, partner_id: int, session_id: str
) -> dict
```

### MongoHistoryManager
```python
async def get_sliding_window(session_id: str, partner_id: int, max_messages: int = 15) -> list[dict]
async def append_messages(session_id: str, partner_id: int, agent_id: int, new_messages: list[ChatMessageModel]) -> None
async def clear_session(session_id: str, partner_id: int) -> None
```

---

## 11. partner_id Threading (Auth → Data)

This is how `partner_id` flows from auth to every query:

```
1. Request arrives with "Authorization: Bearer <token>" header

2. AuthMiddleware.dispatch()
   ├── Validates token via Redis cache or gRPC to Console Service
   └── Sets: request.state.user = {"partner_id": 42, "user_id": 7, ...}

3. Controller
   ├── partner_id = request.state.user["partner_id"]
   └── Passes partner_id to service method

4. Service
   └── Passes partner_id to repository method

5. Repository
   └── Adds WHERE partner_id = :partner_id to every SQL query
   └── Or filters MongoDB by {"partner_id": partner_id}

Result: Partner 42 can never read or modify Partner 43's data.
```

**API Key auth** follows the same pattern but sets:
```python
request.state.user = {
    "partner_id": payload["partner_id"],
    "api_key_id": payload["id"],
    "is_api_key_auth": True
}
```
