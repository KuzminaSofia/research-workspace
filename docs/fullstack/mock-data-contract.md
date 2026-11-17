# Mock Data & State Fixtures (F7)

**Зачем:** дать frontend возможность собирать все 6 MVP-экранов до готовности backend, и сделать UI states (`Empty` / `Loading` / `Error` / `Success` / `Draft-review`) проверяемыми фикстурами, а не только описанными в Markdown.

**Правило для этого документа:** ни один mock object не вводит новое domain-поле, новый enum-значение статуса или новое правило, которых нет в `04-domain-model.md` или в подтверждённом backend contract из `ui-api-mapping.md` (F6). Decision confidence, Decision Open Questions, Claim review note и EvidenceItem review status уже подтверждены как backend contract в F6 (пока не синхронизированы с `04-domain-model.md`) и больше не считаются UI-only mock. Там, где UI всё ещё показывает данные, не подтверждённые ни domain model, ни F6, фикстура помечена как **UI-only mock, не контракт** — см. раздел "Поля вне domain model" в конце документа.

---

## Соглашение об именовании

### Entity-level fixtures
Файлы: `packages/contracts/fixtures/<entity>.fixtures.ts` — по одному файлу на сущность, экспортируют именованные fixture-объекты по shape'у из `04-domain-model.md`.

Имя fixture'ы = `<entityCamelCase>Fixtures.<variant>`, где `<variant>` — либо конкретное значение domain-статуса (`draft`, `supported`, `queued`, ...), либо описательное имя для сущностей без статуса (`Source`, `EvidenceItem`, `Workspace`).

### Screen-level fixtures
Файлы: `packages/contracts/fixtures/screens/<screen>.fixtures.ts` — по одному файлу на экран (совпадает с chunk'ами 1–6 из `implementation-backlog.md`). Каждый файл собирает screen-level fixture из entity-level fixtures.

Имя fixture'ы = `<screenCamelCase>.<state>`, где `<state>` — одно из пяти: `empty`, `loading`, `error`, `success`, `draftReview`. Если у состояния есть обязательные варианты (например, `Loading` на Research Task Page = `queued` **и** `running`, `Error` = `failed` **и** `cancelled` — оба варианта зафиксированы как обязательные в `UI_State_Map.md` и `implementation-backlog.md`), это отдельные подключи: `<screenCamelCase>.loading.queued`, `<screenCamelCase>.loading.running` и т.д. Ключ `<state>` без суффикса всегда указывает на дефолтный/основной вариант этого состояния.

Ровно те же имена (`empty` / `loading` / `error` / `success` / `draftReview`) используются в тест-файлах и в Storybook-стори — чтобы состояние можно было найти поиском и свести 1:1 с `UI_State_Map.md`, как уже сделано для заголовков `#### State: <Имя>` в `implementation-backlog.md`.

---

## 1. Entity-level mock objects

### Workspace

```
Workspace {
  id
  title
  description
  owner_id
  created_at
  updated_at
}
```

У `Workspace` нет собственного enum-статуса — нужна одна репрезентативная фикстура.

| Fixture | Назначение |
|---|---|
| `workspaceFixtures.default` | единственный Workspace, используется во всех screen-level fixture'ах как контейнер |

### ResearchTask

```
ResearchTask {
  id
  workspace_id
  title
  research_question
  status: draft | in_progress | review | completed | archived
  summary
  created_by
  created_at
  updated_at
}
```

| Fixture | `status` | Назначение |
|---|---|---|
| `researchTaskFixtures.draft` | `draft` | покрытие enum-значения, не показано на текущих wireframe'ах |
| `researchTaskFixtures.inProgress` | `in_progress` | покрытие enum-значения |
| `researchTaskFixtures.review` | `review` | Workspace Overview `Draft-review` (бейдж "Review" у "Market analysis") |
| `researchTaskFixtures.completed` | `completed` | Workspace Overview `Success` ("Technology evaluation") |
| `researchTaskFixtures.archived` | `archived` | покрытие enum-значения, не показано на текущих wireframe'ах |

Все 5 значений `ResearchTask.status` должны присутствовать как отдельные фикстуры — это прямое требование `Success criterion` Chunk 1 в `implementation-backlog.md` ("отдельно нужно проверить рендер всех 5 значений `ResearchTask.status`... не только те два, что видны на самом wireframe").

### Source

```
Source {
  id
  workspace_id
  research_task_id
  type: url | manual_note | pdf | doc | github | video
  title
  url
  author
  published_at
  captured_at
  summary
  raw_content_ref
  reliability_note
}
```

| Fixture | `type` | Назначение |
|---|---|---|
| `sourceFixtures.url` | `url` | основной вариант, используется в Success-фикстурах экранов 2 и 3 |
| `sourceFixtures.pdf` | `pdf` | покрытие enum-значения `type` |
| `sourceFixtures.githubRepo` | `github` | покрытие enum-значения `type` |
| `sourceFixtures.manualNote` | `manual_note` | покрытие enum-значения `type` (без `url`/`author`, т.к. это заметка, а не внешний источник) |
| `sourceFixtures.brokenSummary` | `url` | `summary` недоступен (пустой/не загрузился) — для Source / Evidence Panel `Error`, т.к. превью строится на `Source.summary`, не на `raw_content_ref` (`ui-api-mapping.md`, F6) |

`type: doc` и `type: video` не заведены отдельными фикстурами в этой версии — при необходимости добавляются по тому же шаблону, что `pdf`/`github`, без изменения shape.

### EvidenceItem

```
EvidenceItem {
  id
  source_id
  excerpt
  location
  note
  confidence
  created_at
}
```

| Fixture | Назначение |
|---|---|
| `evidenceItemFixtures.default` | полностью заполненная карточка для Success-состояний экранов 2 и 3 |
| `evidenceItemFixtures.highConfidence` | `confidence` близко к верхней границе — проверка рендера pill'а |
| `evidenceItemFixtures.lowConfidence` | `confidence` близко к нижней границе — проверка рендера pill'а |

**Важно:** review-статус `EvidenceItem` (`status`, `reviewed_by`, `reviewed_at`) в `04-domain-model.md` пока не описан, но по `ui-api-mapping.md` (F6) это уже backend contract, а не UI-only mock — см. раздел "Поля вне domain model" ниже.

### Claim

```
Claim {
  id
  workspace_id
  research_task_id
  text
  status: draft | supported | weak | conflicting | outdated
  confidence
  created_by
  reviewed_by
  created_at
  updated_at
  // связи: many-to-many EvidenceItem, many-to-many Decision
}
```

| Fixture | `status` | `reviewed_by` | Связанные EvidenceItem | Назначение |
|---|---|---|---|---|
| `claimFixtures.draft` | `draft` | `null` | ≥1 (может быть 0, см. `claimFixtures.draftNoEvidence`) | Claim Review Panel `Draft-review`, Research Task Page `Draft-review` |
| `claimFixtures.draftNoEvidence` | `draft` | `null` | 0 | Claim Review Panel `Empty` — проверка disabled `Supported` |
| `claimFixtures.supported` | `supported` | задан | ≥1 (обязательно, правило "supported невозможен без evidence") | Claim Review Panel / Decision Page `Success` |
| `claimFixtures.weak` | `weak` | задан | ≥1 | Decision Page `Success` (линкованный claim "Weak") |
| `claimFixtures.conflicting` | `conflicting` | задан | ≥1 | покрытие enum-значения |
| `claimFixtures.outdated` | `outdated` | задан | ≥1 | покрытие enum-значения |

Домен-правило "`supported` невозможен без evidence" соблюдено на уровне самих фикстур: не существует fixture'ы `claimFixtures.supported*` с пустым списком evidence.

### Decision

```
Decision {
  id
  workspace_id
  title
  decision_text
  status: proposed | accepted | rejected | superseded | needs_review
  rationale
  decided_by
  decided_at
  created_at
  updated_at
  // связи: many-to-many Claim, many-to-many ResearchTask
}
```

| Fixture | `status` | Linked Claims | Назначение |
|---|---|---|---|
| `decisionFixtures.empty` | `proposed` | `[]` | Decision Page `Empty` (только что создан) |
| `decisionFixtures.proposed` | `proposed` | `supported`, `supported`, `weak` | Decision Page / Workspace Overview `Success` |
| `decisionFixtures.accepted` | `accepted` | `supported`, `supported` | Workspace Overview `Success` |
| `decisionFixtures.rejected` | `rejected` | смешанные | покрытие enum-значения текущего MVP UI scope |
| `decisionFixtures.needsReview` | `needs_review` | включает `claimFixtures.draft` | Decision Page `Draft-review` |
| `decisionFixtures.superseded` | `superseded` | любые | **только для проверки, что `StatusPill` не падает на неожиданном значении**; не используется ни в одном screen-level `success`/`draftReview` fixture'е — статус сознательно вне MVP UI scope (Open Contract Question №10 в `ui-api-mapping.md`) |

`decisionFixtures.superseded` существует как entity-level fixture (значение валидно в domain model), но **не подключается** ни к одному screen-level fixture'у ниже — это соответствует правке из `ui-api-mapping.md`: в Workspace Overview `Success` проверяются только `proposed`/`accepted`/`rejected`/`needs_review`.

Поля `Decision.confidence` в `04-domain-model.md` пока нет, но по `ui-api-mapping.md` (F6) оно уже backend contract, ещё не синхронизированный с domain model.

### ResearchRun

```
ResearchRun {
  id
  workspace_id
  research_task_id
  question
  status: queued | running | review_required | completed | failed | cancelled
  mode: manual_sources | search_assisted | follow_up
  artifact_manifest_ref
  started_by
  started_at
  completed_at
  error_message
}
```

| Fixture | `status` | `mode` | `error_message` | Назначение |
|---|---|---|---|---|
| `researchRunFixtures.queued` | `queued` | `manual_sources` | `null` | Research Task Page `Loading` (вариант 1) |
| `researchRunFixtures.running` | `running` | `manual_sources` | `null` | Research Task Page `Loading` (вариант 2) |
| `researchRunFixtures.reviewRequired` | `review_required` | `manual_sources` | `null` | Research Task Page `Draft-review` |
| `researchRunFixtures.completed` | `completed` | `manual_sources` | `null` | Research Task Page `Success` |
| `researchRunFixtures.failed` | `failed` | `manual_sources` | заполнен | Research Task Page `Error` (вариант 1) |
| `researchRunFixtures.cancelled` | `cancelled` | `manual_sources` | `null` | Research Task Page `Error` (вариант 2) — не ошибка по смыслу, но та же UI-ветка |

Все фикстуры используют `mode: manual_sources`, так как это единственный режим в MVP-скоупе (см. `Экраны.md`, `UI_State_Map.md`). Фикстуры для `mode: search_assisted` не заводятся в этой версии — поведение для этого режима остаётся Open Contract Question (см. `ui-api-mapping.md`, зависимость к экрану 2).

---

## 2. Screen-level fixtures

Для каждого экрана — таблица из 5 fixture-ключей (`empty` / `loading` / `error` / `success` / `draftReview`), с указанием, какие entity-level fixtures в них подставлены, и с явной ссылкой на условие из `04-domain-model.md`/`UI_State_Map.md`, которое это состояние воспроизводит.

### `workspaceOverview` (Chunk 1)

| Fixture key | Состав | Domain condition |
|---|---|---|
| `workspaceOverview.empty` | `workspaceFixtures.default`, `researchTasks: []`, `decisions: []` | нет ResearchTask/Decision |
| `workspaceOverview.loading` | тот же Workspace, списки не резолвлены (fixture возвращается с искусственной задержкой в mock-слое) | — |
| `workspaceOverview.error` | тот же Workspace, запрос списков падает (mock-слой возвращает reject) | — |
| `workspaceOverview.success` | `researchTaskFixtures.review`, `researchTaskFixtures.completed`, `decisionFixtures.accepted` | воспроизводит текущий wireframe экрана 1 |
| `workspaceOverview.draftReview` | включает `researchTaskFixtures.review` в списке задач (совпадает с `success`, т.к. Review-бейдж — часть того же списка, отдельного экрана нет) | `ResearchTask.status = review` |

Дополнительно (не отдельный fixture key, а часть проверки `success`): рендер всех 5 значений `ResearchTask.status` и 4 значений `Decision.status` из MVP UI scope (`proposed`/`accepted`/`rejected`/`needs_review`) через отдельный `StatusPill`-стори, не через сам экран — см. `researchTaskFixtures.*` и `decisionFixtures.*` выше.

### `researchTaskPage` (Chunk 2)

| Fixture key | Состав | Domain condition |
|---|---|---|
| `researchTaskPage.empty` | `researchTaskFixtures.inProgress`, `sources: []`, `run: null`, `evidence: []`, `claims: []` | нет `Source`; `Run Research` заблокирован (`mode: manual_sources`) |
| `researchTaskPage.loading.queued` | `researchRunFixtures.queued`, `sources: [sourceFixtures.url]` | `ResearchRun.status = queued` |
| `researchTaskPage.loading.running` | `researchRunFixtures.running`, `sources: [sourceFixtures.url]` | `ResearchRun.status = running` |
| `researchTaskPage.error.failed` | `researchRunFixtures.failed` | `ResearchRun.status = failed`, показан `error_message` |
| `researchTaskPage.error.cancelled` | `researchRunFixtures.cancelled` | `ResearchRun.status = cancelled`, `error_message` не показывается |
| `researchTaskPage.success` | `researchRunFixtures.completed`, `sourceFixtures.url`, `evidenceItemFixtures.default`, `claimFixtures.supported` | `ResearchRun.status = completed` |
| `researchTaskPage.draftReview` | `researchRunFixtures.reviewRequired`, `sourceFixtures.url`, `evidenceItemFixtures.default`, `claimFixtures.draft` | `ResearchRun.status = review_required` |

`researchTaskPage.loading` и `researchTaskPage.error` без суффикса не существуют как отдельные ключи — оба состояния обязательно двухвариантные (см. `UI_State_Map.md` и `implementation-backlog.md`, Chunk 2).

### `sourceEvidencePanel` (Chunk 3)

| Fixture key | Состав | Domain condition |
|---|---|---|
| `sourceEvidencePanel.empty` | `sourceFixtures.url`, `evidence: []` | нет `EvidenceItem` для этого `Source` |
| `sourceEvidencePanel.loading` | те же данные, не резолвлены (искусственная задержка) | — |
| `sourceEvidencePanel.error` | `sourceFixtures.brokenSummary` | `Source.summary` недоступен |
| `sourceEvidencePanel.success` | `sourceFixtures.url`, `evidenceItemFixtures.default`, связанный `claimFixtures.supported` | полностью заполненная карточка |
| `sourceEvidencePanel.draftReview` | тот же состав, что `success`; Approve/Reject/Edit меняют `EvidenceItem.status`/`reviewed_by`/`reviewed_at` — backend contract по `ui-api-mapping.md` (F6) | см. раздел "Поля вне domain model" |

### `claimReviewPanel` (Chunk 4)

| Fixture key | Состав | Domain condition |
|---|---|---|
| `claimReviewPanel.empty` | `claimFixtures.draftNoEvidence` | Supporting Evidence пуст, `Supported` задизейблен |
| `claimReviewPanel.loading` | `claimFixtures.draft`, данные не резолвлены | — |
| `claimReviewPanel.error` | два обязательных под-случая (см. ниже) | — |
| `claimReviewPanel.error.statusSaveFailed` | `claimFixtures.draft`, mock-слой отклоняет `PATCH status` | ошибка сохранения `Claim.status`, optimistic rollback |
| `claimReviewPanel.error.reviewNoteSaveFailed` | `claimFixtures.draft`, mock-слой отклоняет сохранение review note | текст остаётся в поле, не считается сохранённым |
| `claimReviewPanel.success` | `claimFixtures.supported`, ≥1 `evidenceItemFixtures.default` | заполненный review, reviewer указан |
| `claimReviewPanel.draftReview` | `claimFixtures.draft`, ≥1 `evidenceItemFixtures.default` | `Claim.status = draft` |

`claimReviewPanel.error` без суффикса используется как алиас на `claimReviewPanel.error.statusSaveFailed` там, где тесту нужен один дефолтный error-fixture; для полного покрытия обоих случаев из F6-правки (сохранение статуса **и** сохранение review note) используются оба под-ключа.

### `decisionPage` (Chunk 5)

| Fixture key | Состав | Domain condition |
|---|---|---|
| `decisionPage.empty` | `decisionFixtures.empty` | Linked Claims пуст, `Accept` недоступен ("нет claims") |
| `decisionPage.loading` | `decisionFixtures.proposed`, действие в процессе отправки (флаг в mock-слое) | Accept/Send to Review/Reject в процессе |
| `decisionPage.error` | `decisionFixtures.proposed`, mock-слой отклоняет запрос смены статуса | статус остаётся прежним |
| `decisionPage.success` | `decisionFixtures.proposed` (linked: `supported`, `supported`, `weak`) | воспроизводит wireframe экрана 5 |
| `decisionPage.draftReview` | `decisionFixtures.needsReview` (linked включает `claimFixtures.draft`) | `Decision.status = needs_review` |

`decisionFixtures.superseded` намеренно не используется ни в одном из ключей выше (см. Entity-level таблицу `Decision`).

### `followUpQuestionArea` (Chunk 6)

Эта секция встроена в Workspace Overview (не отдельный route), поэтому её fixtures — это ответ mock Ask-эндпоинта, а не отдельная страница.

| Fixture key | Состав | Domain condition |
|---|---|---|
| `followUpQuestionArea.empty` | `sourcesCount: 0, claimsCount: 0, decisionsCount: 0`, ответа ещё нет | Workspace без накопленных Source/Claim/Decision |
| `followUpQuestionArea.loading` | запрос отправлен, ответ не резолвлен | после клика **Ask** |
| `followUpQuestionArea.error` | mock-слой отклоняет запрос ответа | ошибка генерации ответа |
| `followUpQuestionArea.success` | `answerText`, ссылки заполнены, `memorySufficient: true`, ненулевые счётчики | `Memory sufficient? = Yes` |
| `followUpQuestionArea.draftReview` | `answerText`, `memorySufficient: false` | `Memory sufficient? = No`, показан `Create Follow-up Task` |

**Важно:** `answerText`, счётчики и `memorySufficient` подтверждены как backend contract в `POST /workspaces/{workspace_id}/ask` (`ui-api-mapping.md`, F6: `answer_text`, `sources_count`, `claims_count`, `decisions_count`, `memory_sufficient`) — это больше не UI-only mock, хотя как entity в `04-domain-model.md` они не описаны.

---

## 3. Проверка на непротиворечивость domain model

| Проверка | Результат |
|---|---|
| Все статусные значения фикстур (`ResearchTask.status`, `ResearchRun.status`, `Claim.status`, `Decision.status`) взяты только из enum'ов `04-domain-model.md` | ✅ ни одна фикстура не вводит значение вне списка |
| `claimFixtures.supported*` всегда имеет ≥1 связанный `EvidenceItem` | ✅ соблюдает правило "Claim без evidence не может быть supported" |
| Ни у одной `claimFixtures.*`, созданной "агентом" (т.е. использованной в `Draft-review`/`Empty` состояниях до review), `status` не установлен в отличное от `draft` значение по умолчанию | ✅ соблюдает правило "AI-generated claim стартует как draft" |
| `decisionFixtures.needsReview` использует `claimFixtures.draft` среди Linked Claims, но не задаёт логику, при каком составе Accept разблокируется | ✅ соответствует статусу открытого вопроса — критерий "reviewed claims" не решён, фикстура его не решает за backend |
| `EvidenceItem.status`/`reviewed_by`/`reviewed_at`, `Claim.review_note`, `Decision.confidence`/`open_questions` | ✅ подтверждены как backend contract в `ui-api-mapping.md` (F6); в `04-domain-model.md` пока не синхронизированы |
| `decisionFixtures.superseded` существует, но не используется в screen-level `success`/`draftReview` | ✅ соответствует правке F6: `superseded` вне MVP UI scope |
| `researchRunFixtures.*` используют только `mode: manual_sources` | ✅ единственный режим в MVP-скоупе, `search_assisted` оставлен как open question |

---

## 4. Поля вне domain model (UI-only mock, не контракт)

Эти значения нужны экранам, но пока не подтверждены ни в `04-domain-model.md`, ни как backend contract в `ui-api-mapping.md` (F6). Они замоканы отдельно от entity-shape'ов из раздела 1, чтобы не создавать у backend/frontend впечатление, что это уже согласованный API-контракт. Полный список открытых вопросов — в `docs/fullstack/ui-api-mapping.md`, здесь только то, что затрагивает fixtures.

Evidence review result, `Decision confidence`/`Open Questions`, `Claim` review note и данные `/ask` из этой таблицы убраны: по F6 это уже backend contract, а не UI-only mock (см. примечания к соответствующим entity-level и screen-level fixtures в разделах 1–2 выше).

| UI-only mock значение | Экран | Соответствующий Open Contract Question |
|---|---|---|
| Семантика `Send to Review` (mock — no-op с toast) | Decision Page | №3 |
| Критерий разблокировки `Accept` для `needs_review` (mock не решает, кнопка остаётся с пояснительным текстом) | Decision Page | №4 |
| `reviewed_by`/`decided_by`/`created_by` отображаются как готовые display-имена (например, "Alex Ivanov") вместо резолва по ID | Claim Review Panel, Decision Page | №9 (нет сущности `User`) |
| `Decision.status = superseded` | — (не подключён ни к одному screen fixture) | №10 |

Если backend/API contract по любому из этих пунктов будет уточнён, соответствующий mock переносится из этого раздела в раздел 1 как обычное entity-поле, а fixture-имена не меняются — меняется только источник данных внутри mock-слоя.

---

## Критерий готовности этого документа
- frontend может подключить `packages/contracts/fixtures/*` и `packages/contracts/fixtures/screens/*` и реализовать каждый из 6 экранов на предсказуемых, поимённо совпадающих с `UI_State_Map.md`/`implementation-backlog.md` fixtures, не дожидаясь backend.
- backend и frontend называют статусы и поля одинаково: все entity-level fixture-поля 1:1 совпадают по имени и допустимым значениям с `04-domain-model.md`; всё, чего в domain model нет, явно вынесено в раздел "Поля вне domain model" и не выдаётся за согласованный контракт.