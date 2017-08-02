---
title: Предварительная валидация кредитной заявки
---

## HTTP-Method: POST
## PATH: /partners/credit_applications/validate

### Пример запроса:

```

{
 "phone": "79031232299",
 "email": "test@test.com",
 "amount": 10000,
 "registration_date": "2010-08-03",
 "orders_completed": 2,
 "product_type": "airline_tickets",
 "product": {  
   "cabin_type": "economy",
   // business, premium
   "validating_airline": "SU",
   "segments": [
     {
       "flights": [
         {
           "aircraft": "737",
           "arrival_dt": "2016-12-10T10:00:00", // обязательное поле
           "departure_dt": "2016-12-10T08:00:00", // обязательное поле
           "book_code": "Y",
           "orig": "SVO", // обязательное поле
           "dest": "VIE", // обязательное поле
           "flight_number": "606",
           "marketing_ac": "SU",
           "operating_ac": "SU"
         }
       ]
     }
   ]
 }
}

```

### Ответ
```

{
  "result": "declined" //approved
}

```
