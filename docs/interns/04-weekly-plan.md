# Недельный План

План рассчитан на 6-8 недель для первого MVP с возможностью продолжить работу дальше.

## Неделя 1: Продуктовый И Архитектурный Онбординг

Цели:

- понять product problem;
- понять modular monolith и layered architecture;
- изучить DocForge как reference;
- подготовить draft domain model и main screens.

Что готовит backend:

- notes on DocForge boundaries;
- draft ERD;
- proposal по status transitions.

Что готовит fullstack:

- MVP user journey;
- low-fidelity wireframes;
- UI state map.

Что готовит lead:

- финальные architecture decisions;
- review intern drafts;
- first agent/artifact contract review.

## Неделя 2: Базовый Каркас Проекта

Примечание:

Если после Недели 1 остались review comments или contract gaps, начало Недели 2 можно использовать как мягкий transition: сначала закрыть замечания, согласовать shared API shapes и подготовить implementation backlog. Skeleton implementation можно начинать draft PR'ами, когда backend/frontend contract достаточно понятен.

Цели:

- создать backend/frontend skeleton;
- настроить project conventions;
- определить shared API shapes;
- реализовать первый read/write flow.

Что готовит backend:

- package layout;
- initial migrations;
- draft repositories/unit-of-work;
- workspace/research task CRUD.

Что готовит fullstack:

- app shell;
- workspace list;
- workspace overview;
- basic forms.

## Неделя 3: Sources, Evidence, Claims

Цели:

- поддержать создание source;
- поддержать создание evidence;
- поддержать создание и linking claims.

Что готовит backend:

- source/evidence/claim APIs;
- link tables;
- domain tests для claim status rules.

Что готовит fullstack:

- research task page;
- source panel;
- evidence display;
- claim card.

## Неделя 4: Decisions И Export

Цели:

- создать decision from claims;
- показать traceability;
- экспортировать Markdown brief.

Что готовит backend:

- decision APIs;
- decision-claim relations;
- export use case.

Что готовит fullstack:

- decision creation flow;
- decision detail page;
- export preview.

## Неделя 5: Research Run И Artifacts

Цели:

- ввести research run lifecycle;
- писать artifact packages;
- показывать run status и artifacts.

Что готовит backend:

- research run model;
- artifact registry;
- local artifact store adapter;
- worker skeleton.

Что готовит fullstack:

- research run status UI;
- artifact/brief preview;
- review-required state.

Что готовит lead:

- первая версия agent orchestrator;
- structured output schemas;
- guardrail cases.

## Неделя 6: Drafting С Помощью Agent

Цели:

- генерировать source summaries;
- extract evidence;
- draft claims;
- generate decision brief.

Что готовит backend:

- LLM port integration point;
- run step persistence;
- error handling.

Что готовит fullstack:

- claim review workflow;
- evidence review workflow;
- visible draft vs reviewed state.

## Неделя 7: Follow-Up Questions

Цели:

- ask over workspace memory;
- cite sources/evidence/claims;
- create follow-up task, если memory недостаточно.

Что готовит backend:

- workspace search;
- ask use case;
- follow-up task creation.

Что готовит fullstack:

- follow-up question UI;
- answer with citations;
- create follow-up task action.

## Неделя 8: Demo, Evals, Polish

Цели:

- подготовить demo scenario;
- укрепить edge cases;
- проверить architecture drift;
- документировать next steps.

Что должно быть готово:

- end-to-end demo;
- test checklist;
- known limitations;
- next-phase backlog.
