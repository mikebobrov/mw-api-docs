---
title: Возврат
---

## HTTP-Method: POST
## PATH: /partners/payments/refund

### Пример запроса:

```javascript
{
    "amount": 10000,
    "order_id": "partner_order_id"
}

```

### Ответ
HTTP статус `200` в случае если запрос обработан. Любой другой статус говорит о том что запрос обратан не был.
