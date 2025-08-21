# Saga-хореография «Оформление заказа» — реестр событий

## События

Сага хореографическая: каждый сервис реагирует на события и создает свои, централизованного оркестратора нет.

| Этап                | Тип события     | Название события              | Кем публикуется            | Описание                                                                                                           |
|---------------------|-----------------|-------------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------|
| Инициация           | domain          | CheckoutRequested             | BFF/Order (внутр. команда) | Команда от BFF для создания заказа в статусе `PENDING`.                                                            |
| Инициация           | domain          | OrderCreated                  | Order                      | Заказ создан (PENDING). **Inventory** реагирует, пробует резерв. **Notification** может слать «заказ оформляется». |
| Резерв              | domain          | InventoryReservationRequested | Order                      | Триггер для резервирования.                                                                                        |
| Резерв              | domain          | InventoryReserved             | Inventory                  | Резерв выполнен. **Order** меняет статус на `AWAITING_PAYMENT`, публикует `PaymentRequested`.                      |
| Резерв              | failure         | InventoryReservationFailed    | Inventory                  | Резерв не выполнен. **Order** публикует `OrderCanceled` (причина: нет на складе), уведомления покупателю/продавцу. |
| Оплата              | domain          | PaymentRequested              | Order                      | Запросить оплату. **Payment** инициирует запрос к PSP.                                                             |
| Оплата              | domain          | PaymentSucceeded              | Payment                    | Оплата успешна (capture/authorize). **Order** публикует `ShipmentRequested`.                                       |
| Оплата              | failure         | PaymentFailed                 | Payment                    | Оплата отклонена. **Order** публикует `OrderCanceled`. **Inventory** реакция: `ReservationCanceled` (компенсация). |
| Оплата              | timeout/control | PaymentTimedOut               | Timer/Order                | Время на оплату вышло. **Order** → `OrderCanceled`. **Inventory** → `ReservationCanceled`.                         |
| Компенсация резерва | compensation    | ReservationCanceled           | Inventory                  | Разрезервировать товар (после отмены заказа).                                                                      |
| Возврат средств     | compensation    | RefundRequested               | Order/Payment              | Если деньги уже захолдированы/списаны и дальнейшие этапы провалились.                                              |
| Возврат средств     | compensation    | RefundSucceeded               | Payment                    | Возврат прошёл успешно. Уведомление покупателю.                                                                    |
| Доставка            | domain          | ShipmentRequested             | Order                      | Создать доставку. **Fulfillment** создаёт заявку у провайдера.                                                     |
| Доставка            | domain          | ShipmentDispatched            | Fulfillment                | Передано в доставку, есть трек. Уведомление покупателю/продавцу.                                                   |
| Доставка            | domain          | ShipmentDelivered             | Fulfillment                | Доставлено. **Order** → `OrderCompleted`.                                                                          |
| Доставка            | failure         | ShipmentFailed                | Fulfillment                | Ошибка/невозможность доставки. **Order** может инициировать `RefundRequested` и `ReservationCanceled`.             |
| Завершение          | domain          | OrderCompleted                | Order                      | Сага завершена успешно.                                                                                            |

## Идемпотентность и согласованность

- Все события имеют `eventId`, `correlationId`, `causationId`, версионирование (`v1`), и идемпотентную обработку (
  outbox, ключи повторов).
- Сервисы публикуют события из **outbox** (transactional outbox + CDC), чтобы не терять сообщения. Будем использовать
  Debezium.

## Пример события

```json5
{
  "eventId": "4f7a1a6e-8f5e-4b0b-9c43-6cf7e6e5f9f2", // уникальный айди события UUID
  "eventType": "OrderCreated", // название события
  "eventVersion": "v1", // версионирование
  "occurredAt": "2024-01-21T10:15:30.000Z",
  "correlationId": "c0e2b9d8-0c57-4b1d-9c2e-7c69f4c0f111", // айди всей цепочки бизнес операции
  "causationId": null, // айди родителя, который создал событие, будет null если это начало цепочки
  "producer": {
    "service": "order-service", // название сервиса
    "instance": "order-7f4b" // название пода
  },
  "payload": {
    "orderAmount": "1323.10" // вся необходимая информация для обработки события
  },
  "metadata": { // технические данные для отладки
    "traceId": "123213",
    "spanId": "133213"
  }
}
```
