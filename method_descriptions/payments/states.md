---
title: Получение статуса платежных сессий за период
---

Метод предназначен для получения статуса платежной сессии.

## HTTP-Method: GET
## PATH: /partners/payments/states
## Params:

| название параметра | назначение                                          |
|--------------------|-----------------------------------------------------|
| from               | дата начала периода (формат yyyy-mm-ddTHH:MM:SS)    |
| till               | дата окончания периода (формат yyyy-mm-ddTHH:MM:SS) |

### Пример запроса

```
https://moneywall.io/api/partners/payments/states?from=2017-03-01T00:00:00&till=2017-04-01T01:00:30
```

### Ответ

```javascript
[
  {
    "order_id": "partner_order_id",
    "state": "processing"
  },
  {
    "order_id": "another_partner_order_id",
    "state": "authorized"
  }
]

```
