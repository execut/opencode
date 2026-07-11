# Подписчики и диспетчеры событий
Обязательные опции: (ddd-light + business-events)

Опция полезна, когда необходимо реагировать на бизнес-события для целей вспомогательных поддоменов.

- `ddd`
  - `dispatcher.go` - диспетчер событий:
```go
type Subscriber interface {
    Handle(ctx context.Context, entity Entity, event Event) error
}

type Dispatcher struct {
    subscriberList []Subscriber
}

func NewDispatcher(subscriberList []Subscriber) *Dispatcher {
    return &Dispatcher{subscriberList: subscriberList}
}

func (d *Dispatcher) Dispatch(ctx context.Context, entity Entity) error {
    for _, subscriber := range d.subscriberList {
        for _, event := range entity.EventList() {
            err := subscriber.Handle(ctx, entity, event)
            if err != nil {
                return err
            }
        }
    }

    return nil
}
```
