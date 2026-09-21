# Frontend Implementation Backlog

Источник: `01-product-vision.md`, `02-system-architecture.md`, `03-backend-layered-architecture.md`,
`04-domain-model.md`, `User_Journey.md`, `Экраны.md`, `UI_State_Map.md`, Figma low-fidelity
wireframes ("AI Research & Decision Workspace — Low-Fidelity Wireframes").

Цель backlog'а: разбить MVP UI на chunks, которые можно брать по одному, с mock data там,
где backend ещё не готов, и с минимальным проверяемым deliverable для каждого PR.

## Как пользоваться этим backlog'ом

- Каждый chunk — отдельная ветка / PR.
- Chunk не блокируется backend'ом, если он помечен **Mock: да**. В этом случае данные
  берутся из `packages/contracts` shape'ов, замоканных локально во frontend (fixtures/MSW/in-memory),
  без реального API вызова.
- Domain-статусы (`ResearchTask.status`, `Claim.status`, `Decision.status`, `ResearchRun.status`)
  везде берутся строго из `04-domain-model.md`, а не изобретаются на UI.
- **Каждый chunk (кроме App Shell) содержит ровно 5 явно подписанных состояний из
  `UI_State_Map.md`: `State: Empty`, `State: Loading`, `State: Error`, `State: Success`,
  `State: Draft-review`.** Формат подписи везде одинаковый — заголовок `#### State: <Имя>`,
  чтобы состояние было легко найти поиском/сверить один к одному с `UI_State_Map.md`, а не
  распознавать его из свободного текста. Если для какого-то состояния в конкретном chunk нет
  отдельной визуальной ветки (например, оно эквивалентно другому состоянию), это написано явно
  внутри самого подписанного блока, а не пропуском заголовка.
- Open questions из `Экраны.md` и `UI_State_Map.md` (Decision confidence source, семантика
  Send to Review, критерий "reviewed claims" для Accept, отсутствие review-полей у `EvidenceItem`,
  `Decision.status = superseded`) оформлены в соответствующих chunks как явный раздел
  **Open Contract Questions** (вопрос к backend/domain/API contract + `Current UI behavior` +
  `Blocked decision`), а не как принятое поведение или "не входит в scope" без объяснения — это
  сделано, чтобы не блокировать frontend старт, но и не выдавать открытый вопрос за решение.
- `Empty`, `Loading`, `Error`, `Success`, `Draft-review` — это UI states из `UI_State_Map.md`, а
  не дополнительные domain enum. Frontend не добавляет эти значения в `packages/contracts` или в
  domain model. Domain-статусы (`ResearchTask.status`, `ResearchRun.status`, `Claim.status`,
  `Decision.status`) берутся только из `04-domain-model.md`; там, где конкретный UI state
  обусловлен конкретным domain-статусом, это указано явно (например, "UI State: Error, Domain
  condition: `ResearchRun.status = failed`"), но это не делает `Error` частью
  `ResearchRun.status`.

---

## Chunk 0 — App Shell

**Что это:** общий layout, который переиспользуют все экраны (header, навигация между
Workspace Overview / Research Task Page, route skeleton).

**Expected components:**
- `AppHeader` (название workspace/раздела, breadcrumb-подобный подзаголовок — виден на
  экранах 02–06, напр. "Research Task / Research Workspace");
- `PageContainer` / `PageTitle` (заголовок + подзаголовок страницы — паттерн повторяется на
  экранах 01, 04, 05, 06);
- `StatusPill` (переиспользуемый компонент бейджа статуса — используется на всех 6 экранах:
  task status, run status, claim status, decision status, memory sufficiency);
- `Card` (базовый контейнер секции — используется на всех экранах);
- `Button` (primary/secondary/ghost варианты — Create, Add, Run, Ask, Approve/Reject и т.д.);
- `EmptyState`, `LoadingSkeleton`, `ErrorState` — три переиспользуемых UI-kit компонента, которые
  каждый из chunks 1–6 использует для своих `State: Empty` / `State: Loading` / `State: Error`,
  чтобы не изобретать разметку заново в каждом экране.
- базовый роутинг между Workspace Overview и Research Task Page.

**State: Not applicable.** Этот chunk — UI-kit, а не экран с данными, поэтому 5-state модель
(`Empty` / `Loading` / `Error` / `Success` / `Draft-review`) к нему не применяется напрямую.
Вместо этого проверяется, что каждый из трёх компонентов (`EmptyState`, `LoadingSkeleton`,
`ErrorState`) достаточно универсален (заголовок + описание + опциональная CTA-кнопка /
Retry-кнопка), чтобы его переиспользовали все 6 экранов без кастомной разметки под каждый
конкретный случай. `StatusPill` должен поддерживать весь набор enum-значений, которые
встречаются в domain model: `ResearchTask.status`, `Claim.status`, `Decision.status`,
`ResearchRun.status`, плюс кастомный "Memory sufficient? / Insufficient" со экрана 6.

**Success criterion:**
- `AppHeader`, `PageContainer`, `StatusPill`, `Card`, `Button`, `EmptyState`, `LoadingSkeleton`,
  `ErrorState` собраны как переиспользуемые компоненты (Storybook-страница или демонстрационный
  route) и хотя бы один реальный экран (Workspace Overview) их использует вместо inline-разметки.

**Mock:** да, полностью — это чистый UI-kit, backend не нужен.

---

## Chunk 1 — Workspace Overview

**Экран:** `01 Workspace Overview` (главная точка входа, шаг 2 и точка возврата после шага 9
user journey).

**Expected components:**
- `WorkspaceHero` (название workspace + одна строка описания);
- `ResearchTaskList` + `ResearchTaskCard` (title, research_question, `StatusPill` со статусом
  task, клик → Research Task Page);
- `btn-create-task` (Create Research Task — открывает форму/модалку, форма может быть заглушкой
  в этом chunk);
- `DecisionList` + `DecisionCard` (title, короткий rationale-тизер, `StatusPill` decision status,
  клик → Decision Page);
- `AskWorkspaceCard` — встроенный вход в Follow-up Question Area (инпут + кнопка Ask; сама логика
  ответа — в Chunk 6, здесь только точка входа/переход).

**States (источник: `UI_State_Map.md`, раздел "1. Workspace Overview"):**

#### State: Empty
Workspace только что создан — ни одной `ResearchTask`, ни одного `Decision` ещё не существует.
Вместо списков показывается placeholder-сообщение ("No research tasks yet") с призывом нажать
**Create Research Task**. Секция **Ask Workspace** остаётся видимой, но с подсказкой, что
задавать вопросы пока не о чем.

#### State: Loading
Первичная загрузка списков Research Tasks и Decisions при открытии workspace — вместо карточек
показываются skeleton-плейсхолдеры (`LoadingSkeleton` из Chunk 0).

#### State: Error
Не удалось получить данные workspace (сетевая ошибка / API недоступен) — вместо списков
показывается сообщение об ошибке и кнопка **Retry** (`ErrorState` из Chunk 0).

#### State: Success
Текущий wireframe (экран 1) — заполненные списки Research Tasks (Market analysis / Review,
Technology evaluation / Completed) и одно Decision со статусом Accepted. Отдельно нужно
проверить рендер всех 5 значений `ResearchTask.status` и всех 5 значений `Decision.status`
в `StatusPill` (не только те два, что видны на самом wireframe).

#### State: Draft-review
`ResearchTask.status = review` отображается бейджем **Review** прямо в списке (уже видно на
макете у "Market analysis"). Это сигнализирует пользователю, что task ждёт его внимания, не
блокируя переход в остальные разделы workspace.

**Success criterion:**
- на mock-данных (список из 2–3 task, 1–2 decision) страница рендерит `State: Success` целиком;
  отдельными mock-наборами проверяются `State: Empty`, `State: Loading`, `State: Error`.

**Mock:** да. Данные — фикстуры по shape `ResearchTask` и `Decision` из `04-domain-model.md`
(без реального API); `State: Loading` / `State: Error` эмулируются искусственной
задержкой/флагом в mock-слое.

---

## Chunk 2 — Research Task Page

**Экран:** `02 Research Task Page` (покрывает шаги 3–7 user journey одной страницей:
Sources → Run → Evidence → Claims → Summary/Create Decision).

**Expected components:**
- `TaskHeader` (title, question, `StatusPill` task status, `btn-run-research`);
- `SourcesSection` + `SourceCard` (title, "URL · Author · Published date · Reliability note",
  `btn-view`, `btn-add-source`);
- `ResearchRunCard` (текущий статус run по `ResearchRun.status`, счётчик "N run(s)");
- `EvidenceColumn` + `EvidenceCardCompact` (excerpt + `btn-review` → Source/Evidence Panel);
- `ClaimsColumn` + `ClaimCardCompact` — двухблочная карточка:
  - верх: status pill, confidence, текст claim, linked evidence excerpt + confidence
    (соответствует many-to-many Claim↔EvidenceItem из domain model);
  - низ: панель review-статусов (`Supported / Weak / Conflicting / Outdated`, возврат в `Draft` —
    единый контрол, как явно указано в `Экраны.md`) + ссылка "Open full review →" на Claim Review
    Panel;
- `SummarySection` (summary text + `btn-create-decision`, ведёт на Decision Page).

**States (источник: `UI_State_Map.md`, раздел "2. Research Task Page"; здесь состояния
завязаны в первую очередь на `ResearchRun.status`):**

#### State: Empty
Секция **Sources** пуста — ни один source ещё не добавлен. Поведение `btn-run-research`
зависит от `ResearchRun.mode`: для `mode = manual_sources` (единственный режим MVP) кнопка
заблокирована/с подсказкой "Add at least one source to run research". Поведение для
`mode = search_assisted` не определено в этом chunk — см. **Open Contract Questions** ниже.
Секции Evidence / Claims / Summary тоже пусты, пока активен только `manual_sources`.

#### State: Loading
`ResearchRun.status = queued` или `running`. Блок Research Run показывает соответствующий бейдж
("Queued" / "Running"), секции Evidence и Claims показывают заглушку "Waiting for run to
complete" вместо карточек.

#### State: Error
Два обязательных варианта:
- `ResearchRun.status = failed` — бейдж **Failed** + текст из `ResearchRun.error_message`,
  кнопка **Run Research** заменяется на **Retry Run**;
- `ResearchRun.status = cancelled` — та же ветка UI, но другая семантика (осознанная отмена,
  не сбой): бейдж **Cancelled**, кнопка **Re-run**, `error_message` не показывается.

#### State: Success
Текущий wireframe (экран 2) — `ResearchRun.status = completed`, Evidence и Claims заполнены
карточками, доступна кнопка **Create Decision** в блоке Summary.

#### State: Draft-review
`ResearchRun.status = review_required` — именно это состояние показано на текущем макете
(бейдж "Review Required" на Research Run, бейдж "Draft" на карточке Claim). Пользователю нужно
перейти в Evidence/Claim панели и подтвердить результаты, прежде чем они станут доверенным
знанием.

### Open Contract Questions

- **Search-assisted empty state**: Как должен вести себя `State: Empty` секции Sources и
  кнопка `btn-run-research`, когда `ResearchRun.mode = search_assisted` и sources ещё не
  добавлены заранее?
  - Current UI behavior: в этом chunk реализуется и покрывается тестами только
    `mode = manual_sources`; ветка для `search_assisted` не реализуется и не мокается.
  - Blocked decision: нельзя фиксировать, разрешён ли запуск run без sources для
    `search_assisted`, и какой текст/состояние показывать вместо "Add at least one source to
    run research", пока это не уточнено в domain/API contract.

**Success criterion:**
- на mock task с 1 source, 1 run (`review_required`), 1 evidence, 1 claim (`draft`) страница
  рендерит `State: Draft-review` целиком без обращения к backend; отдельными mock-фикстурами
  проверяются `State: Empty` (нет source), `State: Loading` (`queued`/`running`), `State: Error`
  (`failed` и `cancelled` отдельно) и `State: Success` (`completed`).

**Mock:** да, для всех данных чтения. Кнопки `Run Research` / `Add Source` в этом chunk могут
быть no-op (заглушка клика, без реального запуска agent run) — интеграция с реальным
`POST /research-runs` не входит в scope этого chunk.

---

## Chunk 3 — Source / Evidence Panel

**Экран:** `03 Source Evidence Panel` (шаг 5 user journey — Review Evidence).

**Expected components:**
- `SourcePreviewCard` (Source: тип/название, meta — домен/путь, published date, content preview,
  `btn-open-source` со ссылкой на оригинал);
- `EvidenceReviewCard`:
  - excerpt текста evidence;
  - три info-pill: Confidence, Location, Note (все поля `EvidenceItem` из domain model —
    явно требуется в `Экраны.md`, что все поля видны в самой карточке evidence, а не в source
    preview);
  - review-вопрос ("Does this evidence accurately represent the source?");
  - `btn-approve` / `btn-reject` / `btn-edit`;
  - строка "Linked Claim: …" (связь Evidence → Claim).

**States (источник: `UI_State_Map.md`, раздел "3. Source / Evidence Panel"):**

#### State: Empty
Для данного source ещё не извлечено ни одного `EvidenceItem` (run ещё выполняется или
завершился без находок по этому источнику) — вместо карточки Evidence Item показывается
сообщение "No evidence extracted yet from this source".

#### State: Loading
Идёт загрузка содержимого source (`Source.summary`) или конкретного evidence item — панель
Source и панель Evidence Item показывают skeleton вместо текста.

#### State: Error
Не удалось загрузить содержимое source (`Source.summary` недоступен) — вместо
превью показывается сообщение об ошибке; кнопка **Open original source** остаётся активной как
fallback-путь для ручной проверки.

#### State: Success
Текущий wireframe (экран 3) — Source и Evidence Item полностью заполнены: excerpt, Confidence,
Location, Note присутствуют в карточке Evidence Item, действия Approve/Reject/Edit доступны.

#### State: Draft-review
Наличие кнопок **Approve / Reject / Edit** само по себе отражает продуктовый принцип "только
проверенные человеком evidence... становятся знанием команды": evidence item, извлечённый
агентом, должен быть явно подтверждён человеком. Точный способ, которым результат
Approve/Reject должен становиться частью domain-состояния, не определён — см.
**Open Contract Questions** ниже.

### Open Contract Questions

`04-domain-model.md` не определяет у `EvidenceItem` review-статус (аналог `Claim.status`) и не
определяет audit-поля вроде `reviewed_by` / `reviewed_at`. Из этого следуют открытые вопросы:

- **Evidence review status**: Нужен ли отдельный review-status для `EvidenceItem`, аналогичный
  `Claim.status`, и если да — какие значения он должен принимать?
  - Current UI behavior: Approve/Reject/Edit меняют только локальное UI-состояние карточки
    (например, визуальное выделение "approved"/"rejected" в рамках текущей сессии).
  - Blocked decision: нельзя вводить на фронте новый domain enum или помечать это состояние
    как domain status `EvidenceItem`, пока статус не появится в domain model.
- **Evidence review persistence**: Как backend должен персистить результат Approve / Reject /
  Edit для `EvidenceItem`, учитывая отсутствие review-status и audit-полей?
  - Current UI behavior: результат клика не сохраняется между перезагрузками страницы, никакой
    `PATCH`/`PUT` вызов не выполняется.
  - Blocked decision: нельзя предполагать конкретный endpoint или payload для persist-логики.
- **Evidence review audit**: Нужны ли `reviewed_by`, `reviewed_at` или эквивалентные
  audit-поля для `EvidenceItem`?
  - Current UI behavior: `ReviewerMeta`-подобный блок для Evidence в этом chunk не показывается,
    так как источника данных для него нет.
  - Blocked decision: нельзя рендерить `reviewed_by`/`reviewed_at` как если бы это уже были
    поля `EvidenceItem`.
- **Evidence review API**: Какой endpoint/mutation contract будет использоваться для сохранения
  результата review, когда он появится?
  - Current UI behavior: кнопки Approve/Reject/Edit вызывают no-op заглушку вместо реального
    API-вызова.
  - Blocked decision: нельзя фиксировать конкретный контракт запроса (метод, путь, тело) в этом
    chunk.

**Success criterion:**
- на mock evidence + связанном mock source страница показывает `State: Success` (все поля
  `EvidenceItem` и source meta одновременно, side-by-side, как в макете); Approve/Reject
  переводят карточку в `State: Draft-review` UI-состояние (без API); отдельными фикстурами
  проверяются `State: Empty` (нет evidence для source) и `State: Error` (source content
  недоступен).

**Mock:** да. Реальная persist-логика (`PATCH` review-статуса) не входит — только UI-состояние.

---

## Chunk 4 — Claim Review Panel

**Экран:** `04 Claim Review Panel` (шаг 6 user journey — Validate Claims).

**Expected components:**
- `ClaimSummaryCard` (текст claim, `StatusPill` status, confidence pill);
- `SupportingEvidenceList` + `EvidenceListItem` (source name + excerpt + "Open Source" pill,
  может показывать несколько evidence — domain model допускает many-to-many Claim↔Evidence);
- `ReviewNoteField` (textarea + `btn-save`);
- `SetStatusPanel` (правый сайдбар): кнопки `Supported / Weak / Conflicting / Outdated / Draft` —
  единый набор, взаимоисключающий выбор (radio-подобное поведение, не multi-select);
- `ReviewerMeta` (Reviewer: …, Status: …).

**States (источник: `UI_State_Map.md`, раздел "4. Claim Review Panel"):**

#### State: Empty
Секция Supporting Evidence пуста — у claim ещё нет привязанных evidence items. По правилу
"Claim без evidence не может быть `supported`" кнопка **Supported** в этом состоянии
задизейблена/помечена как недоступная, с подсказкой "Link evidence to mark as Supported".

#### State: Loading
Идёт загрузка claim и связанных evidence items при открытии панели — карточки Claim и
Supporting Evidence показывают skeleton.

#### State: Error
Не удалось сохранить выбранный статус (сетевая ошибка при клике на
Supported/Weak/Conflicting/Outdated) — показывается toast/сообщение об ошибке, выбранный статус
визуально откатывается к предыдущему значению.

#### State: Success
Текущий wireframe (экран 4) — claim имеет заполненную секцию Supporting Evidence, статус выбран
пользователем, review note сохранена, указан reviewer.

#### State: Draft-review
`Claim.status = draft` — стартовое состояние по правилу "AI-generated claim стартует как draft".
Именно оно показано на текущем макете (бейдж "Draft" на карточке Claim и "Status: Draft" в блоке
Set status справа), пока reviewer не выберет финальный статус.

**Success criterion:**
- на mock claim с 1 evidence item страница позволяет переключать статус между всеми 5 значениями
  enum `Claim.status` (переход из `State: Draft-review` в reviewed `State: Success`); отдельной
  фикстурой (claim без evidence) проверяется `State: Empty` с задизейбленной кнопкой Supported;
  искусственная задержка/флаг ошибки в mock-слое проверяют `State: Loading` и `State: Error`.

**Mock:** да, полностью.

---

## Chunk 5 — Decision Page

**Экран:** `05 Decision Page` (шаги 7–9 user journey: Create Decision → Decision Brief →
Review/Accept).

**Expected components:**
- `DecisionHeaderCard` (decision title + `StatusPill` decision status);
- `DecisionTextCard` / `RationaleCard`;
- `ReviewActionsCard` (`btn-accept`, `btn-send-review`, `btn-reject` + explanatory текст про
  human override правило из domain model);
- `LinkedClaimsCard` (список claim-строк со статус-иконкой ✓/△, ссылка "Open Evidence →");
- `DecisionConfidenceCard`;
- `OpenQuestionsCard`.

**States (источник: `UI_State_Map.md`, раздел "5. Decision Page"):**

#### State: Empty
Секция Linked Claims пуста — decision только что создан (сразу после нажатия
**Create Decision** на Research Task Page), claims для привязки ещё не выбраны. Поля Decision
text / Rationale тоже пусты, кнопка **Accept** недоступна по определению (нет claims).

#### State: Loading
Отправка действия Accept / Send to Review / Reject — соответствующая кнопка показывает
индикатор загрузки, остальные кнопки временно неактивны.

#### State: Error
Не удалось сохранить изменение статуса decision (сетевая ошибка при Accept/Reject) — сообщение
об ошибке отображается рядом с блоком Review, статус decision остаётся прежним (например,
Proposed), пользователь может повторить действие.

#### State: Success
Текущий wireframe (экран 5) — `Decision.status = proposed`, Linked Claims заполнены (Supported
/ Supported / Weak), Confidence и Open Questions присутствуют, действия Accept/Send to
Review/Reject доступны.

#### State: Draft-review
`Decision.status = needs_review` — кнопка **Accept** недоступна или сопровождается пояснением о
том, что accepted decision требует reviewed claims, кроме явного human override (домен:
`04-domain-model.md`). Пример на макете показывает Linked Claims, среди которых есть claim в
статусе `draft`.

### Open Contract Questions

- **Decision confidence source**: Откуда frontend должен получать `Decision confidence`?
  - Готовое поле API?
  - Агрегация из claims?
  - Artifact manifest?
  - Другой backend-derived value?
  - Current UI behavior: `Decision.confidence` приходит готовым числом из mock-фикстуры;
    фронт ничего не вычисляет.
  - Blocked decision: нельзя реализовывать на фронте алгоритм агрегации confidence из claims
    или любой другой источник, пока это не зафиксировано backend/domain contract'ом.
- **Send to Review transition**: Какое доменное изменение должен выполнять `Send to Review`?
  - Меняет ли он `Decision.status`?
  - Если да, на какое значение?
  - Или это отдельная команда/notification без изменения status?
  - Current UI behavior: `btn-send-review` — визуальная заглушка (loading-индикатор +
    подтверждающий toast), без какого-либо предположения об изменении `Decision.status`.
  - Blocked decision: нельзя реализовывать реальный `Decision.status` transition для этой
    кнопки, пока семантика не определена.
- **Accept eligibility**: Что именно означает "reviewed claims" для разрешения `Accept` у
  `Decision`?
  - Должны ли быть reviewed все Linked Claims?
  - Достаточно ли определённого подмножества?
  - Считается ли `weak` reviewed?
  - Считаются ли `conflicting` и `outdated` reviewed?
  - Current UI behavior: в `State: Draft-review` (`Decision.status = needs_review`) `btn-accept`
    сопровождается пояснительным текстом о том, что accepted decision требует reviewed claims
    (домен: `04-domain-model.md`), но фронт не решает самостоятельно, при каком именно составе
    Linked Claims кнопка должна разблокироваться.
  - Blocked decision: нельзя выбирать критерий разблокировки `btn-accept` (ни "хотя бы один
    non-draft claim", ни "все claims должны быть reviewed", ни любой другой вариант) до
    появления backend/API contract.
- **Human override semantics**: Как human override должен быть выражен в API/domain contract и
  какие ограничения он снимает для Accept?
  - Current UI behavior: явного UI для human override в этом chunk нет.
  - Blocked decision: нельзя добавлять кнопку/флаг "override" или предполагать, как он повлияет
    на доступность `Accept`, пока это не определено contract'ом.
- **Superseded decision UI**: Нужно ли отражать `Decision.status = superseded` в MVP UI? Если
  да, где именно и каким визуальным состоянием?
  - Current UI behavior: `superseded` не рендерится ни в одном state этого chunk.
  - Blocked decision: нельзя добавлять для `superseded` ни отдельный UI state, ни визуальное
    представление в `StatusPill`, пока `UI_State_Map.md` не даст ответа (сознательно оставлено
    out of scope для F1–F4).

**Success criterion:**
- на mock decision (status `proposed`) с 2 supported + 1 weak claim страница рендерит
  `State: Success`; отдельная фикстура с decision `needs_review` и claim в статусе `draft` среди
  Linked Claims проверяет `State: Draft-review` (бейдж needs_review виден, пояснительный текст
  про reviewed claims показан у `btn-accept`) — фикстура не проверяет конкретное правило
  блокировки `btn-accept`, так как критерий "reviewed claims" остаётся Open Contract Question
  выше; фикстура с пустым Linked Claims проверяет `State: Empty` (здесь `btn-accept` недоступен
  по отдельному, уже зафиксированному правилу — "нет claims", а не по критерию "reviewed
  claims").

**Mock:** да, полностью.

---

## Chunk 6 — Follow-up Question Area

**Экран:** `06 Follow-up Question - Embedded in Workspace` (шаг 10 user journey). Согласно
`Экраны.md`, это embedded-секция/модалка, вызываемая из Workspace Overview, **не отдельный route**.

**Expected components:**
- `AskQuestionInput` (текстовое поле + `btn-ask`);
- `WorkspaceAnswerCard` (текстовый ответ + `btn-view-evidence` / `btn-view-claims` /
  "View Sources →" + счётчики "Sources · N / Claims · N / Decisions · N");
- `MemorySufficiencyCard` (`StatusPill` "Sufficient / Insufficient" + пояснительный текст +
  `btn-create-follow-up-task`, который переиспользует форму Create Research Task из Chunk 1,
  отдельной формы под follow-up не создаём).

**States (источник: `UI_State_Map.md`, раздел "6. Follow-up Question Area"):**

#### State: Empty
В workspace ещё нет накопленных Sources/Claims/Decisions (состояние сразу после создания
workspace, до первого research run). Поле вопроса и кнопка Ask видны, но счётчики Sources /
Claims / Decisions показывают 0, с подсказкой "Add research first to get answers from memory".

#### State: Loading
После нажатия Ask — блок Workspace Answer показывает индикатор "Generating answer..." вместо
текста ответа, счётчики временно скрыты или показаны как есть.

#### State: Error
Не удалось сгенерировать ответ (ошибка LLM-вызова или недоступность памяти workspace) — в блоке
Workspace Answer показывается сообщение об ошибке и кнопка Retry.

#### State: Success
Текущий wireframe (экран 6) — вопрос задан, Workspace Answer заполнен текстом, ссылки View
Evidence / View Claims / View Sources доступны, счётчики Sources / Claims / Decisions
отображены. Сюда же относится ветка "Memory sufficient? = Yes" (без CTA follow-up).

#### State: Draft-review
Блок **Memory sufficient? = No**: ответ дан, но система явно помечает его как неполный и
предлагает **Create Follow-up Task**, не выдавая ответ за окончательное принятое знание
(соответствует принципу "только проверенное человеком становится знанием команды"). Это
состояние — вариант `State: Success` с дополнительным review-сигналом, а не отдельная ветка
загрузки/ошибки.

**Success criterion:**
- компонент встраивается в Workspace Overview (Chunk 1) как collapsible/modal-секция; на
  mock-ответе с `insufficient` показывается `State: Draft-review` (виден
  `btn-create-follow-up-task`, клик по нему открывает ту же форму, что и `btn-create-task` из
  Chunk 1); отдельными фикстурами проверяются `State: Empty` (нулевые счётчики),
  `State: Loading` (после клика Ask) и `State: Error` (ответ не сгенерирован).

**Mock:** да, полностью. `Ask` в этом chunk возвращает заранее заданный mock-ответ, реальный
вызов workspace Q&A backend endpoint не входит в scope.

---

## Сводная таблица

| Chunk | Экран | State: Empty | State: Loading | State: Error | State: Success | State: Draft-review | Mock |
|---|---|---|---|---|---|---|---|
| 0 | App Shell | N/A (UI-kit) | N/A (UI-kit) | N/A (UI-kit) | N/A (UI-kit) | N/A (UI-kit) | Да |
| 1 | Workspace Overview | Нет tasks/decisions | Skeleton списков | Retry на ошибке загрузки | Заполненные списки (wireframe) | Бейдж Review у task | Да |
| 2 | Research Task Page | Нет sources | `queued`/`running` | `failed` + `cancelled` | `completed` (wireframe) | `review_required` (wireframe) | Да |
| 3 | Source / Evidence Panel | Нет evidence у source | Skeleton source/evidence | Source content недоступен | Заполненная карточка (wireframe) | Approve/Reject/Edit до подтверждения | Да |
| 4 | Claim Review Panel | Нет supporting evidence | Skeleton claim/evidence | Не сохранился статус | Заполненный review (wireframe) | `Claim.status = draft` (wireframe) | Да |
| 5 | Decision Page | Нет linked claims | Отправка Accept/Reject | Не сохранился статус decision | `proposed` (wireframe) | `needs_review` | Да |
| 6 | Follow-up Question Area | Нулевые счётчики памяти | "Generating answer..." | Ответ не сгенерирован | Ответ + Memory sufficient = Yes (wireframe) | Memory sufficient = No | Да |

Все 7 chunks можно стартовать параллельно на mock data, не дожидаясь backend и agent-service.
Первая точка реальной интеграции с backend (замена fixtures на реальные API-вызовы через
`packages/contracts`) — отдельный backlog, не входит в этот документ.