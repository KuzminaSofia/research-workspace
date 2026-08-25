# Задачи На Неделю 25-01

Цель недели: перейти от первого discovery-этапа к понятным MVP contracts и implementation backlog без спешки с кодом.

Frontend/fullstack уже подготовил F1-F4: user journey, screens, UI state map и claim card. Backend в начале недели закрывает review comments по B1-B4, после этого обе дорожки работают параллельно: backend уточняет contract/data boundaries, fullstack переводит принятые экраны в план реализации.

Эта неделя является мягким входом в Неделю 2 из общего roadmap. Если contract/backlog будут готовы раньше, можно начать skeleton implementation небольшим draft PR. Если нет, достаточно качественно закрыть документы и open questions: для MVP важнее синхронизировать backend/frontend contract, чем rushed skeleton.

## Backend Intern

### B5. Закрыть Review Comments В PR #2

Зачем:

- привести B1-B4 к точной архитектурной формулировке;
- убрать места, где API contract случайно вводит новые domain rules;
- подготовить backend contract, на который сможет опереться fullstack.

Что сделать:

- переписать определение `domain` через business entities, invariants, domain rules и policies;
- описать `reconciler` через конкретный remote job flow: какое состояние получает, что проверяет, что обновляет, когда считает job завершенной;
- у всех list endpoints явно показать response как коллекцию, например `{ items: [...] }`;
- заменить `string` в status/type/mode fields на конкретные enum names и allowed values;
- убрать `reviewed_by`, `decided_by`, `started_by`, `created_by` из request body там, где backend должен брать actor из authenticated user context;
- добавить validation/error cases для scope integrity: нельзя связывать Claim/Evidence/Decision из разных workspaces или research tasks;
- не вводить правило "Decision нельзя создать без claims", если оно не зафиксировано как domain decision;
- исправить B4: accepted decision требует reviewed claim или explicit human override; outdated claim дает review signal/candidate; agent draft сохраняется как draft/unreviewed, но не считается accepted knowledge.

Где хранится:

- `docs/backend/b-questions.md` в PR #2.

Готово, когда:

- все GitHub review comments обработаны;
- в PR оставлен короткий summary-комментарий, что именно изменилось;
- документ можно использовать как вход для `api-contract-v1.md`.

### B6. Backend API Contract V1

Зачем:

- дать frontend понятные request/response shapes;
- зафиксировать enum values, error cases и actor rules до начала implementation;
- отделить transport-level API от domain rules.

Что сделать:

- описать endpoints для Workspaces, Research Tasks, Sources, Evidence, Claims, Decisions, Research Runs;
- для каждого list endpoint использовать единый collection envelope: `{ items: [...] }`;
- в начале документа вынести enum definitions: `ResearchTaskStatus`, `SourceType`, `ClaimStatus`, `DecisionStatus`, `ResearchRunStatus`, `ResearchRunMode`, `UpdateType`;
- отдельно описать правило actor fields: клиент не передает `created_by`, `reviewed_by`, `decided_by`, `started_by`; backend берет их из authenticated context;
- отдельно описать scope integrity checks для связей:
  - Claim -> Evidence;
  - Decision -> Claim;
  - Decision -> ResearchTask;
- в конце оставить `Open Questions`, если domain decision еще не принят.

Где хранится:

- `docs/backend/api-contract-v1.md`.

Готово, когда:

- fullstack может по документу понять, какие данные нужны каждому экрану;
- нет placeholder'ов вроде `status=и тут что-то`;
- request body не принимает actor fields от клиента.

### B7. Backend Skeleton Plan

Зачем:

- подготовить реализацию на следующую итерацию без архитектурного дрейфа;
- заранее определить package layout, migrations, repositories и первые use cases.

Что сделать:

- предложить структуру backend package по слоям: `domain`, `application`, `infrastructure`, `api`;
- перечислить первые migrations/tables для MVP;
- описать первые repositories и Unit of Work boundary;
- выбрать первый read/write flow для реализации: Workspace CRUD + Research Task CRUD;
- указать, какие domain tests нужны до API implementation.

Где хранится:

- `docs/backend/backend-skeleton-plan.md`.

Готово, когда:

- понятно, с каких файлов начинать backend implementation;
- видно, какие правила должны жить в domain tests;
- API layer остается тонким и не содержит business logic.

Опционально, если B5-B7 готовы без спешки:

- создать draft PR с минимальным backend package layout;
- не реализовывать весь CRUD, если contract еще обсуждается.

## Fullstack Intern

### F5. Frontend Implementation Backlog

Зачем:

- перевести принятые F1-F4 документы в конкретные implementation tasks;
- подготовить работу так, чтобы frontend не ждал весь backend, а мог стартовать на mock data.

Что сделать:

- разбить MVP UI на первые implementation chunks:
  - app shell;
  - Workspace Overview;
  - Research Task Page;
  - Source/Evidence Panel;
  - Claim Review Panel;
  - Decision Page;
  - Follow-up Question Area;
- для каждого chunk указать expected components, states и минимальный success criterion;
- отдельно отметить, какие части можно делать на mock data до готовности backend.

Где хранится:

- `docs/fullstack/implementation-backlog.md`.

Готово, когда:

- понятно, какие UI tasks можно брать по одной;
- у каждой задачи есть небольшой deliverable, который можно проверить в PR.

### F6. UI To API Mapping

Зачем:

- связать экраны с backend contract;
- заранее найти места, где UI требует данных, которых пока нет в domain/API docs.

Что сделать:

- для каждого из 6 экранов указать:
  - какие entities нужны;
  - какие endpoints нужны;
  - какие fields отображаются;
  - какие loading/error/empty/review states зависят от backend;
- вынести contract dependencies и open questions:
  - review status для `EvidenceItem`;
  - источник `Decision confidence`;
  - что делает кнопка `Send to Review`;
  - точный критерий `reviewed claims` для Accept.

Где хранится:

- `docs/fullstack/ui-api-mapping.md`.

Готово, когда:

- backend intern может открыть документ и понять, какие fields нужны frontend;
- open questions сформулированы как вопросы к contract, а не как случайные UI-допущения.

### F7. Mock Data And State Fixtures

Зачем:

- дать frontend возможность собирать экраны до готовности backend;
- сделать UI states проверяемыми, а не только описанными в Markdown.

Что сделать:

- описать mock objects для Workspace, ResearchTask, Source, EvidenceItem, Claim, Decision, ResearchRun;
- для каждого основного экрана подготовить fixture names:
  - empty;
  - loading;
  - error;
  - success;
  - draft/review;
- проверить, что mock data не противоречит `docs/architecture/04-domain-model.md`.

Где хранится:

- `docs/fullstack/mock-data-contract.md`.

Готово, когда:

- frontend может реализовывать экраны на предсказуемых fixtures;
- backend и frontend одинаково называют statuses и fields.

Опционально, если F5-F7 готовы без спешки:

- создать draft PR с app shell или Workspace Overview на mock data;
- не ждать backend, но и не фиксировать случайные API-допущения в коде.

## Совместная Задача

### Contract Walkthrough В Конце Недели

Зачем:

- синхронизировать backend и fullstack до начала реализации skeleton;
- поймать contract gaps раньше, чем они станут конфликтами в коде.

Что подготовить:

- backend показывает `api-contract-v1.md` и `backend-skeleton-plan.md`;
- fullstack показывает `implementation-backlog.md`, `ui-api-mapping.md` и `mock-data-contract.md`;
- вместе фиксируем unresolved open questions отдельным списком.

Где хранится итог:

- `docs/interns/06-week-2026-08-25-assignments.md`;
- новые backend/fullstack deliverable files из задач B6-B7 и F5-F7.

Готово, когда:

- backend и fullstack используют одни и те же names/statuses/fields;
- skeleton implementation можно начинать без пересогласования базовых contracts.
