---
title: Описание запросов
---
Moneywall Partner API предназначен для партнеров для создания заявок на оформление кредита и проведения возвратов по выданным кредитам

Все вызовы методов осуществляются по протоколу `https`

Авторизации в API происходит с помощью токена, который передается в HTTP-заголовке `Authorization`

| Окружение | Путь к API     | Описание |
| -----------| ----------------| ----------|
| **staging**    | `https://dev.moneywall.io/api` | _Окружение предназначенное, для тестирования и разработки_ |
| **production** | `https://moneywall.io/api`     | _Рабочее окружение_ |

### Список методов
* [Начало платежной сессии](method_descriptions/payments/init)
* [Предварительная проверка кредитной заявки](method_desciprtions/credit_applications/validate)
* [Возврат](method_descriptions/payments/refund)
* [Получение статуса платежной сессии](method_descriptions/payments/status)
