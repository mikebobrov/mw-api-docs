---
title: Создание кредитной заявки
---

Метод предназначен для создания кредитной заявки (начало сессии выдачи кредита)

## HTTP-Method: POST
## PATH: /partners/payments/init

### Пример запроса:

```

{
 "phone": "79031232299",
 "email": "test@test.com",
 "amount": 10000,
 "calculated_credit_cost": 12000, // опциональный параметр (если преварительный расчет стоимости кредита происходит на стороне партнера, данный параметр позволяет передать стоимость, которая была расчитана на стороне партнера)
 "success_callback_url": "http://yoursite.com/callbacks/success",
 "failure_callback_url": "http://yoursite.com/callbacks/failure",
 "redirect_url": "http://yoursite.com/order/234",
 "order_id": "partner_order_id", // идентификатор сессии со стороны партнера (по нему можно будет получать статус)
 "web_mode": "standalone", // в каком режиме будет открыт диалог заполнения заявки moneywall (standalone / iframe)
 "original_provider_order_id": "original_provider_order_id", // идентификатор заказа в системе партнера 
 "expires_at": "2017-07-01T00:00:00", // таймлити для заказа (опциональный параметр)
 // опционально - детали продукта, под который выдается кредит
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
           "arrival_dt": "2016-12-10T10:00:00",
           "departure_dt": "2016-12-10T08:00:00",
           "book_code": "Y",
           "orig": "SVO",
           "dest": "VIE",
           "flight_number": "606",
           "marketing_ac": "SU",
           "operating_ac": "SU"
         }
       ]
     }
   ],
   "passengers": [
     {
       "first_name": "IVAN",
       "middle_name": "FEDOROVICH",
       "last_name": "KRUZENSHTERN",
       "document_type": "local_passport",
       // local_passport, foreign_passport, birth_certificate, national_passport
       "document_number": "2349287349",
       "birth_date": "1950-10-03"
     }
   ]
 },
 // опционально - паспорт заемщика
 "local_passport": { 	
 	"birth_date": "1987-07-18", 	
 	"issue_date": "2001-01-01",
 	"number": "4444333222",
 	"first_name": "Иван",
 	"last_name": "Иванов",
 	"middle_name": "Александрович",
 	"sex": "male", // female
  "issuing_authority": "ОВД Района Калитники",
  "authority_code": "663402"  
 },
 // адрес прописки
 "address": {
    "appartment": "616",
    "city": "Москва",
    "house": "3",
    "index": "111123",
    "street": "ул. Китобойцев"
  },
 // опционально - загранпаспорт заемщика
 "international_passport": {
 	"country_code": "RU",
 	"birth_date": "1987-07-18",
 	"expiration_date": "2020-01-01",
 	"number": "723902034",
 	"first_name": "Ivanov",
 	"last_name": "Ivan"
 }
}

```

### Ответ
```

{
  "id": 10,
  "amount": 10000,
  "frame_url": "https://dev.moneywall.io/frame/credit_applications/10/payment_graph"
}

```
