# MakunAI — Component Deep Dive

> Companion to `01_SYSTEM_OVERVIEW.md`. Covers each module's internals.

---

## 1. Agents Module (`src/components/agents/`)

### Purpose
CRUD management of AI agent configurations. An agent is a reusable config bundle — model, prompt, tools, medium — that powers both text chat and voice.

### Layer Pattern
Every component follows: **Controller → Service → Repository → Model**

```
AgentController (/ai/v1/agents)
    ↓
AgentService (business logic, validation)
    ↓
AgentRepository (SQLAlchemy queries)
    ↓
Agent (ORM model → ai_agents table)
```

### Agent Lifecycle
```
CREATE (draft) → UPDATE config → PUBLISH → ACTIVE
                                    ↓
                              DEACTIVATE (soft-delete)
```

- Agents start as `is_active=False`, `is_published=False`
- Publishing requires a non-empty `system_prompt`
- Deactivation is soft — row stays for audit

### Key Schema: `AgentConfig`
This is the **runtime object** that flows through the system during chat/voice:
```
DB Row → AgentRepository → AgentConfig → AgentService.build_llm_request() → LLMRequest
```

It carries: `partner_id`, `agent_id`, `provider`, `model`, `temperature`, `max_tokens`, `system_prompt`, `tools[]`, `context_text`, `medium`, `max_tool_calls_per_turn`, `sliding_window`, `agent_config`.

> **Schema rename (PR #21):** The `ai_agents.channel` column was renamed to `medium` (migration `b798831ec8fa_v3.py`). The `Medium` enum (`text` / `voice`) and a parallel `Interface` enum (`chat` / `whatsapp` / `pstn` / `telegram` / `email`) now live in [src/components/common/channel.py](../../src/components/common/channel.py). `INTERFACE_SUPPORTED_MEDIUMS` in the same file is the authoritative map of which mediums are valid per interface.

### API Endpoints
| Method | Path | Action |
|---|---|---|
| GET | `/ai/v1/agents` | List agents (paginated) |
| POST | `/ai/v1/agents` | Create draft agent |
| GET | `/ai/v1/agents/{id}` | Get agent config |
| PUT | `/ai/v1/agents/{id}` | Update agent |
| DELETE | `/ai/v1/agents/{id}` | Soft-delete (deactivate) |
| POST | `/ai/v1/agents/{id}/publish` | Publish to production |

---

## 2. Chat Module (`src/components/chat/`)

### Purpose
Streaming text chat with SSE (Server-Sent Events). Handles the full user turn loop: load agent → build LLM request → stream response → handle tool calls → loop.

### SSE Stream Events
| Event Type | Fields | When |
|---|---|---|
| `delta` | `content` | Each text chunk from LLM |
| `tool_call` | `tool_name`, `tool_call_id`, `arguments` | LLM requests a tool |
| `tool_result` | `tool_name`, `tool_call_id`, `result` | Emitted when a tool has finished executing (specifically used for streaming skin scan report card) |
| `transfer` | `from_agent_id`, `to_agent_id`, `to_agent_name` | Agent-to-agent transfer |
| `handoff` | `session_id`, `farewell`, `context` | Escalation to human |
| `done` | `total_tokens`, `finish_reason` | Stream complete |
| `error` | `message` | Any failure |

### Chat Turn Flow (Critical Path)
```
1. Controller receives POST /ai/v1/chat/message
     ├── Extract partner_id from auth
     └── Return SSE StreamingResponse

2. ChatService.stream_chat_message(req)
     ├── Load AgentConfig from DB
     ├── Build LLMRequest (system prompt + context + attachments[])
     └── Enter TOOL LOOP:

3. TOOL LOOP (max N iterations):
     ├── LLMCore.generate_response_stream():
     │     ├── Fetch sliding-window history
     │     ├── REHYDRATE: walk stored attachment_ref blocks →
     │     │             LLMContentBlockBuilder rebuilds full multimodal
     │     │             blocks from the digital asset bytes
     │     ├── BUILD USER TURN: combine request.new_message +
     │     │                    request.attachments[] → provider-specific
     │     │                    content blocks; store the history-safe
     │     │                    version (attachment_ref) for the DB write
     │     └── Stream from provider
     ├── TextChunkEvent → yield delta to client
     ├── ToolCallEvent → execute tool
     │   ├── SILENT mode  → append result as tool message, rebuild request, continue loop
     │   ├── INJECT mode  → inject result into system prompt, continue loop
     │   ├── TRANSFER     → load new agent, yield transfer event, stream from new agent
     │   └── HANDOFF      → yield farewell + handoff event, END stream
     ├── EndOfTurnEvent → yield done, EXIT loop
     └── If no tool call in iteration → EXIT loop
```

### Session History (MongoDB)
- Each session = one document in `chat_transcripts`
- Messages are appended with `$push` + upsert
- Sliding window (`$slice: -N`) used for LLM context, with pair-aware truncation in Python to strip orphaned `role: tool` messages and prevent 400 errors.
- Indexed on `(session_id, partner_id)` unique
- Multimodal `content`: a list of blocks (`text` / `attachment_ref`). Base64 image and PDF data are **never** written here — `LLMContentBlockBuilder.to_history_ref()` replaces them with `attachment_ref` blocks before the DB write, and `LLMCore._rehydrate_history_message()` rebuilds the full blocks on the next LLM call.
- Each message carries `interface` and `medium` fields (see `common/channel.py`).

### Media Pipeline (`src/components/chat/media/`)

New sub-module added in PR #21. Lets a chat message carry image / document / audio / video attachments, sourced from either the direct upload API or the WhatsApp webhook.

#### Layers
```
chat/media/
├── controllers.py     # POST /ai/v1/chat/attachments (multipart)
├── services.py        # ChatAttachmentService — upload, resolve, delete
├── content_blocks.py  # LLMContentBlockBuilder — provider block translation
├── preprocessor.py    # preprocess_image() — EXIF, RGB, resize, JPEG compress
├── helpers.py         # validate_mime_and_size(), magic-byte detection
├── schemas.py         # ChatAttachment, AttachmentMetadata, AttachmentUploadResponse
├── constants.py       # ALLOWED_MIME_TYPES, MAX_FILE_SIZES, ATTACHMENT_REDIS_KEY, WA_GRAPH_API_BASE
└── errors.py          # AttachmentNotFoundError, UnsupportedMimeTypeError,
                       # FileTooLargeError, MimeMismatchError, WhatsAppMediaDownloadError
```

#### Upload → Resolve → Use Flow
```
1. UPLOAD (direct API or WhatsApp webhook)
     ├── validate_mime_and_size()  # ALLOWED_MIME_TYPES + MAX_FILE_SIZES, magic-byte sniff
     ├── DigitalAssetService.create_digital_asset()  # stores file in S3-compatible bucket
     └── Redis SET chat_attachment:{asset_id} → AttachmentMetadata JSON (TTL 7d)
     → returns AttachmentUploadResponse{asset_id, filename, media_type, mime_type, url(24h)}

2. CHAT TURN INCLUDES asset_id
     Client → POST /ai/v1/chat/message  with attachments=[{asset_id, …}]

3. RESOLVE (inside LLMContentBlockBuilder, called by LLMCore)
     ├── ChatAttachmentService.resolve_attachment(asset_id, partner_id)
     │     ├── Read AttachmentMetadata from Redis (raise AttachmentNotFoundError if missing)
     │     ├── Enforce partner ownership (raise PermissionError on mismatch)
     │     ├── Try byte-cache: chat_attachment_bytes:{asset_id} (TTL 30m)
     │     └── On miss: presigned download → preprocess_image() (if image) → cache bytes
     └── Build provider blocks (see "Multimodal Support" in §3 LLM Module)

4. CLEAR SESSION (DELETE /ai/v1/chat/session/{id}/clear)
     └── delete_session_attachments(): per asset_id → delete digital asset
         + Redis metadata key + Redis byte-cache key
```

#### Image Preprocessing (`preprocessor.preprocess_image`)
- EXIF auto-rotation, RGB conversion.
- Resize so the longest side ≤ **1568 px** (tightest of OpenAI 2048 / Anthropic 1568 to stay safe across both providers).
- Iterative JPEG re-encode at quality **85 → 70** (floor at Q70 to keep alphanumeric text legible) targeting **~1.5 MB**.
- Output `ProcessedImage{data, mime_type, original_size, processed_size, width, height, was_resized}`. Skipped when the image already fits both dimension and size thresholds.

#### Error Classes
| Error | Raised when |
|---|---|
| `AttachmentNotFoundError` | Redis metadata key missing for the requested `asset_id` (expired or never uploaded). |
| `UnsupportedMimeTypeError` | MIME type not in `ALLOWED_MIME_TYPES`. |
| `FileTooLargeError` | File exceeds the per-bucket `MAX_FILE_SIZES` limit. |
| `MimeMismatchError` | Declared `Content-Type` does not match the magic-byte detected type. |
| `WhatsAppMediaDownloadError` | Graph API media download failed after `WA_MAX_RETRIES` (3) attempts with exponential backoff. |

### API Endpoints
| Method | Path | Action |
|---|---|---|
| POST | `/ai/v1/chat/message` | Stream chat (SSE) |
| POST | `/ai/v1/chat/attachments` | Upload a file attachment (multipart) |
| GET | `/ai/v1/chat/session/{id}/history` | Get session messages |
| DELETE | `/ai/v1/chat/session/{id}/clear` | Clear session history (also deletes its attachments) |
| GET | `/ai/v1/chat/sessions` | List partner sessions |

---

## 3. LLM Module (`src/components/llm/`)

### Purpose
Provider-agnostic LLM abstraction. Handles streaming, history management, tool call assembly, and context overflow protection.

### Architecture
```
LLMCore (orchestrator)
├── ContextOrchestrator (parallel history, RAG, compression)
├── LLMContentBlockBuilder (multimodal attachment rehydration)
├── ToolOutputCompactor (payload truncation)
├── LLMProviderFactory (registry + cache)
│   ├── OpenAIProvider (BaseLLMProvider)
│   └── AnthropicProvider (BaseLLMProvider)
└── Context overflow guard
```

### `LLMCore.generate_response_stream()` Flow
```
1. Orchestrate context via ContextOrchestrator (history + RAG + summary_blocks).
2. Build message array: [system_prompt, ...resolved_history, user_message].
3. Guard context overflow (trim oldest non-system messages if still too large).
4. Fire user message DB write as background task (asyncio.create_task)
5. Get provider from factory → stream response
6. Accumulate text chunks OR tool call deltas
7. On stream end:
   ├── If tool calls → save assistant tool_calls message → yield ToolCallEvents
   └── If text → save assistant reply → yield EndOfTurnEvent
```

### Provider Factory (`LLMProviderFactory`)
- **Registry pattern** — providers register at import time
- **Two-level cache** — in-process dict + Redis presence flag
- **Auto-detection** — `get_provider_for_model("gpt-4o")` → OpenAI
- **Pluggable** — call `LLMProviderFactory.register(type, class)` to add providers

### `BaseLLMProvider` Interface
Every provider must implement:
```python
async def stream(messages, model, temperature, max_tokens, tools) -> AsyncIterator[StreamDelta]
```

`StreamDelta` types: `text`, `tool_call_delta`, `usage`, `stop`

### Context Overflow Guard
- Looks up model context limit from `MODEL_CTX_LIMITS`
- Calculates character budget = `limit × 0.85 × 4 chars/token`
- If over budget: drops oldest non-system messages
- If system prompt alone exceeds budget: raises `ContextOverflowError`

### Error Handling in `generate_response_stream()`

The provider fetch and stream loop both run inside the same `try` block so all failures are caught cleanly:

| Exception | Cause | User sees | Logged at |
|---|---|---|---|
| `MissingPartnerApiKeyError` | `use_custom_api_key=True` but no key in `meta_config` | Generic `"I'm having trouble..."` message | `ERROR` with partner ID, provider, and fix hint |
| `RateLimitError` | Provider 429 | Rate-limit fallback message | `WARNING` |
| `ProviderUnavailableError` | Provider timeout / network | Unavailable fallback message | `ERROR` |
| `LLMBaseError` (catch-all) | Any other LLM failure | Generic fallback message | `ERROR` |

> **Design decision:** `MissingPartnerApiKeyError` details are intentionally kept out of the SSE stream. The end user sees the same generic message they'd see for any transient error. Operators diagnose via logs.

### Multimodal Support (added in PR #21)

The LLM module now handles user turns that carry attachments. `LLMRequest` gains an `attachments: list[ChatAttachment] | None` field, and `new_message` is optional (defaults to `""`) so a turn can be media-only.

#### Content Block Builder (`chat/media/content_blocks.py`)

`LLMContentBlockBuilder.build_user_content(text, attachments, partner_id, provider, model, is_history)` translates each `ChatAttachment` reference into provider-specific blocks:

| Provider | Image block | Document block | Audio/Video |
|---|---|---|---|
| **OpenAI** | `image_url` with `data:{mime};base64,…`, `detail=AUTO` (or `LOW` when replaying history). For non-vision models (not in `VISION_CAPABLE_MODELS`) returns a `[IMAGE ATTACHMENT: …]` text descriptor instead. | PDFs/.txt extracted to a `text` block (PDFs via `pypdf` if installed). | `[AUDIO/VIDEO ATTACHMENT: filename]` text descriptor (transcription is handled separately). |
| **Anthropic** | Native `image` block with `source.type=base64`, only for supported MIMEs (`image/jpeg`, `png`, `webp`, `gif`). | Native `document` block for `application/pdf`; other docs extracted to text. | Same text-descriptor fallback as OpenAI. |

`to_history_ref(content, attachments)` walks the produced block list and replaces every non-text block with a small `attachment_ref` block (`{type, asset_id, filename, media_type, mime_type}`) so MongoDB never stores base64 payloads.

#### History Rehydration

In `LLMCore.generate_response_stream()`:
- `_rehydrate_history_message(message, partner_id, provider, model)` — for every historical message containing `attachment_ref` blocks, fetches the bytes via `ChatAttachmentService.resolve_attachment()` and rebuilds the full multimodal blocks (with `is_history=True`, so OpenAI images use `detail=LOW` to keep older turns cheap).
- `_build_user_turn(request)` — assembles the current turn's `(content_for_llm, content_for_db)` tuple from `request.new_message` + `request.attachments[]`.

#### Image Token Estimation (`llm/image_tokens.py`)

For pre-flight cost / context budgeting:

- `estimate_openai_image_tokens(image_bytes, detail=AUTO)` — implements OpenAI's tile formula:
  - `LOW`: flat **85** tokens.
  - `HIGH`: scale longest side to ≤ 2048 px → scale shortest side up to ≥ 768 px → count 512 × 512 tiles → `tokens = tiles × 170 + 85`.
  - `AUTO`: resolves to `HIGH` when either dimension > 512 px, else `LOW`.
- `estimate_anthropic_image_tokens(image_bytes)` — `tokens = (width × height) ÷ 750`.
- Both return `ImageTokenEstimate{tokens, detail, tiles, width, height}` and fall back to a `len(bytes) / 750` byte-density approximation when PIL is not installed.

The `ImageDetail` enum (`LOW`, `HIGH`, `AUTO`) and the matching constants live in `llm/constants.py`.

#### Extracted Static Helpers (`llm/static_methods.py`)

Pure functions extracted from `LLMCore` for testability:
- `estimate_content_tokens()` / `estimate_message_tokens()` — token estimation including images.
- `to_llm_message()` — strips persistence-only fields (`created_at`, `interface`, `medium`, `attachment_ref` payload) before handing a stored message to a provider.

---

## 4. Tools Module (`src/components/tools/`)

### Purpose
Tool definition management and runtime execution. When the LLM calls a function, this module resolves it, validates arguments, dispatches it, and returns the result.

### Tool Model Fields
| Field | Purpose |
|---|---|
| `tool_name` | Function name the LLM uses (e.g. `check_availability`) |
| `display_name` | Human-readable label shown in the UI |
| `tool_description` | Tells the LLM when to call this tool |
| `json_schema` | JSON Schema for input validation |
| `is_global` | Boolean flag indicating if the tool is globally available to all partners |
| `target_service` | Where to send the request: `MAGLO`, `BABBLER`, `WEBHOOK`, `INTERNAL`, `INTEGRATION` |
| `api_endpoint` | URL path (supports `{param}` templating) |
| `method` | HTTP method (GET/POST/PUT/PATCH/DELETE) |
| `tool_type` | Functional classification: `ACTION`, `LOOKUP`, `TRANSFER`, `HANDOFF`, `TRIGGER`, `CUSTOM` |
| `tool_category` | Ownership category: `CUSTOM` (user-created) or `PARTNER` (integration-managed) |
| `tool_config` | Arbitrary JSON config specific to this tool instance |
| `tool_headers` | Custom HTTP headers (e.g. API keys) injected into outbound calls |
| `delegation_mode` | What to do with the result (see below) |
| `auth_credential_id` | ID of `PartnerToolCredential` used for authenticating outbound calls |
| `target_agent_id` | For TRANSFER mode — which agent to switch to |
| `handoff_message` | For HANDOFF mode — farewell message |

> **Note on Agent Assignment:** There is NO `agent_id` column on the `Tool` model. Tools are linked to agents via the `ai_agent_tools` many-to-many association table. This allows tools (especially integration tools) to be shared across multiple agents under the same partner without duplicating tool records.

> **Note on `_to_read_schema`:** All API responses for tools (GET, POST, PUT) pass through `ToolServiceStaticMethods._to_read_schema()` in `src/components/tools/static_methods.py`. Any new field added to the `Tool` ORM model **must** also be explicitly mapped here, otherwise it will always be returned as `null` regardless of what is stored in the database.

### Delegation Modes (Critical Concept)
| Mode | Behavior |
|---|---|
| **SILENT** | Run tool → append result as `role=tool` message → loop back to LLM |
| **INJECT** | Run tool → inject result into system prompt invisibly → re-call LLM |
| **TRANSFER** | Run tool → switch to a different agent → continue session with new agent |
| **HANDOFF** | Send farewell message → emit handoff event → end stream (escalate to human) |

### `ToolExecutor.execute_tool()` Flow
```
1. Look up assigned Tool row for agent_id by tool_name
2. Parse + validate arguments against json_schema
3. Route by delegation_mode:
   - HANDOFF -> return result immediately (no HTTP call)
   - INTEGRATION -> delegate to IntegrationHandler
   - HTTP tool -> call via pooled httpx client from ToolServiceClients
4. Log execution to tool_execution_logs (in finally block)
5. Return ToolExecutionResult
```

### ToolServiceClients
Pre-configured `httpx.AsyncClient` pools for each target service:
- `MAGLO` → `http://maglo-service`
- `BABBLER` → `http://babbler-service`
- `INTERNAL` → `http://makun-internal`
- `WEBHOOK` → no base URL (full URL per request)

### API Endpoints
| Method | Path | Action |
|---|---|---|
| GET | `/ai/v1/tools` | List global tools |
| GET | `/ai/v1/agents/{agent_id}/tools` | List agent's assigned tools |
| POST | `/ai/v1/tools` | Create global tool |
| PUT | `/ai/v1/tools/{tool_id}` | Update tool (supports clearing fields with `null` + includes `tool_name` collision check) |
| POST | `/ai/v1/tools/{tool_id}/publish` | Publish tool |
| DELETE | `/ai/v1/tools/{tool_id}` | Deactivate tool |
| PUT | `/ai/v1/agents/{agent_id}/tools` | Bulk assign tool IDs to an agent. This **replaces** the entire list of assigned tools for the agent (inserting new links and deleting omitted ones). |
| DELETE | `/ai/v1/agents/{agent_id}/tools/{tool_id}`| Removes a single tool assignment from the agent. |

### Tool Authentication Injection Flow

Outbound HTTP tool calls support secure, automatic authentication token injection via the `ToolAuthResolver` and `ToolExecutor`:

1. **Stored Credentials Mapping**: Tools map to encrypted `PartnerToolCredential` records via the `auth_credential_id` column.
2. **Credential Decryption**: At runtime, if `auth_credential_id` is set, `ToolAuthResolver` decrypts the credentials dynamically via `CredentialVault` using AES-256 (Fernet symmetric encryption).
3. **Caching**: Decrypted auth data (headers and query parameters) is cached in Redis (`tool_credential:{credential_id}`) with a configurable TTL to minimize vault decryption latency.
4. **Resolution Output (`headers, query_params`)**:
   - **`BEARER_TOKEN`**: Injected as an HTTP Authorization header (e.g. `Authorization: Bearer <token>`).
   - **`API_KEY`**: Injected as an arbitrary HTTP header using a custom configured header name (e.g., `X-Api-Key: <key>`).
   - **`BASIC_AUTH`**: Base64 encoded username/password pair injected as `Authorization: Basic <base64>`.
   - **`QUERY_PARAM`**: Injected as a URL query parameter using the custom parameter name configured in `header_name` (e.g. `?api_key=<key>`).
5. **401 Cache Invalidation & Retry**: If an outbound HTTP request returns a `401 Unauthorized` status code, `ToolExecutor` immediately invalidates the Redis cache for that credential, fetches a fresh token, and retries the HTTP request once.

---

## 5. Voice Module (`src/components/voice/`)

### Purpose
Real-time voice AI calls via LiveKit. Runs as a **separate worker process** that handles STT → LLM → TTS pipelines.

### Architecture
```
Browser (WebRTC)
    ↕ LiveKit Cloud
    ↕
makunai-livekit-worker (separate Docker container)
    └── VoiceSessionHandler
        ├── DTLN NoiseSuppression (ONNX, CPU)
        ├── Silero VAD (hardened, noisy mode)
        ├── Deepgram STT
        ├── LLMCoreAdapter → LLMCore (shared)
        └── TTS (Cartesia or Deepgram, configurable)
```

### Voice Call Lifecycle
```
1. Client: POST /ai/v1/voice/sessions
   └── VoiceService.initiate_call()
       ├── Validate agent is published
       ├── Create CallSession row in Postgres
       ├── Build RoomMetadata JSON
       ├── Create LiveKit room (with metadata)
       ├── Generate caller JWT token
       └── Return {room_name, token, livekit_url}

2. Client connects to LiveKit with token (WebRTC)

3. LiveKit dispatches job → worker entrypoint
   └── VoiceSessionHandler.setup()
       ├── Connect to room
       ├── Parse RoomMetadata from room.metadata
       ├── Load AgentConfig from DB
       ├── Build AgentSession (VAD + STT + LLM + TTS)
       └── Register event hooks
   └── VoiceSessionHandler.run()
       ├── Mark session IN_PROGRESS
       ├── Say greeting message
       └── Handle user turns (with filler detection + noise filtering)

4. On disconnect:
   └── Finalize session (duration, status)
   └── Delete LiveKit room
```

### Noise & Filler Handling
- **DTLN**: Removes background noise (fan, traffic) before VAD
- **Silero VAD**: Hardened settings to prevent false triggers
- **Hard skip phrases**: "okay", "yes", "got it" etc. → never sent to LLM
- **Short utterances**: If LLM is busy (lock held), fragments are dropped
- **Fillers**: For real messages, a random filler ("Let me check...", "One moment...") plays while LLM generates

### API Endpoints
| Method | Path | Action |
|---|---|---|
| POST | `/ai/v1/voice/token` | Get LiveKit JWT for browser |
| POST | `/ai/v1/voice/sessions` | Create voice session |
| POST | `/ai/v1/voice/sessions/{id}/end` | End voice session |
| GET | `/ai/v1/voice/health` | LiveKit health check |

---

## 6. Integrations Module (`src/components/integrations/`)

### Purpose
Connect third-party services (Google Calendar, Slack, Calendly, Razorpay) to agents. Handles OAuth2 flows, API key storage, credential encryption, and tool auto-registration.

### Supported Providers
| Provider | Auth Type | Adapter | Tools |
|---|---|---|---|
| Google Calendar | OAuth2 | `GoogleCalendarAdapter` | Calendar operations |
| Slack | OAuth2 | `SlackAdapter` | Message sending |
| Calendly | OAuth2 | `CalendlyAdapter` | Scheduling |
| Razorpay | API Key | `RazorpayAdapter` | Payment operations |

### Integration Flow
```
1. Connect Credential:
   ├── OAuth2: oauth_initialize() → redirect → oauth_callback() → encrypt & store
   └── API Key: register_api_key_credential() → encrypt & store

2. Enable on Agent:
   └── enable_provider_for_agent()
       ├── Validate no duplicate active integration (idempotent: returns existing if found)
       ├── Verify active credential exists
       ├── Create or fetch PartnerIntegration row
       └── Auto-register all provider tools (namespaced: "google_calendar__list_events")
           └── Uses `upsert_integration_tool` to either create the tool or update an existing one (by partner_id and tool_name), and then links it to the agent via `ai_agent_tools`.

3. Tool Execution at Runtime:
   └── ToolExecutor detects target_service=INTEGRATION
       └── IntegrationHandler.execute()
           ├── Resolve provider from tool name prefix
           ├── Load active integration + credential
           ├── Decrypt credential (auto-refresh if expired)
           └── Dispatch to provider adapter
```

### Credential Vault
- Uses `cryptography.Fernet` for symmetric encryption
- Stores: `encrypted_data`, `iv` (nonce), `key_version`
- Auto-refresh: checks expiry → re-exchanges OAuth refresh token if needed

### Integration Data Model
```
IntegrationTemplate (seed data — one per provider)
    - Stores dynamic configurations: `provider_prefix`, `adapter_key`, `oauth_config`, `client_config`, `base_config`
    ↓
PartnerCredential (encrypted tokens per partner)
    ↓
PartnerIntegration (enabled on specific agent, links template + credential)
    - Tracks operational state: `status`, `sync_status`, `health_status`, `last_sync_error`, `last_sync_at`
    - Tracks MCP configurations: `negotiated_protocol_version`, `resource_indicator`
    ↓
Tool (auto-registered, namespaced tools on the agent)
```
*Note: Database models are centralized in `src/components/integrations/model.py` (promoted from the old `models/` subdirectory) to simplify internal package imports.*

### WhatsApp Inbound Webhook (added in PR #21)

The integrations controller now exposes Meta's two-step webhook contract under `/ai/v1/integrations/whatsapp/webhook` and feeds inbound messages into the chat pipeline.

| Step | Method | What happens |
|---|---|---|
| Verify | `GET` | Meta sends `?hub.mode=subscribe&hub.verify_token=…&hub.challenge=…`. If `hub.verify_token` matches `ENV.WHATSAPP_VERIFY_TOKEN`, return `200 text/plain` echoing `hub.challenge`. Otherwise `403`. |
| Receive | `POST` | Validate `X-Hub-Signature-256` HMAC-SHA256 against `ENV.WHATSAPP_APP_SECRET` over the raw body. On mismatch, silently `200` (never `4xx`, or Meta will retry). |

Routing inside the POST handler:
- `text` → forwarded as a text-only chat turn.
- `image` / `document` / `audio` / `video` (`_WA_ATTACHMENT_TYPES`) → `ChatAttachmentService.upload_from_whatsapp(media_id, mime_type, partner_id, session_id, wa_token)` → resulting `ChatAttachment` appended to a chat turn.
- `sticker` / `reaction` / `unsupported` (`_WA_SILENT_TYPES`) → acknowledged `200`, not sent to the LLM.
- Unknown types → acknowledged `200`.

`session_id` is `wa:{partner_id}:{wa_from_number}` so each WhatsApp user has one stable thread per partner. The constructed `ChatMessageRequest` carries `interface=WHATSAPP, medium=TEXT`. Phone number → `(partner_id, agent_id, wa_token)` resolution lives in `_resolve_whatsapp_routing()` (currently a stub awaiting WABA-aware lookup).

### Model Context Protocol (MCP) Client Integration
The integrations module supports dynamic ingestion of third-party tools via the Model Context Protocol (MCP) using a standard JSON-RPC 2.0 interface.
- **Dynamic Client Registration (DCR):** Managed by `McpDcrService` to negotiate authentication handshakes and exchange scopes before acquiring a template.
- **MCP Adapter:** Implements `BaseIntegrationAdapter` via `McpAdapter`, converting native LLM tool execution schemas into standard MCP `tools/call` JSON-RPC payloads.
- **Error Mapping:** MCP protocol faults are robustly caught and raised as custom domain errors: `McpConnectionError` (handshake fails), `McpToolCallError` (tool execution fails), `McpToolDiscoveryError` (pagination/schema failure), and `McpDcrError` (registration fails).

---

## 7. Partner Configuration Module (`src/components/partner_config/`)

### Purpose
CRUD management of per-partner operational settings, limits, and integration overrides. Enables multi-tenant customization of rate-limiting gates, daily budgets, and custom provider API keys.

### Hot-Path Fallback (Fall-Open)
For unprovisioned partners, the module uses a fall-open mechanism (`_build_default_partner_config`) rather than throwing errors. System-wide default variables are read from environment configurations in `ENV`:
- **`chat_rate_per_min`** (default: 60)
- **`daily_token_limit`** (default: 100,000)
- **`rate_limit_enabled`** (default: `False` / disabled by default)
- **`idempotency_enabled`** (default: `False` / disabled by default)
- **`use_custom_api_key`** (default: `False` — always use global env keys)

### Redis Integration
When configuration values are read or updated, they are synced into Redis to support low-latency hot-path checks. All three keys are also warmed at service startup via `sync_all_configs_to_redis()`:
- Rate limits write to: `makun_ai:config:chat_rate_per_min:{partner_id}`
- Metadata config writes to: `makun_ai:config:meta_config:{partner_id}`
- API key routing flag writes to: `makun_ai:config:use_custom_api_key:{partner_id}` (`"true"` = True, `"false"` = False)

### Partner-Specific LLM Keys Routing

Partners can bring their own LLM credentials (e.g. `openai_api_key`, `anthropic_api_key`) by storing them in `meta_config` **and** enabling the `use_custom_api_key` flag. The flag is the authoritative switch — `meta_config` keys are ignored unless the flag is on.

#### Two-Step Redis Lookup (inside `LLMProviderFactory.get_provider`)
```
1. Read flag:  Redis.get("makun_ai:config:use_custom_api_key:{partner_id}")
     ├─ key missing (cold cache):
     │     └─ DB fallback → read PartnerConfig row
     │     └─ warm flag key AND meta_config key in Redis
     └─ key present:
           └─ "false" → use_custom = False
           └─ "true"  → use_custom = True

2. If use_custom = True:
     ├─ Read: Redis.get("makun_ai:config:meta_config:{partner_id}")
     ├─ Extract openai_api_key / anthropic_api_key from JSON
     └─ Key present → use partner key ✅
     └─ Key absent / empty → raise MissingPartnerApiKeyError 🛑 (no fallback)

   If use_custom = False:
     └─ custom_api_key = None → global OPENAI_API_KEY / ANTHROPIC_API_KEY used ✅
```

#### `MissingPartnerApiKeyError`
Raised strictly when `use_custom_api_key=True` but the partner has not yet set the corresponding key in `meta_config`. This is a **hard stop** — the system must never silently fall back to the global key and bill against it when the partner intended to use their own account. The error propagates out of the factory and is caught in `LLMCore.generate_response_stream()`, which:
- Logs the full detail (partner ID, provider, fix instruction) at `ERROR` level
- Yields the generic user-facing fallback (`"I'm having trouble responding right now."`) to the SSE stream — the end user sees no config detail

#### Cached Provider Instances
Provider instances are keyed by `(provider_type, custom_api_key)` in a local in-process dict, so:
- `(OPENAI, None)` → the shared global-key instance
- `(OPENAI, "sk-partner-xyz")` → a separate instance with isolated HTTP connection pool

Under `OpenAIProvider`, client connections are pooled in `_shared_clients` mapped by API key, preserving TCP/TLS socket reuse.

#### Partner Config `use_custom_api_key` REST API
| Field | Type | Default | Description |
|---|---|---|---|
| `use_custom_api_key` | `bool` | `False` | Toggle. When `True`, LLM calls route via keys in `meta_config`. When `False`, global env keys are always used. |

To enable: `PATCH /ai/v1/partner/config` `{ "use_custom_api_key": true, "meta_config": { "openai_api_key": "sk-..." } }`
To revert: `PATCH /ai/v1/partner/config` `{ "use_custom_api_key": false }`

---

## 8. Context Module (`src/components/context/`)

The Context Module isolates context window management from the core LLM execution layer. It orchestrates history fetching, Retrieval-Augmented Generation (RAG) fetching, structured session facts, conversational summarization (compression), tool output compaction, and lazy media loading to prevent context limit exhaustion.

### 8.1 Context Orchestrator
The `ContextOrchestrator` runs before every LLM API call. It executes context retrieval asynchronously and in parallel:
- Fetches the base conversational history from MongoDB (`MongoHistoryManager`).
- Fetches fast ephemeral state from the `RedisHotTier`.
- Retrieves structured memory facts using the `SessionMemoryManager`.
- Submits history to the `CompressionService` if the uncompressed turn count exceeds limits.
- Concurrently fetches relevant RAG documents if `rag_enabled=True`.
- Applies the `ToolOutputCompactor` to inline tool outputs.
- Returns a fully-resolved list of LLM-ready messages (with RAG docs and session facts injected as system instructions).

### 8.2 Session Memory Manager (Structured Facts)
The `SessionMemoryManager` implements a structured key-value store for session facts, primarily backed by Redis.
- **Conflict Policy:** Facts are categorized by `source` (`confirmed` or `inferred`). A Lua script ensures that `inferred` facts cannot overwrite `confirmed` facts unless explicitly forced.
- **Persistence:** By default, acts as an ephemeral Redis cache, with a fallback to MongoDB on cache misses. A write-behind Celery task (`persist_session_fact`) can optionally sync facts durably to MongoDB if enabled.
- **Eviction:** Bounded by `ENV.MAX_SESSION_FACTS`. Drops oldest inferred facts when the limit is reached.
- **Tools:** The native `remember_detail` tool allows the LLM to write facts dynamically, providing a mechanism for long-term state tracking beyond the sliding conversational window.

### 8.3 Compression Service
Maintains the `summary_blocks` MongoDB collection. When a conversation exceeds the `max_raw_turns` policy for a given medium, the `CompressionService` batches the oldest `N` turns, calls a cheaper, deterministic LLM (`gpt-4o-mini`, temperature `0.1`), and stores the resulting summary. The orchestrator rehydrates these summary blocks into the LLM context prompt to preserve long-term memory without eating tokens.
*(Note: PostgreSQL was initially considered for cold-tier archival of trimmed raw messages but was deprecated. MongoDB bounds now definitively control raw message retention.)*

### 8.4 Tool Output Compactor
Tool responses containing large payloads (e.g. database rows, long text responses) are automatically compacted:
- Arrays exceeding `COMPACTOR_MAX_ARRAY_ITEMS` are truncated.
- Strings exceeding `COMPACTOR_MAX_STRING_LENGTH` are truncated.
- Total JSON output size is capped at `COMPACTOR_MAX_OUTPUT_CHARS`.

### 8.5 Lazy Media Rehydration System (LMRS)
Historical media attachments are not blindly injected into the context window as costly base64 encodings or image URLs. Instead:
- Historical media is replaced by a lightweight `attachment_ref` descriptor (~25 tokens).
- A synthetic tool (`fetch_attachment`) is automatically injected into the LLM's available tools.
- If the LLM determines it needs to see a historical image to answer the user's question, it calls `fetch_attachment(attachment_id)`.
- The system intercepts this tool call and injects the actual heavy image block back into the current prompt for that specific turn only.

### 8.6 Medium Policies
Configured in `policies.py`, each interface medium dictates context rules:
- **TEXT**: Larger history windows, summarization enabled, deeper RAG chunks.
- **VOICE**: Tighter history bounds to minimize TTFB latency, fewer RAG chunks, stricter LLM output generation caps to prevent excessive monologue.

### 8.7 Strict Tool Call Visibility
To prevent raw JSON tool blocks from leaking to client interfaces (e.g., SSE streaming UI or Text-to-Speech engines):
- `LLMEngine.run_turn` strictly swallows all `ToolCallEvent` and `ToolResultEvent` chunks.
- Adapters (like `llm_adapter` and `llm_core_bridge`) no longer listen for tool events.
- To bypass TTFB caches during tool execution in voice without exposing JSON, `EndOfTurnData` carries a `had_tools: bool` flag.
- Voice UI timing gaps (e.g. silence during a `TRANSFER`) are bridged internally inside `LLMEngine` by yielding safe `TextChunkEvent(" Just a moment... ")` fillers, ensuring the core enforces the security boundary.

---

## 9. Prompt Builder Module (`src/components/prompt_builder/`)

### Purpose
An interactive LLM-assisted tool for generating robust, specialized system prompts based on industry templates. It helps partners quickly configure high-quality agents without writing system prompts from scratch.

### Flow
```
1. Get Templates:    Client calls GET /ai/v1/prompts/templates
2. Generate Prompt:  Client calls POST /ai/v1/prompts/generate with template_id and business_details
3. LLM Generation:   PromptBuilderService uses LLMAdapter to compile the meta-prompt and generate the prompt
4. Apply to Agent:   Client calls PUT /ai/v1/agents/{agent_id} with the new system_prompt
```

### Industry Templates
Templates are seeded via `seed_agent_templates()` during startup. They are stored in the `agent_prompt_templates` table.
Available templates include: `ECOMMERCE`, `HEALTHCARE`, `REAL_ESTATE`, `FINANCE`, `LEGAL`, `FOOD`, `HOSPITALITY`, `HR`, `SALES`, `SUPPORT`, `EDUCATION`, `SKIN_ANALYZER`, and a `GENERAL` fallback.

Each template contains:
- **`schema_definition`**: A JSON Schema describing the specific business details required from the user (e.g., `store_name`, `return_policy`).
- **`system_prompt_template`**: A Jinja-like template string defining the agent's core identity, guardrails, and tone, into which the LLM generation merges the user's business details.

### LLM Adapter & Meta-Prompts
The generation is handled by `LLMPromptBuilderAdapter`. It constructs a **meta-prompt** (a prompt that writes prompts) that instructs the LLM to assume the role of an expert prompt engineer. The LLM merges the `system_prompt_template` with the provided `business_details` to produce the final, ready-to-use system prompt for the new agent.

### API Endpoints
| Method | Path | Action |
|---|---|---|
| GET | `/ai/v1/prompts/templates` | List available industry templates |
| POST | `/ai/v1/prompts/generate` | Generate a system prompt from a template and business details |

---

## 10. Retrieval-Augmented Generation (RAG) Module (`src/components/rag/`)

### Purpose
Provides a robust, asynchronous, and scalable knowledge base system. It handles everything from multi-format document ingestion to semantic vector search with strict agent-level access controls.

### Architecture Layering
The module strictly follows the **Controller → Service → Repository** architectural pattern.
- **Controllers**: Thin HTTP layer responsible for request validation, routing, and returning formatted responses. No business logic.
- **Services**: Orchestrates external services (Digital Assets), caches (Redis), task queues (Celery), and database operations.
- **Repositories**: Abstracts away all SQLAlchemy Core operations, ensuring atomic transactions and optimized queries for vector handling.

### The Ingestion Pipeline
The ingestion pipeline is entirely asynchronous and non-blocking.

#### Upload Phase
1. Files (PDF, DOCX, XLSX, CSV) are validated in memory to enforce size limits (20MB) and allowed types.
2. A hash is generated to instantly detect duplicate documents.
3. The raw file is pushed to the Digital Asset Service (S3/DigitalOcean Spaces) and a `PENDING` record is created in the DB.
4. A Celery background task is dispatched using the digital asset's presigned URL.

#### Background Processing (`tasks.py`)
The Celery worker executes the pipeline without touching the local disk (stateless scaling):
1. **Downloading**: Fetches the file into memory using the presigned URL.
2. **Parsing (`ingestion/parsers`)**: Extracts text dynamically based on the file extension using specialized parsers (e.g., PyMuPDF, `csv` module).
3. **Chunking (`ingestion/chunkers`)**: Splits the text into semantic chunks (default 500 tokens with 50 token overlap).
4. **Embedding (`embeddings/`)**: Generates 1536-dimensional vectors using OpenAI `text-embedding-3-small`.
5. **Saving**: Atomically saves all chunks and metadata to the vector database.

> **Progress Tracking:** The Celery task updates a Redis key at each stage (`STARTING`, `PARSING`, `EMBEDDING`, etc.). The frontend subscribes to the `/progress` Server-Sent Events (SSE) endpoint to display a live loading bar.

### Management & Reliability
- **Cancellation**: Users can cancel an ongoing upload. The system revokes the Celery task, marks the database status as `FAILED`, and notifies the frontend via the SSE stream.
- **Retry Mechanism**: If a document fails processing, the user can retry. The system bypasses the upload phase, generates a fresh presigned URL from the existing digital asset, and re-dispatches the Celery task.
- **Agent Assignment**: Documents are uploaded at the partner level. They can optionally be mapped to specific AI Agents, ensuring agents only retrieve knowledge they are explicitly granted access to.

### Vector Retrieval (`retrieval/`)
When an agent needs to answer a question, the `KnowledgeBaseRetriever`:
1. Embeds the user's query.
2. Performs a similarity search (Cosine Similarity) against the vector database.
3. **Respects Boundaries**: Retrieves only universal chunks (unassigned) AND chunks explicitly mapped to the querying agent.
4. Returns the top `K` most relevant chunks with their source file names and page numbers for inline citation formatting.

### API Endpoints
| Method | Path | Action |
|---|---|---|
| POST | `/ai/v1/knowledge/upload` | Async document upload |
| GET | `/ai/v1/knowledge/documents/{task_id}/progress` | Stream upload progress (SSE) |
| POST | `/ai/v1/knowledge/documents/{task_id}/cancel` | Cancel document upload |
| GET | `/ai/v1/knowledge/documents` | List partner documents (paginated) |
| POST | `/ai/v1/knowledge/documents/{id}/assign` | Assign document to agent |
| POST | `/ai/v1/knowledge/documents/{id}/unassign` | Remove document from agent |
| POST | `/ai/v1/knowledge/documents/{id}/retry` | Retry failed upload |
| DELETE | `/ai/v1/knowledge/documents/{id}` | Soft delete document |
| POST | `/ai/v1/knowledge/query` | Query the knowledge base |

---

## 10. Skin Analysis Module (`src/components/skin_analysis/`)

### Purpose
Dermatology and skincare assistant backend support. It provides vision-based skin concern analysis, overlays bounding boxes on identified areas, queries the knowledge base for RAG recommendations, and estimates treatment trajectory outcomes over a 12-week timeline.

### Components & Internals
Every component follows the standard layer pattern: **Controller → Service → Repository → Model**.

```
SkinAnalysisController (/ai/v1/skin-analysis)
    ↓
SkinAnalysisService (business logic coordination)
    ├─ SkinEngineFactory → LLMVisionSkinEngine (Vision pipeline)
    ├─ ImageAnnotationRenderer (Draws Pillow bounding boxes & uploads)
    └─ KnowledgeBaseRetriever (RAG lookup)
        ↓
SkinScanRepository (SQLAlchemy queries)
        ↓
SkinScan (ORM model → makun_ai_skin_scans table)
```

#### 1. Analysis Pipeline (Vision)
- **Vision Engine**: The `LLMVisionSkinEngine` receives the `asset_id` representing the uploaded user image. It sends the image (base64) along with structured formatting instructions to the vision-capable LLM.
- **Concern Detection**: The LLM detects skin attributes (acne, redness, wrinkles, oiliness, pores, etc.) with confidence scores, qualitative bands, and bounding box coordinates `(x, y, width, height)`.

#### 2. RAG & Recommendations
- The `SkinAnalysisService` extracts detected concerns and calls the `KnowledgeBaseRetriever` to fetch relevant treatment guides.
- A secondary LLM call evaluates the RAG chunks and structures product/routine recommendations.

#### 3. Image Annotation Renderer
- The `ImageAnnotationRenderer` retrieves the original image bytes from `DigitalAssetService`.
- It overlays bounding boxes on the image using Python Pillow (`PIL`) with custom colors matching the severity or concern type.
- The annotated image is uploaded as a new digital asset, generating `annotated_asset_id` and `annotated_image_url`.

#### 4. Trajectory Projection Heuristic
- Projected trajectory math resides in `static_methods.py`. It computes numerical projected overall scores for weeks 4, 8, and 12 using a conservative improvement curve based on product usage.

### API Endpoints
| Method | Path | Action |
|---|---|---|
| GET | `/ai/v1/skin-analysis/scans` | List skin scans for a session (paginated) |
| GET | `/ai/v1/skin-analysis/scans/{scan_id}` | Retrieve a full skin scan report by ID |

---

## 11. Agent Orchestration Module (`src/components/orchestration/`)

### Purpose
The Orchestration module manages multi-agent delegation. It allows a "parent" agent to invoke a "child" sub-agent to fulfill a specific sub-task in an isolated ReAct loop, wait for the result, and continue its own execution. 

### Architecture
- **Route Definitions (`AgentRoute`)**: Configured parent-to-child relationships stored in `agent_routes`. A parent can only delegate to explicit, active child routes.
- **Dynamic Tool Generation**: The orchestrator dynamic tool builder maps active `AgentRoutes` into a single LLM schema tool (`delegate_to_agent`) injected dynamically into the parent agent's system prompt and tool definitions.
- **`SubAgentRunner`**: The core execution engine for running the sub-agent. It executes an isolated ReAct loop with a hard timeout, bypassing database history tracking in favor of ephemeral state to prevent context bleeding. 
- **Telemetry (`OrchestrationRepository`)**: Background Celery tasks (`write_invocation_log`) record invocation metrics (`agent_invocation_log` table) and raw findings (`agent_invocation_findings` table) for monitoring usage and failures.

### Multi-Tier Prevention
To prevent infinite recursive delegation (e.g., Parent -> Child A -> Child B -> Child A), the system enforces a strict 2-tier limit. The `SubAgentRunner` automatically filters out the `delegate_to_agent` tool from the child agent's available tools before execution.


