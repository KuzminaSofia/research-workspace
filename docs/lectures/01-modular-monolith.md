# Мини-Лекция: Модульный Монолит

## Короткое Определение

Модульный монолит - это одно deployable приложение с сильными внутренними границами между модулями.

Это не "грязный монолит". Это осознанная архитектура:

```text
один deployable backend
много внутренних modules
четкие dependencies
явные contracts
```

## Почему Мы Его Используем

Для этого проекта modular monolith - хороший старт, потому что:

- команда маленькая;
- нужно понимать систему целиком;
- продуктовые границы еще будут уточняться;
- мы хотим clean architecture без distributed-system overhead.

## Отличие От Microservices

Microservices разделяют runtime deployment. Modular monolith сначала разделяет code ownership и dependencies.

Позже можно вынести отдельные modules:

- agent worker;
- search service;
- artifact service;
- LLM gateway.

Но не стоит платить этой сложностью до того, как продукт действительно этого потребует.

## Направление Зависимостей

```text
API / Worker
  -> Application
    -> Domain

Infrastructure implements Application ports.
```

Domain не должен знать про:

- FastAPI;
- SQLAlchemy;
- OpenAI SDK;
- S3;
- RabbitMQ;
- Next.js.

## Ментальная Модель

Когда добавляем feature, спрашиваем:

1. Какое здесь domain rule?
2. Какой use case?
3. Какие ports нужны use case?
4. Какой adapter реализует каждый port?
5. Что должен expose API?
6. Что должен показать UI?

Это защищает код от превращения в набор route handlers и database calls.
