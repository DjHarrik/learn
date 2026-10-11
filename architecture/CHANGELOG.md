# Architecture Docs — Changelog

Track every update to the architecture documentation here. Newest first.
## 2026-08-31 — Complete Memory Management Implementation & Security Fixes

**Author:** Vivek
**Files changed:**
- `src/components/context/session_memory.py`, `src/components/sessions/tasks.py` — Implemented `SessionMemoryManager` with Redis backing and MongoDB fallback for structured session memory, complete with Lua-script conflict resolution (confirmed vs inferred facts) and a Celery background sweeper for idle session cleanup.
- `src/components/context/context_orchestrator.py` — Consolidated memory flows. The `ContextOrchestrator` now pulls from `MongoHistoryManager` (warm tier), `SessionMemoryManager` (structured facts), `RedisHotTier` (fast cache), and `RAGService` in parallel to build LLM prompts.
- `src/components/llm/engine.py`, `src/components/chat/services.py`, `src/components/voice/adapters/` — Enforced strict tool call visibility limits. All raw `ToolCallEvent` and `ToolResultEvent` JSON blocks are now securely swallowed by `LLMEngine.run_turn`. Replaced voice-adapter tool injection points with an `EndOfTurnData.had_tools` boolean flag to trigger cache-bypass logic and UX fillers ("Just a moment...") without leaking JSON to the TTS engine or SSE client.
- `src/components/chat/repositories.py` — Fully deprecated the PostgreSQL cold-tier archival system in favor of keeping all durable chat history and compressed summary blocks natively within MongoDB. Trimmed messages are cleanly evicted from the hot window cache.
- `docs/architecture/` — Updated `01_SYSTEM_OVERVIEW.md` and `02_COMPONENT_DEEP_DIVE.md` to reflect the removal of Postgres chat storage and the final memory management data flows.

## 2026-08-10 — Global Tools, Document Parsing, and Vector Storage Optimizations

**Author:** Aditya Tripathi (Vivek)
**Files changed:**
- `src/components/tools/models.py`, `schemas.py`, `static_methods.py`, `controllers.py` — Added `is_global` boolean flag to the Tool models and schemas. Updated list endpoints to fetch global tools alongside partner-specific tools.
- `src/components/llm/engine.py`, `src/core/container.py` — Injected `ToolExecutionLogRepository` into the `LLMEngine` to automatically intercept and write implicit RAG context events (`RagContextEvent`) into the PostgreSQL `makun_ai_tool_execution_logs` table, granting parity with explicit tool execution history in the dashboard.
- `src/components/rag/ingestion/parsers/pdf_parser.py` — Re-architected PDF extraction to automatically detect pages with images (`page.get_images()`) and offload them to OpenAI's Vision API (`gpt-4o-mini`). Accurately extracts complex tabular layouts embedded in images into Markdown without increasing image resolution or payload size.
- `src/components/rag/repositories.py` — Optimized vector database cleanup routines by converting the document chunk deletion sequence from a soft-delete (`UPDATE deleted_at`) to a hard-delete (`DELETE FROM makun_ai_document_chunks`), instantaneously reclaiming pgvector database space when files are removed.

## 2026-07-22 — Agent Orchestration, Routing, and Sub-Agent Delegation

**Author:** Vivek Anand
**Files changed:**
- `src/components/orchestration/` — Implemented the `SubAgentRunner` for isolating sub-agent ReAct loops. Added `OrchestrationRepository` and Celery tasks for async invocation telemetry logging into `agent_invocation_log` and `agent_invocation_findings`.
- `src/components/agents/services.py`, `controllers.py`, `repositories.py`, `models.py` — Added parent-to-child agent route management (REST endpoints, models, and DB lookups). Enabled dynamic generation of the `delegate_to_agent` tool within orchestrator system prompts.
- `src/components/tools/executor.py` — Added native execution handling of the `delegate_to_agent` tool, hooking directly into the `SubAgentRunner` and handling multi-tier limitations.
- `src/components/voice/adapters/llm_core_bridge.py` — Registered `delegate_to_agent` in the voice adapter to emit "Just a moment..." streaming wait messages.
- `alembic/versions/976f7c0c7b67_add_agent_invocation_log_tables.py` — Resolved PostgreSQL `DatatypeMismatchError` by adding `postgresql_using` CAST logic for legacy String-to-UUID column migrations.
- `src/core/container.py`, etc. — Added comprehensive static type hinting (`Any`, `Optional`, explicit dict mappings) across `agents`, `orchestration`, `tools`, and `voice` modules to strictly comply with "Mandatory Coding Standards v1.0".

## 2026-07-15 — Operational Reliability, Idempotency, and Coding Standards

**Author:** Vivek Anand
**Files changed:**
- `src/components/chat/services.py`, `constants.py` — Implemented strict idempotency locking and caching using Redis to prevent duplicate execution and billing during transient retries. Extracted constants and added strict type hints to comply with 1.0 Coding Standards.
- `src/components/history_manager/manager.py` — Fixed OpenAI context truncation errors (hard 400s) by making the sliding window "pair-aware" and stripping orphaned `role: "tool"` results when the assistant's `tool_calls` message falls outside the MongoDB `$slice`. Added strict type hints.
- `src/components/tools/executor.py` — Addressed a silent data loss compliance gap by replacing the silent `except Exception: pass` block with targeted `SQLAlchemyError` catching and explicit Datadog telemetry logging via `logger.error`.
- `src/components/sessions/tasks.py`, `constants.py` — Updated the `sweep_idle_sessions` Celery task to transition idle session statuses to `CLOSED` (`idle_timeout`) instead of executing a destructive SQL `delete`, preserving hot telemetry in PostgreSQL. Created session status constants and applied explicit types.


## 2026-07-09 — Integrations Module Refactor & MCP Foundations

**Author:** Vivek Anand
**Files changed:**
- `src/components/integrations/model.py` — Centralized database models from `models/` directory, adding MCP protocol fields (`negotiated_protocol_version`, `resource_indicator`) and integration lifecycle/health state tracking to `PartnerIntegration`. Added dynamic config fields (`oauth_config`, `client_config`, `base_config`, `adapter_key`, `provider_prefix`) to `IntegrationTemplate`.
- `alembic/versions/0560bd113320_v14.py` — Generated schema migration for the new integration models and patched `status` column with `server_default='ACTIVE'` to prevent `NotNullViolationError` on upgrade.
- `src/components/integrations/services/` — Enforced clean internal module exports by creating `__init__.py` and routing `IntegrationService` via `integration_service.py` to decouple component boundaries and fix Celery Worker/Beat `ImportError`.
- `src/components/integrations/controllers/` — Refactored FastAPI endpoints returning complex `Union` schemas by explicitly passing `response_model=None` to `router` decorators, fixing the `FastAPIError: Invalid args for response field!` crash.
- `src/components/integrations/schemas/responses.py` — Updated `IntegrationTemplateDetailRead` to allow `provider_prefix` as nullable (`str | None`), fixing Pydantic validation errors on legacy database records.
- `docs/architecture/` — Synced all data model enhancements, new service architecture export conventions, and FastAPI `response_model=None` practices into `02_COMPONENT_DEEP_DIVE.md`, `03_DATA_FLOWS_AND_DEV_GUIDE.md`, and `04_API_CONTRACTS.md`.

## 2026-06-19 — Skin Analysis Feature Implementation & Streaming Integration

**Author:** Vivek Anand
**Files changed:**
- `src/components/skin_analysis/` (New Component) — Implemented the full Skin Analysis module including vision engine for concern detection, Pillow-based `ImageAnnotationRenderer` for drawing bounding boxes on user uploaded photos, `SkinAnalysisService` for consolidating RAG documents & product recommendations, and Postgres DB ORM mapping.
- `src/components/prompt_builder/constants.py` — Registered `SKIN_ANALYZER` category in the `TemplateCategory` enum, defined the default `SKIN_ANALYZER` seed template instructions containing the critical instructions for `analyze_skin` tool, and appended it to the `ALL_TEMPLATES` seeder.
- `src/components/llm/schemas.py`, `src/components/llm/engine.py` — Defined `ToolResultEvent` and updated the ReAct loop to yield tool results immediately upon completion.
- `src/components/chat/schemas.py`, `src/components/chat/services.py` — Defined `ToolResultStreamEvent` and mapped the tool result event into the SSE stream so that the frontend can dynamically render the detailed report card and annotated images.
- `docs/architecture/` — Documented the new component structure, DI mappings, Postgres schema, REST endpoints, and SSE data flows in `01_SYSTEM_OVERVIEW.md`, `02_COMPONENT_DEEP_DIVE.md`, `03_DATA_FLOWS_AND_DEV_GUIDE.md`, and `04_API_CONTRACTS.md`.

## 2026-06-12 — RAG Module Stateless Ingestion & Constants Refactoring

**Author:** Vivek Anand
**Files changed:**
- `src/components/rag/constants.py` — Centralized all hardcoded values, including `IngestionProgressEnum` for SSE tracking, Redis TTLs, timeouts, and standard API response messages.
- `src/components/rag/tasks.py` — Replaced hardcoded status strings and progress values with enums. Streamlined ingestion to run purely in memory from bytes, ensuring stateless horizontal scalability without local disk I/O.
- `src/components/rag/controllers.py` — Moved heavy business logic (Celery cancellation, Redis progress polling) out of the controller into the service layer. Cleaned up unused imports.
- `src/components/rag/services.py` — Added `stream_document_progress` (SSE generator) and `cancel_document`. Fixed JSON serialization errors on `da_response.url` Pydantic `HttpUrl` objects.
- `docs/architecture/02_COMPONENT_DEEP_DIVE.md` — Added Section 9 documenting the complete RAG module architecture, ingestion pipeline, retry mechanism, and vector retrieval flow.

## 2026-06-10 — Database Schema Prefix & Migration Fixes

**Author:** Vivek Anand
**Files changed:**
- `src/database/constants.py` — Renamed all database tables and MongoDB collections to use the `makun_ai_` prefix.
- `src/components/voice/models.py` — Updated `agent_id` ForeignKey reference in `CallSession` to dynamically use the `AGENT_TABLE_NAME` constant instead of a hardcoded string `ai_agents.id` to prevent migration reference errors.
- `alembic/versions/27728e9fa7fc_v10.py` — Rewrote the Alembic `v10` migration script to safely use `op.rename_table()` instead of destructive drop/create commands to preserve existing data during the table rename process.

## 2026-06-04 — Context Module (Orchestrator, Compression, LMRS)

**Author:** Vivek Anand
**Files changed:**
- `01_SYSTEM_OVERVIEW.md` — Added `Context Orchestration` to Core Capabilities. Added `context/` directory to Project Structure. Added `ContextOrchestrator`, `CompressionService`, and `ToolOutputCompactor` to Dependency Injection (Container). Added `summary_blocks` collection to MongoDB schema.
- `02_COMPONENT_DEEP_DIVE.md` — Added **Section 8. Context Module** documenting the `ContextOrchestrator` (parallel RAG/history), `CompressionService` (conversational memory summarization), `ToolOutputCompactor` (payload truncation), Lazy Media Rehydration System (LMRS), and Medium Policies.

## 2026-05-29 — Agent Template Assistant & Prompt Builder (Feature & Bugfixes)

**Author:** Vivek Anand
**Files changed:**
- `alembic/versions/bbbfd18605c6_v8.py`, `alembic/env.py` — Added database schema migrations for the new `agent_prompt_templates` table.
- `src/components/prompt_builder/` (New Component) — Implemented the Prompt Builder module with REST controllers, services, repositories, and models for managing industry-specific templates and generating system prompts.
- `src/components/prompt_builder/seeds/` — Added database seeder and 12 industry template definitions (Ecommerce, Education, Finance, Healthcare, Real Estate, etc.).
- `src/components/prompt_builder/llm_adapter.py` — Implemented LLM-assisted prompt generation and review pipelines for interactive prompt building.
- `src/components/agents/` — Updated schemas, models, controllers, and services to integrate with the new template-based prompt generation flow.
- `src/core/container.py`, `src/routes/base.py`, `src/main.py`, `src/database/constants.py` — Wired the prompt builder module into the dependency injection container, registered routes, and added table constants.
- `src/components/partner_config/repositories.py` — Fixed a bug in `update_partner_config` where `exclude_none=True` was incorrectly discarding explicit `null` updates from the `PATCH` payload. Changed to `exclude_unset=True` so fields can be properly cleared.
- `src/components/tools/repositories.py`, `src/components/tools/static_methods.py` — Fixed `tool_headers` being silently dropped during `POST` creation (missing in DB instantiation) and falsely masked as `null` in API responses (missing in `_to_read_schema` mapping).
- `docs/architecture/` — Updated `02_COMPONENT_DEEP_DIVE.md` and `04_API_CONTRACTS.md` to reflect `tool_headers` in API payloads and model definitions.

---

## 2026-05-26 — Partner API Key Routing Flag & Strict Key Enforcement

**Author:** Vivek Anand
**Files changed:**
- `alembic/env.py` — Configured database schema migration environment parameters.
- `docs/architecture/01_SYSTEM_OVERVIEW.md` — Updated Startup Sequence to reflect that `sync_all_configs_to_redis()` (including `use_custom_api_key`) now runs during `startup_event`; updated the `partner_configs` table description to mention the new `use_custom_api_key` column.
- `docs/architecture/02_COMPONENT_DEEP_DIVE.md` — Rewrote the **Partner-Specific LLM Keys Routing** sub-section (§7) to document the new `use_custom_api_key` flag, the two-step Redis lookup flow (flag → key), strict enforcement with no silent fallback, and the `MissingPartnerApiKeyError` exception. Updated the **LLM Module** error-handling section to document the new exception class and its user-abstracted stream handling. Corrected REST API update endpoint references from `PUT` to `PATCH`.
- `docs/architecture/03_DATA_FLOWS_AND_DEV_GUIDE.md` — Documented partner config default variables (`DEFAULT_CHAT_RATE_PER_MIN`, `DEFAULT_DAILY_TOKEN_LIMIT`, etc.) in the environment variables reference.
- `docs/architecture/04_API_CONTRACTS.md` — Added detailed partner config REST API contract documentation under Section 8.
- `src/config/settings.py` — Configured CORS origins and allowed hosts parsing from environment variables.
- `src/core/container.py` — Wired dependency bindings for the partner configuration service; passed `partner_config_repository` to `LLMProviderFactory.init_cache` at container startup (moved definitions above factory cache definition to avoid sequential class evaluation `NameError`).
- `src/core/environment.py` — Configured partner configuration operational default values in `ENV`.
- `src/database/constants.py` — Added table name constant for the `partner_configs` table.
- `src/main.py` — Registered `sync_all_configs_to_redis()` on application startup event.
- `src/middlewares/__init__.py` — Enabled dynamic middleware initialization for `AllowedHostsMiddleware` and CORS using settings.
- `src/middlewares/logging_request_and_response.py` — Handled streaming and non-JSON responses to prevent 500 error on chat message SSE stream responses.
- `src/routes/base.py` — Bound the partner config controller router to the application.
- `alembic/versions/b9d99442be35_v7.py` — Alembic database schema migration to add `use_custom_api_key` to `partner_configs` table.

**What changed (code):**
- `src/components/partner_config/` (New Component) — Implemented REST controller with full Swagger annotations, service layer, repositories, models, errors, Pydantic schemas (carrying `openai_api_key` & `anthropic_api_key` under `meta_config`), and static helper methods.
- `src/components/partner_config/constants.py` — Added `DEFAULT_USE_CUSTOM_API_KEY = False`.
- `src/components/partner_config/models.py` — Added `use_custom_api_key` boolean column (`server_default="false"`).
- `src/components/partner_config/repositories.py` — `_build_default_partner_config` and `create_partner_config` now include the flag. Added proper logging across all database operations, with sensitive credentials masking in logging outputs (`meta_config` keys like `openai_api_key` and `anthropic_api_key`) to prevent credential leaks in service logs. Resolved missing imports for `Result` and `Optional`.
- `src/components/partner_config/schemas.py` — Added request/response payload schemas (`PartnerConfigCreate`, `PartnerConfigUpdate`, `PartnerConfigResponse`) and success/error wrapper schemas (`PartnerConfigSuccessResponse`, `PartnerConfigErrorResponse`) carrying custom keys `openai_api_key` and `anthropic_api_key`.
- `src/components/partner_config/services.py` — Added `_write_use_custom_api_key_to_redis()` helper, wired into `get`, `create`, `update`, and `sync_all_configs_to_redis`.
- `src/components/llm/factory.py` — Replaced unconditional meta_config fetch with a two-step check: (1) read `use_custom_api_key` flag from Redis; (2) fetch custom key only when flag is `True`. DB fallback warms both Redis keys on a cache miss. Raises `MissingPartnerApiKeyError` if flag is `True` but key is absent — no silent fallback. Fixed imports of `PartnerConfig` and `PartnerConfigRepository` to use the `src.` prefix. Injected the `PartnerConfigRepository` singleton during container startup to avoid dynamic container lookups on database cache misses. Added detailed logging statements for all key operations and warnings/errors near exceptions inside `get_provider()`.
- `src/components/llm/exceptions.py` — Added `MissingPartnerApiKeyError` with partner ID, provider name, and actionable message for operators.
- `src/components/llm/core.py` — Moved `LLMProviderFactory.get_provider()` call inside the stream `try` block so `MissingPartnerApiKeyError` is caught at the same level; yields the generic `FALLBACK["generic"]` message to the end-user (detail stays in logs only).
- `src/components/llm/providers/openai_llm.py` — Updated provider backend to resolve correct API key from factory.


---

## 2026-05-25 — Tool Group, Tool Headers Bug Fixes & Alembic Migration Hardening

**Author:** Vivek Anand
**Files changed:**
- `02_COMPONENT_DEEP_DIVE.md` — Extended the Tool Model Fields table to document `display_name`, `tool_type`, `tool_category`, `tool_headers`, and `tool_group`. Added a critical developer note on `_to_read_schema`: any new field on the ORM model must be explicitly mapped in `ToolServiceStaticMethods._to_read_schema()` or it will always return `null` in API responses.

**Backend bug fixes (`src/components/tools/`):**
- `repositories.py` — `create_tool` was not passing `tool_headers` or `tool_group` to the `Tool()` ORM constructor, causing both fields to be silently dropped on creation.
- `static_methods.py` — `_to_read_schema()` was not mapping `tool_headers` or `tool_group` from the ORM model, so all GET/POST/PUT responses always returned `null` for these fields even when the database had correct values.
- `controllers.py` — Added `tool_group` as an optional query parameter to both `GET /ai/v1/tools` and `GET /ai/v1/agents/{agent_id}/tools` endpoints.
- `services.py` / `repositories.py` — Propagated `tool_group` filter through the full service → repository stack for both `list_tools` and `list_agent_tools`.

---

## 2026-05-22 — Tool Assignment and Integration Bug Fixes

**Author:** Vivek Anand
**Files changed:**
- `02_COMPONENT_DEEP_DIVE.md` — Updated the Tools section to explicitly document that tools are linked to agents via the `ai_agent_tools` many-to-many table rather than an `agent_id` column. Clarified that `PUT /ai/v1/agents/{agent_id}/tools` acts as a bulk replacement of assigned tools, and documented the idempotent auto-registration behavior in `enable_provider_for_agent()` which prevents duplicate integration tools.
- `01_SYSTEM_OVERVIEW.md` — Updated the database schema section to reflect that tools are globally scoped and added the `agent_tools` mapping table.
- `04_API_CONTRACTS.md` — Documented the new global `GET /ai/v1/tools` endpoint and the agent assignment endpoint `PUT /ai/v1/agents/{agent_id}/tools`.

---

## 2026-05-18 — Outbound Tool Authentication & Query Parameter Support

**Author:** Vivek Anand
**Files changed:**
- `01_SYSTEM_OVERVIEW.md` — Added the new `partner_tool_credentials` table to the database schema reference.
- `02_COMPONENT_DEEP_DIVE.md` — Documented the new `auth_credential_id` field in the Tool model and added a new subsection detailing the **Tool Authentication Injection Flow** with support for the new `QUERY_PARAM` credential type and 401 retry cache invalidation.
- `03_DATA_FLOWS_AND_DEV_GUIDE.md` — Added `PartnerToolCredential` to the architecture glossary.
- `04_API_CONTRACTS.md` — Updated the Tools API payload contract (`ToolCreate`) to include the optional `auth_credential_id` field.

---

## 2026-05-13 — Agent and Tool Configuration Expansion

**Author:** Vivek Anand
**Files changed:**
- `02_COMPONENT_DEEP_DIVE.md` — Added `agent_config` to `AgentConfig` and `tool_config` to Tool Model Fields. Noted the fix for partial updates and the new collision check on PUT tool operations.
- `04_API_CONTRACTS.md` — Added `agent_config` and `tool_config` to respective request payloads. Added the new `GET /ai/v1/agents/available-models` endpoint. Reflected that `provider`, `model`, `medium`, and `max_tool_calls_per_turn` are now fully updatable fields in the PUT Agent payload. Added a note that partial updates properly respect explicit `null` values.

---

## 2026-05-13 — Media support in chat (documents PR #21)

**Author:** Vivek Anand (code), Anmol (docs)
**PR:** documents code from [#21](https://github.com/MakunaiGlobal/makun-ai/pull/21) (`implemented media support in chat`) by @vivek-makunai
**Files changed:**
- `01_SYSTEM_OVERVIEW.md` — Multimodal + WhatsApp capabilities; `chat/media/`, `digital_assets/`, `common/channel.py` in the project tree; new DI providers (`digital_asset_service`, `chat_attachment_service`, `llm_content_block_builder`); `chat.media.controllers` wired; `ai_agents.channel` → `medium` and new `digital_assets` table.
- `02_COMPONENT_DEEP_DIVE.md` — `AgentConfig` field renamed `channel` → `medium`; new **Media Pipeline** subsection (upload / resolve / preprocess / errors); chat turn flow shows attachment rehydration and `_build_user_turn`; new **Multimodal Support** subsection under LLM Module (content block builder per provider, history rehydration, `image_tokens.py`, `static_methods.py`); new **WhatsApp Inbound Webhook** subsection under Integrations.
- `03_DATA_FLOWS_AND_DEV_GUIDE.md` — Chat data-flow extended with attachment resolution + content blocks; new **WhatsApp Inbound** end-to-end flow; sub-module convention added to the *Adding a New Component* guide; three `WHATSAPP_*` env vars; glossary entries for Interface, Medium, Attachment, Content Block, Digital Asset; `History channel` row replaced with `History tagging` (interface + medium).
- `04_API_CONTRACTS.md` — `ChatMessageRequest` gains `interface`, `medium`, `attachments[]`; new `POST /ai/v1/chat/attachments`; new `GET`/`POST /ai/v1/integrations/whatsapp/webhook`; `chat_transcripts` example shows multimodal `content` + `attachment_ref` blocks + per-message `interface`/`medium`; four new chat-attachment error rows plus `WhatsAppMediaDownloadError`; `agents.channel` and `voice/sessions.channel` → `medium`; **Key Method Signatures** adds `LLMRequest.attachments`, `LLMCore._build_user_turn`, `LLMCore._rehydrate_history_message`, full `ChatAttachmentService` API, and `LLMContentBlockBuilder` API.

> **Caveat:** PR #21 is open at time of writing. Two unresolved review comments may force a follow-up doc edit — (1) `Interface.CHAT` may be renamed to something more specific (`IN_APP`), (2) `src/components/common/channel.py` may be renamed since it no longer contains a `Channel` symbol.

---

## 2026-05-13 — Initial Release

**Author:** Anmol
**Files created:**
- `README.md` — Index, quick links, maintenance guide
- `01_SYSTEM_OVERVIEW.md` — Architecture, tech stack, Docker, DI, DB schema, auth, routes
- `02_COMPONENT_DEEP_DIVE.md` — Agents, Chat, LLM, Tools, Voice, Integrations internals
- `03_DATA_FLOWS_AND_DEV_GUIDE.md` — End-to-end flows, patterns, env vars, dev quickstart
- `04_API_CONTRACTS.md` — Request/response schemas, error codes, MongoDB shapes, SSE protocol

---

<!-- 
Template for new entries:

## YYYY-MM-DD — Short Description

**Author:** Name
**PR:** #123
**Files changed:**
- `0X_FILE.md` — What was added/changed and why

-->
