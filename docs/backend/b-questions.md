# B1 - Границы В DocForge

## Слои

Имена слоёв ниже взяты из DocForge. В нашем проекте им соответствуют `domain/`, `application/`, `infrastructure/` и `api/` из `docs/architecture/02-system-architecture.md`.

 1. domain/ (laws layer)
    - `domain/` содержит бизнес-понятия и правила, которые остаются истинными независимо от HTTP, базы данных, очередей и внешних сервисов. В domain нет импортов FastAPI, SQLAlchemy, queue-клиентов или storage-адаптеров. Domain не описывает, как другие слои вызывают друг друга; он фиксирует, какое состояние бизнес-объекта считается корректным.
    - Состав domain:
        - **business entities**: `Workspace`, `ResearchTask`, `Source`, `EvidenceItem`, `Claim`, `Decision`, `ResearchRun`, `Update`;
        - **invariants**: условия, которые обязаны выполняться в любом допустимом состоянии. Например: `supported` claim не существует без хотя бы одной evidence и human review; `accepted` decision имеет reviewed claim или explicit human override; связанные Claim/Evidence/Decision не выходят за пределы одного workspace/research task; agent draft не считается accepted knowledge;
        - **domain rules**: правила жизненного цикла и переходов. Например: claim проходит `draft -> supported | weak | conflicting | outdated`; decision проходит `proposed -> accepted | rejected | needs_review | superseded`; `outdated` claim даёт review signal для связанных decisions, но не меняет их статус автоматически;
        - **policies**: именованные проверки, которые применяют domain rules к конкретному сценарию: `ClaimReviewPolicy`, `DecisionAcceptancePolicy`, `ScopeIntegrityPolicy`, `ResearchTaskLifecyclePolicy`, `ResearchRunLifecyclePolicy`, `EvidenceReviewPolicy`, `AgentBoundaryPolicy`.
    - Пример: claim нельзя пометить `supported`, если у него нет ни одной evidence или он не прошёл human review. Это правило истинно и для Postgres, и для файла на диске, и для in-memory repository в тестах.
 2. services/ (application layer)
    - Координирует сценарии использования (use cases): в каком порядке вызывать правила (из domain), обращаться к базе данных, публиковать сообщения в очередь. Три основные подгруппы:
        - use-case сервисы (auth_service, prediction_submission_service) описывают весь сценарий от начала до конца.
        - mappers (ИЗ файла mappers.py) представляет классы (например, User) в 3 разных видах (формах), в зависимости от того, в каком слое кода мы сейчас находимся.
        - reconciler - application-компонент, который доводит remote job до terminal state. Он не содержит domain rules, а оркестрирует вызовы:
            1. Получает из repository пачку remote job записей в статусе `pending` (`job_id`, `run_id`, `external_job_id`, `attempt`, `idempotency_key`).
            2. Для каждой записи запрашивает состояние у внешнего сервиса (agent-service/ML-service) и проверяет: job ещё выполняется, завершилась успешно, завершилась ошибкой или отменена; есть ли готовый artifact/result.
            3. Если job ещё выполняется - оставляет `pending` и планирует следующий poll с backoff, ничего не записывая в product state.
            4. Если job `succeeded` - идемпотентно сохраняет artifact и draft-results (sources/evidence/draft claims) через use case, помечает remote job как `succeeded` и переводит `ResearchRun` в `review_required`. Draft сохраняется, но не становится accepted knowledge.
            5. Если job `failed` - сохраняет `error_message`, помечает job как `failed` и переводит `ResearchRun` в `failed`.
            6. Если job `cancelled` - помечает job как `cancelled` и переводит `ResearchRun` в `cancelled`.
            7. Считает job завершённой только после перехода в terminal state: `succeeded`, `failed` или `cancelled`. Повторный poll terminal job не выполняет.
            8. Все записи по одной job выполняет в одной Unit of Work; повторная обработка защищена `idempotency_key`.
        - Для нашего проекта полный жизненный цикл `ResearchRun`: `queued -> running -> (review_required | failed | cancelled)`. `review_required` означает, что агент закончил и draft-results сохранены, но человек ещё не проверил их. `completed` выставляется после завершения human review, а не в момент ответа агента.
 3. api/ (http layer)
    - Принимает HTTP-запрос, вызывает нужный use case из services/, оформляет ответ в JSON. Так же есть поддержка ошибок, например, InsufficientBalanceError.
 4. db/ (database layer)
    - Работа с базами данных (PostgreSQL, ORM-модели*): r - read, w - write, rw - read&write и тд.

    *тут имею в виду таблиц базы данных в виде Python классов.

 5. inference/contracts.py 
    - Файл с описанием единого формата данных, в котором сервисный слой передаёт формы вида "Вопрос - Ответ" для общения с внешним ML-сервисом (Docling, Datalab и т.д.), чтобы не переписывать код при смене или добавлении сервиса.

## 3 примера хороших границ

1. Ошибка InsufficientBalanceError (недостаточно средств, domain/exceptions.py) это обычные Python классы. Внутри самого исключения нет ни ничего про HTTP. Перевод в конкретный HTTP статус (например, 404) происходит только в api/errors.py.

2. Пользователь для базы дынных (UserORM), пользователь для основных правил (User), пользователь для ответа сайту (DTO) - это три разных объекта с похожим смыслом, но они специально не путаются между собой для платформенности.

3. Разделение сессий в базе данных:
    - get_db_session() - сессия для записи. Если всё прошло гладко, то коммит, всё сохраняется, иначе откат.
    - get_read_session() - сессия только для чтения.
    - get_plain_session() - сессия без автоматики, нужна в случае, когда логика транзакции многоуровневая.

## 3 вещи, которые стоит адаптировать для нашего проекта

1. Более строгое расположение правил. Например, в docforge проверка на "хватает ли пользователю на задачу" находится в файле, отвечающем за сохранение данных, а не в чистом domain/.
2. Разделение W, R и WR операций с базой данных. Отдельно смотреть на данные и менять их для мультизадачности с базой данных.
3. Единое место, где ошибки становятся HTTP ответами, чтобы каждый раз не писать try & except.

# B2 - Черновик Доменной Модели

## ERD
- https://miro.com/welcomeonboard/ZmlRRGl4anp4NjdTSGZKY1d4NCt2S3BaMVRwb1ZKUTJEWjF3V2lzVFdoTEk4NTNVRVMyL3BTUjNvRHlXaHdOTzNQQWhYNzZpa2U3UTZTZFAyNEx3c04zZWxQRysvYXdQcllqNnQzU0tnb2k2S28zbmhxeDBDdDY4K2g5Sm9BUE85RHYwazJmckV3alB5NmozR3Jxd21nPT0hdjE=?share_link_id=914409870422

## Список таблиц
1. Workspace - рабочее пространство проекта
2. ResearchTask - исследовательский вопрос внутри workspace
3. Source - источник информации (например, статья или пдф)
4. EvidenceItem - конкретный фрагмент Source
5. Claim - проверяемое утверждение, извлеченное из исследования
6. Decision - решение команды на основе claims
7. ResearchRun - один запуск агента
8. updates - событие, которое может изменить актуальность sources, claims или decisions

## many-to-many link tables
1. claim_evidence
2. decision_claims
3. decision_research_tasks

## status fields и allowed values

| Таблица      |  Поле  | Значения                                                               |
| ------------ | :----: | ---------------------------------------------------------------------- |
| ResearchTask | status | draft, in_progress, review, completed, archived                        |
| Source       |  type  | url, manual_note, pdf, doc, github, video                              |
| EvidenceItem | status | pending, approved, rejected                                            |
| Claim        | status | draft, supported, weak, conflicting, outdated                          |
| Decision     | status | proposed, accepted, rejected, superseded, needs_review                 |
| ResearchRun  | status | queued, running, review_required, completed, failed, cancelled         |
| ResearchRun  |  mode  | manual_sources, search_assisted, follow_up                             |
| Update       |  type  | source_changed, new_source, claim_review, decision_review, manual_note |

# B3 - Черновик API Contracts

## Enums

Имена enum используются в request/response и в query-параметрах. Allowed values совпадают с B2.

| Enum | Allowed values |
| --- | --- |
| `ResearchTaskStatus` | `draft`, `in_progress`, `review`, `completed`, `archived` |
| `SourceType` | `url`, `manual_note`, `pdf`, `doc`, `github`, `video` |
| `EvidenceReviewStatus` | `pending`, `approved`, `rejected` |
| `ClaimStatus` | `draft`, `supported`, `weak`, `conflicting`, `outdated` |
| `DecisionStatus` | `proposed`, `accepted`, `rejected`, `superseded`, `needs_review` |
| `ResearchRunStatus` | `queued`, `running`, `review_required`, `completed`, `failed`, `cancelled` |
| `ResearchRunMode` | `manual_sources`, `search_assisted`, `follow_up` |
| `UpdateType` | `source_changed`, `new_source`, `claim_review`, `decision_review`, `manual_note` |

`EvidenceReviewStatus` формализует уже существовавшие в B3 значения `pending/approved/rejected` для review-состояния EvidenceItem.

## Collection Envelope

Все list endpoints возвращают коллекцию в едином envelope:

```json
{ "items": [] }
```

Пустой список - это `{ "items": [] }`, а не `null` и не отсутствие поля.

## Actor Fields

Клиент никогда не передаёт `created_by`, `reviewed_by`, `decided_by`, `started_by` и `owner_id` в request body. Backend получает actor из authenticated user context (Bearer token) и сам заполняет эти поля в response. Если клиент передал actor-поле в body, API возвращает `422 unknown_actor_field` (request schema работает с `extra=forbid`).

## Scope Integrity

Все связи проверяются на backend до записи. Если сущность не существует - `404 not_found`; если существует, но принадлежит другому workspace/research task - `422 scope_mismatch`.

| Связь | Проверка | Ошибка |
| --- | --- | --- |
| Evidence -> Source | `evidence.source_id` указывает на source из path | `404 not_found` |
| Claim -> Evidence | `evidence.source.research_task_id == claim.research_task_id` и `workspace_id` совпадают | `422 scope_mismatch` |
| Decision -> Claim | `claim.workspace_id == decision.workspace_id` | `422 scope_mismatch` |
| Decision -> ResearchTask | `research_task.workspace_id == decision.workspace_id` | `422 scope_mismatch` |

### Workspaces

 - Стиль "*?:" означает необязательность пункта

**Создать workspace**  
POST /workspaces  
body: { title: string, description?: string }  
response: { id, title, description, owner_id, created_at, updated_at }  
errors: 422 validation (пустой title)  
  
**Получить список workspaces**  
GET /workspaces  
response: { items: [{ id, title, description, owner_id, created_at }]}
errors: 401 unauthenticated  
  
**Получить один workspace**  
GET /workspaces/{workspace_id}  
response: { id, title, description, owner_id, created_at, updated_at}  
errors: 404 not found  
  
**Обновить workspace**  
PATCH /workspaces/{workspace_id}  
body: {title: string, description?: string}  
response: { id, title, description, updated_at }  
errors: 404 not found, 422 validation  
  
**Задать вопрос по workspace**  
POST /workspaces/{workspace_id}/ask  
body: { question: string }  
response: { answer_text, sources_count, claims_count, decisions_count, memory_sufficient, evidence_link: "/api/v1/workspaces/{workspace_id}/evidence", claims_link: "/api/v1/workspaces/{workspace_id}/claims", sources_link: "/api/v1/workspaces/{workspace_id}/sources" }  
errors: 404 workspace not found, 502 memory unavailable  

### Research Tasks  
  
**Создать research task**  
POST /workspaces/{workspace_id}/research-tasks  
body: { title: string, research_question: string }  
response: { id, workspace_id, title, research_question, status: ResearchTaskStatus, created_by, created_at }  
errors: 404 workspace not found, 422 validation  
  
**Список research tasks в workspace**  
GET /workspaces/{workspace_id}/research-tasks?status=ResearchTaskStatus  
response: { items: [{ id, title, status, summary, created_at }]}
errors: 404 workspace not found  
  
**Получить одну research task**  
GET /research-tasks/{task_id}  
response: { id, workspace_id, title, research_question, status: ResearchTaskStatus, summary, created_by, created_at, updated_at }  
errors: 404  
  
**Изменить статус**  
PATCH /research-tasks/{task_id}  
body: { status?: ResearchTaskStatus, summary?: string }  
response: { id, status: ResearchTaskStatus, summary, updated_at }  
errors: 404 not found, 409 invalid_transition, 422 validation  
  
### Sources  
  
**Добавить source**  
POST /research-tasks/{task_id}/sources  
body: { type: SourceType, title: string, url?: string, author?: string, published_at?: datetime, summary?: string }  
response: { id, workspace_id, research_task_id, type: SourceType, title, url, summary, captured_at }  
errors: 404 task not found, 422 validation (неверный SourceType; для url/github/video обязателен url)  
  
**Импорт source из URL**  
POST /research-tasks/{task_id}/sources/import  
body: { url: string }  
response: { id, type, title, url, summary, captured_at } (summary может быть заполнен позже, асинхронно)  
errors: 404 task not found, 422 некорректный url, 502 не удалось получить содержимое  
  
**Список sources в task**  
GET /research-tasks/{task_id}/sources  
response: { items: [{ id, type, title, url, summary, reliability_note }]}  
errors: 404  
  
**Получить один source**  
GET /sources/{source_id}  
response: { id, workspace_id, research_task_id, type: SourceType, title, url, author, published_at, captured_at, summary, reliability_note }  
errors: 404  

**Список sources в workspace**  
GET /workspaces/{workspace_id}/sources?type=SourceType&research_task_id=uuid  
response: { items: [{ id, research_task_id, type: SourceType, title, url, summary, reliability_note }]}  
errors: 404 workspace not found, 422 validation (неизвестный SourceType)  
  
### Evidence  
  
**Добавить evidence к source**  
POST /sources/{source_id}/evidence  
body: { excerpt: string, location: string, note?: string, confidence: number }  
response: { id, source_id, excerpt, location, note, confidence, created_at, status: EvidenceReviewStatus, reviewed_by?, reviewed_at? }  
errors: 404 source not found, 422 validation (пустой excerpt)  
  
**Список evidence по source**  
GET /sources/{source_id}/evidence  
response: { items: [{ id, excerpt, location, confidence, created_at, status: EvidenceReviewStatus, reviewed_by?, reviewed_at? }] } 
errors: 404 source not found  
  
**Получить одно evidence**  
GET /evidence/{evidence_id}  
response: { id, source_id, excerpt, location, note, confidence, created_at, status: EvidenceReviewStatus, reviewed_by?, reviewed_at? } 
errors: 404   

**Список evidence в workspace**  
GET /workspaces/{workspace_id}/evidence?status=EvidenceReviewStatus&research_task_id=uuid  
response: { items: [{ id, source_id, excerpt, location, confidence, status: EvidenceReviewStatus, reviewed_by?, reviewed_at? }]}  
errors: 404 workspace not found, 422 validation (неизвестный EvidenceReviewStatus)  
  
**Изменить статус evidence (review)**
PATCH /evidence/{evidence_id}
body: { status: EvidenceReviewStatus }
response: { id, status: EvidenceReviewStatus, reviewed_by, reviewed_at }
errors: 404 not found, 409 invalid_transition (review возможен только из pending), 422 validation (allowed: approved, rejected)

### Claims  
**status:** `ClaimStatus` (`draft` | `supported` | `weak` | `conflicting` | `outdated`)  
**confidence:** read-only; при ручном создании claim через API равен `null`, agent draft может заполнить значение.  

**Создать claim**  
POST /research-tasks/{task_id}/claims  
body: { text: string }  
response: { id, workspace_id, research_task_id, text, status: ClaimStatus, confidence, created_by, created_at }  
errors: 404 task not found, 422 validation (пустой text)  
  
**Список claims по research task**  
GET /research-tasks/{task_id}/claims?status=ClaimStatus
response: { items: [{ id, text, status: ClaimStatus, confidence, evidence_count, review_note? }]}  
errors: 404 task not found, 422 validation (неизвестный ClaimStatus)  

**Список claims в workspace**  
GET /workspaces/{workspace_id}/claims?status=ClaimStatus&research_task_id=uuid  
response: { items: [{ id, research_task_id, text, status: ClaimStatus, confidence, evidence_count, review_note? }]}  
errors: 404 workspace not found, 422 validation (неизвестный ClaimStatus)  
  
**Получить один claim (с evidence)**  
GET /claims/{claim_id}  
response: { id, text, status: ClaimStatus, confidence, reviewed_by, evidence: [{ id, excerpt, source_id, source_title, confidence }], created_at, updated_at, review_note? }  
errors: 404 not found  
  
**Связать claim с evidence**  
POST /claims/{claim_id}/evidence  
body: { evidence_id: string }  
response: { claim_id, evidence_id, created_at }  
errors: 404 claim or evidence not found, 409 already_linked, 422 scope_mismatch (evidence принадлежит другому research task/workspace)

**Отвязать evidence от claim**
DELETE /claims/{claim_id}/evidence/{evidence_id}
response: 204 No Content
errors: 404 not found, 409 нельзя отвязать последнюю evidence у supported claim
  
**Изменить статус claim (review)**  
PATCH /claims/{claim_id}  
body: { status: ClaimStatus, review_note?: string }  
response: { id, status: ClaimStatus, reviewed_by, updated_at, review_note? }  
errors: 404 not found, 409 evidence_required (supported без evidence), 409 invalid_transition, 422 validation (неизвестный ClaimStatus)  
  
### Decisions  
  
**confidence:** read-only агрегированное значение; в v1 возвращается `null`, пока правило агрегации не зафиксировано в `api-contract-v1.md` (OQ-2).  
  
**Создать decision**  
POST /workspaces/{workspace_id}/decisions  
body: { title: string, decision_text: string, rationale?: string, claim_ids?: string[], research_task_ids?: string[], open_questions?: string[] }  
response: { id, workspace_id, title, decision_text, status: DecisionStatus, rationale, decided_by, created_at, confidence, open_questions?, research_task_ids? }  
errors: 404 workspace not found, 422 scope_mismatch (claim или research task принадлежит другому workspace), 422 validation  
  
**Список decisions в workspace**  
GET /workspaces/{workspace_id}/decisions?status=DecisionStatus  
response: { items: [{ id, title, status: DecisionStatus, decided_at, claims_count, research_tasks_count }]}  
errors: 404 workspace not found, 422 validation (неизвестный DecisionStatus)  
  
**Получить одно decision (с claims)**  
GET /decisions/{decision_id}  
response: { id, title, decision_text, status: DecisionStatus, rationale, decided_by, decided_at, claims: [{ id, text, status: ClaimStatus, reviewed_by? }], research_task_ids: string[], created_at, updated_at, confidence, open_questions?, override_reason? }  
errors: 404 not found  
  
**Принять/изменить статус decision**  
PATCH /decisions/{decision_id}  
body: { status: DecisionStatus, override_reason?: string, open_questions?: string[] }  
response: { id, status: DecisionStatus, decided_by, decided_at, updated_at, confidence, override_reason? }  
errors: 404 not found, 409 reviewed_claim_required (accepted без reviewed claim и без override_reason), 409 invalid_transition, 422 validation (неизвестный DecisionStatus)  
  
**Экспорт decision brief в Markdown**  
GET /decisions/{decision_id}/export  
response: text/markdown (файл )  
errors: 404  

**Связать decision с claim**
POST /decisions/{decision_id}/claims
body: { claim_id: string }
response: { decision_id, claim_id, created_at }
errors: 404 not found, 409 already_linked, 422 scope_mismatch

**Связать decision с research task**
POST /decisions/{decision_id}/research-tasks
body: { research_task_id: string }
response: { decision_id, research_task_id, created_at }
errors: 404 not found, 409 already_linked, 422 scope_mismatch
  
### Research Runs  
  
**Запустить research run**  
POST /research-tasks/{task_id}/research-runs  
body: { question: string, mode: ResearchRunMode }  
response: { id, workspace_id, research_task_id, question, status: ResearchRunStatus, mode: ResearchRunMode, started_by, started_at }  
errors: 404 task not found, 422 validation (неверный ResearchRunMode), 429 too_many_active_runs  
  
**Список research runs по task**  
GET /research-tasks/{task_id}/research-runs  
response: { items: [{ id, status: ResearchRunStatus, mode: ResearchRunMode, started_at, completed_at, error_message }]}  
errors: 404 task not found  
  
**Получить статус run**  
GET /research-runs/{run_id}  
response: { id, status: ResearchRunStatus, mode: ResearchRunMode, artifact_manifest_ref, started_at, completed_at, error_message }  
errors: 404 not found  
  
**Отменить run**  
POST /research-runs/{run_id}/cancel  
response: { id, status: ResearchRunStatus, completed_at }  
errors: 404 not found, 409 run_terminal  

# B4 - Domain Rules
1. Claim может в статус supported если:
   1. у claim есть хотя бы одна связанная evidence
   2. claim прошёл проверку человеком (review)
2. Decision может стать accepted по одному из двух сценариев:
   1. decision связано хотя бы с одним claim, который был проверен человеком (reviewed claim)
   2. human override: человек принимает decision вручную, без reviewed claims
3. Что происходит, когда claim становится outdated
   1. сам claim переходит в статус outdated
   2. все decisions, которые ссылались на этот claim, тоже помечаются на пересмотр (candidates for review) т.е это сигнал для команды, что решение стоит перепроверить, а не обязательно автоматическая смена статуса decision на needs_review
   3. создаётся соответствующая запись в таблице апдейтов, чтобы сохранить историю
4. Что означает agent-generated draft:
   1. это данные (sources, evidence, claims), которые backend получает от agent-service как draft-results. Backend сохраняет их со статусом draft/unreviewed, после чего ResearchRun переходит в review_required. draft данные хранятся в системе, но не считаются подтверждённым знанием команды (accepted knowledge) до прохождения review человеком.

> Примечание: точный критерий `reviewed claim` (наличие `reviewed_by`/`reviewed_at` или отдельный `review_status`) фиксируется в `docs/backend/api-contract-v1.md`. В B4 не вводится эквивалентность `reviewed claim == status != draft`.

# B5 - Summary Изменений

- `domain` переписан через business entities, invariants, domain rules и policies; явно зафиксировано, что domain не зависит от HTTP, БД, queue и storage.
- `reconciler` описан как конкретный remote job flow: получение `pending`, проверка внешнего статуса, идемпотентное сохранение результата, обновление job и `ResearchRun`, terminal states и retry/backoff.
- Все list endpoints используют единый envelope `{ items: [...] }`.
- В B3 добавлены именованные enums и allowed values; `string` в status/type/mode и query-параметрах заменён на `ResearchTaskStatus`, `SourceType`, `EvidenceReviewStatus`, `ClaimStatus`, `DecisionStatus`, `ResearchRunStatus`, `ResearchRunMode`.
- Actor fields (`created_by`, `reviewed_by`, `decided_by`, `started_by`, `owner_id`) не принимаются в request body; backend берёт их из authenticated context.
- Добавлен единый раздел Scope Integrity для связей Evidence -> Source, Claim -> Evidence, Decision -> Claim, Decision -> ResearchTask.
- Создание decision не требует claim_ids: `claim_ids?` остаётся опциональным, ограничение действует только для перехода в `accepted`.
- B4 исправлен: accepted decision требует reviewed claim или explicit human override; outdated claim даёт review signal/candidate, а не автоматическую смену статуса decision; agent draft сохраняется как draft/unreviewed и не считается accepted knowledge.
- Добавлены endpoints для связи Decision -> Claim и Decision -> ResearchTask, а также отвязки Evidence от Claim.
- `ResearchRun` использует `started_at`/`completed_at` без добавления `created_at`.