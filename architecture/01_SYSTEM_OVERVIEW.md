
# MakunAI — System Overview & Architecture

> **Version:** 1.0 · **Last Updated:** 2026-05-13
> **Service Name:** `makunai-service` · **Port:** 7000 (HTTP) / 50050 (gRPC)

---

## 1. What is MakunAI?

MakunAI is a **multi-tenant AI agent platform** that lets partners (businesses) create, configure, and deploy AI agents across **text chat**, **real-time voice**, and **WhatsApp** interfaces, with support for image / document / audio / video attachments on chat turns. Each agent is fully configurable — model, prompt, tools, integrations — and can be embedded into any product via REST API.

### Core Capabilities (Implemented)
| Capability | Status | Description |
|---|---|---|
| Agent CRUD & Publishing | ✅ Done | Create, update, publish, deactivate AI agents |
| Streaming Text Chat (SSE) | ✅ Done | Real-time LLM streaming with Server-Sent Events |
| Multimodal Chat Attachments | ✅ Done | Image / document / audio / video uploads (direct API and WhatsApp), rehydrated into provider-specific multimodal blocks at LLM call time |
| WhatsApp Inbound Webhook | ✅ Done | Meta Cloud API webhook verify + signed inbound message routing into the chat pipeline |
| Tool Execution Engine | ✅ Done | LLM function calling with 4 delegation modes |
| Real-Time Voice Calls | ✅ Done | LiveKit-powered voice with STT/TTS/VAD pipeline |
| Third-Party Integrations | ✅ Done | OAuth2 + API Key credential flow (Google Calendar, Slack, Calendly, Razorpay) |
| Agent Prompt Builder | ✅ Done | LLM-assisted system prompt generation via industry templates |
| Multi-LLM Provider Support | ✅ Done | OpenAI + Anthropic with pluggable factory |
| gRPC Auth | ✅ Done | JWT + API Key auth via Console Service gRPC |
| Conversation History | ✅ Done | MongoDB sliding-window history with per-session transcripts and compressed summary blocks (no PostgreSQL) |
| Context Orchestration | ✅ Done | Parallelized history, structured memory, hot tier cache, RAG fetching, LLM turn compression, tool output compaction, and lazy media rehydration |
| RAG / Knowledge Base | ✅ Done | Multi-format asynchronous document ingestion with vector search and agent-level access |
| Skin Analysis Engine | ✅ Done | Vision-based skin analysis, concern bounding boxes annotation, product & routine recommendations, and trajectory projection |
| Agent Orchestration | ✅ Done | Parent-to-child sub-agent delegation, isolated ReAct loops, multi-tier protection |
| Celery Workers | ✅ Scaffold | Worker + Beat configured, placeholder tasks |

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| **Language** | Python 3.11+ |
| **Web Framework** | FastAPI (async, ASGI) |
| **DI Container** | `dependency-injector` (DeclarativeContainer) |
| **ORM** | SQLAlchemy 2.0 (async, `asyncpg`) |
| **Relational DB** | PostgreSQL 16 |
| **Document DB** | MongoDB 6 (via `motor` async driver) |
| **Cache / Broker** | Redis 5+ |
| **Background Tasks** | Celery 5 (worker + beat) |
| **Migrations** | Alembic |
| **LLM SDKs** | `openai`, `anthropic` |
| **Image Processing / Rendering** | Pillow (PIL) |
| **Voice** | LiveKit Agents SDK, Deepgram STT, Cartesia/Deepgram TTS, Silero VAD, DTLN noise suppression |
| **Auth** | gRPC to Console Service (JWT validation, API Key validation) |
| **HTTP Client** | `httpx` (pooled async clients) |
| **Containerization** | Docker + Docker Compose |
| **Encryption** | `cryptography` (Fernet, for credential vault) |

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                  │
│   Browser / Mobile / External LLM / MCP Client                  │
└──────────┬──────────────────────────────┬───────────────────────┘
           │ REST (HTTP/SSE)              │ WebRTC (LiveKit)
           ▼                              ▼
┌─────────────────────┐        ┌─────────────────────┐
│  makunai-service     │        │  makunai-livekit-    │
│  (FastAPI :7000)     │        │  worker              │
│                      │        │  (LiveKit Agent)     │
│  ┌───────────────┐   │        │                      │
│  │ Auth Middleware│   │        │  VoiceSessionHandler │
│  │ (gRPC → Redis)│   │        │  ├─ Silero VAD       │
│  └───────┬───────┘   │        │  ├─ DTLN Noise       │
│          ▼           │        │  ├─ Deepgram STT     │
│  ┌───────────────┐   │        │  ├─ LLMCore Adapter  │
│  │ Route Modules │   │        │  └─ TTS (Cartesia/DG)│
│  │ /ai/v1/...    │   │        └──────────┬───────────┘
│  └───────┬───────┘   │                   │
│          ▼           │                   │
│  ┌───────────────┐   │                   │
│  │  Services     │◄──┼───────────────────┘
│  │  (Business    │   │  (shared Container, DB, LLMCore)
│  │   Logic)      │   │
│  └───────┬───────┘   │
│          ▼           │
│  ┌───────────────┐   │
│  │  Repositories │   │
│  └──┬─────────┬──┘   │
│     │         │      │
│     ▼         ▼      │
│  Postgres   MongoDB  │
└─────────────────────┘

     ┌──────────┐   ┌──────────┐   ┌──────────────┐
     │  Redis   │   │ Celery   │   │ Console Svc  │
     │  Cache   │   │ Worker   │   │ (gRPC :50051)│
     └──────────┘   └──────────┘   └──────────────┘
```

---

## 4. Docker Services

Defined in `docker-compose.yml` — all on `makunai_console` network:

| Service | Container | Purpose | Port |
|---|---|---|---|
| `makunai-service` | FastAPI app | Main API server | 7000 |
| `celery-worker` | Celery worker | Background task execution | — |
| `celery-beat` | Celery beat | Periodic task scheduling | — |
| `makunai-livekit-worker` | LiveKit agent | Voice call handling | — |
| `makunai-redis` | Redis | Cache + Celery broker | 6378→6379 |
| `makunai-mongo-db` | MongoDB 6 | Chat transcript storage | 27016→27017 |
| `makunai-postgres` | PostgreSQL 16 | Relational data | 4000→5432 |

---

## 5. Project Structure

```
makun-ai/
├── src/
│   ├── main.py                    # FastAPI app entry point
│   ├── config/
│   │   ├── settings.py            # Pydantic settings (local/dev)
│   │   ├── swagger.py             # Swagger UI config
│   │   └── log.py                 # Logging config
│   ├── core/
│   │   ├── container.py           # DI Container (all wiring)
│   │   ├── environment.py         # ENV class (all env vars)
│   │   ├── db.py                  # PostgreSQL session manager
│   │   ├── db_mongo.py            # MongoDB session manager
│   │   ├── doc_db.py              # MongoDB document DB helper
│   │   └── redis.py               # Redis pool setup
│   ├── components/
│   │   ├── agents/                # Agent CRUD + config
│   │   ├── chat/                  # Text chat streaming (SSE)
│   │   │   └── media/             # Attachment upload, resolve, preprocessing,
│   │   │                          # provider content blocks (added in PR #21)
│   │   ├── context/               # Orchestrator, session memory, hot tier, compression, compaction, policies
│   │   ├── digital_assets/        # Generic file storage (S3-backed)
│   │   ├── llm/                   # LLM abstraction layer
│   │   │                          # (incl. image_tokens.py, static_methods.py)
│   │   ├── partner_config/        # Per-partner rate limit/token configuration (added in v7)
│   │   ├── prompt_builder/        # Agent template assistant & LLM prompt generation
│   │   ├── tools/                 # Tool execution engine
│   │   ├── voice/                 # Voice call pipeline
│   │   ├── integrations/          # Third-party integrations + WhatsApp webhook
│   │   ├── rag/                   # RAG (async document ingestion & vector retrieval)
│   │   ├── skin_analysis/         # Vision-based skin analysis, bounding boxes annotation, and recommendations
│   │   ├── common/                # Shared helpers (incl. channel.py — Interface/Medium enums)
│   │   ├── health/                # Health check endpoint
│   │   └── security/              # robots.txt
│   ├── middlewares/
│   │   ├── authentication.py      # JWT + API Key auth
│   │   ├── context.py             # Request context
│   │   └── __init__.py            # Middleware registration
│   ├── routes/
│   │   └── base.py                # Route registration
│   ├── database/
│   │   ├── base.py                # SQLAlchemy MetaData base
│   │   └── constants.py           # Table names
│   ├── grpc_client/               # gRPC client to Console
│   ├── makunai_celery/            # Celery app + tasks
│   ├── loggers/                   # Structured loggers
│   └── pub/                       # Generated protobuf stubs
├── alembic/                       # DB migrations
├── docs/                          # Documentation
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml                 # Poetry dependencies
└── .env.example                   # Environment template
```

---

## 6. Dependency Injection (Container)

All services are wired through `src/core/container.py` using `dependency-injector.DeclarativeContainer`. This is the **single source of truth** for all component wiring.

```
Container
├── logger               → MakunaiServiceLogger (Singleton)
├── db                   → DatabaseSessionManager (Singleton)
├── mongo_db             → DocDatabaseSessionManager (Singleton)
├── redis_pool           → RedisCache.init_redis_pool (Resource)
│
├── agent_repository     → AgentRepository (Singleton)
├── agent_service        → AgentService (Factory)
│
├── chat_transcript_repo → ChatTranscriptRepository (Singleton)
├── history_manager      → MongoHistoryManager (Singleton)
│
├── session_memory_manager → SessionMemoryManager (Singleton)
├── redis_hot_tier       → RedisHotTier (Singleton)
├── compression_service  → CompressionService (Singleton)
├── context_orchestrator → ContextOrchestrator (Singleton — injected with history, hot_tier, session_memory, and rag_service)
├── tool_output_compactor→ ToolOutputCompactor (Singleton)
│
├── digital_asset_repository → DigitalAssetRepository (Singleton)
├── digital_asset_service    → DigitalAssetService (Factory)
├── chat_attachment_service  → ChatAttachmentService (Singleton — no per-request state;
│                              all mutable state lives in Redis + Postgres)
├── llm_content_block_builder → LLMContentBlockBuilder (Singleton — captures the live
│                              attachment service at startup)
├── llm_core             → LLMCore (Singleton — injected with content_block_builder,
│                              context_orchestrator, and tool_output_compactor)
│
├── tool_repository      → ToolRepository (Singleton)
├── tool_execution_log   → ToolExecutionLogRepository (Singleton)
├── tool_service_clients → ToolServiceClients (Singleton)
├── tool_service         → ToolService (Factory)
├── tool_executor        → ToolExecutor (Singleton)
│
├── credential_vault     → CredentialVault (Singleton)
├── integration_*_repo   → 3 integration repos (Singleton)
├── integration_handler  → IntegrationHandler (Singleton)
├── integration_service  → IntegrationService (Factory)
│
├── parser_registry      → DocumentParserRegistry (Singleton)
├── document_chunker     → BaseDocumentChunker (RecursiveCharacterTextSplitter)
├── embedding_provider   → BaseEmbeddingProvider (OpenAIEmbeddingProvider)
├── rag_repository       → VectorRepository (Singleton)
├── rag_service          → RAGService (Factory)
├── knowledge_retriever  → KnowledgeBaseRetriever (Singleton)
│
├── chat_service         → ChatService (Factory — injected with attachment_service)
├── livekit_transport    → LiveKitTransport (Singleton)
├── call_session_repo    → CallSessionRepository (Singleton)
├── voice_service        → VoiceService (Factory)
│
├── skin_scan_repository → SkinScanRepository (Singleton)
├── skin_engine_factory  → SkinEngineFactory (Singleton)
├── image_annotation_renderer → ImageAnnotationRenderer (Singleton)
└── skin_analysis_service → SkinAnalysisService (Factory — injected with DI dependencies)
```

**Wired modules** (auto-injected via `@inject`):
- `middlewares.authentication`
- `middlewares.logging_request_and_response`
- `components.agents.controllers`
- `components.chat.controllers`
- `components.chat.media.controllers`
- `components.voice.controllers`
- `components.tools.controllers`
- `components.integrations.controllers`
- `components.partner_config.controllers`
- `components.rag.controllers`
- `components.skin_analysis.controllers`

---

## 7. Database Schema

### PostgreSQL Tables

| Table | Model | Purpose |
|---|---|---|
| `ai_agents` | `Agent` | Agent configuration (prompt, model, settings). Has a `medium` column (`text` / `voice`) - replaces the old `channel` column dropped by Alembic revision `b798831ec8fa_v3`. |
| `ai_tools` | `Tool` | Tool definitions available globally to a partner |
| `agent_tools` | `AgentTool` | Junction table assigning global tools to specific agents |
| `ai_call_sessions` | `CallSession` | Voice call billing/lifecycle records |
| `tool_execution_logs` | `ToolExecutionLog` | Audit log of every tool execution |
| `integration_templates` | `IntegrationTemplate` | Available provider definitions |
| `partner_credentials` | `PartnerCredential` | Encrypted OAuth/API credentials |
| `partner_tool_credentials` | `PartnerToolCredential` | Encrypted credentials (API Key, Bearer, Basic, Query Param) linked to Tools |
| `partner_integrations` | `PartnerIntegration` | Active integrations per agent |
| `partner_configs` | `PartnerConfig` | Per-partner operational configuration (rate limits, daily token limits, custom provider API keys, `use_custom_api_key` routing flag) |
| `agent_prompt_templates` | `AgentPromptTemplate` | Industry-specific templates for the prompt builder assistant |
| `digital_assets` | `DigitalAsset` | Pointers to files stored in S3-compatible storage; backs chat attachments and other uploads |
| `rag_documents` | `Document` | Uploaded document metadata, processing status, and digital asset linkage |
| `rag_vectors` | `Vector` | Document chunks and pgvector embeddings |
| `makun_ai_skin_scans` | `SkinScan` | Vision-based skin scans, bounding boxes, recommendations, and projected trajectory |
| `agent_routes` | `AgentRoute` | Defines parent-to-child delegation paths between agents |
| `agent_invocation_log` | `AgentInvocationLog` | Audit log for sub-agent invocations |
| `agent_invocation_findings` | `AgentInvocationFindings` | Stored findings generated by a sub-agent |

### MongoDB Collections

| Collection | Document Model | Purpose |
|---|---|---|
| `chat_transcripts` | `ChatTranscriptModel` | Full conversation history per session. Messages carry `interface` + `medium` and may use multimodal `content` (text + `attachment_ref` blocks). |
| `summary_blocks` | `SummaryBlockModel` | Compressed representations of older conversational turns, maintained by the CompressionService. |

**Base Model:** All PG tables extend `MetaData` which provides: `id`, `created_by`, `updated_by`, `deleted_by`, `created_at`, `updated_at`, `deleted_at`, `description`, `additional_info`.

---

## 8. Authentication Flow

```
Client Request
    │
    ▼
AuthMiddleware.dispatch()
    │
    ├── Path in excluded_paths? → bypass
    │
    ├── Has "API-KEY" header?
    │   └── Validate via gRPC → api_key_service_client
    │       └── Cache in Redis ("api_key:{key}")
    │       └── Set request.state.user = {partner_id, api_key_id, is_api_key_auth}
    │
    └── Has "Authorization: Bearer <token>" header?
        └── Check Redis cache ("token:{token}")
        └── If miss → Validate via gRPC → auth_service_client
        └── Set request.state.user = {user_id, partner_id, ...}
```

**Multi-tenancy:** Every API operation is scoped by `partner_id` extracted from the authenticated request state. One partner can never access another partner's agents, tools, or data.

---

## 9. API Routes

All routes are registered in `src/routes/base.py`:

| Prefix | Router | Module |
|---|---|---|
| `/ai/v1/agents` | `agents_router` | Agent CRUD & publish |
| `/ai/v1/chat` | `chat_router` | Text chat streaming |
| `/ai/v1/voice` | `voice_router` | Voice session lifecycle |
| `/ai/v1/tools` | `tools_router` | Tool CRUD & publish |
| `/ai/v1/integrations` | `integration_router` | Credentials & OAuth |
| `/ai/v1/integrations` | `agent_integration_router` | Per-agent integrations |
| `/ai/v1/partner/config` | `partner_config_router` | Partner Configuration |
| `/ai/v1/prompts` | `prompt_builder_router` | LLM prompt builder |
| `/ai/v1/knowledge` | `rag_router` | Async ingestion and vector search |
| `/ai/v1/skin-analysis` | `skin_analysis_router` | View skin scans history and reports |
| `/health` | `health_router` | Health check |
| `/robots.txt` | `robot_router` | SEO robots |

---

## 10. Startup Sequence

```python
# src/main.py
container = Container()         # 1. Wire all dependencies
container.check_dependencies()  # 2. Validate (placeholder)

app = FastAPI(container=container, middleware=allowed_middlewares)
Router.register_all_routes(app) # 3. Mount all routers

@app.on_event("startup")
async def startup_event():
    await container.init_resources()                             # 4. Initialise async resources (Redis pool)
    await container.tool_service_clients().initialize()          # 5. Create HTTP pools
    async with container.db().session() as session:
        await seed_agent_templates(session)                          # 5. Seed default templates
    await partner_config_service.sync_all_configs_to_redis()     # 8. Warm Redis with all partner configs
                                                                 #    (rate limits, meta_config, use_custom_api_key)

@app.on_event("shutdown")
async def shutdown_event():
    await postgres_session_manager.close()                       # 8. Cleanup
```
