# Сага (saga)

## Описание
**Saga-based Orchestration** — это паттерн управления длительными транзакционными процессами, обеспечивающий согласованность данных в распределённых системах.

## Комбинации с другими опциями
* **scenario-of-transaction** — определяет стратегию обработки транзакций внутри саги (компенсирующие транзакции, откат, частичные подтверждения)
* **ddd-light** — базовая реализация с минимальным набором доменных концепций
* **ddd-high** — полная реализация с rich-domain и сложными инвариантами
* **cqrs** — разделение запросов и команд при выполнении шагов саги

## Основные компоненты
* **Orchestrator** — координирует выполнение шагов саги
* **Choreographer** — обрабатывает события и запускает последующие шаги
* **Компенсирующие транзакции** — механизмы отката при ошибках
* **Event Store** — хранилище событий саги

## Варианты реализации
* **Orchestration Saga** — централизованный контроллер процесса
* **Choreography Saga** — событийно-ориентированная модель
* **Compensating Transactions** — паттерн компенсирующих действий

## Преимущества
* Обработка длительных бизнес-процессов
* Отказоустойчивость и масштабируемость
* Атомарность операций на уровне бизнес-процессов
* Возможность отката частичных изменений
* Гибкая обработка ошибок

## Ограничения
* Сложность реализации и поддержки
* Необходимость обработки race conditions
* Потребность в надёжной системе сообщений
* Возможные проблемы с производительностью

## Пример реализации
```go
// Файл saga/ordersaga.go
package saga

import (
    "context"
    "github.com/example/domain"
    "github.com/example/ports"
)

type OrderSaga struct {
    OrderRepository    ports.OrderRepository
    PaymentGateway     ports.PaymentGateway
    EventBus          ports.EventBus
    TransactionPolicy  ports.TransactionPolicy
}

func (s *OrderSaga) Execute(ctx context.Context, orderID uuid.UUID) error {
    // Шаг 1: Создание заказа
    if err := s.OrderRepository.Create(ctx, orderID); err != nil {
        return err
    }
    
    // Шаг 2: Обработка платежа
    if err := s.PaymentGateway.Process(ctx, orderID); err != nil {
        // Компенсация при ошибке
        s.OrderRepository.Cancel(ctx, orderID)
        return err
    }
    
    // Шаг 3: Подтверждение заказа
    if err := s.OrderRepository.Confirm(ctx, orderID); err != nil {
        // Компенсация при ошибке
        s.PaymentGateway.Refund(ctx, orderID)
        return err
    }
    
    // Публикация события завершения
    s.EventBus.Publish(domain.OrderCompletedEvent{OrderID: orderID})
    return nil
}
```

## Рекомендации по внедрению
* Использовать только для действительно длительных процессов
* Тщательно проектировать компенсирующие действия
* Реализовать мониторинг состояния саг
* Обеспечить надёжное хранение состояния процесса
* Предусмотреть механизмы восстановления после сбоев

## Структура файлов
```
saga/
├── ordersaga.go          # Реализация саги для заказов
├── saga.go              # Базовый интерфейс саги
├── compensators.go      # Компенсирующие действия
├── eventstore.go        # Реализация хранилища событий
└── middleware/          # Вспомогательные компоненты
    ├── retry.go
    └── timeout.go
```