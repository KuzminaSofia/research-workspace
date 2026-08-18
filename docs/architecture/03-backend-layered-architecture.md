# Слоистая Архитектура Backend

Backend следует Clean Architecture / Hexagonal Architecture в прагматичной форме.

## Слои

### Domain

Содержит бизнес-понятия и правила:

- `Workspace`
- `ResearchTask`
- `Source`
- `EvidenceItem`
- `Claim`
- `Decision`
- `ResearchRun`
- `ArtifactManifest`

В domain должны жить правила вроде:

- draft claim не может стать supported без evidence;
- accepted decision должен ссылаться хотя бы на один reviewed claim;
- outdated claim может пометить связанные decisions как `needs_review`;
- claims, созданные агентом, стартуют в статусе `draft`.

### Application

Содержит workflow/use cases:

- создать workspace;
- создать research task;
- добавить source;
- запустить research run;
- проверить claim;
- создать decision;
- задать вопрос по workspace;
- экспортировать decision brief.

Application use cases зависят от ports, а не от конкретной infrastructure.

Примеры ports:

```text
ResearchRepositoryPort
UnitOfWorkPort
ArtifactStorePort
LLMPort
SearchPort
QueuePort
ClockPort
```

### Infrastructure

Реализует ports:

- SQLAlchemy/PostgreSQL repositories;
- local filesystem или S3 artifact store;
- OpenAI/другой LLM adapter;
- Postgres full-text search;
- RabbitMQ/Redis queue adapter;
- migration scripts.

### API

Тонкий HTTP-слой:

- request/response schemas;
- auth dependency;
- валидация transport-level input;
- вызов use case;
- маппинг результата use case в response.

API не должен содержать бизнес-логику.

### Agent Service Boundary

Agent service живет отдельным runtime-компонентом и не является внутренним слоем backend. Он отвечает за research pipeline:

- планирует run;
- ingest sources;
- извлекает evidence;
- создает draft claims;
- связывает claims с evidence;
- генерирует decision brief;
- сохраняет artifacts;
- переводит результат в human review.

Agent service общается с backend через queue/internal API/contracts. Результат agent считается draft до human review.

## Пример Flow

```text
POST /research-runs
  -> api router валидирует request
  -> StartResearchRun use case создает ResearchRun(status=queued)
  -> UnitOfWork commits DB state
  -> QueuePort публикует ResearchRunRequested
  -> agent-service получает message
  -> agent-service запрашивает context у backend internal API
  -> AgentOrchestrator выполняет bounded steps
  -> agent-service пишет artifact package
  -> agent-service отправляет draft-results в backend
  -> backend валидирует и сохраняет sources/evidence/draft claims
  -> run status становится review_required
```

## Тестовая Стратегия

- Domain tests: чистые unit tests без DB.
- Application tests: use cases с fake ports.
- Infrastructure tests: repository/storage/queue adapters.
- API tests: route-level behavior.
- System tests: один end-to-end demo flow.
