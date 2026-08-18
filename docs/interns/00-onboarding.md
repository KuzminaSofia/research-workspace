# Онбординг

AI Research & Decision Workspace

Мы строим систему, которая помогает команде сохранять исследование как структурированную память.

Основная идея:

```text
research question -> sources -> evidence -> claims -> decisions -> updates
```

Проект не является просто AI-чатом и не является просто генератором отчета. Главная продуктовая ценность - traceability: каждый важный claim должен быть связан с evidence, а каждое decision должно показывать, почему оно было принято.

## Что Понять В Первую Очередь

Сначала прочитать:

1. `docs/architecture/01-product-vision.md`
2. `docs/architecture/02-system-architecture.md`
3. `docs/architecture/03-backend-layered-architecture.md`
4. `docs/architecture/04-domain-model.md`

Затем прочитать:

5. `docs/interns/01-reading-list.md`
6. `docs/interns/02-first-assignments.md`
7. `docs/interns/03-code-review-rules.md`
8. `docs/interns/04-weekly-plan.md`

## Инженерный Принцип

Мы строим feature by feature, но архитектурные границы важны с первого дня.

Хороший код в этом проекте:

- держит API тонким;
- не кладет domain logic в routers;
- держит LLM/search/storage за интерфейсами;
- имеет tests для business rules;
- создает читаемые artifacts для research runs.

## Референсный Проект

Используйте DocForge как референс по слоистой структуре:

`https://github.com/KuzminaSofia/docforge`

Обратите внимание на:

- `domain/`
- `services/`
- `api/`
- `db/`
- `messaging/`
- `workers/`
- `storage/`
- `inference/contracts.py`

Изучите границы и адаптируйте подход под этот проект
