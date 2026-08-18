# Доменная Модель

Этот документ описывает первую доменную модель для MVP.

## Сущности

### Workspace

Контейнер для исследовательской области.

Поля:

- `id`
- `title`
- `description`
- `owner_id`
- `created_at`
- `updated_at`

### ResearchTask

Конкретный исследовательский вопрос внутри workspace.

Поля:

- `id`
- `workspace_id`
- `title`
- `research_question`
- `status`: `draft`, `in_progress`, `review`, `completed`, `archived`
- `summary`
- `created_by`
- `created_at`
- `updated_at`

### Source

Источник информации, использованный в исследовании.

Поля:

- `id`
- `workspace_id`
- `research_task_id`
- `type`: `url`, `manual_note`, `pdf`, `doc`, `github`, `video`
- `title`
- `url`
- `author`
- `published_at`
- `captured_at`
- `summary`
- `raw_content_ref`
- `reliability_note`

### EvidenceItem

Конкретный фрагмент source.

Поля:

- `id`
- `source_id`
- `excerpt`
- `location`
- `note`
- `confidence`
- `created_at`

### Claim

Проверяемое утверждение, извлеченное из исследования.

Поля:

- `id`
- `workspace_id`
- `research_task_id`
- `text`
- `status`: `draft`, `supported`, `weak`, `conflicting`, `outdated`
- `confidence`
- `created_by`
- `reviewed_by`
- `created_at`
- `updated_at`

Связи:

- many-to-many с evidence items;
- many-to-many с decisions;
- связь с updates.

### Decision

Решение команды на основе claims.

Поля:

- `id`
- `workspace_id`
- `title`
- `decision_text`
- `status`: `proposed`, `accepted`, `rejected`, `superseded`, `needs_review`
- `rationale`
- `decided_by`
- `decided_at`
- `created_at`
- `updated_at`

Связи:

- many-to-many с claims;
- many-to-many с research tasks;
- связь с updates.

### ResearchRun

Один контролируемый agent/research запуск.

Поля:

- `id`
- `workspace_id`
- `research_task_id`
- `question`
- `status`: `queued`, `running`, `review_required`, `completed`, `failed`, `cancelled`
- `mode`: `manual_sources`, `search_assisted`, `follow_up`
- `artifact_manifest_ref`
- `started_by`
- `started_at`
- `completed_at`
- `error_message`

### Update

Событие, которое может изменить актуальность sources, claims или decisions.

Поля:

- `id`
- `workspace_id`
- `type`: `source_changed`, `new_source`, `claim_review`, `decision_review`, `manual_note`
- `title`
- `description`
- `affected_source_id`
- `affected_claim_id`
- `affected_decision_id`
- `created_at`

## Ключевые Правила

- Claim без evidence не может быть `supported`.
- AI-generated claim стартует как `draft`.
- Decision может стать `accepted` только с reviewed claims или с явным human override.
- Outdated claim должен подсвечивать связанные decisions как candidates for review.
- ResearchRun output не становится accepted knowledge автоматически.
