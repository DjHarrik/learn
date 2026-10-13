# makun-ai documentation set — index

This folder holds developer documentation for the `makun-ai` service (`C:/Users/hp/Desktop/makun-ai-feature-dev-2026`) and its GitOps configuration repo (`C:/Users/hp/Desktop/makun-ai-oc-config-master`). It is written from the code, not from the older company documents in `docs/architecture/`, and it records where the two disagree.

## The four phases

The set is produced in four phases. **Phase 1** is the project map (`00-project-map.md`): what exists, where it is, what it is built with, how work enters and leaves the system, and what is broken or inconsistent. **Phase 2** is one deep document per module group: every class and public method, the tables, collections and Redis keys it reads and writes, and its error classes. **Phase 3** is one flow document per endpoint family or job family: a single request, call, webhook or task traced end to end across packages with file and line numbers. **Phase 4** is a learning path for a new developer plus a glossary of every project-specific term and acronym. Only Phase 1 exists today.

## Documents

| Document | Phase | Status | What it covers |
|---|---|---|---|
| `00-README.md` | - | complete | This index, the rules, and quick links |
| `00-project-map.md` | 1 | complete (verified 2026-10-10) | Product and repos, tech stack with locked versions, entry points, request/job/voice/webhook/streaming lifecycles, external services, data stores, architecture diagram, folder tree, learning order, key files, known gaps and doc contradictions, verification notes |
| `10-infra-and-config.md` | 2 | not started | `Dockerfile`, `docker-compose.yml`, CI workflows, `k8s/`, `pyproject.toml`/`poetry.lock`, `src/core/environment.py`, `src/config/`, `src/core/app_env.py` |
| `11-core-runtime.md` | 2 | not started | `src/main.py`, `src/core/container.py`, DB/Mongo/Redis managers, `src/routes/`, `src/middlewares/`, `src/components/common/`, `src/loggers/`, `src/exceptions/`, `src/grpc_client/`, `src/grpc_interceptor/`, `src/components/health/`, `src/components/security/` |
| `12-agents-sessions-partner-config-db-constants.md` | 2 | not started | `src/components/agents/`, `sessions/`, `partner_config/`, `db_constants/`, `rbac/`, `src/database/` |
| `13-chat-context-history.md` | 2 | not started | `src/components/chat/` (incl. `media/`), `context/`, `history_manager/` |
| `14-llm-orchestration.md` | 2 | not started | `src/components/llm/` (engine, core, factory, providers), `orchestration/` |
| `15-tools-integrations.md` | 2 | not started | `src/components/tools/` (executor, clients, credentials), `integrations/` (vault, OAuth2, MCP, adapters) |
| `16-voice-core.md` | 2 | not started | `src/components/voice/` controllers, services, repositories, worker, pipeline, platform, ports, constants |
| `17-voice-session-adapters.md` | 2 | not started | `src/components/voice/session/` and `src/components/voice/adapters/` |
| `18-voice-transcript.md` | 2 | not started | `src/components/voice/transcript/` |
| `19-campaign-cdr.md` | 2 | not started | `src/components/campaign/` and `src/components/cdr/` |
| `20-rag-skin-prompt-builder.md` | 2 | not started | `src/components/rag/`, `skin_analysis/`, `prompt_builder/`, `email_service/`, `templates/` |
| `21-celery-jobs.md` | 2 | not started | `src/makunai_celery/` and every `tasks.py` |
| `22-oc-config-repo.md` | 2 | not started | The `makun-ai-oc-config` repo: ArgoCD Application, Kubernetes Secret (key names only), lint workflow |
| `50-flow-*.md` | 3 | not started | One document per endpoint or job family (chat turn, attachments, agent CRUD, voice LiveKit call, voice SIP relay call, campaign dispatch tick, CDR webhooks, RAG ingestion, OAuth connect, MCP discovery, skin analysis, prompt builder, session sweeps) |
| `90-learning-path.md` | 4 | not started | Ordered reading plan with exercises, built from section 9 of the map |
| `91-glossary.md` | 4 | not started | Every project-specific term and acronym (partner, agent, DID, CDR, hot tier, fallback flow, disposition, relay, RoomMetadata, ...) |

## Rules this documentation follows

- The code is confidential company property. These documents stay with the code and are not published elsewhere.
- No secret value is ever written down: no API keys, passwords, tokens, HMAC secrets, connection strings with credentials or certificates, including the hardcoded defaults in `src/core/environment.py` and every value in `oc-config/dev/makun-ai-env-secret.yaml`. Secrets are named by environment-variable name only.
- No source file is changed. The only files created or modified are Markdown files under `docs/`.
- Every non-obvious claim cites a file path and line number. A claim about a library is marked "verified" only when the library source was read at the locked version in a scratch virtualenv; otherwise it is marked "not verified".
- Library versions are the ones locked in `poetry.lock` (179 packages). The scratch virtualenv used for verification matches the lock for the packages whose source was read (starlette 0.46.2, fastapi 0.115.14, celery 5.6.3, kombu 5.6.2, dependency-injector 4.49.0, livekit-agents 1.8.2, livekit-plugins-turn-detector 1.6.10, starlette-context 0.3.6, redis 5.3.1, sqlalchemy 2.0.49, anthropic 0.21.3, openai 2.31.0, pydantic 2.13.4 and the others in map section 12.2), but it is not a faithful install of the lock: asyncpg is 0.31.0 (locked 0.30.0), pillow 11.3.0 (10.4.0), livekit-api 1.2.1 (1.2.0), prometheus-client 0.26.0 (0.25.0), grpcio 1.84.0 (1.80.0), and in total 79 of the 179 locked packages are at another version and 8 (including opencv-python-headless, langchain and grpcio-tools) are not installed. A library claim is "verified" only when the installed version equals the locked one (map section 12.3).
- Both repositories are covered: paths are relative to `C:/Users/hp/Desktop/makun-ai-feature-dev-2026`; paths in `C:/Users/hp/Desktop/makun-ai-oc-config-master` are prefixed `oc-config/`.
- The documents do not explain how to run the project.
- The text is written for a developer who knows Python basics and FastAPI/SQLAlchemy basics only; every acronym is expanded on first use and every sentence is about this code.

## Where things are

| What | Where |
|---|---|
| FastAPI app, startup and shutdown hooks | `src/main.py` |
| Dependency-injection container (every repository, service and resource) | `src/core/container.py` |
| Environment variables (`ENV` class, URL builders, `validate_env_vars`) | `src/core/environment.py` |
| Router registration (25 component routers, all under `/ai/v1/` except `GET /health`; `/robots.txt` is added in `src/main.py:35`) | `src/routes/base.py` |
| Middleware list in runtime order | `src/middlewares/__init__.py` |
| Celery routes, default queue and beat schedule | `src/makunai_celery/celery_config.py` |
| Celery app, queues and attached container | `src/makunai_celery/celery_app.py` |
| LiveKit voice worker process | `src/components/voice/worker.py` |
| Table and collection name constants | `src/database/constants.py` |
| Local multi-process start (API, 3 Celery pools, beat, Redis, Mongo, Postgres, voice worker) | `docker-compose.yml` |
| Container image | `Dockerfile` |
| Kubernetes manifests in the main repo (two campaign workers + KEDA) | `k8s/campaign-dispatch-worker.yaml` |
| Schema migrations (34, linear) | `alembic/env.py`, `alembic/versions/` |
| CI workflows | `.github/workflows/build.yaml`, `.github/workflows/deployment.yaml` |
| Older company documentation | `docs/architecture/` |
| ArgoCD Application for the dev cluster | `oc-config/argocd/apps/makun-ai-dev.yaml` |
| Kubernetes Secret for the dev cluster (key names only; values are secrets) | `oc-config/dev/makun-ai-env-secret.yaml` |
| oc-config lint workflow | `oc-config/.github/workflows/pr-validate.yaml` |
| This map | `docs/00-project-map.md` |
