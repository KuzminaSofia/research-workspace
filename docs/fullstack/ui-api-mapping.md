# UI → API Mapping (F6)

**Зачем:** связать каждый из 6 MVP-экранов с backend contract (entities, endpoints, fields, states) и заранее найти места, где UI требует данных или поведения, которых пока нет ни в `04-domain-model.md`, ни в каком-либо опубликованном API-контракте.

**Важная оговорка для backend intern'а:** ни в одном исходном документе нет опубликованной REST API-спецификации — в `02-system-architecture.md` описаны только внутренние endpoints `backend ↔ agent-service`. Все endpoints ниже — это **предположение**, выведенное из domain model + wireframes, а не скопированный существующий контракт. Раздел "Endpoints" нужно читать как *"вот что экрану нужно, чтобы это где-то существовало"*, а не как уже согласованный API. Там, где поле вообще отсутствует в `04-domain-model.md`, это явно помечено, а не тихо предполагается.

Статусные enum'ы ниже всегда берутся из domain model, а не изобретаются на фронте:
- `ResearchTask.status`: `draft`, `in_progress`, `review`, `completed`, `archived`
- `ResearchRun.status`: `queued`, `running`, `review_required`, `completed`, `failed`, `cancelled`
- `Claim.status`: `draft`, `supported`, `weak`, `conflicting`, `outdated`
- `Decision.status`: `proposed`, `accepted`, `rejected`, `superseded`, `needs_review`

---

## 1. Workspace Overview

**Фрейм в Figma:** `01 Workspace Overview`

### Entities
- `Workspace`
- `ResearchTask` (список)
- `Decision` (список)

### Нужные endpoints
| Метод | Endpoint (предположительно) | Для чего |
|---|---|---|
| GET | `/workspaces/{workspace_id}` | title + description в hero |
| GET | `/workspaces/{workspace_id}/research-tasks` | список Research Tasks |
| GET | `/workspaces/{workspace_id}/decisions` | список Decisions |
| POST | `/workspaces/{workspace_id}/research-tasks` | кнопка **Create Research Task** |
| POST | `/workspaces/{workspace_id}/ask` | инпут **Ask Workspace** (общий контракт с экраном 6) |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Workspace.title` | заголовок hero | Workspace |
| `Workspace.description` | подзаголовок hero | Workspace |
| `ResearchTask.title` | заголовок task-card | ResearchTask |
| `ResearchTask.research_question` | подзаголовок task-card | ResearchTask |
| `ResearchTask.status` | `status-pill` на task-card | ResearchTask |
| `Decision.title` | заголовок decision-card | Decision |
| `Decision.rationale` (тизер) | строка-описание на decision-card | Decision |
| `Decision.status` | `status-pill` на decision-card | Decision |

### Состояния, зависящие от backend
- **Loading** — идут начальные вызовы `GET /research-tasks` + `GET /decisions` → skeleton-списки.
- **Empty** — оба list-endpoint'а вернули `[]` → плейсхолдер + CTA "Create Research Task"; карточка Ask Workspace остаётся видимой с подсказкой.
- **Error** — любой из GET-вызовов упал (сеть/API недоступен) → сообщение об ошибке + Retry.
- **Success** — списки заполнены; все 5 значений `ResearchTask.status` и все 5 значений `Decision.status` должны корректно рендериться в `StatusPill`, а не только те два, что видны на wireframe.
- **Draft/review** — `ResearchTask.status = review` → бейдж "Review" на карточке этой task, чисто визуально (без отдельного endpoint'а).

---

## 2. Research Task Page

**Фрейм в Figma:** `02 Research Task Page`

### Entities
- `ResearchTask`
- `Source` (список)
- `ResearchRun` (последний)
- `EvidenceItem` (список, превью)
- `Claim` (список, с inline-контролами review)

### Нужные endpoints
| Метод | Endpoint (предположительно) | Для чего |
|---|---|---|
| GET | `/research-tasks/{id}` | title, research_question, status |
| GET | `/research-tasks/{id}/sources` | секция Sources |
| POST | `/research-tasks/{id}/sources` | **+ Add Source** |
| POST | `/research-tasks/{id}/research-runs` | **Run Research** / **Retry Run** / **Re-run** |
| GET | `/research-tasks/{id}/research-runs/latest` | карточка статуса Research Run |
| GET | `/research-tasks/{id}/evidence` | колонка Evidence |
| GET | `/research-tasks/{id}/claims` | колонка Claims (включая excerpt связанного evidence на каждый claim) |
| PATCH | `/claims/{id}/status` | inline-кнопки **Supported/Weak/Conflicting/Outdated/Draft** на карточке claim |
| POST | `/decisions` (с `research_task_id`) | **Create Decision** |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `ResearchTask.title`, `research_question` | question-card | ResearchTask |
| `ResearchTask.status` | status-pill в header | ResearchTask |
| `Source.title`, `url`, `author`, `published_at`, `reliability_note` | source-card | Source |
| `ResearchRun.status` | status-pill Research Run | ResearchRun |
| `ResearchRun.error_message` | показывается только в состоянии failed | ResearchRun |
| `ResearchRun.mode` | определяет, какой текст Empty-состояния применяется (см. ниже) | ResearchRun |
| `EvidenceItem.excerpt` | превью на evidence-card | EvidenceItem |
| `Claim.text`, `status`, `confidence` | claims-card | Claim |
| Связанные `EvidenceItem.excerpt`, `confidence`, `Source.title` | подблок "Linked evidence" на карточке claim | связь EvidenceItem × Claim |

### Состояния, зависящие от backend
- **Empty** — ещё нет ни одного `Source`.
  - Для `ResearchRun.mode = manual_sources` (единственный режим в MVP): **Run Research** заблокирован/сопровождается подсказкой, пока нет ни одного source — это устоявшееся доменное правило.
  - Для `ResearchRun.mode = search_assisted`: допустим ли пустой список Sources перед запуском — **открытый вопрос** (см. ниже), не решено, вне скоупа MVP.
- **Loading** — `ResearchRun.status = queued` или `running` → секции Evidence/Claims показывают "Waiting for run to complete".
- **Error** — `ResearchRun.status = failed` (показывается `error_message`, кнопка становится **Retry Run**) либо `status = cancelled` (без `error_message`, кнопка становится **Re-run**) — два разных backend-состояния маппятся в один UI-state.
- **Success** — `ResearchRun.status = completed`; Evidence/Claims заполнены; **Create Decision** доступна.
- **Draft/review** — `ResearchRun.status = review_required`; карточки claim показывают бейдж `Draft`.

---

## 3. Source / Evidence Panel

**Фрейм в Figma:** `03 Source Evidence Panel`

### Entities
- `Source`
- `EvidenceItem`

### Нужные endpoints
| Метод | Endpoint (предположительно) | Для чего |
|---|---|---|
| GET | `/sources/{id}` | карточка Source (title, url, published_at, превью содержимого через `raw_content_ref`) |
| GET | `/evidence/{id}` | карточка Evidence Item (excerpt, location, note, confidence) |
| POST | `/evidence/{id}/approve` | **Approve** — ⚠️ см. open question, поля для хранения результата пока нет |
| POST | `/evidence/{id}/reject` | **Reject** — ⚠️ тот же пробел |
| PATCH | `/evidence/{id}` | **Edit** |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Source.title`, `url`, `published_at` | source-header / source-meta | Source |
| содержимое по `Source.raw_content_ref` | source-content-preview | Source |
| `EvidenceItem.excerpt` | текстовый блок evidence | EvidenceItem |
| `EvidenceItem.confidence` | пилюля "Confidence 0.86" | EvidenceItem |
| `EvidenceItem.location` | пилюля "Location: section 4.2" | EvidenceItem |
| `EvidenceItem.note` | пилюля "Note: ..." | EvidenceItem |
| связанный `Claim.text` | строка "Linked Claim: ..." внизу | Claim (через связь EvidenceItem↔Claim) |

### Состояния, зависящие от backend
- **Empty** — для данного `Source` ещё не извлечён ни один `EvidenceItem`.
- **Loading** — идёт загрузка содержимого по `Source.raw_content_ref` или самого evidence item.
- **Error** — `raw_content_ref` недоступен/битая ссылка → сообщение об ошибке, "Open original source" остаётся как fallback.
- **Success** — Source + Evidence Item полностью заполнены, Approve/Reject/Edit доступны.
- **Draft/review** — Approve/Reject/Edit существуют, чтобы обеспечить принцип "только проверенный человеком evidence становится знанием", но **у `EvidenceItem` нет поля review-состояния, куда можно было бы записать результат** — вынесено как open question ниже.

---

## 4. Claim Review Panel

**Фрейм в Figma:** `04 Claim Review Panel`

### Entities
- `Claim`
- `EvidenceItem` (supporting evidence, many-to-many)

### Нужные endpoints
| Метод | Endpoint (предположительно) | Для чего |
|---|---|---|
| GET | `/claims/{id}` | карточка Claim (text, status, confidence) |
| GET | `/claims/{id}/evidence` | список Supporting Evidence |
| PATCH | `/claims/{id}/status` | **Set status**: Supported / Weak / Conflicting / Outdated / Draft (один контрол) |
| PATCH или POST | `/claims/{id}/review-note` | **Save** review note — ⚠️ см. open question, поля для этого пока нет |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Claim.text` | claim-card | Claim |
| `Claim.status` | status-pill | Claim |
| `Claim.confidence` | confidence-pill | Claim |
| `EvidenceItem.excerpt`, `Source.title` | элемент Supporting Evidence | EvidenceItem |
| свободный текст-комментарий reviewer'а | textarea "Review note" | ⚠️ нет в domain model |
| `Claim.reviewed_by` | "Reviewer: ..." | Claim |

### Доменные правила, которые должны быть видны в UI
- `supported` невозможен без связанного evidence → контрол **Supported** задизейблен, пока Supporting Evidence пуст.
- AI-generated claims стартуют как `draft`.

### Состояния, зависящие от backend
- **Empty** — ещё нет привязанного Supporting Evidence → контрол **Supported** задизейблен с подсказкой.
- **Loading** — идёт загрузка claim + evidence при открытии панели.
- **Error** — не удалось записать статус (сетевая ошибка при клике на Supported/Weak/Conflicting/Outdated) → toast + оптимистичный статус откатывается назад.
- **Success** — Supporting Evidence заполнен, статус выбран, review note сохранена, reviewer показан.
- **Draft/review** — `Claim.status = draft` — стартовое состояние до того, как reviewer выберет финальное значение.

---

## 5. Decision Page

**Фрейм в Figma:** `05 Decision Page`

### Entities
- `Decision`
- `Claim` (связанные, many-to-many)

### Нужные endpoints
| Метод | Endpoint (предположительно) | Для чего |
|---|---|---|
| GET | `/decisions/{id}` | Decision text, rationale, status, decided_by |
| GET | `/decisions/{id}/claims` | список Linked Claims (со статусом каждого claim) |
| POST | `/decisions/{id}/accept` | **Accept** |
| POST | `/decisions/{id}/send-to-review` | **Send to Review** — ⚠️ см. open question, семантика не определена |
| POST | `/decisions/{id}/reject` | **Reject** |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| `Decision.title`, `decision_text` | decision-header-card, секция Decision | Decision |
| `Decision.rationale` | секция Rationale | Decision |
| `Decision.status` | status-pill | Decision |
| `Decision.decided_by` | строка "Reviewer / decided_by" | Decision |
| `Claim.text`, `Claim.status` (на каждый linked claim) | список Linked Claims | Claim |
| значение "Decision confidence" | карточка Confidence в Decision Brief | ⚠️ нет в domain model — см. open question |
| текст "Open Questions" | карточка Open Questions в Decision Brief | ⚠️ нет в domain model — см. open question |

### Состояния, зависящие от backend
- **Empty** — Decision только что создан, Linked Claims ещё нет → Accept недоступен по определению (нет claims).
- **Loading** — Accept / Send to Review / Reject в процессе отправки → кнопка показывает спиннер, остальные недоступны.
- **Error** — вызов смены статуса упал → сообщение об ошибке рядом с блоком Review, статус не изменился, можно повторить.
- **Success** — `Decision.status = proposed`, Linked Claims заполнены со смешанными статусами, показаны Confidence и Open Questions, Accept/Send to Review/Reject доступны.
- **Draft/review** — `Decision.status = needs_review`; **Accept** недоступен/сопровождается пояснением по доменному правилу "accepted decision требует reviewed claims, кроме явного human override" — но точный критерий проверки не решён (open question ниже).

---

## 6. Follow-up Question Area

**Фрейм в Figma:** `06 Follow-up Question - Embedded in Workspace` (встроенная секция/модалка в Workspace Overview, **не отдельный route**).

### Entities
- Агрегированное представление счётчиков `Source`, `Claim`, `Decision` по `Workspace`.
- Обмен вопрос/ответ и сигнал "memory sufficient" — ⚠️ **ни для того, ни для другого нет определённой сущности/схемы в `04-domain-model.md`**, хотя сам use case ("задать вопрос по workspace") назван в `03-backend-layered-architecture.md`. См. open question ниже.

### Нужные endpoints
| Метод | Endpoint (предположительно) | Для чего |
|---|---|---|
| POST | `/workspaces/{id}/ask` | отправка вопроса → возвращает текст ответа, ссылки, счётчики, флаг достаточности |
| POST | `/workspaces/{id}/research-tasks` | **Create Follow-up Task** (переиспользует flow Create Research Task из экрана 1, отдельного endpoint'а нет) |

### Отображаемые поля
| Поле | UI-элемент | Сущность |
|---|---|---|
| текст вопроса (ввод пользователя) | ask-input | нет (это payload запроса) |
| текст ответа | блок Workspace Answer | ⚠️ сущность не определена |
| ссылки View Evidence / View Claims / View Sources | evidence-links | производные, не поле сущности |
| счётчики Sources / Claims / Decisions | строка metrics | агрегированные счётчики, не поле сущности напрямую |
| "Memory sufficient?" Yes/No | пилюля memory-status | ⚠️ сущность/поле не определены |

### Состояния, зависящие от backend
- **Empty** — в Workspace 0 Sources/Claims/Decisions (сразу после создания, до первого run) → счётчики показывают 0, показана подсказка.
- **Loading** — после нажатия **Ask** → плейсхолдер "Generating answer..." вместо текста ответа.
- **Error** — не удалось сгенерировать ответ (ошибка LLM-вызова / память недоступна) → сообщение об ошибке + Retry.
- **Success** — ответ заполнен, ссылки активны, счётчики показаны; сюда же относится ветка `Memory sufficient? = Yes` (без дополнительного CTA).
- **Draft/review** — `Memory sufficient? = No` → ответ показан, но помечен как неполный, показан CTA **Create Follow-up Task** (это вариант Success с добавленным review-сигналом, а не отдельная ветка loading/error).

---

## Contract Dependencies & Open Questions

Это вопросы к backend/domain/API контракту, а не решения, которые фронтенд принял самостоятельно. Ничего из перечисленного ниже нельзя считать решённым, пока не подтверждено.

### 1. Review status для `EvidenceItem` (экраны 3, 2)
В `04-domain-model.md` у `EvidenceItem` нет поля review-состояния (аналога `Claim.status`) и нет audit-полей вроде `reviewed_by` / `reviewed_at`. Действиям **Approve / Reject / Edit** на экране 3 некуда записывать свой результат.
- Появится ли у `EvidenceItem` enum `status` (например, `pending` / `approved` / `rejected`)?
- Появятся ли `reviewed_by` / `reviewed_at`?
- **Текущее поведение UI:** Approve/Reject/Edit отрисованы как действия без подтверждённого сохраняемого эффекта.
- **Заблокировано:** нельзя реализовывать реальный переход Approve/Reject, пока это не появится в контракте.

### 2. Источник `Decision confidence` (экран 5)
В `04-domain-model.md` поле `confidence` есть у `Claim` и `EvidenceItem`, но не у `Decision`. На wireframe "Decision confidence" показан как отдельное значение в Decision Brief.
- Это реальное API-поле у `Decision`?
- Агрегируется на клиенте или на сервере из связанных `Claim.confidence`?
- Берётся из `ArtifactManifest` / вывода агента?
- **Текущее поведение UI:** трактуется как непрозрачное число из API; фронтенд ничего не вычисляет.
- **Заблокировано:** никакую логику агрегации нельзя строить на фронте, пока источник не определён.

### 3. Семантика `Send to Review` (экран 5)
Значения `Decision.status` — это `proposed`, `accepted`, `rejected`, `superseded`, `needs_review`; отдельного `in_review` нет, а на wireframe decision уже показан в статусе `Proposed` рядом с этой кнопкой.
- Меняет ли кнопка `Decision.status`? На какое значение — `needs_review`?
- Или она просто уведомляет/назначает reviewer'а без смены статуса?
- Может, `proposed` сам по себе уже означает "ожидает review", и кнопка делает что-то совсем другое (например, отправляет уведомление)?
- **Текущее поведение UI:** кнопка — визуальная заглушка (loading-индикатор + подтверждающий toast), без предполагаемого перехода статуса.
- **Заблокировано:** нельзя привязывать к этой кнопке реальное изменение статуса, пока workflow не определён.

### 4. Точный критерий "reviewed claims" для Accept (экран 5)
Доменное правило: *"Decision может стать `accepted` только с reviewed claims или с явным human override"* — но "reviewed" не определено через `Claim.status`.
- Должны быть non-`draft` **все** Linked Claims?
- Достаточно хотя бы одного non-`draft` claim?
- Считается ли `weak` reviewed? А `conflicting` / `outdated`?
- **Текущее поведение UI:** в состоянии `needs_review` кнопка **Accept** показывает пояснительный текст про правило, но логика разблокировки не реализована.
- **Заблокировано:** нельзя выбрать критерий разблокировки кнопки Accept, пока это не зафиксировано.

### 5. Механизм human override для Accept (экран 5)
То же доменное правило допускает "явный human override", обходящий требование reviewed claims, но ни в API, ни в UI формы для этого нет.
- Это флаг в запросе на accept? Отдельная проверка роли/права? Диалог подтверждения?
- **Текущее поведение UI:** контрола для override нет ни на wireframe, ни в backlog.
- **Заблокировано:** нельзя добавлять элемент override, пока это не определено.

### 6. Поле review note у `Claim` (экран 4)
На wireframe есть textarea **Review note** + **Save**, но в полях `Claim` из `04-domain-model.md` (`id`, `workspace_id`, `research_task_id`, `text`, `status`, `confidence`, `created_by`, `reviewed_by`, `created_at`, `updated_at`) поля для свободного текста нет.
- Должно ли у `Claim` появиться поле `review_note`, или это отдельная запись `Update`/комментарий?
- **Заблокировано:** нельзя подключать сохранение к кнопке **Save**, пока не появится поле/endpoint для этого.

### 7. Поле "Open Questions" у `Decision` (экран 5)
В Decision Brief есть карточка **Open Questions** ("What evidence could change this decision?"), но у `Decision` нет соответствующего поля в domain model.
- Это свободный текст, который пишет пользователь, или часть Decision Brief artifact, сгенерированная агентом?
- **Заблокировано:** до подтверждения источника считать это статическим/mock-содержимым.

### 8. Контракт Ask/Answer для Workspace (экран 6)
Use case "задать вопрос по workspace" назван в `03-backend-layered-architecture.md`, но нигде в документах нет схемы запроса/ответа: ни сущности для вопроса, ни для ответа, ни для флага "Memory sufficient", ни для агрегации счётчиков Sources/Claims/Decisions.
- Какая точная форма ответа (`answer_text`, `sources_count`, `claims_count`, `decisions_count`, `memory_sufficient`, ссылки)?
- Что определяет `memory_sufficient` — порог, суждение LLM, что-то ещё?
- **Заблокировано:** фронтенд может только мокать этот экран, пока не опубликован контракт ответа.

### 9. Нет сущности `User`/аккаунта в domain model
`Workspace.owner_id`, `ResearchTask.created_by`, `Claim.created_by`/`reviewed_by`, `Decision.decided_by`, `ResearchRun.started_by` ссылаются на пользователя, но в `04-domain-model.md` нет ни сущности `User`/аккаунта, ни auth-контракта.
- Это непрозрачные ID, которые фронтенд резолвит через отдельный identity/auth-сервис, или API должен сразу возвращать display name (как на wireframe, например "Alex Ivanov")?
- **Заблокировано:** имена reviewer/decided_by на wireframe захардкожены; фронтенду нужно знать реальный источник до реализации этого.

### 10. `Decision.status = superseded` — вопрос скоупа MVP
Присутствует в domain model, но явно не проработан ни в одном из 5 UI-состояний ни в `UI_State_Map.md`, ни в `implementation-backlog.md`. Там это прямо помечено как *out of scope для F1–F4 / requires clarification*, а не тихо проигнорировано.
- Нужно ли вообще отражать этот статус в MVP UI? Если да — где именно и каким состоянием?
- **Заблокировано:** нельзя добавлять вариант `StatusPill` или поведение экрана под `superseded`, пока на это нет ответа.

---

## Критерий готовности этого документа
- backend intern может открыть этот файл и понять, по каждому экрану, какие именно entities/fields/endpoints нужны фронтенду — без чтения фронтенд-кода.
- Каждый пробел выше сформулирован как вопрос к контракту (текущее поведение UI + что заблокировано), а не как молчаливое допущение фронтенда.
