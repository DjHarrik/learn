# MakunAI — Data Flow, Patterns & Developer Guide

> Companion to `01_SYSTEM_OVERVIEW.md` and `02_COMPONENT_DEEP_DIVE.md`.

---

## 1. End-to-End Data Flow: Text Chat

This is the most important flow in the system. Every text interaction follows this exact path.

```
┌──────────┐     POST /ai/v1/chat/message        ┌──────────────┐
│  Client   │ ──────────────────────────────────► │ AuthMiddleware│
│ (Browser) │     {agent_id, session_id, message} │              │
└──────────┘                                      └──────┬───────┘
                                                         │ request.state.user = {partner_id}
                                                         ▼
                                                  ┌──────────────┐
                                                  │ChatController │
                                                  │ process_chat_ │
                                                  │ message()     │
                                                  └──────┬───────┘
                                                         │ payload.partner_id = partner_id
                                                         │ return SSE StreamingResponse
                                                         ▼
                                                  ┌──────────────┐
                                                  │ ChatService   │
                                                  │ stream_chat_  │
                                                  │ message()     │
                                                  └──────┬───────┘
                                                         │
                    ┌────────────────────────────────────┤
                    ▼                                    ▼
             ┌──────────────┐                    ┌──────────────┐
             │ AgentService  │                    │   LLMCore     │
             │ load_agent_   │                    │ generate_     │
             │ config()      │                    │ response_     │
             │ build_llm_    │                    │ stream()      │
             │ request()     │                    └──────┬───────┘
             └──────────────┘                           │
                                                        ▼
                    ┌───────────────────────────────────────────────┐
                    │                 TOOL LOOP                     │
                    │                                               │
                    │  TextChunk? → yield delta → SSE to client    │
                    │  ToolCall?  → ToolExecutor.execute_tool()    │
                    │              ├── SILENT → append result,     │
                    │              │    rebuild request, continue   │
                    │              ├── INJECT → embed in prompt,   │
                    │              │    continue                    │
                    │              ├── TRANSFER → switch agent     │
                    │              └── HANDOFF → farewell, end     │
                    │  EndOfTurn? → yield done → EXIT              │
                    └───────────────────────────────────────────────┘
```

### Data Objects Through the Pipeline
```
ChatMessageRequest          # Controller input (partner_id, agent_id, session_id, message,
                            #                   interface, medium, attachments[])
    ↓
AgentConfig                 # Loaded from DB — runtime config for this turn
    ↓
LLMRequest                  # Packaged for LLMCore (ModelConfig + session + message + attachments[])
    ↓
LLMCore.generate_response_stream():
    ├── ContextOrchestrator.build_context() ────┬───► (MongoDB sliding-window history fetch)
    │                                           ├───► (Vector DB RAG fetch if enabled)
    │                                           └───► (ToolOutputCompactor for truncation)
    ├── Rehydrate stored attachment_ref blocks ─► ChatAttachmentService (LMRS fetch if tool called)
    │      (via LLMContentBlockBuilder)           (Redis byte cache → S3 fallback)
    ├── _build_user_turn(): text + attachments ─┬───► provider content blocks (image_url / image / document)
    │                                           └───► history-safe blocks (attachment_ref) for DB write
    ↓
[StreamDelta]               # Raw chunks from provider (text/tool_call_delta/usage/stop)
    ↓
[LLMStreamEvent]            # Normalized events (TextChunkEvent/ToolCallEvent/EndOfTurnEvent)
    ↓
[ChatStreamEvent]           # Controller-level events (delta/tool_call/done/error/transfer/handoff)
    ↓
SSE text/event-stream       # Wire format: "data: {json}\n\n" ... "data: [DONE]\n\n"
```

### Attachment Upload Side-Channel

`attachments[]` items are not uploaded inline with the chat message. The client first runs:

```
1. POST /ai/v1/chat/attachments  (multipart, per file)
       form: session_id, file
   → ChatAttachmentService.upload_from_file()
       ├── validate_mime_and_size (ALLOWED_MIME_TYPES + MAX_FILE_SIZES + magic bytes)
       ├── DigitalAssetService.create_digital_asset() → S3-compatible bucket
       └── Redis SET chat_attachment:{asset_id} = AttachmentMetadata (TTL 7d)
   ← 201 { asset_id, filename, media_type, mime_type, url(24h presigned) }

2. POST /ai/v1/chat/message
       body: { agent_id, session_id, message, attachments: [{asset_id, …}] }
   → flows through the pipeline above, where _build_user_turn resolves bytes
     (30 min byte cache; on miss → presigned download + image preprocess).
```

---

## 1b. End-to-End Data Flow: WhatsApp Inbound

Meta calls our webhook for every inbound message on a connected WhatsApp Business number. Text and media both fan into the same `ChatService.stream_chat_message()` used by the direct API.

```
Meta WhatsApp Cloud API
    │
    │ POST /ai/v1/integrations/whatsapp/webhook
    │ X-Hub-Signature-256: sha256=<hmac(WHATSAPP_APP_SECRET, raw_body)>
    ▼
┌────────────────────────────────────────────────────────────┐
│ IntegrationController.whatsapp_webhook()                   │
│  ├── HMAC verify (mismatch → 200 silent drop)              │
│  ├── Parse entry[].changes[].value.messages[0]             │
│  ├── _resolve_whatsapp_routing(phone_number_id)            │
│  │     → (partner_id, agent_id, wa_token)                  │
│  ├── session_id = f"wa:{partner_id}:{wa_from}"             │
│  ├── if type ∈ {image, document, audio, video}:            │
│  │     ChatAttachmentService.upload_from_whatsapp(         │
│  │         media_id, mime_type, partner_id, session_id,    │
│  │         wa_token)                                       │
│  │     → Graph API 2-step download (retry x3, backoff)     │
│  │     → digital asset row + Redis metadata                │
│  │     → ChatAttachment ref appended to attachments[]      │
│  ├── if type ∈ {sticker, reaction, unsupported}:           │
│  │     log + return 200 (no LLM call)                      │
│  └── if text or attachments present:                       │
│        ChatService.stream_chat_message(                    │
│            ChatMessageRequest{                             │
│                interface=WHATSAPP, medium=TEXT,            │
│                message=<text>, attachments=[...]})         │
│        — stream is drained server-side (no SSE response)   │
└────────────────────────────────────────────────────────────┘
       │
       └─► (downstream identical to §1 Text Chat flow)
```

Response is **always** `200`, regardless of internal outcome, because Meta retries any non-200 response.

---

## 2. End-to-End Data Flow: Voice Call

```
┌──────────┐  POST /voice/sessions    ┌──────────────┐
│  Client   │ ──────────────────────► │VoiceController│
└─────┬────┘                          └──────┬───────┘
      │                                      │
      │                                      ▼
      │                               ┌──────────────┐
      │                               │ VoiceService  │
      │                               │ initiate_call │
      │                               └──────┬───────┘
      │                                      │
      │                    ┌─────────────────┼──────────────────┐
      │                    ▼                 ▼                  ▼
      │             Load Agent        Create CallSession   Create LiveKit
      │             Config            in Postgres          Room + Token
      │                                                         │
      │◄────────────── Return {token, livekit_url, room_name} ──┘
      │
      │  WebRTC connect with token
      ▼
┌──────────┐          ┌──────────────────────────┐
│ LiveKit   │ ──────► │ makunai-livekit-worker    │
│ Cloud     │  job    │                          │
└──────────┘         │ VoiceSessionHandler       │
                     │ ├── connect to room       │
                     │ ├── parse RoomMetadata    │
                     │ ├── load AgentConfig      │
                     │ ├── build pipeline:       │
                     │ │   DTLN → VAD → STT      │
                     │ │   → LLMCore → TTS       │
                     │ ├── say greeting          │
                     │ └── handle turns:         │
                     │     User speaks           │
                     │     → STT transcribes     │
                     │     → noise/filler filter │
                     │     → LLMCore generates   │
                     │     → TTS speaks back     │
                     └──────────────────────────┘
```

---

## 3. End-to-End Data Flow: Integration Tool

```
LLM calls tool: "google_calendar__list_events"
    ↓
ToolExecutor.execute_tool()
    │ target_service == "INTEGRATION"
    ▼
IntegrationHandler.execute()
    ├── Resolve provider: "google_calendar__list_events" → "GOOGLE_CALENDAR"
    ├── Load PartnerIntegration (is_active check)
    ├── Load PartnerCredential
    ├── Vault: decrypt + auto-refresh if expired
    └── GoogleCalendarAdapter.execute(tool_name, args, credentials, config)
        └── HTTP call to Google Calendar API
            └── Return result dict to LLM
```

```

---

## 3a. End-to-End Data Flow: MCP Client Discovery & Registration

```
Client calls POST /ai/v1/integrations/discover-mcp
    │ payload: { provider_name, issuer_url, mcp_base_url, ... }
    ▼
IntegrationController.discover_mcp_provider()
    │
    ▼
McpDcrService.discover_and_register()
    ├── 1. Fetch `.well-known/openid-configuration` (if DCR applicable)
    ├── 2. Perform Dynamic Client Registration (DCR) handshake via HTTP POST
    ├── 3. Save resulting `client_id` / `client_secret` in Vault (CredentialService)
    ├── 4. Open JSON-RPC 2.0 connection to `mcp_base_url` (McpAdapter)
    ├── 5. Call `tools/list` on the MCP server to fetch tool schemas
    └── 6. Save as a new `IntegrationTemplate` (IntegrationTemplateService)
```

---

## 3b. End-to-End Data Flow: Document Ingestion (RAG)

```
┌──────────┐  POST /knowledge/upload   ┌──────────────┐
│  Client   │ ──────────────────────► │RAGController  │
└─────┬────┘   {file, partner_id}     └──────┬───────┘
      │                                      │
      │                                      ▼
      │                               ┌──────────────┐
      │                               │ RAGService    │
      │                               │ ingest_       │
      │                               │ document()    │
      │                               └──────┬───────┘
      │                                      │
      │                  ┌───────────────────┼─────────────────────┐
      │                  ▼                   ▼                     ▼
      │           Generate hash      Upload to Digital        Create PENDING
      │           (Check dupes)      Asset Service (S3)       DB Document
      │                                      │                     │
      │◄─────── Return {document_id, task_id} ─────────────────────┘
      │
      │  Subscribe via SSE (/progress)
      ▼
┌──────────┐          ┌──────────────────────────┐
│ Redis     │ ◄────── │ makunai-worker (Celery)   │
│ (Progress)│         │                          │
└──────────┘         │ process_document_ingestion│
                      │ ├── Parse bytes          │
                      │ │   (PyMuPDF, csv, etc.) │
                      │ ├── Chunk text           │
                      │ ├── Embed chunks (OpenAI)│
                      │ └── Bulk save vectors    │
                      └──────────────────────────┘
```

---

## 3c. End-to-End Data Flow: Skin Analysis

This flow is triggered when the AI agent executes the `analyze_skin` tool on an uploaded image asset.

```
┌──────────┐      tool_call (analyze_skin)       ┌──────────────┐
│  Client   │ ◄───────────────────────────────── │  LLMEngine   │
│ (Browser) │     tool_result event (SSE)        │  run_turn()  │
└──────────┘                                     └──────▲───────┘
                                                        │
                                                        │ execute_tool()
                                                        ▼
                                                 ┌──────────────┐
                                                 │ ToolExecutor │
                                                 └──────┬───────┘
                                                        │
                                                        ▼
                                          ┌─────────────────────────────┐
                                          │    SkinAnalysisService      │
                                          │    .analyze()               │
                                          └──────┬──────┬────────┬──────┘
                                                 │      │        │
                   ┌─────────────────────────────┘      │        └──────────────────────────────┐
                   ▼                                    ▼                                       ▼
        ┌─────────────────────┐               ┌───────────────────┐                  ┌─────────────────────┐
        │LLMVisionSkinEngine  │               │VectorRepository   │                  │ImageAnnotation-     │
        │.analyze_skin()      │               │(RAG lookup)       │                  │Renderer.draw()      │
        └─────────────────────┘               └───────────────────┘                  └─────────────────────┘
           Vision LLM detects                     Fetches expert                         Draws bounding boxes
           scores + coordinates                   skincare guidelines                    and uploads to S3
```

### Steps in the Pipeline
1. **Tool Invocation**: The user uploads an image, and the LLM detects the image's `asset_id`. It invokes the `analyze_skin` tool.
2. **Tool Execution**: `ToolExecutor` dispatches the call to `SkinAnalysisService.analyze()`.
3. **Vision Analysis**: `SkinAnalysisService` routes the original image to `LLMVisionSkinEngine`, which calls the vision LLM. The LLM returns a structured JSON containing overall skin score, individual attribute scores, and coordinates of detected region findings.
4. **Knowledge Retrieval (RAG)**: The service queries `KnowledgeBaseRetriever` with the detected concerns to retrieve relevant skincare advice.
5. **Recommendation & Trajectory Generation**: A second LLM call consolidates the RAG documents and raw concerns into structured product recommendations. The service computes a 12-week numerical trajectory using a heuristic improvement curve in `static_methods.py`.
6. **Annotation Rendering**: The service invokes `ImageAnnotationRenderer`, which downloads the original image, draws color-coded bounding boxes on detected concerns, and uploads the annotated image as a new digital asset.
7. **Persistence & Return**: The final report is saved to `makun_ai_skin_scans` table and returned as `SkinScanResponse`.
8. **SSE Streaming**: `LLMEngine` catches the tool execution result and yields a `ToolResultEvent`. `ChatService` maps this to `ToolResultStreamEvent` and streams it to the client via Server-Sent Events, prompting the frontend to render the report card.

---


## 3d. End-to-End Data Flow: Sub-Agent Delegation

```
[User Chat] 
    │
    ▼
[ChatService] → [LLMCore] 
                     │
             (LLM outputs tool_call for 'delegate_to_agent')
                     │
                     ▼
             [ToolExecutor]
                     │
                     ▼
             [SubAgentRunner] 
                     │
            (Loads child AgentConfig)
            (Injects parent context summary)
            (Runs isolated ReAct loop with timeout)
                     │
         ┌───────────┴───────────┐
         │                       │
 [ToolServiceClients]    (Async celery task)
 (Child uses tools)      [write_invocation_log]
         │                       │
         └───────────┬───────────┘
                     │
         (Returns SubAgentResult: findings/status)
                     │
                     ▼
             [ToolExecutor] 
             (Formats result as tool output)
                     │
                     ▼
             [LLMCore] (Parent resumes logic)
```

### Steps in the Pipeline
1. **Delegation Prompting**: The parent agent decides to delegate a task based on the `delegate_to_agent` tool in its context, mapped from `agent_routes`.
2. **Execution Trap**: `ToolExecutor` traps the `delegate_to_agent` call and invokes the `SubAgentRunner` directly.
3. **Sub-Agent Loop**: `SubAgentRunner` fetches the child agent config, filters out the delegation tool to prevent recursion, and begins an isolated `LLMEngine.run_turn()` execution.
4. **Context Boundary**: The child agent does NOT save its conversation history to the main MongoDB session transcript. Its tool uses and thoughts are ephemeral.
5. **Yielding Findings**: The child agent finishes, emitting findings. The `SubAgentRunner` returns a `SubAgentResult` object.
6. **Telemetry Logging**: The runner fires an async Celery task `write_invocation_log` to insert a record into the `agent_invocation_log` and `agent_invocation_findings` tables.
7. **Parent Resumption**: The findings are serialized as the tool output back into the parent agent's ReAct loop, which then synthesizes the final response to the user.

---


## 4. Design Patterns Used

### 4.1 Layered Architecture
```
Controller (HTTP layer — request parsing, error mapping)
    ↓
Service (business logic — validation, orchestration)
    ↓
Repository (data access — SQLAlchemy/MongoDB queries)
    ↓
Model (ORM/Pydantic — data shape)
```

### 4.2 Static Methods Delegation
Heavy service classes delegate pure functions to `*StaticMethods` classes for testability:
```python
class AgentService:
    _resolve_system_prompt = staticmethod(AgentServiceStaticMethods._resolve_system_prompt)
```

### 4.3 Provider/Strategy Pattern
LLM providers are pluggable via `BaseLLMProvider` interface + `LLMProviderFactory` registry. Integration adapters follow the same pattern via `BaseIntegrationAdapter`.

### 4.4 Dependency Injection
All dependencies are wired through the DI container. Controllers use `@inject` + `Depends(Provide[Container.xxx])`. No service creates its own dependencies.

### 4.5 Event Streaming
Chat uses discriminated union types for stream events:
```python
ChatStreamEvent = TextChunkStreamEvent | ToolCallStreamEvent | TransferStreamEvent | ...
```

### 4.6 Upsert for History
MongoDB chat transcripts use `$push` + `$setOnInsert` upserts — first message auto-creates the document, subsequent messages append.

---

## 5. Environment Variables Reference

### Core
| Variable | Default | Description |
|---|---|---|
| `SERVICE_NAME` | `MakunAI Service` | Service display name |
| `ENV` | `` | Environment (triggers env validation when set) |

### Partner Config & Middleware Defaults
| Variable | Default | Description |
|---|---|---|
| `DEFAULT_CHAT_RATE_PER_MIN` | `60` | Default fallback rate limit for unprovisioned partners |
| `DEFAULT_DAILY_TOKEN_LIMIT` | `100000` | Default fallback daily token limit |
| `DEFAULT_RATE_LIMIT_ENABLED` | `False` | Whether rate limiting is enabled by default for unprovisioned partners |
| `DEFAULT_IDEMPOTENCY_ENABLED` | `False` | Whether idempotency checks are enabled by default |
| `CORS_ALLOW_ORIGINS` | `None` | Semicolon-separated list of client CORS origins allowed |
| `ALLOWED_HOSTS` | `None` | Semicolon-separated list of HTTP Host headers allowed |

### PostgreSQL
| Variable | Default | Description |
|---|---|---|
| `POSTGRES_DB` | `makunai_db` | Database name |
| `POSTGRES_USER` | `makunai_user` | Username |
| `POSTGRES_PASSWORD` | `makunai_pass` | Password |
| `POSTGRES_HOST` | `makunai-postgres` | Hostname |
| `POSTGRES_PORT` | `5432` | Port |

### MongoDB
| Variable | Default | Description |
|---|---|---|
| `MONGO_DB` | `makunai-service` | Database name |
| `MONGO_USER` | `jaggerbomb` | Username |
| `MONGO_PASSWORD` | `jaggerbomb` | Password |
| `MONGO_HOST` | `makunai-mongo-db` | Hostname |
| `MONGO_PORT` | `27016` | Port |

### Redis
| Variable | Default | Description |
|---|---|---|
| `CACHE_PROTOCOL` | `redis` | `redis` or `rediss` (TLS) |
| `CACHE_HOST` | `makunai-redis` | Hostname |
| `CACHE_PORT` | `6379` | Port |
| `CACHE_PASSWORD` | `pwd` | Password |
| `CACHE_DB` | `1` | Database number |

### Voice / LiveKit
| Variable | Default | Description |
|---|---|---|
| `LIVEKIT_URL` | `https://livekit.makunai.com` | LiveKit server URL |
| `LIVEKIT_API_KEY` | `` | LiveKit API key |
| `LIVEKIT_API_SECRET` | `` | LiveKit API secret |
| `DEEPGRAM_API_KEY` | `` | Deepgram STT key |
| `CARTESIA_API_KEY` | `` | Cartesia TTS key |
| `OPENAI_API_KEY` | `` | OpenAI API key |
| `TTS_PROVIDER` | `cartesia` | TTS backend: `cartesia` or `openai` |
| `STT_PROVIDER` | `deepgram` | STT backend |

### gRPC / Auth
| Variable | Default | Description |
|---|---|---|
| `CONSOLE_GRPC_HOST` | `console-service` | Console service hostname |
| `CONSOLE_GRPC_PORT` | `50051` | Console service port |
| `CA` | `None` | Base64 CA cert (set for TLS, `None` for insecure) |

### WhatsApp (Meta Cloud API — for inbound webhook)
| Variable | Default | Description |
|---|---|---|
| `WHATSAPP_API_TOKEN` | `12345` | Fallback Graph API bearer token used to download media when a per-partner token is not available. Set to a real long-lived access token in production. |
| `WHATSAPP_VERIFY_TOKEN` | `12345` | Shared secret echoed in the GET verify handshake. Must match the value entered in the Meta dashboard under WhatsApp → Configuration → Webhook. |
| `WHATSAPP_APP_SECRET` | `12345` | Meta App Secret used to verify `X-Hub-Signature-256` on inbound POSTs. |

---

## 6. Developer Quickstart

### Prerequisites
- Docker & Docker Compose
- Python 3.11+
- Poetry

### Local Setup
```bash
# 1. Clone
git clone <repo> && cd makun-ai

# 2. Copy env
cp .env.example .env
# Edit .env with real API keys

# 3. Create Docker network
docker network create makunai_console

# 4. Start everything
docker-compose up --build

# Services will be available at:
# API:      http://localhost:7000
# Swagger:  http://localhost:7000/docs
# Postgres: localhost:4000
# MongoDB:  localhost:27016
# Redis:    localhost:6378
```

### Run Migrations
```bash
docker exec -it makunai-service poetry run alembic upgrade head
```

### Adding a New Component
Follow the established pattern:

1. Create directory: `src/components/your_module/`
2. Create files:
   - `models.py` — SQLAlchemy ORM models
   - `schemas.py` — Pydantic request/response schemas
   - `repositories.py` — Data access layer
   - `services.py` — Business logic
   - `controllers.py` — FastAPI router + endpoints
   - `constants.py` — Enums and constants
   - `errors.py` — Custom exceptions
   - `static_methods.py` — Pure functions for testability
   - `__init__.py`
3. Register in `container.py` — add repository (Singleton) + service (Factory)
4. Add wiring module path to `Container.__add_wiring()`
5. Register router in `routes/base.py`

> **Sub-modules:** a component may grow a focused sub-package (e.g. [`src/components/chat/media/`](../../src/components/chat/media/)) that follows the same constants / errors / schemas / services / controllers layout internally. Wire each sub-module's controller into `Container.__add_wiring()` (`..components.chat.media.controllers`) and add any sub-module providers in the parent component's section of `container.py`. Prefer `providers.Singleton` for services that hold no per-request state (mutable state lives in Redis / DB) — see the `chat_attachment_service` comment in `container.py` explaining why a `Factory` would have been wrong here.

### Adding a New LLM Provider
```python
# 1. Create src/components/llm/providers/your_provider.py
class YourProvider(BaseLLMProvider):
    @property
    def provider_name(self) -> str:
        return "your_provider"

    async def stream(self, messages, model, temperature, max_tokens, tools):
        # Yield StreamDelta objects
        ...

# 2. Register in factory.py
LLMProviderFactory.register(ProviderType.YOUR_PROVIDER, YourProvider)

# 3. Add to ProviderType enum in constants.py
```

### Adding a New Integration Provider

**A. Native HTTP Providers:**
```python
# 1. Create adapter: src/components/integrations/adapters/your_adapter.py
class YourAdapter(BaseIntegrationAdapter):
    async def execute(self, tool_name, args, credentials, config) -> dict:
        ...

# 2. Create seed template: src/components/integrations/seeds/your_template.py

# 3. Register in integration_handler.py:
_ADAPTERS["YOUR_PROVIDER"] = YourAdapter
_PREFIX_TO_PROVIDER["your_provider"] = "YOUR_PROVIDER"
```

**B. MCP Providers:**
MCP providers do not require custom Python adapters or seed templates. The standard `McpAdapter` automatically discovers their JSON-RPC `tools/list` schema via the `/discover-mcp` endpoint and executes them dynamically.

---

## 7. Key Conventions

| Convention | Detail |
|---|---|
| **Timestamps** | Stored as Unix ms (`BigInteger`) via `DateTimeUtil.get_current_time()` |
| **Soft deletes** | `is_active=False` + `deleted_by` + `deleted_at`, row kept |
| **Publish gate** | Entities need explicit publish before going live |
| **Partner scoping** | Every query includes `partner_id` filter |
| **Error handling** | Domain errors → custom exceptions → controller maps to HTTP codes |
| **Logging** | Structured via `MakunaiServiceLogger`, prefixed by component |
| **Tool namespacing** | Integration tools prefixed: `{provider}__` (e.g. `slack__send_message`) |
| **Services Export** | Internal modules inside `src/components/` must explicitly export classes via `__init__.py` (e.g. `services/__init__.py`) to prevent deep import coupling. |
| **FastAPI Union Returns** | When route handlers return complex `Union` schemas (e.g. `Union[SuccessResponse, ErrorResponse]`), you MUST specify `response_model=None` in the `@router` decorator to bypass OpenAPI auto-generation crashes (`FastAPIError`). |
| **History tagging** | Every chat message stores an `interface` (`chat` / `whatsapp` / `pstn` / `telegram` / `email`) and a `medium` (`text` / `voice`) — enums in [src/components/common/channel.py](../../src/components/common/channel.py). `INTERFACE_SUPPORTED_MEDIUMS` is the authoritative valid-combo map. Replaces the old `channel` field (renamed in PR #21, migration `b798831ec8fa_v3.py`). |

---

## 8. Glossary

| Term | Meaning |
|---|---|
| **Partner** | A tenant/business using the platform. All data is partner-scoped. |
| **Agent** | An AI agent configuration (prompt + model + tools). Not a running process. |
| **Session** | A conversation thread identified by `session_id`. Spans multiple turns. |
| **Turn** | One user message → one or more LLM responses (including tool calls). |
| **Tool** | A function the LLM can call during a turn. Defined with JSON Schema. |
| **Delegation Mode** | What happens after a tool executes (SILENT/INJECT/TRANSFER/HANDOFF). |
| **Sliding Window** | Last N messages loaded as context for each LLM call. Configurable per agent. |
| **Integration** | A connected third-party service (Google Calendar, Slack, etc.) on an agent. |
| **Credential Vault** | Encrypted storage for OAuth tokens and API keys. |
| **PartnerToolCredential** | Encrypted credential entity used by the Tools Module, supporting Bearer Token, API Key (Header), Basic Auth, OAuth2, and Query Parameter auth schemas. |
| **RoomMetadata** | JSON payload attached to LiveKit rooms — carries all config the worker needs. |
| **Interface** | The delivery surface a message arrives on (`chat`, `whatsapp`, `pstn`, `telegram`, `email`). |
| **Medium** | The communication modality (`text` or `voice`). |
| **Attachment** | A file (image / document / audio / video) uploaded ahead of a chat message via `POST /ai/v1/chat/attachments` and referenced by `asset_id`. |
| **Content Block** | One element of a multimodal `content` list — `{"type": "text", …}` or `{"type": "attachment_ref", …}` in storage; rebuilt into provider-specific blocks (OpenAI `image_url`, Anthropic `image` / `document`) before each LLM call. |
| **Digital Asset** | A stored file (Postgres `digital_assets` row + S3-compatible object) — the backing store for chat attachments and other uploaded media. |
| **ContextOrchestrator** | Coordinates history retrieval, RAG chunk injection, and summarization asynchronously prior to the LLM call. |
| **CompressionService** | Background process that batches older conversation turns and distills them into a compact `SummaryBlock` via `gpt-4o-mini` to save context window tokens. |
| **ToolOutputCompactor** | Truncates over-sized arrays and strings returned by tools to prevent prompt explosion. |
| **LMRS** | Lazy Media Rehydration System. Replaces historical attachments with a compact `attachment_ref` block and exposes a `fetch_attachment` tool. |
| **Parser** | RAG component that extracts text from raw file bytes without saving to disk (e.g., PyMuPDF for PDFs, Python CSV module). |
| **Chunker** | RAG component that splits document text into semantic blocks to optimize vector similarity searches. |
| **Embedding** | RAG component that calls an LLM to convert a text chunk into a high-dimensional vector for database storage. |
