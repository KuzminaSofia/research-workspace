# Первые Задачи

Это задачи для первой фазы проекта. Цель не только написать код, но и понять архитектуру.

## Backend Intern

### Задача B1: Изучить Границы В DocForge

Посмотреть:

- `technical-document-ml-service/app/src/technical_document_ml_service/domain/`
- `technical-document-ml-service/app/src/technical_document_ml_service/services/`
- `technical-document-ml-service/app/src/technical_document_ml_service/api/`
- `technical-document-ml-service/app/src/technical_document_ml_service/db/`
- `technical-document-ml-service/app/src/technical_document_ml_service/inference/contracts.py`

Что нужно подготовить:

- короткая Markdown-заметка: за что отвечает каждый слой;
- 3 примера хороших границ;
- 3 вещи, которые стоит адаптировать для нашего проекта.

### Задача B2: Черновик Доменной Модели

На основе `docs/architecture/04-domain-model.md` подготовить:

- draft ERD;
- список таблиц;
- many-to-many link tables;
- status fields и allowed values.

Что нужно подготовить:

- `docs/interns/submissions/backend-domain-model.md` или сообщение с такой же структурой

### Задача B3: Черновик API Contracts

Подготовить proposed endpoints для:

- workspaces;
- research tasks;
- sources;
- evidence;
- claims;
- decisions;
- research runs.

Для каждого endpoint указать:

- method;
- path;
- request body;
- response shape;
- error cases.

### Задача B4: Domain Rules

Написать правила простым языком:

- когда claim может стать `supported`;
- когда decision может стать `accepted`;
- что происходит, когда claim становится `outdated`;
- что означает agent-generated draft.

## Fullstack Intern

### Задача F1: Понять MVP Flow

Прочитать:

- `docs/architecture/01-product-vision.md`
- `docs/architecture/04-domain-model.md`

Что нужно подготовить:

- короткий user journey от создания workspace до decision brief;
- список screens, нужных для MVP.

### Задача F2: Wireframe Основных Экранов

Подготовить low-fidelity wireframes для:

- workspace overview;
- research task page;
- source/evidence panel;
- claim review panel;
- decision page;
- follow-up question area.

Что нужно подготовить:

- screenshots, Figma, Excalidraw, Markdown sketch или любой понятный формат

### Задача F3: UI State Map

Для каждого основного экрана перечислить:

- empty state;
- loading state;
- error state;
- success state;
- draft/review state.

### Задача F4: Evidence-First Claim Card

Спроектировать claim card, где видно:

- claim text;
- status;
- confidence;
- linked evidence;
- source title;
- review actions.

## Lead

Lead отвечает за первую версию:

- agent run lifecycle;
- artifact contract;
- structured LLM output schemas;
- guardrail cases;
- code review process;
- final architectural decisions.
