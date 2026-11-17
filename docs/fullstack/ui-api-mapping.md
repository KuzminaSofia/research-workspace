# UI → API Mapping (F6)

**Зачем:** связать каждый из 6 MVP-экранов с backend contract'ом — entities (`04-domain-model.md`), endpoints/response shape (`b-questions.md`, разделы B2–B4) и правилами из `02-system-architecture.md` / `03-backend-layered-architecture.md`.

Статусные enum'ы (`04-domain-model.md`):
- `ResearchTask.status`: `draft`, `in_progress`, `review`, `completed`, `archived`
- `ResearchRun.status`: `queued`, `running`, `review_required`, `completed`, `failed`, `cancelled`
- `ResearchRun.mode`: `manual_sources`, `search_assisted`, `follow_up`
- `Claim.status`: `draft`, `supported`, `weak`, `conflicting`, `outdated`
- `Decision.status`: `proposed`, `accepted`, `rejected`, `superseded`, `needs_review`
- `Source.type`: `url`, `manual_note`, `pdf`, `doc`, `github`, `video`
- `Update.type`: `source_changed`, `new_source`, `claim_review`, `decision_review`, `manual_note`

---

## 1. Workspace Overview

**Фрейм в Figma:** `01 Workspace Overview`

### Entities
- `Workspace`
- `ResearchTask` (список)
- `Decision` (список)

### Endpoints
| Метод | Endpoint | Response | Для чего |
|---|---|---|---|
| GET | `/workspaces/{workspace_id}` | `{ id, title, description, owner_id, created_at, updated_at }` | title + description в hero |
| GET | `/workspaces/{workspace_id}/research-tasks` | `{ items: [{ id, title, status, summary, created_at }] }` | список Research Tasks |
| GET | `/workspaces/{workspace_id}/decisions` | `{ items: [{ id, title, status, decided_at, claims_count }] }` | список Decisions |
| POST | `/workspaces/{workspace_id}/research-tasks` | `{ id, workspace_id, title, research_question, status: "draft", created_by, created_at }` | кнопка **Create Research Task** |
| POST | `/workspaces/{workspace_id}/ask` | см. экран 6 | инпут **Ask Workspace** |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Workspace.title` | заголовок hero | Workspace |
| `Workspace.description` | подзаголовок hero | Workspace |
| `ResearchTask.title` | заголовок task-card | ResearchTask |
| `ResearchTask.summary` | подзаголовок task-card | ResearchTask — list-endpoint не отдаёт `research_question`, полный вопрос виден только на экране 2 |
| `ResearchTask.status` | `status-pill` на task-card | ResearchTask |
| `Decision.title` | заголовок decision-card | Decision |
| `Decision.status` | `status-pill` на decision-card | Decision |

List-endpoint `GET /workspaces/{workspace_id}/decisions` не отдаёт `rationale` — тизер на decision-card, который предполагался раньше, показать нечем. Чем заполнить это место на карточке, не решено (см. Open Questions).

### Состояния, зависящие от backend
- **Loading** — идут начальные вызовы `GET /research-tasks` + `GET /decisions` → skeleton-списки.
- **Empty** — оба list-endpoint'а вернули `items: []` → плейсхолдер + CTA "Create Research Task"; карточка Ask Workspace остаётся видимой.
- **Error** — любой из GET-вызовов упал (401 unauthenticated, сеть) → сообщение об ошибке + Retry.
- **Success** — списки заполнены; все значения `status` должны корректно рендериться в `StatusPill`.
- **Draft/review** — `ResearchTask.status = review` → бейдж "Review" на карточке, чисто визуально.

---

## 2. Research Task Page

**Фрейм в Figma:** `02 Research Task Page`

### Entities
- `ResearchTask`
- `Source` (список)
- `ResearchRun` (последний)
- `EvidenceItem` (список, превью)
- `Claim` (список, с inline-контролами review)

### Endpoints
| Метод | Endpoint | Response | Для чего |
|---|---|---|---|
| GET | `/research-tasks/{task_id}` | `{ id, workspace_id, title, research_question, status, summary, created_by, created_at, updated_at }` | title, research_question, status |
| GET | `/research-tasks/{task_id}/sources` | `{ items: [{ id, type, title, url, summary, reliability_note }] }` | секция Sources |
| POST | `/research-tasks/{task_id}/sources` | `{ id, workspace_id, research_task_id, type, title, url, summary, captured_at }` | **+ Add Source** (ручной ввод) |
| POST | `/research-tasks/{task_id}/sources/import` | `{ id, type, title, url, summary, captured_at }` | **+ Add Source → Import from URL**; `summary` может заполниться асинхронно |
| POST | `/research-tasks/{task_id}/research-runs` | `{ id, ..., status: "queued", mode, started_by, started_at }` | **Run Research** / **Retry Run** / **Re-run** |
| GET | `/research-tasks/{task_id}/research-runs` | `{ items: [{ id, status, mode, started_at, completed_at }] }` | список runs; фронт берёт последний по `started_at` |
| GET | `/research-runs/{run_id}` | `{ id, status, mode, artifact_manifest_ref, started_at, completed_at, error_message }` | детальная карточка статуса Research Run (error_message, retry) |
| GET | `/research-tasks/{task_id}/claims?status=` | `{ items: [{ id, text, status, confidence, evidence_count, review_note? }] }` | колонка Claims |
| PATCH | `/claims/{claim_id}` | body `{ status: string, review_note?: string }` → `{ id, status, reviewed_by, updated_at, review_note? }` | inline-кнопки **Supported/Weak/Conflicting/Outdated/Draft**. Errors: `409` — нельзя перевести в `supported` без evidence; `422` — недопустимый переход статуса |
| POST | `/workspaces/{workspace_id}/decisions` | body `{ title, decision_text, rationale?, claim_ids?, open_questions? }` | **Create Decision** — привязывается через `claim_ids`, а не напрямую через `research_task_id` |

Для колонки **Evidence** отдельного endpoint'а на уровне research task нет: `EvidenceItem` существует только "per source" (`GET /sources/{id}/evidence`, экран 3). Как именно наполнять эту колонку данными, не решено (см. Open Questions).

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `ResearchTask.title`, `research_question` | question-card | ResearchTask |
| `ResearchTask.status` | status-pill в header | ResearchTask |
| `Source.title`, `url`, `reliability_note` | source-card в списке | Source — `author`/`published_at` в списке не приходят, доступны только через `GET /sources/{id}` поштучно |
| `ResearchRun.status` | status-pill Research Run | ResearchRun |
| `ResearchRun.error_message` | видно через `GET /research-runs/{id}`, состояние failed | ResearchRun |
| `ResearchRun.mode` | определяет текст Empty-состояния | ResearchRun |
| `Claim.text`, `status`, `confidence`, `evidence_count` | claims-card | Claim — список отдаёт только количество evidence, не excerpt'ы; полный excerpt виден в Claim Review Panel |

### Состояния, зависящие от backend
- **Empty** — ещё нет ни одного `Source`. Для `mode = manual_sources`: **Run Research** заблокирован, пока нет source. Для `search_assisted`/`follow_up` — допустимость пустого списка Sources перед запуском остаётся открытым вопросом (см. ниже).
- **Loading** — последний run в `queued`/`running` → секции Evidence/Claims показывают "Waiting for run to complete".
- **Error** — run в `failed` (кнопка **Retry Run**, показан `error_message`) либо `cancelled` (кнопка **Re-run**, без `error_message`).
- **Success** — run `completed`; Evidence/Claims заполнены; **Create Decision** доступна.
- **Draft/review** — run `review_required`; карточки claim показывают бейдж `Draft`.

---

## 3. Source / Evidence Panel

**Фрейм в Figma:** `03 Source Evidence Panel`

### Entities
- `Source`
- `EvidenceItem`

### Endpoints
| Метод | Endpoint | Response | Для чего |
|---|---|---|---|
| GET | `/sources/{source_id}` | `{ id, workspace_id, research_task_id, type, title, url, author, published_at, captured_at, summary, reliability_note }` | карточка Source |
| GET | `/evidence/{evidence_id}` | `{ id, source_id, excerpt, location, note, confidence, created_at, status, reviewed_by?, reviewed_at? }` | карточка Evidence Item |
| PATCH | `/evidence/{evidence_id}` | body `{ status: "approved" \| "rejected" }` → `{ id, status, reviewed_by, reviewed_at }` | **Approve** и **Reject** — один и тот же вызов с разным значением `status` |

Для действия **Edit** (правка `excerpt`/`location`/`note`) подтверждённого endpoint'а нет — контракт описывает только смену `status`. Пока не решено, иммутабельно ли содержимое evidence после создания.

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Source.title`, `url`, `published_at`, `author` | source-header / source-meta | Source |
| `Source.summary` | source-content-preview | Source — в `04-domain-model.md` у `Source` есть поле `raw_content_ref`, но опубликованный `GET /sources/{id}` его не возвращает; до уточнения превью строится на `summary` |
| `EvidenceItem.excerpt`, `confidence`, `location`, `note` | пилюли на карточке evidence | EvidenceItem |
| `EvidenceItem.status`, `reviewed_by`, `reviewed_at` | бейдж review-статуса | EvidenceItem |

### Состояния, зависящие от backend
- **Empty** — для данного `Source` ещё не извлечён ни один `EvidenceItem`.
- **Loading** — идёт загрузка Source / Evidence Item.
- **Error** — `422`/`404` при получении Source или Evidence → сообщение об ошибке, "Open original source" остаётся fallback'ом.
- **Success** — Source + Evidence Item заполнены; **Approve/Reject** пишут `status`/`reviewed_by`/`reviewed_at` через `PATCH /evidence/{id}`.
- **Draft/review** — `EvidenceItem.status = pending` — стартовое значение до review; Approve/Reject переводят в `approved`/`rejected`.

---

## 4. Claim Review Panel

**Фрейм в Figma:** `04 Claim Review Panel`

### Entities
- `Claim`
- `EvidenceItem` (supporting evidence, many-to-many)

### Endpoints
| Метод | Endpoint | Response | Для чего |
|---|---|---|---|
| GET | `/claims/{claim_id}` | `{ id, text, status, confidence, reviewed_by, evidence: [{ id, excerpt, source_title }], created_at, updated_at, review_note? }` | карточка Claim вместе со Supporting Evidence |
| PATCH | `/claims/{claim_id}` | body `{ status: string, review_note?: string }` → `{ id, status, reviewed_by, updated_at, review_note? }` | **Set status** (Supported/Weak/Conflicting/Outdated/Draft) и **Save** review note — один и тот же вызов. Errors: `409` — `supported` без evidence; `422` — недопустимый переход статуса |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Claim.text` | claim-card | Claim |
| `Claim.status` | status-pill | Claim |
| `Claim.confidence` | confidence-pill | Claim |
| `EvidenceItem.excerpt`, `source_title` | элемент Supporting Evidence (приходит вместе с claim) | EvidenceItem |
| `Claim.review_note` | textarea "Review note" | Claim |
| `Claim.reviewed_by` | "Reviewer: ..." | Claim |

### Доменные правила, которые должны быть видны в UI
- `supported` невозможен без связанного evidence и без факта review человеком — backend проверяет это сам (`409`), но UI должен задизейблить контрол **Supported** оптимистично, пока Supporting Evidence пуст.
- AI-generated claims стартуют как `draft`.
- Если claim переводят в `outdated`, backend подсвечивает связанные decisions как кандидатов на пересмотр (не меняет их статус автоматически) и создаёт запись в `Update`. Как именно это должно отражаться в UI, не решено (см. Open Questions).

### Состояния, зависящие от backend
- **Empty** — нет привязанного Supporting Evidence → **Supported** задизейблен.
- **Loading** — идёт загрузка claim (evidence приходит тем же запросом).
- **Error** — `409`/`422`/сетевая ошибка при PATCH → toast + откат оптимистичного статуса.
- **Success** — Supporting Evidence заполнен, статус выбран, review note сохранена, reviewer показан.
- **Draft/review** — `Claim.status = draft` — стартовое состояние.

---

## 5. Decision Page

**Фрейм в Figma:** `05 Decision Page`

### Entities
- `Decision`
- `Claim` (связанные, many-to-many)

### Endpoints
| Метод | Endpoint | Response | Для чего |
|---|---|---|---|
| GET | `/decisions/{decision_id}` | `{ id, title, decision_text, status, rationale, decided_by, decided_at, claims: [{ id, text, status }], created_at, updated_at, confidence, open_questions?, override_reason? }` | Decision + Linked Claims одним вызовом |
| PATCH | `/decisions/{decision_id}` | body `{ status: string, override_reason?: string, open_questions?: string[] }` → `{ id, status, decided_by, decided_at, updated_at, confidence, override_reason? }` | **Accept** (`status: "accepted"`), **Send to Review** (`status: "needs_review"`), **Reject** (`status: "rejected"`) — один и тот же вызов с разным `status`. Errors: `409` — нельзя `accepted` без хотя бы одного reviewed claim, если нет `override_reason`; `422` — недопустимый переход статуса |
| GET | `/decisions/{decision_id}/export` | `text/markdown` | **Export** Decision Brief в Markdown |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Decision.title`, `decision_text` | decision-header-card, секция Decision | Decision |
| `Decision.rationale` | секция Rationale | Decision |
| `Decision.status` | status-pill | Decision |
| `Decision.decided_by` | строка "Reviewer / decided_by" | Decision |
| `Claim.text`, `Claim.status` (на каждый linked claim) | список Linked Claims | Claim |
| `Decision.confidence` | карточка Confidence в Decision Brief | Decision — поле есть в API-ответах, но в `04-domain-model.md` у `Decision` его нет (см. open questions) |
| `Decision.open_questions` | карточка Open Questions в Decision Brief | Decision — то же самое: реален в API, отсутствует в опубликованной domain model |
| `Decision.override_reason` | пояснение при human override | Decision — то же самое |

### Доменное правило для Accept
Decision может стать `accepted`, если верно любое из двух:
1. хотя бы один (не обязательно все) linked claim прошёл review человеком (non-`draft` статус);
2. явный human override — `PATCH` с непустым `override_reason`, без требования к reviewed claims.

Как именно UI должен предлагать override (форма, модалка, отдельный экран и т.д.), не решено (см. Open Questions).

### Состояния, зависящие от backend
- **Empty** — Decision только создан, Linked Claims ещё нет → Accept недоступен.
- **Loading** — PATCH в процессе → кнопка со спиннером, остальные недоступны.
- **Error** — `409`/`422` при PATCH → сообщение об ошибке рядом с блоком Review, статус не меняется.
- **Success** — `status = proposed`, Linked Claims со смешанными статусами, Confidence и Open Questions показаны, все три действия доступны.
- **Draft/review** — `status = needs_review`; **Accept** доступен только если выполнено правило выше.

---

## 6. Follow-up Question Area

**Фрейм в Figma:** `06 Follow-up Question - Embedded in Workspace` (встроенная секция, не отдельный route).

### Entities
Отдельной сущности для вопроса/ответа не требуется — весь use case описывается response shape'ом `POST /ask`.

### Endpoints
| Метод | Endpoint | Response | Для чего |
|---|---|---|---|
| POST | `/workspaces/{workspace_id}/ask` | `{ answer_text, sources_count, claims_count, decisions_count, memory_sufficient, evidence_link, claims_link, sources_link }` | вопрос → готовый ответ с текстом, счётчиками и ссылками. Errors: `404` workspace not found, `502` memory unavailable |
| POST | `/workspaces/{workspace_id}/research-tasks` | как в экране 1 | **Create Follow-up Task** (переиспользует flow создания task; создаваемый run получает `mode: follow_up`) |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| текст вопроса (ввод пользователя) | ask-input | payload запроса |
| `answer_text` | блок Workspace Answer | поле ответа `/ask` |
| `evidence_link`, `claims_link`, `sources_link` | evidence-links | backend отдаёт готовые ссылки прямо в ответе, фронт ничего не строит сам |
| `sources_count`, `claims_count`, `decisions_count` | строка metrics | прямые поля ответа `/ask`, агрегация на фронте не нужна |
| `memory_sufficient` | пилюля memory-status (Yes/No) | прямое boolean-поле ответа `/ask` |

### Состояния, зависящие от backend
- **Empty** — 0 Sources/Claims/Decisions в Workspace → счётчики 0, подсказка.
- **Loading** — после **Ask** → плейсхолдер "Generating answer...".
- **Error** — `502 memory unavailable` (или сетевая) → сообщение об ошибке + Retry.
- **Success** — `answer_text` заполнен, ссылки активны, счётчики показаны; ветка `memory_sufficient: true` без доп. CTA.
- **Draft/review** — `memory_sufficient: false` → ответ показан с пометкой "неполный" + CTA **Create Follow-up Task**.

---

## Contract Dependencies & Open Questions

### 1. Расхождение domain model ↔ API contract для `EvidenceItem`
В `04-domain-model.md` у `EvidenceItem` только `id, source_id, excerpt, location, note, confidence, created_at`. Реальный API (`b-questions.md`) уже возвращает и меняет `status`, `reviewed_by`, `reviewed_at`. Нужно синхронизировать документ доменной модели с фактическим контрактом — иначе непонятно, что из этого источник истины.

### 2. Расхождение domain model ↔ API contract для `Decision`
В `04-domain-model.md` у `Decision` нет полей `confidence`, `open_questions`, `override_reason` — но они уже есть в опубликованных ответах API и участвуют в domain rule для Accept (`override_reason`). Та же просьба: обновить `04-domain-model.md` или подтвердить, что это временные поля вне модели.

### 3. `Source.raw_content_ref` заявлен в модели, но не возвращается API
`04-domain-model.md` включает `raw_content_ref` в поля `Source`, но `GET /sources/{id}` его не отдаёт. Нужно уточнить: поле скрыто намеренно (например, большое и не нужно на каждый GET), или его забыли добавить в response.

### 4. Нет сущности `User`/аккаунта
`owner_id`, `created_by`, `decided_by`, `reviewed_by`, `started_by` — везде просто ID, ни один response их не резолвит в display name. Не решено: фронт резолвит имена сам через отдельный identity-сервис, или backend должен подставлять имя прямо в ответ.

### 5. `Decision.status = superseded` — вопрос скоупа MVP
Есть в enum'е, но ни один endpoint или domain rule не описывает переход в это состояние и его отображение в UI. Нужен ли он в MVP?

### 6. Нет агрегированного `evidence` по research task
Evidence существует только "per source" (`GET /sources/{source_id}/evidence`), агрегирующего endpoint'а на уровне research task нет. Как получать данные для колонки Evidence на экране 2 — не решено.

### 7. Нет endpoint'а "последний run"
Есть только список (`GET /research-tasks/{id}/research-runs`) и получение по id (`GET /research-runs/{id}`). Стоит ли backend'у дать гарантию сортировки по `started_at`, или добавить отдельный `?latest=true`?

### 8. Связь Decision ↔ Research Task не описана явно
`POST /workspaces/{workspace_id}/decisions` принимает `claim_ids`, но не `research_task_id`, хотя в `04-domain-model.md` у `Decision` есть many-to-many связь с `ResearchTask`. Заполняется ли она автоматически через `research_task_id` переданных claims, или нужен отдельный вызов?

### 9. Допустимость пустого списка Sources для `search_assisted`/`follow_up` run
Для `mode = manual_sources` правило чёткое (нельзя запускать без source). Для остальных двух режимов — не решено.

### 10. `Update` нигде не появляется в UI
`01-product-vision.md` называет `updates` последним звеном главной цепочки продукта (`research question -> sources -> evidence -> claims -> decisions -> updates`), и `04-domain-model.md` описывает `Update` как полноценную сущность. Ни один из 6 экранов её не показывает — ни как список, ни как уведомление. Это осознанно вне MVP-скоупа, или для неё нужен отдельный экран/виджет?

### 11. Edit для `EvidenceItem` не определён
`PATCH /evidence/{id}` описывает только смену `status`. Правка содержимого (excerpt/location/note) — отдельный, пока не описанный сценарий: evidence иммутабельна после создания (кроме статуса), или backend должен добавить content-edit endpoint?

### 12. List-endpoint'ы возвращают меньше полей, чем нужно карточкам
- `research-tasks` list не отдаёт `research_question` (только `summary`).
- `decisions` list не отдаёт `rationale` (только `claims_count`, `decided_at`).
- `claims` list (per task) не отдаёт excerpt'ы evidence, только `evidence_count`.
- `sources` list (per task) не отдаёт `author`/`published_at`.

Действительно ли эти поля не нужны в списках, или их стоит добавить, чтобы избежать дополнительных запросов — не решено.

### 13. Чем заменить тизер rationale на decision-card (экран 1)
`GET /workspaces/{workspace_id}/decisions` не отдаёт `rationale`. Что показывать вместо тизера на карточке — не решено.

### 14. Как показывать в UI, что claim стал `outdated` и затронул decisions (экран 4)
Backend подсвечивает связанные decisions как кандидатов на пересмотр и создаёт запись в `Update`, но как именно это должно отражаться в интерфейсе — не решено.

### 15. Как UI должен предлагать human override при Accept (экран 5)
Поле `override_reason` существует в контракте, но форма его ввода в интерфейсе (где, как, в каком виде) — не решена.

---

## Критерий готовности этого документа
- backend intern может открыть этот файл и понять, по каждому экрану, какие именно entities/fields/endpoints нужны фронтенду — без чтения фронтенд-кода.
- Каждый пробел сформулирован как вопрос к контракту, а не как молчаливое допущение фронтенда.