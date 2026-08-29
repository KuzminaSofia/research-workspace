# B1 - Границы В DocForge

## Слои
 1. domain/ (laws layer)
    - domain/ содержит business entities, инварианты и domain rules и policies, которые не зависят от http, бд, queue и инфраструктуры. Например: claim нельзя пометить подтверждённым, если у него нет ни одного источника. Это правило истинно всегда и неважно, хранятся ли данные в Postgres или в файле на диске.
 2. services/ (application layer)
    - Координирует сценарии использования (use cases): в каком порядке вызывать правила (из domain), обращаться к базе данных, публиковать сообщения в очередь. Три основные подгруппы:
        - use-case сервисы (auth_service, prediction_submission_service) описывают весь сценарий от начала до конца.
        - mappers (ИЗ файла mappers.py) представляет классы (например, User) в 3 разных видах (формах), в зависимости от того, в каком слое кода мы сейчас находимся.
        - reconciler периодически опрашивает записи со статусом pending из таблицы remote job flow. Если в ответе результат готов, то сохраняет результат и переводит задачу в succeedd/failed, иначе оставляем pending. Завершённой задачей считается момент, когда статус job перешёл в succeeded или failed. Для нашего проекта ResearchRun переходит queued -> running -> состояние процесса
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
| Claim        | status | draft, supported, weak, conflicting, outdated                          |
| Decision     | status | proposed, accepted, rejected, superseded, needs_review                 |
| ResearchRun  | status | queued, running, review_required, completed, failed, cancelled         |
| ResearchRun  |  mode  | manual_sources, search_assisted, follow_up                             |
| Update       |  type  | source_changed, new_source, claim_review, decision_review, manual_note |

# B3 - Черновик API Contracts

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
  
### Research Tasks  
  
**Создать research task**  
POST /workspaces/{workspace_id}/research-tasks  
body: { title: string, research_question: string }  
response: { id, workspace_id, title, research_question, status: "draft", created_by, created_at }  
errors: 404 workspace not found, 422 validation  
  
**Список research tasks в workspace**  
GET /workspaces/{workspace_id}/research-tasks?status=  
response: { items: [{ id, title, status, summary, created_at }]}
errors: 404 workspace not found  
  
**Получить одну research task**  
GET /research-tasks/{task_id}  
response: { id, workspace_id, title, research_question, status, summary, created_by, created_at, updated_at }  
errors: 404  
  
**Изменить статус**  
PATCH /research-tasks/{task_id}  
body: { status?: string, summary?: string }  
response: { id, status, summary, updated_at }  
errors: 404, 409 invalid status transition, 422  
  
### Sources  
  
**Добавить source**  
POST /research-tasks/{task_id}/sources  
body: { type: string, title: string, url?: string, author?: string, published_at?: datetime, summary?: string }  
response: { id, workspace_id, research_task_id, type, title, url, summary, captured_at }  
errors: 404 task not found, 422 validation (неверный type)  
  
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
response: { id, workspace_id, research_task_id, type, title, url, author, published_at, captured_at, summary, reliability_note }  
errors: 404  
  
### Evidence  
  
**Добавить evidence к source**  
POST /sources/{source_id}/evidence  
body: { excerpt: string, location: string, note?: string, confidence: number }  
response: { id, source_id, excerpt, location, note, confidence, created_at }  
errors: 404 source not found, 422 validation (пустой excerpt)  
  
**Список evidence по source**  
GET /sources/{source_id}/evidence  
response: { items: [{ id, excerpt, location, confidence, created_at }]}  
errors: 404 source not found  
  
**Получить одно evidence**  
GET /evidence/{evidence_id}  
response: {id, source_id, excerpt, location, note, confidence, created_at }  
errors: 404   
  
### Claims  
**status:** draft | supported | weak | conflicting | outdated  

**Создать claim**  
POST /research-tasks/{task_id}/claims  
body: { text: string }  
response: { id, workspace_id, research_task_id, text, status: "draft", confidence, created_by, created_at }  
errors: 404 task not found, 422 validation (пустой text)  
  
**Список claims по research task**  
GET /research-tasks/{task_id}/claims?status={status} 
response: { items: [{ id, text, status, confidence, evidence_count }]}  
errors: 404 task not found  
  
**Получить один claim (с evidence)**  
GET /claims/{claim_id}  
response: { id, text, status, confidence, reviewed_by, evidence: [{ id, excerpt, source_title }], created_at, updated_at }  
errors: 404 not found  
  
**Связать claim с evidence**  
POST /claims/{claim_id}/evidence  
body: { evidence_id: string }  
response: { claim_id, evidence_id, created_at }  
errors: 404 claim or evidence not found, 409 уже связаны, 422 evidence принадлежит другому research task/workspace
  
**Изменить статус claim (review)**  
PATCH /claims/{claim_id}  
body: { status: string }  
response: { id, status, reviewed_by, updated_at }  
errors: 404 not found, 409 нельзя перевести в supported без evidence, 422 недопустимый переход статуса  
  
### Decisions  
  
**Создать decision**  
POST /workspaces/{workspace_id}/decisions  
body: { title: string, decision_text: string, rationale?: string, claim_ids?: string[] }  
response: { id, workspace_id, title, decision_text, status: "proposed", rationale, decided_by, created_at }  
errors: 404 workspace not found, 422 один из claim_ids принадлежит другому workspace  
  
**Список decisions в workspace**  
GET /workspaces/{workspace_id}/decisions?status=  
response: { items: [{ id, title, status, decided_at, claims_count }]}  
errors: 404 workspace not found  
  
**Получить одно decision (с claims)**  
GET /decisions/{decision_id}  
response: { id, title, decision_text, status, rationale, decided_by, decided_at, claims: [{ id, text, status }], created_at, updated_at }  
errors: 404 not found  
  
**Принять/изменить статус decision**  
PATCH /decisions/{decision_id}  
body: { status: string, override_reason?: string }  
response: { id, status, decided_by, decided_at, updated_at }  
errors: 404 not found, 409 нельзя accepted без хотя бы одного reviewed claim (если нет override_reason), 422 недопустимый переход статуса  
  
**Экспорт decision brief в Markdown**  
GET /decisions/{decision_id}/export  
response: text/markdown (файл )  
errors: 404  
  
### Research Runs  
  
**Запустить research run**  
POST /research-tasks/{task_id}/research-runs  
body: { question: string, mode: string }  
response: { id, workspace_id, research_task_id, question, status: "queued", mode, started_by, started_at }  
errors: 404 task not found, 422 validation (неверный mode), 429 слишком много активных runs  
  
**Список research runs по task**  
GET /research-tasks/{task_id}/research-runs  
response: { items: [{ id, status, mode, started_at, completed_at }]}  
errors: 404 task not found  
  
**Получить статус run**  
GET /research-runs/{run_id}  
response: { id, status, mode, artifact_manifest_ref, started_at, completed_at, error_message }  
errors: 404 not found  
  
**Отменить run**  
POST /research-runs/{run_id}/cancel  
response: { id, status: "cancelled", completed_at }  
errors: 404 not found, 409 run уже завершён/нельзя отменить  

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