# Системная Архитектура

Проект строится как backend modular monolith плюс отдельный agent service. Backend владеет продуктовым состоянием и доменными правилами, а agent service выполняет контролируемые research runs и возвращает draft-результаты через явные контракты.

## Компоненты Во Время Работы

```text
Browser
  |
  v
Frontend Web App
  |
  v
Backend API
  |-- PostgreSQL
  |-- Artifact Storage
  |-- Queue
  |
  +--> Agent Service
        |-- LLM Provider
        |-- Search Provider
        |-- Artifact Storage
```

## Проекты В Репозитории

Целевая структура репозитория:

```text
research-workspace/
  frontend/             # Next.js UI
  backend/              # FastAPI modular monolith, product memory, API, DB ownership
  agent-service/        # отдельный service для research orchestration
  packages/
    contracts/          # shared DTO/event/artifact schemas без бизнес-логики
  web-proxy/            # optional nginx/reverse proxy later
  docs/                 # architecture, interns, lectures
  artifacts/            # local dev artifact storage, игнорируется в production-like окружении
  docker-compose.yml
```

`backend` и `agent-service` не должны импортировать внутренности друг друга. Общими могут быть только нейтральные contracts: event schemas, DTOs, artifact manifest schemas, enum values для wire protocol.

## Структура Backend Package

```text
backend/src/research_workspace/
  api/                  # FastAPI routers, schemas, deps
  domain/               # entities, value objects, domain policies
  application/          # use cases and ports
  infrastructure/       # db, storage, queue, search adapters для product memory
  artifacts/            # artifact manifest/contracts/readers/writers
  core/                 # config, logging, security basics
```

## Структура Agent Service Package

```text
agent-service/src/research_agent/
  api/                  # optional health/internal endpoints
  application/          # start/execute research run use cases
  orchestrator/         # bounded research orchestrator
  steps/                # plan, search/import, summarize, extract, draft, brief
  infrastructure/       # llm, search, artifact storage, backend client, queue consumer
  policies/             # budgets, guardrails, tool permissions
  core/                 # config, logging, tracing
```

## Направление Зависимостей

```text
api / cli
        ↓
application use cases
        ↓
domain

application ports
        ↑
infrastructure adapters
```

Разрешенные примеры:

- `api` вызывает `application`.
- `application` использует `domain` и абстрактные ports.
- `infrastructure` реализует ports.
- `domain` не импортирует FastAPI, SQLAlchemy, OpenAI SDK, queue или storage.
- `agent-service` общается с backend через queue/internal API/contracts, а не через import backend modules.

Запрещенные примеры:

- API router напрямую пишет SQL для бизнес-сценария.
- Domain entity вызывает LLM.
- Agent step напрямую пишет backend ORM models.
- Frontend сам выводит domain rules, которые должен обеспечивать backend.
- Agent service напрямую меняет product DB.

## Контракты Backend И Agent Service

Backend -> Agent Service через queue/event:

```text
ResearchRunRequested
  version
  run_id
  workspace_id
  research_task_id
  mode
  requested_by
  idempotency_key
```

Agent Service -> Backend через internal API:

```text
GET  /internal/research-runs/{run_id}/context
POST /internal/research-runs/{run_id}/steps
POST /internal/research-runs/{run_id}/artifacts
POST /internal/research-runs/{run_id}/draft-results
POST /internal/research-runs/{run_id}/fail
```

Правила контракта:

- все сообщения versioned;
- все mutating calls имеют idempotency key;
- internal API защищен service token;
- backend валидирует draft-results перед записью в DB;
- agent service не может переводить claims в `supported` и decisions в `accepted`.

## System Of Record

- PostgreSQL является источником истины для продуктового состояния.
- Artifact storage является audit/export layer для research runs.
- Retrieval index является пересобираемой проекцией.

## Путь Масштабирования

Архитектура оставляет естественные точки роста:

- `artifact storage` можно перенести с local filesystem на S3;
- `search` можно заменить с Postgres full-text на vector/hybrid retrieval;
- `agent-service` можно масштабировать горизонтально;
- `LLM adapter` может поддерживать несколько providers за одним port.
