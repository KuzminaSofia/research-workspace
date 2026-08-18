# Список Для Чтения

Список намеренно практический. Читайте только те части, которые связаны с вашей текущей задачей.

## Для Всех

- Продуктовый концепт: `ai-research-decision-workspace-intern-brief.md`
- Обзор архитектуры: `docs/architecture/02-system-architecture.md`
- Слоистый backend: `docs/architecture/03-backend-layered-architecture.md`
- Доменная модель: `docs/architecture/04-domain-model.md`
- Мини-лекция: `docs/lectures/01-modular-monolith.md`

## Backend Intern

Темы:

- основы Clean Architecture / Hexagonal Architecture;
- Repository и Unit of Work patterns;
- SQLAlchemy Session и lifecycle транзакций;
- FastAPI routers и структура больших приложений;
- PostgreSQL relations и full-text search;
- background jobs и queues.

Рекомендуемые официальные docs:

- FastAPI Bigger Applications: https://fastapi.tiangolo.com/tutorial/bigger-applications/
- SQLAlchemy ORM: https://docs.sqlalchemy.org/en/20/orm/
- SQLAlchemy Session Basics: https://docs.sqlalchemy.org/en/20/orm/session_basics.html
- PostgreSQL Full Text Search: https://www.postgresql.org/docs/17/textsearch.html
- RabbitMQ Tutorials: https://www.rabbitmq.com/tutorials

## Fullstack Intern

Темы:

- Next.js App Router;
- server и client components;
- loading, error, empty states;
- master-detail UI;
- review workflow UI;
- URL state для search и filters.

Рекомендуемые официальные docs:

- Next.js Learn, App Router: https://nextjs.org/learn/dashboard-app/getting-started
- Next.js Search and Pagination: https://nextjs.org/learn/dashboard-app/adding-search-and-pagination

## Agent / AI Track

Этот track в первую очередь ведет lead, но всем важно понимать общую идею.

Темы:

- agent как bounded workflow;
- structured outputs;
- guardrails;
- tracing;
- память как artifacts, а не скрытая магия.

Рекомендуемые официальные docs:

- OpenAI Agents SDK: https://openai.github.io/openai-agents-python/
- OpenAI Agents Guardrails: https://openai.github.io/openai-agents-python/guardrails/
- Agents SDK Sessions: https://openai.github.io/openai-agents-js/guides/sessions/
