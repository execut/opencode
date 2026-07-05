# Прикладной слой (application-layer)

Обязательные опции: нет

Прикладной слой отвечает за сценарии использования приложения.

Он оркестрирует выполнение операций, но не содержит бизнес-инварианты.

Прикладной слой можно использовать как отдельно, так и совместно с `ddd-light`, `read-model`, `business-events`, `presentation-layer`, `pseudo-cqrs` и `cqrs`.

Эта опция полезна, когда `service.go` становится слишком большим и в нём смешиваются:

* команды;
* запросы;
* транзакции;
* работа с репозиториями;
* вызовы доменных методов;
* обработка событий;
* интеграции;
* DTO для внешнего мира.

## Правила реализации

* Application layer должен содержать use-cases приложения.
* Один обработчик должен реализовывать один сценарий или один метод приложения.
* Application layer может содержать команды, запросы и процессоры.
* Application layer может управлять транзакциями.
* Application layer может загружать агрегаты через репозитории.
* Application layer может вызывать методы агрегатов.
* Application layer может сохранять агрегаты.
* Application layer может вызывать dispatcher для бизнес-событий.
* Application layer может работать с read-model при выполнении queries.
* Application layer не должен содержать бизнес-инварианты.
* Application layer не должен реализовывать HTTP, CLI, gRPC, consumer-логику и другие внешние адаптеры.
* Application layer не должен знать детали SQL, брокеров, HTTP-клиентов и других инфраструктурных реализаций.
* Бизнес-правила должны жить в domain.
* Технические детали должны жить в infrastructure.
* Внешнее взаимодействие должно жить в presentation.

Пример структуры:

```text
/
  application/
    application.go
    commands/
      create_product.go
      rename_product.go
      approve_product.go
      reject_product.go
    queries/
      get_product.go
      list_products.go
    processors/
      auto_approve_product.go
  domain/
    product.go
    repository.go
  repository.go
```

Если отдельный `contract` слой не используется, публичный интерфейс приложения может быть объявлен прямо в `application`.

Например:

```go
type Application interface {
    CreateProduct(ctx context.Context, cmd CreateProduct) error
    RenameProduct(ctx context.Context, cmd RenameProduct) error
    GetProduct(ctx context.Context, query GetProduct) (Product, error)
}
```

Если используется `contract` слой, интерфейсы приложения должны находиться в `contract`, а `application` должен их реализовывать.

## В комбинации с другими опциями

### ddd-light

Самая частая комбинация.

`application-layer` выносит из `service.go` прикладную оркестрацию, а `domain` содержит бизнес-логику.

Командный обработчик должен работать по схеме:

1. принять команду;
2. провалидировать технический формат команды, если нужно;
3. загрузить агрегат через репозиторий;
4. вызвать метод агрегата;
5. сохранить агрегат;
6. вернуть результат или ошибку.

### business-events

Application layer может после сохранения агрегата вызвать dispatcher событий.

Пример порядка:

1. загрузить агрегат;
2. выполнить действие;
3. сохранить агрегат;
4. отправить события в dispatcher;
5. очистить список событий, если это не делает репозиторий.

Конкретный порядок зависит от того, как реализована транзакционность и сохранение событий.

### Подписчики и диспетчеры событий

Application layer может зависеть от dispatcher-интерфейса.

Сам dispatcher может находиться в общем `pkg/ddd` или в инфраструктурном коде конкретного приложения.

Обработчик команды не должен напрямую знать всех подписчиков.

### read-model

Application layer должен разделять command-handlers и query-handlers.

Command-handlers работают с domain.

Query-handlers работают с read-model.

### presentation-layer

Presentation layer должен вызывать application layer, а не domain и не infrastructure напрямую.

Presentation преобразует внешний запрос в command/query и передаёт его в application.

### pseudo-cqrs

`application-layer` является обязательной опцией для `pseudo-cqrs`.

В этом случае в application явно разделяются:

* `commands`;
* `queries`;
* при необходимости `processors`.

### cqrs

В полноценном `cqrs` application layer становится местом обработки команд и запросов.

Также вокруг command-handlers может появиться command bus, inbox, middleware идемпотентности и транзакционная обвязка.

### value-objects

Application layer может создавать value objects из входных DTO перед вызовом домена.

Если ошибка создания VO является бизнес-ошибкой, она должна возвращаться из application наружу.

### optimistic-locking

Application layer должен передавать ожидаемую версию в доменный метод или репозиторий, если команда требует защиты от конкурентной записи.

Обработка конфликта версии должна быть явной ошибкой приложения.

---
