# MakunAI Architecture Documentation

Complete technical documentation for the MakunAI platform. Read in order.

---

## Documents

| # | File | What You'll Learn |
|---|---|---|
| 1 | [System Overview](./01_SYSTEM_OVERVIEW.md) | What MakunAI is, tech stack, high-level architecture, Docker services, project structure, DI container, database schema, auth flow, API routes, startup sequence |
| 2 | [Component Deep Dive](./02_COMPONENT_DEEP_DIVE.md) | Detailed internals of each module: Agents, Chat (SSE streaming), LLM abstraction, Tools (delegation modes), Voice (LiveKit pipeline), Integrations (OAuth + adapters), RAG (Ingestion + Retrieval) |
| 3 | [Data Flows & Dev Guide](./03_DATA_FLOWS_AND_DEV_GUIDE.md) | End-to-end request flows (text chat, voice, integrations, document ingestion), design patterns, env vars reference, developer quickstart, how to add new components/providers, conventions, glossary |
| 4 | [API Contracts & Schemas](./04_API_CONTRACTS.md) | Exact request/response JSON shapes, error→HTTP code mappings, MongoDB document structures, SSE wire protocol, key method signatures, partner_id threading |

---

## Quick Links

- **API Docs (Swagger):** `http://localhost:7000/docs` (when running)
- **Entry Point:** [`src/main.py`](../../src/main.py)
- **DI Container:** [`src/core/container.py`](../../src/core/container.py)
- **Environment Config:** [`src/core/environment.py`](../../src/core/environment.py)
- **Route Registration:** [`src/routes/base.py`](../../src/routes/base.py)
- **Docker Setup:** [`docker-compose.yml`](../../docker-compose.yml)
- **Env Template:** [`.env.example`](../../.env.example)
- **DB Migrations:** [`alembic/`](../../alembic/)

---

## For AI Agents / LLMs Reading This

Start with **01_SYSTEM_OVERVIEW.md** for architecture context, then **02_COMPONENT_DEEP_DIVE.md** for module internals, then **03_DATA_FLOWS_AND_DEV_GUIDE.md** for request flow understanding and development patterns. The codebase follows a strict **Controller → Service → Repository → Model** layering with dependency injection via `container.py`. All data is partner-scoped (multi-tenant). The two critical runtime flows are the **chat tool loop** (Section 2 of Component Deep Dive) and the **voice pipeline** (Section 5).

---

## Keeping Docs Updated

### When to Update

| What You Changed | Which Doc to Update |
|---|---|
| New component/module | `01` (project structure, container) + `02` (new section) + `04` (API contracts) |
| New API endpoint | `04` (request/response shapes, error mapping) |
| New LLM provider | `02` (LLM section) + `03` (provider guide) |
| New integration adapter | `02` (Integrations section) + `03` (integration guide) + `04` (schemas) |
| New DB table | `01` (schema table) |
| New MongoDB collection | `01` (schema table) + `04` (document structure) |
| New delegation mode | `02` (Tools section) + `03` (data flow) + `04` (tool schemas) |
| New SSE event type | `02` (Chat section) + `04` (SSE protocol) |
| New env variable | `03` (env vars reference) |
| Docker service change | `01` (Docker services table) |
| New middleware | `01` (auth flow) |
| Changed method signature | `04` (key method signatures) |

### How to Update

1. **Make your code change** on a feature branch
2. **Update the relevant doc(s)** — see table above
3. **Log the change** in `docs/architecture/CHANGELOG.md`
4. **Include docs in your PR** — reviewers check both code and docs

### Rules
- Docs live **in the repo** — they are versioned with code
- Every PR that changes a public API or adds a module **must update docs**
- Use the same section structure — don't add new top-level sections without team discussion
- Keep examples as real JSON, not pseudocode
