# Полноценный CQRS (cqrs)

Обязательные опции: pseudo-cqrs

`cqrs` — это полноценное разделение write-side и read-side моделей.

В отличие от `pseudo-cqrs`, здесь команды и запросы разделены не только в коде, но и в модели обработки.

Полноценный CQRS нужен, когда:

* команды приходят из внешних систем;
* команды могут быть доставлены повторно;
* нужна идемпотентность обработки команд;
* read-model строится отдельно от write-model;
* read-model может быть eventually consistent;
* есть асинхронные обработчики;
* есть брокер сообщений, outbox, inbox или другие механизмы доставки;
* чтение и запись нужно масштабировать отдельно.

## Правила реализации

* `cqrs` должен строиться поверх `pseudo-cqrs`.
* Команды и запросы должны быть разделены в application layer.
* Команды должны изменять write-model через domain.
* Запросы должны читать read-model.
* Read-model не должна использоваться для выполнения команд.
* Write-model не должна использоваться как DTO для внешнего чтения.
* Команды должны иметь уникальный `command_id` или `idempotency_key`.
* Обработка команд должна быть идемпотентной.
* Для команд должен быть реализован command inbox.
* Command inbox не должен находиться в domain.
* Domain не должен знать про inbox.
* Presentation не должен реализовывать бизнес-логику inbox.
* Infrastructure должна содержать реализацию хранения inbox.
* Application layer должен использовать inbox через интерфейс.
* Command handler должен выполняться через middleware или wrapper, который проверяет inbox.
* Повторная обработка одной и той же команды не должна повторно менять агрегат.
* Повторная обработка уже успешно выполненной команды должна возвращать сохранённый результат или успешный no-op.
* Если команда была начата, но не завершена, стратегия поведения должна быть явной: retry, lock timeout, failed status или ручная обработка.
* Если read-model обновляется асинхронно, API должен учитывать eventual consistency.

## Где размещать inbox

Inbox лучше рассматривать как инфраструктурный механизм на границе application layer.

Рекомендуемое разделение ответственности:

```text
presentation:
  - принимает внешний запрос или сообщение;
  - извлекает command_id / idempotency_key;
  - преобразует payload в команду;
  - вызывает application;
  - подтверждает сообщение только после успешной обработки.

application:
  - содержит интерфейс inbox;
  - оборачивает command-handler в идемпотентную обработку;
  - решает, выполнять команду или вернуть результат уже выполненной команды;
  - управляет транзакционной границей обработки команды.

infrastructure:
  - реализует inbox storage;
  - хранит command_id, тип команды, статус, hash payload, результат, ошибки и timestamps;
  - обеспечивает атомарность проверки и фиксации обработки команды.

domain:
  - ничего не знает про inbox;
  - содержит только бизнес-модель, инварианты, события и правила изменения агрегата.
```

Пример интерфейса:

```go
type CommandInbox interface {
    Start(ctx context.Context, commandID string, commandName string, payloadHash string) (InboxStatus, error)
    Complete(ctx context.Context, commandID string, result []byte) error
    Fail(ctx context.Context, commandID string, reason error) error
}
```

Возможные статусы:

```go
type InboxStatus string

const (
    InboxStatusNew        InboxStatus = "new"
    InboxStatusProcessing InboxStatus = "processing"
    InboxStatusCompleted  InboxStatus = "completed"
    InboxStatusFailed     InboxStatus = "failed"
)
```

Пример структуры:

```text
/
  domain/
    product.go
    event.go
    repository.go

  readmodel/
    product.go
    repository.go
    projector.go

  application/
    application.go
    commands/
      create_product.go
      rename_product.go
      approve_product.go
    queries/
      get_product.go
      list_products.go
    inbox/
      command_inbox.go
      middleware.go

  infrastructure/
    product_repository.go
    product_read_repository.go
    inbox_repository.go
    outbox_repository.go

  presentation/
    http/
      product_handler.go
    consumer/
      product_command_consumer.go
```

## Рекомендуемый порядок обработки команды

1. Presentation принимает внешний запрос или сообщение.
2. Presentation извлекает `command_id` или `idempotency_key`.
3. Presentation маппит payload в command DTO.
4. Presentation вызывает application command-handler.
5. Application через inbox проверяет, не была ли команда уже обработана.
6. Если команда уже выполнена, application возвращает сохранённый результат или no-op.
7. Если команда новая, application начинает обработку.
8. Application загружает агрегат.
9. Application вызывает метод агрегата.
10. Application сохраняет агрегат.
11. Application сохраняет события или outbox, если они используются.
12. Application помечает команду как выполненную в inbox.
13. Presentation подтверждает внешний запрос или сообщение.

Желательно, чтобы шаги изменения агрегата, записи outbox и фиксации inbox находились в одной транзакции.

## В комбинации с другими опциями

### presentation-layer

Presentation layer является входной точкой для команд из внешнего мира.

Для HTTP presentation должен брать `idempotency_key` из заголовка или тела запроса.

Для брокера сообщений presentation должен брать `command_id` из envelope сообщения.

Presentation не должен самостоятельно проверять inbox, кроме технических проверок наличия id.

### application-layer

Application layer является основным местом orchestration для CQRS.

Именно здесь command-handler должен быть обёрнут в inbox/middleware.

Application layer должен зависеть от интерфейса inbox, а не от конкретной БД.

### read-model

Read-model является query-side моделью.

В полноценном CQRS read-model может обновляться отдельно от write-side.

Она может быть eventually consistent.

### business-events

Business-events являются удобным источником для построения read-model.

Агрегат порождает события, а проекторы read-model применяют их к query-side модели.

### Подписчики и диспетчеры событий

Подписчики могут использоваться как проекторы read-model.

Если проекторы работают асинхронно, нужно явно учитывать повторную доставку событий и идемпотентность проекторов.

### outbox

Outbox желательно использовать совместно с `cqrs`, если события должны публиковаться асинхронно.

Outbox решает задачу надёжной публикации событий после изменения write-model.

Inbox решает задачу надёжного приёма и идемпотентной обработки команд.

### value-objects

Команды могут содержать сырые DTO на границе application, но перед вызовом domain все бизнес-значения должны быть преобразованы в value objects.

### optimistic-locking

Optimistic locking может использоваться внутри write-side модели для защиты агрегата от конкурентного изменения.

Inbox не заменяет optimistic locking.

Inbox защищает от повторной доставки команды.

Optimistic locking защищает от конкурентной записи одного агрегата.

### states

Если используется `states`, события и проекторы read-model могут использовать состояние агрегата для построения query-side представлений.

Но read-model не должна получать возможность менять состояние агрегата напрямую.

---
