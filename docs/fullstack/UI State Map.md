Этот документ описывает состояния (empty / loading / error / success / draft-review) для каждого из 6 основных экранов MVP, определённых в `Экраны.md`.

Состояния выведены из:
- статусов и полей сущностей в `04-domain-model.md`;
- ключевых доменных правил (раздел "Ключевые Правила");
- уже готовых wireframe'ов (F2), которые служат эталонным **success state**.

> Примечание по скоупу: `Decision.status = superseded` в domain model существует, но относится к Post-MVP follow-up-циклу (пересмотр решений после новой информации) и не входит в задачи F1–F4. Поэтому он не включён ни в один state ниже — это осознанное ограничение скоупа, а не пропуск.

---

## 1. Workspace Overview

**Основано на:** `Workspace`, список `ResearchTask`, список `Decision`.

- **Empty:**
  Workspace только что создан — ни одной `ResearchTask`, ни одного `Decision` ещё не существует. Вместо списков показывается placeholder-сообщение (например, "No research tasks yet") с призывом нажать **Create Research Task**. Секция **Ask Workspace** остаётся видимой, но с подсказкой, что задавать вопросы пока не о чем.

- **Loading:**
  Первичная загрузка списков Research Tasks и Decisions при открытии Workspace — вместо карточек показываются skeleton-плейсхолдеры.

- **Error:**
  Не удалось получить данные Workspace (сетевая ошибка / API недоступен) — вместо списков показывается сообщение об ошибке и кнопка **Retry**.

- **Success:**
  Текущий wireframe (экран 1) — заполненные списки Research Tasks (Market analysis / Review, Technology evaluation / Completed) и одно Decision со статусом Accepted.

- **Draft/review:**
  `ResearchTask.status = review` отображается бейджем **Review** прямо в списке (уже видно на макете у "Market analysis"). Это сигнализирует пользователю, что task ждёт его внимания, не блокируя переход в остальные разделы Workspace.

---

## 2. Research Task Page

**Основано на:** `ResearchTask`, `Source`, `ResearchRun`, `EvidenceItem`, `Claim`.

- **Empty:**
  Секция **Sources** пуста — ни один source ещё не добавлен. Кнопка **Run Research** заблокирована или сопровождается подсказкой "Add at least one source to run research", поскольку запуск research run без источников бессмысленно для домена. Секции Evidence / Claims / Summary также пусты.

- **Loading:**
  `ResearchRun.status = queued` или `running`. Блок **Research Run** показывает соответствующий бейдж ("Queued" / "Running"), секции Evidence и Claims показывают заглушку "Waiting for run to complete" вместо карточек.

- **Error:**
  `ResearchRun.status = failed`. Блок Research Run показывает бейдж **Failed** и текст из поля `ResearchRun.error_message`. Кнопка **Run Research** заменяется на **Retry Run**.
  Отдельно: `ResearchRun.status = cancelled` — вариант этого же состояния, но с другой семантикой (осознанная отмена, а не сбой): бейдж **Cancelled**, кнопка **Re-run**, `error_message` не показывается, так как это не ошибка.

- **Success:**
  Текущий wireframe (экран 2) — `ResearchRun.status = completed`, Evidence и Claims заполнены карточками, доступна кнопка **Create Decision** в блоке Summary.

- **Draft/review:**
  `ResearchRun.status = review_required` — именно это состояние показано на текущем макете (бейдж "Review Required" на Research Run, бейдж "Draft" на карточке Claim). Пользователю нужно перейти в Evidence/Claim панели и подтвердить результаты, прежде чем они станут доверенным знанием.

---

## 3. Source / Evidence Panel

**Основано на:** `Source`, `EvidenceItem`.

- **Empty:**
  Для данного source ещё не извлечено ни одного `EvidenceItem` (например, run ещё выполняется или завершился без находок по этому источнику) — вместо карточки Evidence Item показывается сообщение "No evidence extracted yet from this source".

- **Loading:**
  Идёт загрузка содержимого source (`raw_content_ref`) или конкретного evidence item — панель Source и панель Evidence Item показывают skeleton вместо текста.

- **Error:**
  Не удалось загрузить содержимое source (например, недоступен `raw_content_ref` или битая ссылка) — вместо превью показывается сообщение об ошибке; кнопка **Open original source** остаётся активной как fallback-путь для ручной проверки.

- **Success:**
  Текущий wireframe (экран 3) — Source и Evidence Item полностью заполнены: excerpt, Confidence, Location, Note присутствуют в карточке Evidence Item, действия Approve/Reject/Edit доступны.

- **Draft/review:**
  Само наличие кнопок **Approve / Reject / Edit** говорит о том, что evidence item по умолчанию находится в непроверенном состоянии — это review-состояние: данные извлечены агентом, но ещё не подтверждены человеком (домен-принцип: "только проверенные человеком evidence... становятся знанием команды").

---

## 4. Claim Review Panel

**Основано на:** `Claim`, связь many-to-many с `EvidenceItem`.

- **Empty:**
  Секция **Supporting Evidence** пуста — у claim ещё нет привязанных evidence items. По правилу "Claim без evidence не может быть `supported`" кнопка статуса **Supported** в этом состоянии задизейблена или явно помечена как недоступная, с подсказкой типа "Link evidence to mark as Supported".

- **Loading:**
  Идёт загрузка claim и связанных evidence items при открытии панели — карточки Claim и Supporting Evidence показывают skeleton.

- **Error:**
  Не удалось сохранить выбранный статус (сетевая ошибка при клике на Supported/Weak/Conflicting/Outdated) — показывается toast/сообщение об ошибке, выбранный статус визуально откатывается к предыдущему значению.

- **Success:**
  Текущий wireframe (экран 4) — claim имеет заполненную секцию Supporting Evidence, статус выбран пользователем, review note сохранена, указан reviewer.

- **Draft/review:**
  `Claim.status = draft` — стартовое состояние по правилу "AI-generated claim стартует как draft". Именно оно показано на текущем макете (бейдж "Draft" на карточке Claim и "Status: Draft" в блоке Set status справа), пока reviewer не выберет финальный статус.

---

## 5. Decision Page

**Основано на:** `Decision`, связь many-to-many с `Claim`.

- **Empty:**
  Секция **Linked Claims** пуста — decision только что создан (сразу после нажатия **Create Decision** на Research Task Page), claims для привязки ещё не выбраны. Поля Decision text / Rationale также пусты, кнопка **Accept** недоступна по определению (нет claims).

- **Loading:**
  Отправка действия **Accept / Send to Review / Reject** — соответствующая кнопка показывает индикатор загрузки, остальные кнопки временно неактивны.

- **Error:**
  Не удалось сохранить изменение статуса decision (сетевая ошибка при Accept/Reject) — сообщение об ошибке отображается рядом с блоком Review, статус decision остаётся прежним (например, Proposed), пользователь может повторить действие.

- **Success:**
  Текущий wireframe (экран 5) — `Decision.status = proposed`, Linked Claims заполнены (Supported / Supported / Weak), Confidence и Open Questions присутствуют, действия Accept/Send to Review/Reject доступны.

- **Draft/review:**
  `Decision.status = needs_review` — кнопка **Accept** недоступна или сопровождается явным пояснением "Accepted decisions require reviewed claims unless a human override is explicit" (текст уже присутствует под кнопками на макете), пока среди Linked Claims остаётся хотя бы один в статусе `draft`.

---

## 6. Follow-up Question Area

**Основано на:** агрегированные данные Workspace — `Source`, `Claim`, `Decision`; use case "задать вопрос по workspace".

- **Empty:**
  В Workspace ещё нет накопленных Sources/Claims/Decisions (состояние сразу после создания Workspace, до первого research run). Поле вопроса и кнопка **Ask** видны, но счётчики Sources / Claims / Decisions показывают 0, с подсказкой вида "Add research first to get answers from memory".

- **Loading:**
  После нажатия **Ask** — блок **Workspace Answer** показывает индикатор "Generating answer..." вместо текста ответа, счётчики временно скрыты или показаны как есть.

- **Error:**
  Не удалось сгенерировать ответ (ошибка LLM-вызова или недоступность памяти workspace) — в блоке Workspace Answer показывается сообщение об ошибке и кнопка **Retry**.

- **Success:**
  Текущий wireframe (экран 6) — вопрос задан, Workspace Answer заполнен текстом, ссылки View Evidence / View Claims / View Sources доступны, счётчики Sources / Claims / Decisions отображены.

- **Draft/review:**
  Блок **Memory sufficient? = No** — это review-сигнал: ответ дан, но система явно помечает его как неполный и предлагает **Create Follow-up Task**, не выдавая ответ за окончательное принятое знание (соответствует принципу "только проверенное человеком становится знанием команды").

---

## Проверка на полноту

Пройдено по каждому полю/статусу из `04-domain-model.md`, которое должно быть отражено в UI:

| Domain-статус / поле | Где отражено |
|---|---|
| `ResearchTask.status` (draft/in_progress/review/completed/archived) | Экран 1 (список Tasks), draft/review state |
| `ResearchRun.status` (queued/running/review_required/completed/failed/cancelled) | Экран 2 — все 6 значений явно расписаны по состояниям |
| `ResearchRun.error_message` | Экран 2, error state |
| `EvidenceItem` (excerpt/location/note/confidence) | Экран 3, success state |
| `Claim.status` (draft/supported/weak/conflicting/outdated) | Экран 4, draft/review + success state |
| `Decision.status` (proposed/accepted/rejected/needs_review) | Экран 5, success + draft/review state |
| Правило "Claim без evidence не может быть supported" | Экран 4, empty state |
| Правило "AI-generated claim стартует как draft" | Экран 4, draft/review state |
| Правило "Accepted decision требует reviewed claims" | Экран 5, draft/review state |
| Use case "задать вопрос по workspace" | Экран 6, все состояния |

`Decision.status = superseded` сознательно не включён (вне скоупа MVP-задач F1–F4, относится к Post-MVP follow-up-циклу).
