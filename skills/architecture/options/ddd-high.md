# DDD high

Обязательные опции: ddd-light, business-events, application-layer, read-model, repository, states, subscribers-and-dispatchers, value-objects, pseudo-cqrs, optimistic-locking

DDD high - это самый навороченный вариант архитектуры на базе подхода DDD

Есть слои:
- Contract
- Application
- Domain
- Infrastructure
- Presentation

## Структура папок
Простая и проверенная структура. Все слои конкретного bounded context помещаем в папку контекста:

- `<context>/` — папка ограниченного контекста, где <context> - название контекста (в репозитории-примере это promotions). Контексты обычно лежат в папке service
  - `domain/` — папка для доменного (domain) слоя
    - `events.go` — доменные события и их структуры.
  - `application/` — оркестрация: use-cases, обработчики команд/процессоров, контракты
    - `contract/` - интерфейсы, контракт приложения
      - `application.go` - находится основной интерфейс приложения Application. В него тут встраиваются три других: Commands, Processors, Queries.
      - `commands.go` - интерфейс для команд (операций записи) контекста
      - `processors.go` - интерфейс для процессоров (фоновых операций), нескольких команд
      - `queries.go` - интерфейс для запросов
    - `queries/` - реализация операций чтения. Тут находятся обработчики (хандлеры), которые реализуют интерфейсы запросов из контракта 
    - `commands/` - реализация команд. Тут находятся обработчики (хандлеры), которые реализуют интерфейсы запросов из контракта
    - `processors/` - реализация процессоров из контракта
  - `infrastructure` — адаптеры: реализации репозиториев, внешние клиенты, БД и API-клиенты
  - `presentation` — API / CLI / handlers — слой взаимодействия с внешним миром
- `pkg` - пакет для обобщённого кода, который можно переиспользовать в разных контекстах
  - `ddd` - общий код для подхода ddd, тоже разделяется по слоям:
    - `domain` - общий код для бизнес-логики
    - `infrastructure` - общий код для инфраструктуры

## Обработчики
Внутри прикладного слоя есть обработчики нескольких видов: команды, запросы и процессоры. Каждый обработчик реализует
один метод приложения из контракта. Прикладной слой реализуется через встраивание в него обработчиков (хандлеров) нескольких видов.
Каждый обработчик - это реализация одного метода из контракта.
Пример обработчика:
```go
type PromotionActualizeHandler struct {
//...
}

func NewPromotionActualizeHandler(...) PromotionActualizeHandler {
return PromotionActualizeHandler{repository: repository, promotionRepository: promotionRepository}
}

func (h *PromotionActualizeHandler) PromotionActualize(ctx context.Context, cmd contract.PromotionActualize) error {
//...
}
```
## Хандлеры прикладного слоя

Хандлеры встраиваются в приложение через конструктор. Пример:
```go
func NewApplication(productGroupRepository domain.ProductGroupRepository, promotionRepository domain.PromotionRepository) contract.Application {
  return &App{
    //...
    appProcessors: appProcessors{
        processors.NewPromotionActualizeHandler(productGroupRepository, promotionRepository),
    },
  }
}
```
