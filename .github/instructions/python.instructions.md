---
description: 'Guidelines for building Python backend, service, and LLM applications'
applyTo: ["src/**/*.py", "apps/**/*.py", "services/**/*.py", "api/**/*.py", "workers/**/*.py"]
---

# Python Applications - Copilot Instructions

## Project Overview

This document provides implementation rules for production Python systems, including:

- REST APIs built with FastAPI or Flask
- Django applications and internal platforms
- Background workers and service processes
- GenAI and agentic workflows built with LangChain or LangGraph

All Python code must target **Python 3.11+**, use **strict typing**, and follow a clean separation of concerns so the same architecture scales from a small microservice to a multi-agent orchestration platform.

---

## Environment & Dependency Control

### Runtime Baseline

- Python version must be **3.11 or newer**
- Do not generate code for Python 3.10 or older unless explicitly required
- Prefer modern typing syntax:
  - `list[str]` instead of `List[str]`
  - `str | None` instead of `Optional[str]`
  - `typing.Self`, `typing.TypedDict`, `typing.Protocol`, and `typing.Annotated` where appropriate

### Dependency Management Rules

Use a modern dependency manager. Prefer:

1. **uv** for fast installs, lockfile management, and isolated execution
2. **Poetry** when the project standard already uses Poetry
3. Do **not** default to raw `pip install` workflows for managed applications

**Preferred commands with uv**
```bash
uv sync
uv add fastapi pydantic-settings sqlalchemy
uv run pytest
uv run ruff check .
uv run ruff format .
uv run mypy .
```

**Preferred commands with Poetry**
```bash
poetry install
poetry add fastapi pydantic-settings sqlalchemy
poetry run pytest
poetry run ruff check .
poetry run ruff format .
poetry run mypy .
```

### Dependency Rules

- Commit lockfiles
- Separate runtime and development dependencies
- Pin major versions for frameworks and infra-sensitive packages
- Avoid unbounded upgrades in shared platforms
- Do not add libraries when the standard library or existing stack already solves the problem

### Configuration & Secrets

Use **Pydantic Settings** for all runtime configuration.

- All environment access must flow through a typed settings object
- Do not scatter `os.getenv()` calls throughout the codebase
- Never hardcode secrets, tokens, API keys, or connection strings
- Support environment-specific overrides through `.env`, deployment variables, or secret stores

```python
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    app_name: str = "python-service"
    environment: str = "local"
    log_level: str = "INFO"
    database_url: str
    openai_api_key: SecretStr | None = None
```

### Configuration Rules

- Instantiate settings once at process startup
- Inject settings into services instead of reading globals directly
- Validate required configuration at startup, not lazily during request handling
- Treat configuration models as part of the application contract

> **Required:** Every deployable Python service must have a typed settings module.

---

## Universal Project Architecture

### Standard Layout

Use a structure that separates HTTP concerns, domain logic, persistence, orchestration, and infrastructure.

```text
project-root/
|-- pyproject.toml
|-- uv.lock or poetry.lock
|-- README.md
|-- .env.example
|-- docker/
|   `-- Dockerfile
|-- migrations/
|-- scripts/
|-- tests/
|   |-- unit/
|   |-- integration/
|   |-- contract/
|   `-- conftest.py
`-- src/
    `-- app/
        |-- main.py                 # process entrypoint
        |-- core/
        |   |-- config.py           # Pydantic settings
        |   |-- logging.py          # structured logging setup
        |   |-- security.py         # auth helpers, hashing, JWT utils
        |   `-- exceptions.py       # shared exception types
        |-- api/
        |   |-- routers/            # FastAPI or Flask route modules
        |   |-- dependencies.py     # DI providers / request-scoped wiring
        |   |-- middleware.py
        |   `-- error_handlers.py
        |-- schemas/
        |   |-- requests/           # input validation models
        |   |-- responses/          # API response models
        |   `-- domain/             # typed internal contracts
        |-- services/
        |   |-- domain/             # business logic
        |   |-- agents/             # LLM agents / graph nodes
        |   `-- workflows/          # orchestration layer
        |-- repositories/
        |   |-- interfaces.py
        |   `-- sql/
        |-- models/
        |   |-- orm/                # SQLAlchemy or Django ORM models
        |   `-- enums.py
        |-- db/
        |   |-- session.py
        |   |-- base.py
        |   `-- migrations/
        |-- clients/
        |   |-- http/
        |   |-- queue/
        |   `-- llm/
        `-- observability/
            |-- tracing.py
            |-- metrics.py
            `-- audit.py
```

### Layer Responsibilities

**API Layer**
- Owns routing, request parsing, response serialization, auth enforcement, and HTTP status mapping
- Must remain thin
- Must not contain business rules, ORM query logic, or prompt construction

**Service Layer**
- Owns business workflows and use-case orchestration
- Coordinates repositories, external clients, and agents
- Returns typed domain results, not framework-specific response objects

**Repository Layer**
- Owns persistence queries and database mutation logic
- Hides ORM details from services
- Must not leak raw database concerns into routers

**Schema Layer**
- Owns input and output validation contracts
- Separates external API models from internal domain state when needed

**Agent / Workflow Layer**
- Owns LLM prompt orchestration, graph transitions, tool invocation, and state movement
- Must use explicit state models
- Must not couple orchestration logic to transport concerns

### Scaling Guidance

**For a simple microservice**
- `services/` may contain only one or two domain services
- `repositories/` may be minimal or absent for stateless integrations
- `workflows/` may not exist

**For a larger platform or LLM system**
- Split `services/domain`, `services/agents`, and `services/workflows`
- Add dedicated `clients/llm` and `observability/`
- Promote graph state and tool contracts into first-class schema modules

> **Rule:** Start with the full separation of concerns even if some folders are initially thin. This prevents rewrites when the service grows.

---

## Automated Quality & Linting

### Required Tooling

Use:

- **Ruff** for linting, formatting, import ordering, and general static rules
- **Mypy** for strict type checking
- **Pytest** for testing

Do not introduce overlapping tools unless there is a clear gap.

- Prefer Ruff over separate Black, Flake8, and isort setups
- Keep formatting deterministic and enforced in CI

### Ruff Rules

- Maximum line length: **88**
- Enable import sorting
- Enforce unused import and dead-code detection
- Enforce modern Python syntax when safe
- Fail CI on lint violations

**Recommended pyproject configuration**
```toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "N", "RUF"]
ignore = []

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### Mypy Rules

- Type hints are required for all public functions, methods, and service boundaries
- New modules should pass mypy with no errors
- Use `TypedDict`, `Protocol`, `Literal`, and generics when they improve contract safety
- Avoid `Any`; use `object` or explicit narrowing instead
- Database, HTTP, and LLM client wrappers must expose typed interfaces

**Recommended mypy configuration**
```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_unused_ignores = true
warn_return_any = true
disallow_any_generics = true
no_implicit_optional = true
```

### Naming & Style Rules

| Construct | Convention | Example |
|-----------|-----------|---------|
| Modules | `snake_case` | `user_service.py` |
| Packages | `snake_case` | `repositories` |
| Classes | `PascalCase` | `PaymentService` |
| Functions / Variables | `snake_case` | `build_response` |
| Constants | `UPPER_SNAKE_CASE` | `DEFAULT_TIMEOUT_SECONDS` |
| Private helpers | leading underscore | `_parse_payload` |
| Async functions | verb-based names | `fetch_customer` |

### Coding Standards

- Prefer explicit return types
- Prefer small, composable functions
- Do not hide side effects in utility functions
- Do not use mutable default arguments
- Do not use bare `except:`
- Do not use synchronous blocking calls inside async code paths
- Do not use framework globals as hidden dependencies when explicit injection is possible

---

## Framework Interoperability

### FastAPI and Flask

#### FastAPI Rules

- Use async endpoints for I/O-bound work
- Keep routers thin and delegate to services
- Use `Depends()` for dependency injection
- Validate request and response payloads with Pydantic models
- Use lifespan hooks for startup and shutdown resources
- Return typed response models for public APIs

```python
from typing import Annotated, Protocol

from fastapi import APIRouter, Depends
from pydantic import BaseModel

router = APIRouter()


class HealthResponse(BaseModel):
    status: str

class HealthService(Protocol):
    async def ping(self) -> None: ...

class DefaultHealthService:
    async def ping(self) -> None:
        return None

def get_health_service() -> HealthService:
    return DefaultHealthService()

@router.get("/health", response_model=HealthResponse)
async def get_health(
    service: Annotated[HealthService, Depends(get_health_service)],
) -> HealthResponse:
    await service.ping()
    return HealthResponse(status="ok")
```

#### Flask Rules

- Flask is acceptable for simpler services, admin utilities, or legacy-aligned integrations
- Use application factories
- Keep route handlers thin
- Move business logic into services
- Do not let blueprints become service layers
- For I/O-heavy workloads, document the concurrency model clearly and avoid pretending synchronous handlers are async-safe

#### Async / Await Rules

- Use `async def` only when the framework, driver, and call chain are actually async
- Use async database and HTTP clients when operating in async applications
- Never call blocking SDKs directly from async request handlers without offloading
- Avoid mixing sync SQLAlchemy sessions into async service paths

### Django

Use a **fat models, thin views** approach.

- Domain rules belong in models, managers, querysets, or dedicated service modules
- Views should orchestrate request parsing, auth, and response rendering only
- Use Django Forms, DRF serializers, or Pydantic adapters consistently based on the stack
- Validate every migration before merge
- Review data migrations for reversibility, locking risk, and production runtime impact

#### Django Rules

- Keep ORM logic out of templates and views
- Prefer `select_related` and `prefetch_related` for predictable query behavior
- Use custom managers or querysets for reusable query patterns
- Do not embed complex business rules in signals unless unavoidable
- Add migration checks in CI:
  - `makemigrations --check`
  - `migrate --plan`
  - integration tests against a real test database

### GenAI / LLMs: LangChain and LangGraph

#### State Management Rules

- Graph and agent state must be explicit and typed
- Use `TypedDict`, Pydantic models, or dataclasses for state contracts
- Do not pass opaque dictionaries through complex workflows without schema guarantees

```python
from typing import TypedDict


class ResearchState(TypedDict):
    query: str
    sources: list[str]
    summary: str | None
```

#### Workflow Rules

- Separate prompt templates, tool definitions, graph nodes, and output parsers
- Make tool input contracts explicit
- Normalize model outputs before business logic consumes them
- Design workflows so every node has deterministic inputs and typed outputs
- Persist trace identifiers across node execution

#### Streaming and Observability Rules

- Prefer event streaming for long-running agent workflows
- Emit structured events for:
  - graph start
  - node transition
  - tool invocation
  - model completion
  - retry
  - failure
- Integrate **LangSmith** or equivalent tracing for prompt and execution analytics
- Log latency, token usage, tool errors, and fallback paths

> **Rule:** LLM workflows are production workflows. They require the same typed contracts, observability, and failure handling as any other distributed system.

---

## Robust Testing & Error Frameworks

### Testing Strategy

Use **Pytest** as the default test framework.

Structure tests by scope:

- `tests/unit/` for isolated business logic
- `tests/integration/` for database and infrastructure interaction
- `tests/contract/` for API or schema-level guarantees

### Test Rules

- Use fixtures for setup reuse
- Use async fixtures for async services and clients
- Mock external endpoints, queues, LLM providers, and cloud SDKs in unit tests
- Do not call live external services in the default test suite
- Use factories or builders for test data generation
- Keep tests deterministic and parallel-safe

```python
import pytest


@pytest.fixture
async def user_service() -> object:
    return ...
```

### Isolation Rules

- Unit tests must not depend on the network
- Unit tests must not require shared environment state
- Integration tests must manage setup and teardown explicitly
- Database tests should run against disposable databases or transaction rollbacks
- LLM workflow tests should validate graph behavior with mocked model outputs before any live-eval layer is introduced

### Error Handling Framework

Create a predictable exception hierarchy.

```python
class AppError(Exception):
    """Base application exception."""


class ValidationError(AppError):
    """Raised for domain validation failures."""


class ExternalDependencyError(AppError):
    """Raised when an upstream dependency fails."""
```

### Exception Mapping Rules

- Domain exceptions map to clear, stable API responses
- Validation failures return `400` or `422`
- Missing resources return `404`
- Upstream dependency failures return `502` or `503`
- Unhandled failures return `500` with sanitized payloads
- Never leak stack traces, secrets, SQL, or prompt internals to clients

**FastAPI example**
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()


@app.exception_handler(ValidationError)
async def handle_validation_error(
    request: Request, exc: ValidationError
) -> JSONResponse:
    return JSONResponse(status_code=422, content={"detail": str(exc)})
```

### Testing Error Paths

Every service should test:

- happy path
- invalid input path
- external dependency failure path
- not-found or empty-result path
- timeout or retry-exhaustion behavior when relevant

---

## Containerization & Observability

### Docker Rules

Use multi-stage builds with minimal runtime images.

- Prefer `python:3.11-slim` or newer slim variants
- Run as a non-root user
- Install only runtime dependencies in the final stage
- Keep build tools out of the runtime image
- Use `.dockerignore`
- Do not copy the entire repository blindly if a smaller context works

```dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

FROM python:3.11-slim AS runtime

RUN useradd -r -u 1001 appuser
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY src ./src

ENV PATH="/app/.venv/bin:$PATH"
USER appuser

CMD ["python", "-m", "src.app.main"]
```

### Runtime Container Rules

- Use explicit health checks where the platform supports them
- Keep images small and reproducible
- Fail fast on invalid configuration
- Do not write persistent application state to the container filesystem

### Logging Rules

Use structured JSON logging for all deployed services.

- Logs must be machine-parseable
- Include correlation IDs, request IDs, trace IDs, environment, service name, and severity
- Emit timestamps in UTC
- Use stdout/stderr for container-native logging
- Avoid ad hoc string concatenation in logs

**Required log fields**
- `timestamp`
- `level`
- `service`
- `environment`
- `message`
- `trace_id`
- `request_id` when applicable

### Observability Rules

- Emit metrics for request counts, latency, error rates, retries, and queue depth where relevant
- Emit traces for API boundaries, database operations, outbound HTTP calls, and LLM/tool orchestration
- Integrate cleanly with Datadog, ELK, OpenTelemetry, or the platform standard
- Do not log secrets, tokens, raw credentials, or protected personal data

### LLM-Specific Observability

For LangChain and LangGraph workloads, additionally record:

- model name
- token usage
- prompt version
- tool selection
- graph transition timing
- fallback or retry activation
- final outcome classification

---

## Implementation Checklist

When generating Python code in this repository, always verify the following:

1. The code targets Python 3.11+
2. Dependencies are managed by uv or Poetry, not ad hoc pip usage
3. Runtime config is typed with Pydantic Settings
4. Routers, services, repositories, and schemas are separated cleanly
5. Ruff and mypy compatibility are preserved
6. Async code only uses non-blocking dependencies
7. Django changes include migration validation
8. LLM workflows use typed state and structured tracing
9. Tests isolate external systems and cover error paths
10. Containers run with slim, non-root, production-safe defaults
11. Logging is structured and cloud-ingestible
12. No secrets or environment-specific constants are hardcoded

---

## General Rules for Copilot

✔ Generate typed, production-grade Python code
✔ Keep transport, business logic, and persistence concerns separate
✔ Prefer explicit contracts over implicit dictionaries and globals
✔ Use structured logging, deterministic tests, and validated configuration
✔ Design code so it can scale from a single API to a workflow-driven platform

❌ Do not generate untyped service boundaries
❌ Do not place business logic in routers, views, or controllers
❌ Do not scatter environment variable reads across modules
❌ Do not mix blocking I/O into async request paths
❌ Do not return raw exceptions or framework-default error payloads in production
❌ Do not treat LLM code as exempt from architecture, testing, or observability standards