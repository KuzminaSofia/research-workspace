# Backend Skeleton Plan

План первой implementation-итерации backend. Цель - начать реализацию без архитектурного дрейфа и не смешать domain rules с HTTP/DB.

Документ опирается на:

- `docs/architecture/02-system-architecture.md`;
- `docs/architecture/03-backend-layered-architecture.md`;
- `docs/architecture/04-domain-model.md`;
- `docs/backend/api-contract-v1.md`.

## 1. Scope Skeleton

В первую итерацию входят:

- package layout по слоям;
- domain entities/enums/policies для MVP;
- первая migration;
- repositories и Unit of Work;
- Workspace CRUD;
- ResearchTask CRUD;
- domain tests до API implementation.

Не входят в первую итерацию:

- agent-service, queue и reconciler;
- artifact storage;
- Source/Evidence/Claim/Decision/ResearchRun endpoints;
- auth provider и полноценный RBAC;
- pagination;
- export decision brief.

Эти части проектируются в `api-contract-v1.md`, но реализуются следующими итерациями.

## 2. Package Layout

```text
backend/
  pyproject.toml
  alembic.ini
  migrations/
    env.py
    versions/
      0001_initial.py
  src/research_workspace/
    __init__.py
    core/
      __init__.py
      config.py
      logging.py
      security.py
    domain/
      __init__.py
      enums.py
      value_objects.py
      exceptions.py
      entities/
        __init__.py
        workspace.py
        research_task.py
        source.py
        evidence_item.py
        claim.py
        decision.py
        research_run.py
        update.py
        artifact_manifest.py
      policies/
        __init__.py
        claim_review.py
        decision_acceptance.py
        evidence_review.py
        scope_integrity.py
        research_task_lifecycle.py
        research_run_lifecycle.py
        agent_boundary.py
    application/
      __init__.py
      ports/
        __init__.py
        repositories.py
        unit_of_work.py
        clock.py
        id_generator.py
      dto/
        __init__.py
        workspace.py
        research_task.py
      use_cases/
        __init__.py
        workspace/
          __init__.py
          create_workspace.py
          list_workspaces.py
          get_workspace.py
          update_workspace.py
        research_task/
          __init__.py
          create_research_task.py
          list_research_tasks.py
          get_research_task.py
          update_research_task.py
    infrastructure/
      __init__.py
      db/
        __init__.py
        base.py
        models.py
        session.py
      repositories/
        __init__.py
        workspace.py
        research_task.py
      uow.py
    api/
      __init__.py
      main.py
      deps.py
      errors.py
      schemas/
        __init__.py
        common.py
        workspace.py
        research_task.py
      routers/
        __init__.py
        workspaces.py
        research_tasks.py
  tests/
    conftest.py
    domain/
      test_workspace.py
      test_research_task.py
      test_scope_integrity.py
      test_claim_review.py
      test_decision_acceptance.py
      test_evidence_review.py
      test_research_run_lifecycle.py
    application/
      test_workspace_use_cases.py
      test_research_task_use_cases.py
    infrastructure/
      test_repositories.py
      test_uow.py
      test_constraints.py
    api/
      test_workspaces_api.py
      test_research_tasks_api.py
```

## 3. Layer Rules

### Domain

- Содержит entities, value objects, enums, policies и domain exceptions.
- Не импортирует FastAPI, SQLAlchemy, Pydantic, queue/storage-клиенты.
- Не знает про транзакции, HTTP-статусы и JSON.
- Domain entity меняет состояние только через методы, которые проверяют invariants.
- Domain exception не содержит HTTP-кода.

### Application

- Содержит use cases, DTO и ports.
- Зависит от domain и абстрактных ports.
- Управляет Unit of Work: одна транзакция на один write use case.
- Не содержит SQL и HTTP-кода.
- Принимает `Actor` и `Clock` как зависимости, а не читает глобальный контекст.

### Infrastructure

- Реализует ports: repositories, Unit of Work, clock, id generator.
- Содержит SQLAlchemy models и Alembic migrations.
- Маппит ORM models в domain entities и обратно.
- Не содержит business rules.

### API

- Содержит FastAPI routers, Pydantic schemas, dependencies и error mapping.
- Валидирует только transport-level input: типы, длины, enum values, обязательность полей.
- Вызывает use case и маппит результат в response schema.
- Не содержит SQL, транзакций и domain rules.
- Actor берётся из `api/deps.py` и передаётся в use case.

### Dependency direction

```text
api -> application -> domain
                 ^
                 |
          infrastructure (ports implementation)
```

## 4. First Domain Model

### Enums

`domain/enums.py`:

```python
class ResearchTaskStatus(str, Enum):
    DRAFT = "draft"
    IN_PROGRESS = "in_progress"
    REVIEW = "review"
    COMPLETED = "completed"
    ARCHIVED = "archived"


class SourceType(str, Enum):
    URL = "url"
    MANUAL_NOTE = "manual_note"
    PDF = "pdf"
    DOC = "doc"
    GITHUB = "github"
    VIDEO = "video"


class EvidenceReviewStatus(str, Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"


class ClaimStatus(str, Enum):
    DRAFT = "draft"
    SUPPORTED = "supported"
    WEAK = "weak"
    CONFLICTING = "conflicting"
    OUTDATED = "outdated"


class DecisionStatus(str, Enum):
    PROPOSED = "proposed"
    ACCEPTED = "accepted"
    REJECTED = "rejected"
    SUPERSEDED = "superseded"
    NEEDS_REVIEW = "needs_review"


class ResearchRunStatus(str, Enum):
    QUEUED = "queued"
    RUNNING = "running"
    REVIEW_REQUIRED = "review_required"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"


class ResearchRunMode(str, Enum):
    MANUAL_SOURCES = "manual_sources"
    SEARCH_ASSISTED = "search_assisted"
    FOLLOW_UP = "follow_up"


class UpdateType(str, Enum):
    SOURCE_CHANGED = "source_changed"
    NEW_SOURCE = "new_source"
    CLAIM_REVIEW = "claim_review"
    DECISION_REVIEW = "decision_review"
    MANUAL_NOTE = "manual_note"
```

### Entities

В skeleton реализуются все MVP entities в domain, но БД и API в первой итерации поднимаются только для `Workspace` и `ResearchTask`:

- `Workspace(id, title, description, owner_id, created_at, updated_at)`;
- `ResearchTask(id, workspace_id, title, research_question, status, summary, created_by, created_at, updated_at)`;
- `Source`, `EvidenceItem`, `Claim`, `Decision`, `ResearchRun`, `Update` - как domain-модели без ORM-таблиц и endpoints.

Правила для `Workspace` и `ResearchTask`:

- `title` и `research_question` обрезаются по краям и не могут быть пустыми;
- `title` не длиннее 200 символов;
- новый `ResearchTask` всегда создаётся в `draft`;
- `archived` - terminal state;
- доменные методы: `Workspace.rename()`, `Workspace.change_description()`, `ResearchTask.change_status()`, `ResearchTask.update_summary()`.

Остальные entities получают ORM-модели и endpoints в следующих итерациях. Enums, exceptions и policies создаются сразу, чтобы API contract не расходился с кодом.

### Policies

- `ClaimReviewPolicy` - проверяет, что `supported` claim имеет evidence и reviewed_by.
- `DecisionAcceptancePolicy` - проверяет reviewed claim или explicit override.
- `EvidenceReviewPolicy` - проверяет переход `pending -> approved | rejected`.
- `ScopeIntegrityPolicy` - проверяет workspace/research task связи.
- `ResearchTaskLifecyclePolicy` - проверяет переходы ResearchTask.
- `ResearchRunLifecyclePolicy` - проверяет переходы run и terminal states.
- `AgentBoundaryPolicy` - запрещает agent service переводить claim в `supported` и decision в `accepted`.

Policies принимают domain entities и возвращают domain exceptions. Они не ходят в БД; необходимые данные передаются в policy явно.

## 5. Initial Migration

### Общие правила схемы

- PostgreSQL.
- Primary keys: `uuid`, default `gen_random_uuid()`.
- Timestamps: `timestamptz`, значения в UTC.
- Enum-поля: `varchar` + `CHECK`, а не PostgreSQL `ENUM`. Это упрощает добавление значений в миграциях.
- `confidence`: `numeric(4,3)` с `CHECK (confidence >= 0 AND confidence <= 1)`.
- `owner_id`/actor fields: `uuid` без FK на users. Источник identity - auth provider; users-проекция появится позже.
- Для `sources` и `claims` используется composite FK на `research_tasks(id, workspace_id)`, чтобы БД не приняла сущность из чужого workspace.

### `workspaces`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK, default `gen_random_uuid()` |
| `title` | `varchar(200)` | not null, `CHECK (length(btrim(title)) > 0)` |
| `description` | `text` | null |
| `owner_id` | `uuid` | not null |
| `created_at` | `timestamptz` | not null |
| `updated_at` | `timestamptz` | not null |

Index: `(owner_id, created_at DESC)`.

### `research_tasks`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK |
| `workspace_id` | `uuid` | FK `workspaces(id)` on delete cascade, not null |
| `title` | `varchar(200)` | not null, non-empty |
| `research_question` | `text` | not null, non-empty |
| `status` | `varchar(32)` | not null, `CHECK` по `ResearchTaskStatus` |
| `summary` | `text` | null |
| `created_by` | `uuid` | not null |
| `created_at` | `timestamptz` | not null |
| `updated_at` | `timestamptz` | not null |

Constraints/indexes:

- `UNIQUE (id, workspace_id)` - нужен для composite FK из `sources`/`claims`;
- index `(workspace_id, status)`;
- index `(workspace_id, created_at DESC)`.

### `sources`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK |
| `workspace_id` | `uuid` | not null |
| `research_task_id` | `uuid` | not null |
| `type` | `varchar(32)` | not null, `CHECK` по `SourceType` |
| `title` | `varchar(200)` | not null |
| `url` | `text` | null |
| `author` | `varchar(200)` | null |
| `published_at` | `timestamptz` | null |
| `captured_at` | `timestamptz` | not null |
| `summary` | `text` | null |
| `raw_content_ref` | `text` | null |
| `reliability_note` | `text` | null |

Constraints:

- FK `(research_task_id, workspace_id)` -> `research_tasks(id, workspace_id)` on delete cascade;
- `CHECK (type NOT IN ('url', 'github', 'video') OR url IS NOT NULL)`;
- index `(research_task_id, captured_at DESC)`.

### `evidence_items`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK |
| `source_id` | `uuid` | FK `sources(id)` on delete cascade, not null |
| `excerpt` | `text` | not null, non-empty |
| `location` | `text` | not null |
| `note` | `text` | null |
| `confidence` | `numeric(4,3)` | not null, `0..1` |
| `status` | `varchar(16)` | not null, default `pending`, `CHECK` по `EvidenceReviewStatus` |
| `reviewed_by` | `uuid` | null |
| `reviewed_at` | `timestamptz` | null |
| `created_at` | `timestamptz` | not null |

Constraints:

- `CHECK (status = 'pending' OR (reviewed_by IS NOT NULL AND reviewed_at IS NOT NULL))`;
- index `(source_id, created_at)`.

### `claims`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK |
| `workspace_id` | `uuid` | not null |
| `research_task_id` | `uuid` | not null |
| `text` | `text` | not null, non-empty |
| `status` | `varchar(32)` | not null, `CHECK` по `ClaimStatus` |
| `confidence` | `numeric(4,3)` | null, `0..1` |
| `created_by` | `uuid` | not null |
| `reviewed_by` | `uuid` | null |
| `review_note` | `text` | null |
| `created_at` | `timestamptz` | not null |
| `updated_at` | `timestamptz` | not null |

Constraints:

- FK `(research_task_id, workspace_id)` -> `research_tasks(id, workspace_id)` on delete cascade;
- `UNIQUE (id, workspace_id)` - для будущих composite FK из decisions;
- index `(research_task_id, status)`;
- index `(workspace_id, updated_at DESC)`.

### `claim_evidence`

| Column | Type | Constraints |
| --- | --- | --- |
| `claim_id` | `uuid` | FK `claims(id)` on delete cascade |
| `evidence_id` | `uuid` | FK `evidence_items(id)` on delete cascade |
| `created_at` | `timestamptz` | not null |

Constraints/indexes:

- PK `(claim_id, evidence_id)`;
- index `(evidence_id)`.

Scope Claim -> Evidence проверяется в `ScopeIntegrityPolicy`: evidence через source должен принадлежать тому же research task и workspace.

### `decisions`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK |
| `workspace_id` | `uuid` | FK `workspaces(id)` on delete cascade, not null |
| `title` | `varchar(200)` | not null, non-empty |
| `decision_text` | `text` | not null, non-empty |
| `status` | `varchar(32)` | not null, `CHECK` по `DecisionStatus` |
| `rationale` | `text` | null |
| `confidence` | `numeric(4,3)` | null, `0..1` |
| `decided_by` | `uuid` | null |
| `decided_at` | `timestamptz` | null |
| `override_reason` | `text` | null |
| `open_questions` | `jsonb` | not null, default `'[]'` |
| `created_at` | `timestamptz` | not null |
| `updated_at` | `timestamptz` | not null |

Constraints/indexes:

- `UNIQUE (id, workspace_id)` - для будущих composite FK;
- index `(workspace_id, status)`;
- index `(workspace_id, created_at DESC)`.

### `decision_claims`

| Column | Type | Constraints |
| --- | --- | --- |
| `decision_id` | `uuid` | FK `decisions(id)` on delete cascade |
| `claim_id` | `uuid` | FK `claims(id)` on delete cascade |
| `created_at` | `timestamptz` | not null |

Constraints/indexes:

- PK `(decision_id, claim_id)`;
- index `(claim_id)`.

Scope Decision -> Claim проверяется в `ScopeIntegrityPolicy`.

### `decision_research_tasks`

| Column | Type | Constraints |
| --- | --- | --- |
| `decision_id` | `uuid` | FK `decisions(id)` on delete cascade |
| `research_task_id` | `uuid` | FK `research_tasks(id)` on delete cascade |
| `created_at` | `timestamptz` | not null |

Constraints/indexes:

- PK `(decision_id, research_task_id)`;
- index `(research_task_id)`.

Scope Decision -> ResearchTask проверяется в `ScopeIntegrityPolicy`.

### `research_runs`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK |
| `workspace_id` | `uuid` | not null |
| `research_task_id` | `uuid` | not null |
| `question` | `text` | not null, non-empty |
| `status` | `varchar(32)` | not null, `CHECK` по `ResearchRunStatus` |
| `mode` | `varchar(32)` | not null, `CHECK` по `ResearchRunMode` |
| `artifact_manifest_ref` | `text` | null |
| `started_by` | `uuid` | not null |
| `started_at` | `timestamptz` | null |
| `completed_at` | `timestamptz` | null |
| `error_message` | `text` | null |

Constraints/indexes:

- FK `(research_task_id, workspace_id)` -> `research_tasks(id, workspace_id)` on delete cascade;
- index `(research_task_id, started_at DESC NULLS LAST)`;
- index `(status)` для reconciler.

`created_at` не добавляется: в domain model его нет. `started_at` заполняется, когда run переходит в `running`.

### `updates`

| Column | Type | Constraints |
| --- | --- | --- |
| `id` | `uuid` | PK |
| `workspace_id` | `uuid` | FK `workspaces(id)` on delete cascade, not null |
| `type` | `varchar(32)` | not null, `CHECK` по `UpdateType` |
| `title` | `varchar(200)` | not null |
| `description` | `text` | null |
| `affected_source_id` | `uuid` | FK `sources(id)` on delete cascade |
| `affected_claim_id` | `uuid` | FK `claims(id)` on delete cascade |
| `affected_decision_id` | `uuid` | FK `decisions(id)` on delete cascade |
| `created_at` | `timestamptz` | not null |

Constraint:

- `CHECK (affected_source_id IS NOT NULL OR affected_claim_id IS NOT NULL OR affected_decision_id IS NOT NULL)`.

### Later migrations

`remote_jobs` (инфраструктурная таблица reconciler) появится вместе с ResearchRun flow:

```text
remote_jobs
  id, run_id, external_job_id, status, attempt,
  idempotency_key, last_polled_at, error_message, created_at, updated_at
```

## 6. Repositories And Unit Of Work

### Ports

`application/ports/repositories.py`:

```python
class WorkspaceRepository(Protocol):
    def add(self, workspace: Workspace) -> None: ...
    def get(self, workspace_id: UUID) -> Workspace | None: ...
    def list_for_owner(self, owner_id: UUID) -> list[Workspace]: ...
    def update(self, workspace: Workspace) -> None: ...


class ResearchTaskRepository(Protocol):
    def add(self, task: ResearchTask) -> None: ...
    def get(self, task_id: UUID) -> ResearchTask | None: ...
    def list_by_workspace(
        self,
        workspace_id: UUID,
        status: ResearchTaskStatus | None = None,
    ) -> list[ResearchTask]: ...
    def update(self, task: ResearchTask) -> None: ...
```

`application/ports/unit_of_work.py`:

```python
class UnitOfWork(Protocol):
    workspaces: WorkspaceRepository
    research_tasks: ResearchTaskRepository

    def __enter__(self) -> "UnitOfWork": ...
    def __exit__(self, exc_type, exc, tb) -> None: ...
    def commit(self) -> None: ...
    def rollback(self) -> None: ...
```

### Unit of Work boundary

- Один write use case = один `UnitOfWork`.
- API не открывает и не коммитит транзакции.
- Read use case использует read-only session и не вызывает `commit()`.
- Repository возвращает domain entities, а не ORM models.
- Маппинг в DTO выполняется внутри UoW, до закрытия session.
- При исключении UoW делает rollback.
- Вложенные транзакции в первой итерации не используются.

### SQLAlchemy implementation

- `infrastructure/db/session.py`:
  - `get_write_session()` - транзакция на write use case;
  - `get_read_session()` - read-only;
  - `get_plain_session()` - для случаев, когда транзакция управляется снаружи.
- `infrastructure/uow.py` - `SqlAlchemyUnitOfWork`.
- `infrastructure/repositories/workspace.py` и `research_task.py` - маппинг ORM <-> domain.

## 7. First Read/Write Flow

### Workspace CRUD

| Use case | Endpoint | Transaction |
| --- | --- | --- |
| `CreateWorkspace` | `POST /workspaces` | write |
| `ListWorkspaces` | `GET /workspaces` | read |
| `GetWorkspace` | `GET /workspaces/{id}` | read |
| `UpdateWorkspace` | `PATCH /workspaces/{id}` | write |

Последовательность `POST /workspaces`:

```text
API schema validation
  -> Actor from dependency
  -> CreateWorkspace.execute(command, actor, clock)
  -> Workspace.create(...)
  -> uow.workspaces.add(workspace)
  -> uow.commit()
  -> WorkspaceDTO
  -> 201 response
```

### ResearchTask CRUD

| Use case | Endpoint | Transaction |
| --- | --- | --- |
| `CreateResearchTask` | `POST /workspaces/{id}/research-tasks` | write |
| `ListResearchTasks` | `GET /workspaces/{id}/research-tasks` | read |
| `GetResearchTask` | `GET /research-tasks/{id}` | read |
| `UpdateResearchTask` | `PATCH /research-tasks/{id}` | write |

Последовательность `PATCH /research-tasks/{id}`:

```text
API schema validation
  -> Actor from dependency
  -> UpdateResearchTask.execute(task_id, command, actor, clock)
  -> uow.research_tasks.get(task_id)
  -> ResearchTask.change_status(new_status, clock)
  -> ResearchTaskLifecyclePolicy / transition table
  -> uow.research_tasks.update(task)
  -> uow.commit()
  -> ResearchTaskDTO
  -> 200 response
```

### Error mapping

- Domain exceptions -> `api/errors.py` -> HTTP status + error envelope.
- `NotFoundError` -> `404 not_found`.
- `ForbiddenError` -> `403 forbidden`.
- `ValidationError` -> `422 validation_error`.
- `InvalidTransitionError` -> `409 invalid_transition`.
- `ScopeMismatchError` -> `422 scope_mismatch`.
- Неизвестное исключение -> `500 internal_error` без утечки stack trace.

## 8. Domain Tests Before API

Domain tests не используют БД, FastAPI и Pydantic.

### Workspace

- `test_workspace_title_is_trimmed`;
- `test_workspace_title_cannot_be_empty`;
- `test_workspace_title_cannot_exceed_200_chars`;
- `test_workspace_owner_is_required`;
- `test_workspace_update_changes_updated_at`.

### ResearchTask

- `test_research_task_starts_as_draft`;
- `test_research_task_question_cannot_be_empty`;
- `test_research_task_title_cannot_be_empty`;
- `test_research_task_allows_forward_transition`;
- `test_research_task_rejects_unknown_transition`;
- `test_research_task_archived_is_terminal`;
- `test_research_task_summary_can_be_updated`.

### Scope Integrity

- `test_claim_cannot_link_evidence_from_another_task`;
- `test_claim_cannot_link_evidence_from_another_workspace`;
- `test_decision_cannot_link_claim_from_another_workspace`;
- `test_decision_cannot_link_research_task_from_another_workspace`;
- `test_same_scope_link_is_allowed`.

### Claim Review

- `test_supported_claim_requires_evidence`;
- `test_supported_claim_requires_reviewer`;
- `test_claim_can_be_reviewed_with_evidence_and_reviewer`;
- `test_draft_claim_clears_review_fields`;
- `test_outdated_claim_emits_review_signal`;
- `test_outdated_claim_does_not_change_decision_status`.

### Decision Acceptance

- `test_decision_can_be_created_without_claims`;
- `test_decision_cannot_be_accepted_without_reviewed_claim`;
- `test_decision_can_be_accepted_with_reviewed_claim`;
- `test_decision_can_be_accepted_with_override_reason`;
- `test_override_requires_non_empty_reason`;
- `test_accept_sets_decided_by_and_decided_at`;
- `test_proposed_decision_keeps_decided_by_null`.

### Evidence Review

- `test_pending_evidence_can_be_approved`;
- `test_pending_evidence_can_be_rejected`;
- `test_approved_evidence_cannot_be_re_reviewed_in_v1`;
- `test_review_sets_reviewed_by_and_reviewed_at`.

### Research Run Lifecycle

- `test_run_starts_as_queued`;
- `test_queued_run_can_start`;
- `test_running_run_can_require_review`;
- `test_review_required_run_can_complete_after_human_review`;
- `test_terminal_run_cannot_be_cancelled`;
- `test_agent_draft_is_not_accepted_knowledge`;
- `test_agent_cannot_mark_claim_supported_or_decision_accepted`.

## 9. Application And Infrastructure Tests

### Application tests с fake UoW

- create workspace;
- list workspaces только для actor-а;
- get missing workspace -> `NotFoundError`;
- update workspace;
- create research task в несуществующем workspace -> `NotFoundError`;
- create research task с чужим workspace -> `ForbiddenError`;
- list research tasks с фильтром по статусу;
- update research task с недопустимым transition -> `InvalidTransitionError`;
- commit вызывается ровно один раз на write use case;
- rollback при исключении.

### Infrastructure tests

- repository roundtrip Workspace/ResearchTask;
- composite FK не позволяет создать source/claim с чужим `workspace_id`;
- unique constraint на `research_tasks(id, workspace_id)`;
- `claim_evidence` PK запрещает duplicate link;
- UoW rollback откатывает изменения;
- read session не видит незакоммиченные изменения.

### API tests

- `401` без токена;
- `422` при неизвестном enum;
- `422 unknown_actor_field` при попытке передать `created_by` в body;
- actor fields в response заполнены из токена;
- list endpoints возвращают `{ "items": [...] }`;
- `404`/`403` для чужих workspace;
- `409 invalid_transition` для запрещённого перехода;
- error envelope соответствует контракту.

## 10. Implementation Order

1. `backend/pyproject.toml`, `core/config.py`, `core/logging.py`.
2. `domain/enums.py`, `domain/exceptions.py`, `domain/value_objects.py`.
3. Все `domain/entities/*` для MVP; первыми - `workspace.py` и `research_task.py`.
4. `domain/policies/*` + domain tests.
5. `infrastructure/db/models.py`, `infrastructure/db/base.py`, `infrastructure/db/session.py`.
6. `migrations/versions/0001_initial.py`.
7. `application/ports/*`, `infrastructure/repositories/*`, `infrastructure/uow.py`.
8. Infrastructure tests для repositories/UoW/constraints.
9. `application/use_cases/workspace/*`, `application/dto/workspace.py`.
10. `api/schemas/workspace.py`, `api/routers/workspaces.py`, `api/deps.py`, `api/errors.py`, `api/main.py`.
11. Workspace API tests.
12. `application/use_cases/research_task/*`, `application/dto/research_task.py`.
13. `api/schemas/research_task.py`, `api/routers/research_tasks.py`.
14. ResearchTask API tests.
15. OpenAPI snapshot check against `api-contract-v1.md` для Workspace/ResearchTask.

## 11. Definition Of Done

- `pytest tests/domain` проходит без PostgreSQL и FastAPI.
- `pytest tests/application` проходит с fake UoW.
- `alembic upgrade head` и `alembic downgrade base` работают на чистой БД.
- Workspace CRUD и ResearchTask CRUD соответствуют `api-contract-v1.md`.
- Все list endpoints возвращают `{ "items": [...] }`.
- Actor fields берутся из authenticated context и не принимаются в body.
- Scope integrity проверяется policy и покрыта тестами.
- API layer не содержит business logic: router только валидирует input, вызывает use case и маппит response.
- Domain не импортирует FastAPI/SQLAlchemy/Pydantic.
- Ошибки маппятся в единый error envelope.

## 12. Risks And Follow-Ups

- Нет users-таблицы: `owner_id`/actor fields хранятся как UUID от auth provider. Если понадобится referential integrity, добавить users-проекцию отдельной миграцией.
- Composite FK защищает `sources`/`claims`, но связи `claim_evidence`, `decision_claims`, `decision_research_tasks` проверяются application-level `ScopeIntegrityPolicy`. Нужны integration tests на cross-scope link.
- Transition tables в `api-contract-v1.md` требуют domain sign-off (OQ-6).
- `Decision.confidence` и `EvidenceReviewStatus` пока contract-level; domain model нужно синхронизировать (OQ-2, OQ-4).
- Pagination не входит в skeleton; list endpoints нужно будет расширить без изменения envelope.
