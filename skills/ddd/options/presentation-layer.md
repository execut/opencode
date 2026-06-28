# Слой представления (presentation-layer)

Обязательные опции: application-layer

Слой представления отвечает за взаимодействие приложения с внешним миром.

В этом слое находятся:

* HTTP handlers;
* gRPC handlers;
* CLI commands;
* message consumers;
* cron entrypoints;
* webhook handlers;
* GraphQL resolvers;
* другие внешние адаптеры.

Presentation layer должен преобразовывать внешний запрос в command/query приложения и вызывать application layer.

## Правила реализации

* Presentation layer не должен содержать бизнес-логику.
* Presentation layer не должен обращаться к domain напрямую.
* Presentation layer не должен обращаться к repository напрямую.
* Presentation layer не должен управлять бизнес-инвариантами.
* Presentation layer не должен знать детали сохранения агрегатов.
* Presentation layer должен отвечать за парсинг внешнего запроса.
* Presentation layer должен отвечать за преобразование внешних DTO в команды и запросы application layer.
* Presentation layer может отвечать за валидацию технического формата: обязательные поля, формат JSON, query-параметры, заголовки, route params.
* Presentation layer может отвечать за маппинг ошибок application layer во внешний протокол: HTTP status, gRPC code, CLI exit code.
* Presentation layer может отвечать за авторизацию на уровне внешнего протокола, если это не бизнес-правило домена.
* Presentation layer может извлекать request id, idempotency key, command id, user id и другие метаданные запроса.
* Presentation layer не должен реализовывать транзакции.
* Presentation layer не должен реализовывать SQL-запросы.
* Presentation layer не должен создавать бизнес-события.

Пример структуры:

```text
/
  presentation/
    http/
      product_handler.go
      mapper.go
    cli/
      product_command.go
    consumer/
      product_consumer.go
  application/
    commands/
    queries/
  domain/
```

Если используется `contract` слой, presentation должен знать только `contract`.

Если отдельный `contract` слой не используется, presentation должен знать только публичный интерфейс `application`.

## В комбинации с другими опциями

### application-layer

Базовая комбинация.

Presentation вызывает application.

Application вызывает domain, read-model и infrastructure через интерфейсы.

### ddd-light

Presentation не должен обращаться к domain напрямую, даже если `ddd-light` используется без полноценного `contract` слоя.

Внешние DTO должны маппиться в команды application layer.

### read-model

Presentation не должен читать read-model напрямую.

Запросы на чтение должны идти через query-handlers application layer.

### pseudo-cqrs

Presentation может иметь разные endpoint'ы или CLI-команды для команд и запросов.

Например:

* `POST /products` вызывает command-handler;
* `PATCH /products/{id}/name` вызывает command-handler;
* `GET /products/{id}` вызывает query-handler;
* `GET /products` вызывает query-handler.

Но presentation не должен сам решать, из какой модели читать данные.

### cqrs

В полноценном CQRS presentation может быть входной точкой для команд.

Presentation должен извлечь `command_id` или `idempotency_key` из внешнего запроса и передать его в application layer.

Если команда приходит из брокера сообщений, consumer находится в presentation layer.

Consumer должен:

1. прочитать сообщение;
2. извлечь command id;
3. преобразовать payload в команду application layer;
4. вызвать application;
5. подтвердить сообщение только после успешной обработки;
6. при ошибке вернуть ошибку в механизм retry/dead-letter.

Сам inbox не должен быть реализован как бизнес-логика presentation layer. Presentation только извлекает идентификатор команды и передаёт его дальше.

### value-objects

Presentation не должен создавать value objects, если это приводит к дублированию бизнес-валидации.

Обычно presentation принимает внешние строки, числа и JSON, а application layer создаёт VO перед вызовом domain.

Исключение: технические value objects внешнего протокола, не относящиеся к domain.

### optimistic-locking

Presentation может извлекать ожидаемую версию из HTTP-заголовка, query-параметра, body или сообщения брокера.

Но проверка версии должна выполняться не в presentation, а в application/domain/repository в зависимости от выбранной реализации optimistic locking.

---
