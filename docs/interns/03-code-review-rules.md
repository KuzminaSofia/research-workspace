# Правила Ревью Кода

Code review проверяет, что implementation сохраняет продуктовое поведение и архитектурные границы.

## Общие Правила

- Держите changes достаточно маленькими для review.
- Добавляйте tests для domain/application behavior.
- Не прячьте business logic в API routers.
- Не обращайтесь к DB напрямую из frontend.
- Не вызывайте LLM/search/storage из domain code.
- Используйте typed request/response DTOs.
- Предпочитайте explicit status transitions вместо free-form strings.

## Чеклист Для Backend Review

- Соблюдает ли code направление зависимостей?
- Проверены ли domain rules без DB?
- Можно ли тестировать use cases с fake ports?
- Явно ли оформлены DB transactions?
- Понятно ли смоделированы many-to-many relations?
- Маппятся ли errors в API responses на API boundary?
- Учитывается ли user/workspace authorization?

## Чеклист Для Fullstack Review

- Может ли user понять цепочку от source до decision?
- Обработаны ли loading, empty и error states?
- Видно ли evidence рядом с claims?
- Отличаются ли визуально draft claims от reviewed claims?
- Не выглядит ли AI output как accepted truth?
- Не делает ли page domain logic, которая должна быть на backend?

## Чеклист Для Agent Review

- Каждый generated claim ссылается на evidence или явно говорит, что evidence не хватает?
- Результаты проходят schema validation?
- Prompt/tool outputs считаются untrusted?
- Artifacts записываются для каждого run?
- Можно ли audit run позже?
- Останавливается ли system или просит review, если evidence недостаточно?
