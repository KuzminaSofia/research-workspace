# Backend API Contract V1

Документ фиксирует transport-level contract для MVP. Источники истины:

- domain model: `docs/architecture/04-domain-model.md`;
- границы слоёв: `docs/architecture/03-backend-layered-architecture.md`;
- черновик B3 и domain rules B4: `docs/backend/b-questions.md`.

API contract не вводит новые domain rules. Если для endpoint нужно правило, которого нет в domain model, оно вынесено в `Open Questions`.

## 1. Conventions

### Base path

```text
/api/v1
```

### Transport

- Request/response: `application/json; charset=utf-8`.
- Export decision brief: `text/markdown; charset=utf-8`.
- Все ID - UUID в строковом виде.
- Все timestamps - ISO 8601 в UTC, например `2026-10-31T12:00:00Z`.
- Nullable-поля всегда присутствуют в response и равны `null`, если значения нет.
- Неизвестные поля в request body запрещены: `422 unknown_field`.

### Authentication

```http
Authorization: Bearer <token>
```

Backend извлекает authenticated actor из токена. Actor не передаётся в request body.

### Collection envelope

Все list endpoints возвращают единый envelope:

```json
{
  "items": []
}
```

Пустая коллекция - это `{ "items": [] }`.

### Error envelope

```json
{
  "error": {
    "code": "scope_mismatch",
    "message": "Evidence belongs to another research task",
    "details": {
      "claim_id": "3f0c...",
      "evidence_id": "8a21..."
    }
  }
}
```

`details` опционален. `message` предназначен для разработчика, а не для показа пользователю как финальный текст.

### HTTP statuses

| Status | Когда используется |
| --- | --- |
| `200 OK` | успешный read/update |
| `201 Created` | создана сущность или link |
| `202 Accepted` | операция принята и продолжится асинхронно (source import) |
| `204 No Content` | успешное удаление связи |
| `401 Unauthorized` | нет или невалиден токен |
| `403 Forbidden` | actor не имеет доступа к workspace |
| `404 Not Found` | сущность не существует |
| `409 Conflict` | нарушен lifecycle/уникальность связи |
| `422 Unprocessable Entity` | validation error или scope mismatch |
| `429 Too Many Requests` | превышен лимит активных research runs |
| `502 Bad Gateway` | внешний сервис недоступен |

## 2. Enums

| Enum | Allowed values | Где используется |
| --- | --- | --- |
| `ResearchTaskStatus` | `draft`, `in_progress`, `review`, `completed`, `archived` | ResearchTask |
| `SourceType` | `url`, `manual_note`, `pdf`, `doc`, `github`, `video` | Source |
| `EvidenceReviewStatus` | `pending`, `approved`, `rejected` | EvidenceItem |
| `ClaimStatus` | `draft`, `supported`, `weak`, `conflicting`, `outdated` | Claim |
| `DecisionStatus` | `proposed`, `accepted`, `rejected`, `superseded`, `needs_review` | Decision |
| `ResearchRunStatus` | `queued`, `running`, `review_required`, `completed`, `failed`, `cancelled` | ResearchRun |
| `ResearchRunMode` | `manual_sources`, `search_assisted`, `follow_up` | ResearchRun |
| `UpdateType` | `source_changed`, `new_source`, `claim_review`, `decision_review`, `manual_note` | Update |

`EvidenceReviewStatus` - contract-level enum, который формализует review-состояние EvidenceItem. Добавление этого поля в `04-domain-model.md` отслеживается в `Open Questions` OQ-4.

`UpdateType` в v1 не имеет публичных endpoints. Записи `Update` создаются backend-ом как audit trail.

## 3. Actor Rules

Клиент не передаёт в request body:

- `created_by`;
- `reviewed_by`;
- `decided_by`;
- `started_by`;
- `owner_id`.

Правила:

1. Backend берёт actor из authenticated context.
2. `owner_id` workspace = actor, создавший workspace.
3. `created_by` research task / source / evidence / claim = actor, создавший сущность.
4. `reviewed_by` evidence / claim = actor, выполнивший review action.
5. `decided_by` decision = actor, который перевёл decision в `accepted` или `rejected`.
6. `started_by` research run = actor, запустивший run.
7. Actor-поля в response доступны только для чтения.
8. Если клиент передал actor-поле в body, API возвращает `422 unknown_actor_field`.
9. Agent service не может вызывать публичные review endpoints и не может переводить claim в `supported` или decision в `accepted`. Это правило действует и для internal API.

## 4. Scope Integrity

Все связи проверяются до записи.

| Связь | Проверка | Ошибка |
| --- | --- | --- |
| Evidence -> Source | source из path существует и доступен actor-у | `404 not_found` |
| Claim -> Evidence | `evidence.source.research_task_id == claim.research_task_id` и `workspace_id` совпадают | `422 scope_mismatch` |
| Decision -> Claim | `claim.workspace_id == decision.workspace_id` | `422 scope_mismatch` |
| Decision -> ResearchTask | `research_task.workspace_id == decision.workspace_id` | `422 scope_mismatch` |

Правила:

- Если сущность не существует или недоступна actor-у, возвращается `404 not_found`.
- Если сущность существует, но принадлежит другому workspace/research task, возвращается `422 scope_mismatch`.
- Scope проверяется как при create, так и при link endpoints.
- Backend не доверяет `workspace_id`/`research_task_id` из body: они выводятся из path и проверяются по БД.

## 5. Common Response Objects

### Workspace

```json
{
  "id": "uuid",
  "title": "LLM Agent Observability",
  "description": "Research workspace",
  "owner_id": "uuid",
  "created_at": "2026-10-31T12:00:00Z",
  "updated_at": "2026-10-31T12:00:00Z"
}
```

### ResearchTask

```json
{
  "id": "uuid",
  "workspace_id": "uuid",
  "title": "Evaluation approaches",
  "research_question": "Which evaluation approach fits production?",
  "status": "draft",
  "summary": null,
  "created_by": "uuid",
  "created_at": "2026-10-31T12:00:00Z",
  "updated_at": "2026-10-31T12:00:00Z"
}
```

### Source

```json
{
  "id": "uuid",
  "workspace_id": "uuid",
  "research_task_id": "uuid",
  "type": "url",
  "title": "Official documentation",
  "url": "https://example.com/docs",
  "author": null,
  "published_at": null,
  "captured_at": "2026-10-31T12:00:00Z",
  "summary": null,
  "raw_content_ref": "artifacts/sources/uuid.txt",
  "reliability_note": null
}
```

`summary` и `raw_content_ref` могут быть `null` сразу после import и заполняются асинхронно. Source при этом уже доступен через GET.

### EvidenceItem

```json
{
  "id": "uuid",
  "source_id": "uuid",
  "excerpt": "Relevant excerpt",
  "location": "section 2.1",
  "note": null,
  "confidence": 0.86,
  "status": "pending",
  "reviewed_by": null,
  "reviewed_at": null,
  "created_at": "2026-10-31T12:00:00Z"
}
```

### Claim

```json
{
  "id": "uuid",
  "workspace_id": "uuid",
  "research_task_id": "uuid",
  "text": "Approach X reduces implementation complexity",
  "status": "draft",
  "confidence": 0.78,
  "created_by": "uuid",
  "reviewed_by": null,
  "review_note": null,
  "evidence_count": 0,
  "created_at": "2026-10-31T12:00:00Z",
  "updated_at": "2026-10-31T12:00:00Z"
}
```

`confidence` - read-only поле. При ручном создании claim через API оно равно `null`; agent draft может заполнить его своим значением. Правило агрегации/дефолта для ручных claims в v1 не вводится.

Claim detail дополнительно содержит:

```json
{
  "evidence": [
    {
      "id": "uuid",
      "source_id": "uuid",
      "source_title": "Official documentation",
      "excerpt": "Relevant excerpt",
      "confidence": 0.86
    }
  ]
}
```

### Decision

```json
{
  "id": "uuid",
  "workspace_id": "uuid",
  "title": "Use trace-based evaluation",
  "decision_text": "Store traces and tool calls in the first release",
  "status": "proposed",
  "rationale": "Reviewed claims support this approach",
  "confidence": null,
  "decided_by": null,
  "decided_at": null,
  "override_reason": null,
  "open_questions": [],
  "claims_count": 0,
  "research_tasks_count": 0,
  "created_at": "2026-10-31T12:00:00Z",
  "updated_at": "2026-10-31T12:00:00Z"
}
```

`confidence` - read-only агрегированное значение. Пока правило агрегации не зафиксировано, поле возвращается как `null` (OQ-2).

Decision detail дополнительно содержит:

```json
{
  "claims": [
    {
      "id": "uuid",
      "text": "Approach X reduces implementation complexity",
      "status": "supported",
      "reviewed_by": "uuid"
    }
  ],
  "research_task_ids": ["uuid"]
}
```

### ResearchRun

```json
{
  "id": "uuid",
  "workspace_id": "uuid",
  "research_task_id": "uuid",
  "question": "Which evaluation approach fits production?",
  "status": "queued",
  "mode": "manual_sources",
  "artifact_manifest_ref": null,
  "started_by": "uuid",
  "started_at": null,
  "completed_at": null,
  "error_message": null
}
```

## 6. Endpoints

### 6.1 Workspaces

#### POST /workspaces

Создать workspace.

Body:

```json
{
  "title": "LLM Agent Observability",
  "description": "Research workspace"
}
```

Response `201`: Workspace.

Errors:

| Status | Code | Условие |
| --- | --- | --- |
| `401` | `unauthenticated` | нет/невалиден токен |
| `422` | `validation_error` | пустой `title`, `title` длиннее 200 символов |

#### GET /workspaces

Список workspaces, доступных actor-у.

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "title": "LLM Agent Observability",
      "description": "Research workspace",
      "owner_id": "uuid",
      "created_at": "2026-10-31T12:00:00Z"
    }
  ]
}
```

Errors: `401 unauthenticated`.

#### GET /workspaces/{workspace_id}

Response `200`: Workspace.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### PATCH /workspaces/{workspace_id}

Body:

```json
{
  "title": "Updated title",
  "description": "Updated description"
}
```

Оба поля опциональны, но хотя бы одно должно быть передано.

Response `200`: Workspace.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `422 validation_error`.

#### POST /workspaces/{workspace_id}/ask

Вопрос по накопленной памяти workspace.

Body:

```json
{
  "question": "What did we decide about evaluation?"
}
```

Response `200`:

```json
{
  "answer_text": "The team decided to use trace-based evaluation.",
  "sources_count": 4,
  "claims_count": 7,
  "decisions_count": 1,
  "memory_sufficient": true,
  "evidence_link": "/api/v1/workspaces/uuid/evidence",
  "claims_link": "/api/v1/workspaces/uuid/claims",
  "sources_link": "/api/v1/workspaces/uuid/sources"
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `422 validation_error`, `502 memory_unavailable`.

### 6.2 Research Tasks

#### POST /workspaces/{workspace_id}/research-tasks

Body:

```json
{
  "title": "Evaluation approaches",
  "research_question": "Which evaluation approach fits production?"
}
```

Response `201`: ResearchTask со `status = draft`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (workspace), `422 validation_error`.

#### GET /workspaces/{workspace_id}/research-tasks

Query:

| Параметр | Тип | Обязателен | Описание |
| --- | --- | --- | --- |
| `status` | `ResearchTaskStatus` | нет | фильтр по статусу |

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "title": "Evaluation approaches",
      "status": "draft",
      "summary": null,
      "created_at": "2026-10-31T12:00:00Z"
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `422 validation_error` (неизвестный `ResearchTaskStatus`).

#### GET /research-tasks/{task_id}

Response `200`: ResearchTask.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### PATCH /research-tasks/{task_id}

Body:

```json
{
  "status": "in_progress",
  "summary": "Intermediate summary"
}
```

Оба поля опциональны. `status` проверяется по transition table.

Response `200`:

```json
{
  "id": "uuid",
  "status": "in_progress",
  "summary": "Intermediate summary",
  "updated_at": "2026-10-31T12:00:00Z"
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 invalid_transition`, `422 validation_error`.

### 6.3 Sources

#### POST /research-tasks/{task_id}/sources

Body:

```json
{
  "type": "url",
  "title": "Official documentation",
  "url": "https://example.com/docs",
  "author": null,
  "published_at": null,
  "summary": "Short summary"
}
```

Validation:

- `type` обязателен и должен быть `SourceType`;
- `title` непустой;
- для `url`, `github`, `video` поле `url` обязательно;
- для `manual_note`, `pdf`, `doc` поле `url` опционально.

Response `201`: Source.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (task), `422 validation_error`.

#### POST /research-tasks/{task_id}/sources/import

Импорт source по URL. Создание source и загрузка содержимого выполняются асинхронно.

Body:

```json
{
  "url": "https://example.com/article"
}
```

Response `202`: Source, у которого `summary = null` и `raw_content_ref = null` до завершения обработки.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `422 invalid_url`, `502 source_fetch_failed`.

#### GET /research-tasks/{task_id}/sources

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "type": "url",
      "title": "Official documentation",
      "url": "https://example.com/docs",
      "summary": "Short summary",
      "reliability_note": null
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### GET /sources/{source_id}

Response `200`: Source.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### GET /workspaces/{workspace_id}/sources

Агрегированный список sources workspace для Workspace Overview и Follow-up Question Area.

Query:

| Параметр | Тип | Обязателен | Описание |
| --- | --- | --- | --- |
| `type` | `SourceType` | нет | фильтр по типу |
| `research_task_id` | `uuid` | нет | фильтр по research task |

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "research_task_id": "uuid",
      "type": "url",
      "title": "Official documentation",
      "url": "https://example.com/docs",
      "summary": "Short summary",
      "reliability_note": null
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (workspace), `422 validation_error`.

### 6.4 Evidence

#### POST /sources/{source_id}/evidence

Body:

```json
{
  "excerpt": "Relevant excerpt from the source",
  "location": "section 2.1",
  "note": "Directly supports the claim",
  "confidence": 0.86
}
```

Response `201`: EvidenceItem со `status = pending`, `reviewed_by = null`, `reviewed_at = null`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (source), `422 validation_error` (пустой `excerpt`, `confidence` вне `[0, 1]`).

#### GET /sources/{source_id}/evidence

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "excerpt": "Relevant excerpt from the source",
      "location": "section 2.1",
      "confidence": 0.86,
      "created_at": "2026-10-31T12:00:00Z",
      "status": "pending",
      "reviewed_by": null,
      "reviewed_at": null
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### GET /evidence/{evidence_id}

Response `200`: EvidenceItem.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### GET /workspaces/{workspace_id}/evidence

Агрегированный список evidence workspace.

Query:

| Параметр | Тип | Обязателен | Описание |
| --- | --- | --- | --- |
| `status` | `EvidenceReviewStatus` | нет | фильтр по review-статусу |
| `research_task_id` | `uuid` | нет | фильтр по research task |

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "source_id": "uuid",
      "excerpt": "Relevant excerpt from the source",
      "location": "section 2.1",
      "confidence": 0.86,
      "status": "pending",
      "reviewed_by": null,
      "reviewed_at": null
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (workspace), `422 validation_error`.

#### PATCH /evidence/{evidence_id}

Review evidence. Разрешён только переход из `pending`.

Body:

```json
{
  "status": "approved"
}
```

Allowed values в body: `approved`, `rejected`.

Response `200`:

```json
{
  "id": "uuid",
  "status": "approved",
  "reviewed_by": "uuid",
  "reviewed_at": "2026-10-31T12:00:00Z"
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 invalid_transition`, `422 validation_error`.

### 6.5 Claims

#### POST /research-tasks/{task_id}/claims

Body:

```json
{
  "text": "Approach X reduces implementation complexity"
}
```

Response `201`: Claim со `status = draft`, `confidence = null`, `reviewed_by = null`, `evidence_count = 0`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (task), `422 validation_error` (пустой `text`).

#### GET /research-tasks/{task_id}/claims

Query:

| Параметр | Тип | Обязателен | Описание |
| --- | --- | --- | --- |
| `status` | `ClaimStatus` | нет | фильтр по статусу |

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "text": "Approach X reduces implementation complexity",
      "status": "draft",
      "confidence": 0.78,
      "evidence_count": 0,
      "review_note": null
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `422 validation_error`.

#### GET /claims/{claim_id}

Response `200`: Claim detail с `evidence`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### GET /workspaces/{workspace_id}/claims

Агрегированный список claims workspace.

Query:

| Параметр | Тип | Обязателен | Описание |
| --- | --- | --- | --- |
| `status` | `ClaimStatus` | нет | фильтр по статусу |
| `research_task_id` | `uuid` | нет | фильтр по research task |

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "research_task_id": "uuid",
      "text": "Approach X reduces implementation complexity",
      "status": "draft",
      "confidence": 0.78,
      "evidence_count": 0,
      "review_note": null
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (workspace), `422 validation_error`.

#### POST /claims/{claim_id}/evidence

Связать claim с evidence.

Body:

```json
{
  "evidence_id": "uuid"
}
```

Response `201`:

```json
{
  "claim_id": "uuid",
  "evidence_id": "uuid",
  "created_at": "2026-10-31T12:00:00Z"
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 already_linked`, `422 scope_mismatch`.

#### DELETE /claims/{claim_id}/evidence/{evidence_id}

Отвязать evidence от claim.

Response `204`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 claim_requires_evidence` (если claim в `supported` и это последняя evidence).

#### PATCH /claims/{claim_id}

Review claim. Backend берёт `reviewed_by` из authenticated context.

Body:

```json
{
  "status": "supported",
  "review_note": "Confirmed by two independent sources"
}
```

Response `200`:

```json
{
  "id": "uuid",
  "status": "supported",
  "reviewed_by": "uuid",
  "review_note": "Confirmed by two independent sources",
  "updated_at": "2026-10-31T12:00:00Z"
}
```

Правила:

- `supported` требует `evidence_count > 0` и human review;
- при `status = draft` поля `reviewed_by`/`review_note` очищаются;
- при любом статусе, отличном от `draft`, `reviewed_by` заполняется actor-ом.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 evidence_required`, `409 invalid_transition`, `422 validation_error`.

### 6.6 Decisions

#### POST /workspaces/{workspace_id}/decisions

Body:

```json
{
  "title": "Use trace-based evaluation",
  "decision_text": "Store traces and tool calls in the first release",
  "rationale": "Reviewed claims support this approach",
  "claim_ids": ["uuid"],
  "research_task_ids": ["uuid"],
  "open_questions": ["How long should traces be retained?"]
}
```

`claim_ids` и `research_task_ids` опциональны. Decision можно создать без claims.

Response `201`: Decision detail со `status = proposed`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (workspace), `422 validation_error`, `422 scope_mismatch`.

#### GET /workspaces/{workspace_id}/decisions

Query:

| Параметр | Тип | Обязателен | Описание |
| --- | --- | --- | --- |
| `status` | `DecisionStatus` | нет | фильтр по статусу |

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "title": "Use trace-based evaluation",
      "status": "proposed",
      "decided_at": null,
      "claims_count": 1,
      "research_tasks_count": 1
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `422 validation_error`.

#### GET /decisions/{decision_id}

Response `200`: Decision detail с `claims` и `research_task_ids`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### PATCH /decisions/{decision_id}

Изменить статус decision. Действие **Send to Review** на frontend соответствует `status = needs_review`.

Body:

```json
{
  "status": "accepted",
  "override_reason": "Urgent release decision approved by tech lead",
  "open_questions": ["How long should traces be retained?"]
}
```

Response `200`:

```json
{
  "id": "uuid",
  "status": "accepted",
  "decided_by": "uuid",
  "decided_at": "2026-10-31T12:00:00Z",
  "updated_at": "2026-10-31T12:00:00Z",
  "confidence": null,
  "override_reason": "Urgent release decision approved by tech lead"
}
```

Правила:

- `accepted` разрешён, если хотя бы один linked claim имеет `reviewed_by != null`, либо передан непустой `override_reason`;
- при `accepted`/`rejected` backend заполняет `decided_by` и `decided_at` из actor-а и текущего времени;
- при `proposed`/`needs_review` поля `decided_by`/`decided_at` остаются `null`;
- `override_reason` сохраняется в response и используется как audit trail.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 reviewed_claim_required`, `409 invalid_transition`, `422 validation_error`.

#### POST /decisions/{decision_id}/claims

Body:

```json
{
  "claim_id": "uuid"
}
```

Response `201`: `{ "decision_id": "uuid", "claim_id": "uuid", "created_at": "..." }`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 already_linked`, `422 scope_mismatch`.

#### POST /decisions/{decision_id}/research-tasks

Body:

```json
{
  "research_task_id": "uuid"
}
```

Response `201`: `{ "decision_id": "uuid", "research_task_id": "uuid", "created_at": "..." }`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 already_linked`, `422 scope_mismatch`.

#### GET /decisions/{decision_id}/export

Response `200`: `text/markdown` с decision brief.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

### 6.7 Research Runs

#### POST /research-tasks/{task_id}/research-runs

Body:

```json
{
  "question": "Which evaluation approach fits production?",
  "mode": "manual_sources"
}
```

Response `201`: ResearchRun со `status = queued`, `started_at = null`, `completed_at = null`.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found` (task), `422 validation_error`, `429 too_many_active_runs`.

#### GET /research-tasks/{task_id}/research-runs

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "status": "queued",
      "mode": "manual_sources",
      "started_at": null,
      "completed_at": null,
      "error_message": null
    }
  ]
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### GET /research-runs/{run_id}

Response `200`: ResearchRun.

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`.

#### POST /research-runs/{run_id}/cancel

Response `200`:

```json
{
  "id": "uuid",
  "status": "cancelled",
  "completed_at": "2026-10-31T12:00:00Z"
}
```

Errors: `401 unauthenticated`, `403 forbidden`, `404 not_found`, `409 run_terminal`.

## 7. State Transitions

Transition table - это API-level validation для v1. Если domain decision изменит допустимые переходы, документ обновляется вместе с domain model.

### ResearchTaskStatus

| From | Allowed to |
| --- | --- |
| `draft` | `in_progress`, `archived` |
| `in_progress` | `review`, `archived` |
| `review` | `completed`, `in_progress`, `archived` |
| `completed` | `archived` |
| `archived` | - |

### EvidenceReviewStatus

| From | Allowed to |
| --- | --- |
| `pending` | `approved`, `rejected` |
| `approved` | - |
| `rejected` | - |

Повторный review evidence не входит в v1 (OQ-4).

### ClaimStatus

| From | Allowed to |
| --- | --- |
| `draft` | `supported`, `weak`, `conflicting`, `outdated` |
| `supported` | `weak`, `conflicting`, `outdated` |
| `weak` | `supported`, `conflicting`, `outdated` |
| `conflicting` | `supported`, `weak`, `outdated` |
| `outdated` | `draft`, `supported`, `weak`, `conflicting` |

Дополнительное условие: переход в `supported` требует хотя бы одной linked evidence. Переход в `outdated` создаёт `Update(type = claim_review)` и review signal для связанных decisions; статус decisions автоматически не меняется.

### DecisionStatus

| From | Allowed to |
| --- | --- |
| `proposed` | `accepted`, `rejected`, `needs_review`, `superseded` |
| `needs_review` | `accepted`, `rejected`, `superseded` |
| `accepted` | `superseded` |
| `rejected` | `superseded` |
| `superseded` | - |

Дополнительное условие: переход в `accepted` требует reviewed claim или непустого `override_reason`.

### ResearchRunStatus

| From | Allowed to | Кто меняет |
| --- | --- | --- |
| `queued` | `running`, `cancelled`, `failed` | worker/reconciler, cancel endpoint |
| `running` | `review_required`, `failed`, `cancelled` | worker/reconciler, cancel endpoint |
| `review_required` | `completed` | human review |
| `completed` | - | - |
| `failed` | - | - |
| `cancelled` | - | - |

`review_required` означает, что draft-results сохранены, но не стали accepted knowledge. `completed` выставляется после human review.

## 8. Screen To API Mapping

| Экран | Entities | Endpoints | Ключевые поля и состояния |
| --- | --- | --- | --- |
| Workspace Overview | Workspace, ResearchTask, Decision | `GET /workspaces/{id}`, `GET /workspaces/{id}/research-tasks`, `GET /workspaces/{id}/decisions`, `POST /workspaces/{id}/ask` | `status`, `summary`, `claims_count`, `memory_sufficient`; empty/loading/error/success/review |
| Research Task Page | ResearchTask, Source, EvidenceItem, Claim, ResearchRun | `GET /research-tasks/{id}`, `GET /research-tasks/{id}/sources`, `GET /research-tasks/{id}/claims`, `GET /research-tasks/{id}/research-runs` | `ResearchTaskStatus`, `ResearchRunStatus`, `error_message`; empty/loading/error/success/review |
| Source / Evidence Panel | Source, EvidenceItem | `GET /sources/{id}`, `GET /sources/{id}/evidence`, `PATCH /evidence/{id}` | `excerpt`, `location`, `confidence`, `status`, `reviewed_by`; actions Approve/Reject |
| Claim Review Panel | Claim, EvidenceItem | `GET /claims/{id}`, `PATCH /claims/{id}` | `text`, `status`, `confidence`, `evidence[]`, `review_note`, `reviewed_by`; Supported недоступен без evidence |
| Decision Page | Decision, Claim, ResearchTask | `GET /decisions/{id}`, `PATCH /decisions/{id}`, `GET /decisions/{id}/export` | `status`, `claims[]`, `research_task_ids`, `confidence`, `open_questions`, `override_reason`; Accept/Send to Review/Reject |
| Follow-up Question Area | Workspace memory | `POST /workspaces/{id}/ask`, `POST /workspaces/{id}/research-tasks` | `answer_text`, counts, `memory_sufficient`, links; empty/loading/error/success/draft |

## 9. Error Codes

| Code | HTTP | Значение |
| --- | --- | --- |
| `unauthenticated` | `401` | нет/невалиден токен |
| `forbidden` | `403` | нет доступа к workspace |
| `not_found` | `404` | сущность не существует |
| `validation_error` | `422` | невалидное поле, неизвестный enum, пустой обязательный текст |
| `unknown_field` | `422` | неизвестное поле в body |
| `unknown_actor_field` | `422` | клиент передал actor-поле |
| `scope_mismatch` | `422` | сущность принадлежит другому workspace/research task |
| `already_linked` | `409` | связь уже существует |
| `invalid_transition` | `409` | переход статуса запрещён |
| `evidence_required` | `409` | claim нельзя перевести в `supported` без evidence |
| `claim_requires_evidence` | `409` | нельзя отвязать последнюю evidence у `supported` claim |
| `reviewed_claim_required` | `409` | decision нельзя принять без reviewed claim или override |
| `run_terminal` | `409` | research run уже в terminal state |
| `too_many_active_runs` | `429` | превышен лимит активных runs |
| `memory_unavailable` | `502` | сервис памяти/LLM недоступен |
| `invalid_url` | `422` | некорректный URL для import |
| `source_fetch_failed` | `502` | не удалось получить содержимое source |

## 10. Open Questions

### OQ-1. Точный критерий reviewed claim

В v1 decision acceptance проверяет `claim.reviewed_by != null`. Нужно domain decision:

- достаточно ли `reviewed_by` без отдельного `review_status`;
- считается ли reviewed claim только `supported`/`weak`/`conflicting`/`outdated`, или любой claim с заполненным `reviewed_by`;
- нужно ли требовать reviewed все linked claims или хотя бы один.

### OQ-2. Источник Decision confidence

В domain model у Decision нет собственного `confidence`. В v1 поле read-only и возвращается как `null`. Нужно решить:

- агрегировать ли confidence из linked claims;
- хранить ли отдельное поле Decision;
- генерировать ли значение в agent-service как часть decision brief.

### OQ-3. Семантика Send to Review

В v1 кнопка Send to Review соответствует `PATCH /decisions/{id}` со `status = needs_review`. Нужно подтвердить, что `needs_review` - правильный статус для этого действия, и не нужен ли отдельный `in_review`.

### OQ-4. Review-состояние EvidenceItem

`EvidenceReviewStatus`, `reviewed_by`, `reviewed_at` добавлены в API contract, но отсутствуют в `04-domain-model.md`. Нужно синхронизировать domain model и решить:

- допустим ли повторный review (`approved -> rejected`);
- нужен ли `review_note` для evidence.

### OQ-5. Связь Decision -> ResearchTask

В domain model есть link table `decision_research_tasks`, но не описано, как она используется в UI. В v1 добавлены `research_task_ids` при создании и link endpoint. Нужно подтвердить, что decision может ссылаться на несколько research tasks и что link не меняет статус decision.

### OQ-6. Transition rules

Transition tables в разделе 7 - это v1 contract. Нужно подтвердить на domain уровне:

- допустимы ли обратные переходы ResearchTask (`review -> in_progress`);
- можно ли возвращать claim из `outdated` в `draft`;
- является ли `superseded` terminal state.

### OQ-7. Permissions и workspace membership

В v1 предполагается, что authenticated actor имеет доступ к workspace, если он его owner. Полноценные роли/membership не входят в MVP. Нужно решить, как проверять доступ, когда появится несколько участников.

### OQ-8. Pagination

В v1 все list endpoints возвращают полный `{ items: [...] }` без пагинации. Нужно определить формат pagination до появления больших workspace.

### OQ-9. Асинхронный import source

В v1 `POST /sources/import` возвращает `202` и Source с `summary = null`. Нужно определить:

- нужен ли отдельный `processing_status`;
- как frontend узнаёт о завершении обработки (polling GET source или отдельный endpoint).

### OQ-10. Публичность Update

`Update` пишется backend-ом как audit trail, но публичных endpoints в v1 нет. Нужно решить, нужен ли экран истории изменений в MVP.

### OQ-11. Actor для internal API

Публичные review endpoints запрещены для agent service. Нужно зафиксировать, какие internal endpoints доступны agent service и как передаётся service identity.
