# Mandatory Coding Standards

**Version:** 1.0
**Scope:** All services under MakunAI codebases
**Status:** Non-negotiable — enforced at code review and CI level

These standards exist to ensure every engineer — present or future — can read, extend, and debug any part of the system without tribal knowledge. Violations should block PR approval unless explicitly justified in writing.

## 1. Type Hints Are Mandatory

Type hints are the first line of defense against runtime errors in a dynamically typed language, and they double as inline documentation.

- Every function parameter must have a type hint.
- Every function must declare an explicit return type, including `None` where applicable.
- Local variables should be annotated wherever the inferred type isn't immediately obvious from the right-hand side.
- Use `Optional[X]` or `X | None` explicitly — never rely on implicit `None` defaults.
- For async functions, annotate the coroutine's resolved type, not `Coroutine` itself (FastAPI/async handlers should return the resolved model).
- Use `TypedDict`, `Protocol`, or Pydantic models over raw `dict[str, Any]` for structured data.
- Run `mypy` (or `pyright`) in strict mode as part of CI; treat type errors as build failures.

```python
async def get_agent_documents(agent_id: UUID, include_archived: bool = False) -> list[Document]:
    active_documents: list[Document] = await document_repository.fetch_by_agent(agent_id, include_archived)
    return active_documents
```

## 2. Clear and Descriptive Naming

Names are the primary interface between code and the humans reading it. Ambiguity here compounds across every file that imports it.

- No cryptic, abbreviated, or single-letter names outside of tight loop counters (`i`, `j` in trivial iteration only).
- File, class, method, variable, and constant names must be self-describing without requiring the reader to open another file.
- Boolean variables/functions should read as predicates: `is_active`, `has_permission`, `should_retry`.
- Avoid overloaded generic names like `data`, `result`, `obj`, `handler` unless scoped tightly and unambiguous.
- Async functions performing I/O should signal it where useful: `fetch_`, `persist_`, `stream_` prefixes over vague `get_`/`process_`.

| Avoid | Prefer |
|---|---|
| `svc.py` | `knowledge_base_service.py` |
| `mgr.py` | `upload_cancellation_manager.py` |
| `tmp`, `data1` | `raw_transcript`, `partner_commission_record` |
| `fn()` | `process_uploaded_document()` |

## 3. Docstrings Are Required

Docstrings are the contract of a function — they describe intent, not implementation.

- Every public class, method, function, and service module requires a docstring.
- Use a consistent style (Google or NumPy) across the entire codebase — don't mix.
- Docstrings must cover: purpose, parameters, return value, exceptions raised, and any side effects (DB writes, external API calls, cache invalidation, emitted events).
- Private/internal helper functions may use shorter one-line docstrings but should not be skipped entirely.

```python
def calculate_partner_commission(lead_value: Decimal, commission_rate: Decimal) -> Decimal:
    """
    Calculate the commission owed to a partner for a converted lead.

    Args:
        lead_value: Monetary value of the converted lead, in base currency.
        commission_rate: Partner's commission rate as a decimal fraction (e.g., 0.10 for 10%).

    Returns:
        Commission amount rounded to two decimal places.

    Raises:
        ValueError: If lead_value or commission_rate is negative.
    """
```

## 4. File Organization and Separation of Concerns

Structure should make it obvious where new logic belongs without discussion.

- Standard layered structure per module/domain:

```text
<module>/
    controllers/    # Route handlers, request/response mapping only
    services/       # Business logic, orchestration
    repositories/   # Data access, query logic
    schemas/        # Pydantic/DTO request-response models
    models/         # ORM/database models
    constants.py
    exceptions.py
```

- Controllers must not contain business logic — they delegate to services.
- Services must not contain raw SQL/ORM queries — they delegate to repositories.
- No file should exceed a reasonable single-responsibility scope; if a service file handles more than one clear domain concern (e.g., commission calculation AND partner notification), split it.
- Shared cross-module logic belongs in a `common/` or `shared/` package, never duplicated per module.

## 5. Minimize Code Duplication & Use Common Components

- **Always use common components for common works.**
- Duplicate logic (even 3+ line blocks repeated across files) must be extracted into a shared function, service, or base class.
- Similar query patterns (e.g., paginated fetch, soft-delete filtering) should be consolidated into repository base classes or query builders.
- Copy-paste across MakunAI and CollegeHai for genuinely shared logic (e.g., currency formatting, forex conversion) should live in a shared internal package, not be reimplemented per project.
- Any duplication that survives review must include an inline comment justifying why abstraction wasn't appropriate.

## 6. Module-Level Constants & No Hardcoded Values

- **Never compare static values directly.**
- All magic numbers, strings, timeouts, static flags, and thresholds must live in `<module>/constants.py` and be imported from there.
- Constants shared across multiple modules belong in an application-level `shared/constants.py`, clearly separated from module-local ones.
- Use `Final` typing for constants to prevent accidental reassignment.
- Environment-dependent values (API keys, URLs) belong in config/settings management, never in `constants.py`.

```python
# voice_agent/constants.py
from typing import Final

MAX_SPEECH_LATENCY_MS: Final[int] = 800
DEFAULT_VOICE_MODEL: Final[str] = "eleven-labs-turbo"
SESSION_TIMEOUT_SECONDS: Final[int] = 300
```

## 7. Async and Concurrency Standards

Relevant specifically to your async-first FastAPI/AsyncIO stack.

- Never mix blocking I/O (`requests`, synchronous DB drivers) inside `async def` functions — use `httpx`, async SQLAlchemy, or Motor consistently.
- All background tasks (Celery, asyncio tasks) must have explicit error handling and retry policy documented.
- Use `asyncio.gather` with `return_exceptions=True` deliberately, not by default — silent exception swallowing must be justified.
- Streaming responses (voice, chat) must have explicit backpressure or timeout handling to prevent resource leaks.

## 8. Error Handling and Observability

- Never use bare `except:` — always catch specific exception types.
- Define domain-specific exceptions in `<module>/exceptions.py` rather than raising generic `Exception`.
- Every raised exception in a service layer should be caught and translated to a proper HTTP/response error at the controller layer — no leaking of internal stack traces to clients.
- All external API calls (Razorpay, WhatsApp, Gemini, LiveKit) must have structured logging on failure, including request ID for traceability.
- Use structured logging (JSON logs) with consistent fields: `timestamp`, `service`, `request_id`, `severity`, `message`.

## 9. Testing Standards

- Every service-layer function must have corresponding unit tests covering the happy path, at least one edge case, and one failure case.
- Repository layer tests should use a test database/fixtures, never mocks that hide real query bugs.
- Integration tests are mandatory for any external integration (payment, WhatsApp, calendar) with recorded/mocked responses for CI reliability.
- Target minimum coverage thresholds should be enforced in CI (e.g., 80%+ for services layer).

## 10. Security and Data Handling

- No secrets, API keys, or credentials in code or version control — use environment variables or a secrets manager.
- All user-facing input must be validated at the schema layer (Pydantic/DRF serializers) before reaching business logic.
- PII and financial data (commission records, forex values) must be logged only in redacted form.
- Role-based access checks must happen at the controller layer before any service invocation, never assumed implicitly.

## 11. API Design Consistency

- All endpoints must be documented via OpenAPI/Swagger with request/response schemas explicitly typed.
- Versioning strategy (`/v1/`, `/v2/`) must be applied consistently across all exposed APIs.
- Error responses must follow a single standardized shape (`error_code`, `message`, `details`) across the entire platform.

## 12. Function Length & Maintainability Priorities

- **Do not make a function very heavy or very long.** If a function is getting long, it must be broken down into smaller, highly maintainable private helper functions.
- Every implementation decision, when in conflict, should be resolved in this order:
  1. Readability
  2. Maintainability
  3. Testability
  4. Scalability
  5. Reusability
  6. Performance micro-optimization (only after the above are satisfied)

A new engineer should be able to understand any module's purpose within minutes, without a walkthrough.

## 13. Pre-Submission Review Checklist

- [ ] All functions have complete type hints, including return types.
- [ ] All classes, methods, and modules have meaningful docstrings.
- [ ] Naming is descriptive and self-explanatory throughout.
- [ ] No unjustified duplication exists, common components are utilized.
- [ ] Files follow single-responsibility and layered architecture.
- [ ] Constants are defined in `constants.py`, no hardcoded or static values used directly.
- [ ] Functions are kept short and broken into smaller parts for maintainability.
- [ ] No blocking calls inside async functions.
- [ ] Errors are caught with specific exception types and logged with context.
- [ ] Unit and integration tests cover new logic, including failure paths.
- [ ] No secrets or credentials committed.
- [ ] API endpoints are documented and follow versioning conventions.
- [ ] Code is easy to read, test, and maintain by someone unfamiliar with it.
