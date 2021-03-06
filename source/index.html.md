---
title: Moneywall API | Документация

language_tabs: # must be one of https://git.io/vQNgJ
  - shell

toc_footers:

search: true
---
<h1 id='v1'> API v1 </h1>

<h2 id='introduction'>Общая информация</h2>

Moneywall – сервис онлайн-кредитования, который встраивается на страницы оплаты партнеров и позволяет покупателям приобретать товары в кредит.

API работает по протоколу HTTPS, все данные представлены в формате JSON.

Базовый URL – `https://dev.moneywall.io/api/partners`

Простой сценарий подачи и прохождения кредитной заявки:

  1. Вызов метода [Начала платежной сессии](#payment-init)
  2. Вывод фрейма по ссылке, полученной из результата запроса
  3. Ввод пользователем информации
  4. Перенаправление пользователя на `redirect_url` либо из `failure_redirect_url`
  5. Ожидание решения по заявке
  6. Вызов `success_callback_url` или `failure_callback_url` при одобрении или отклонении заявки соответственно

<h3 id='formula'>Формула расчета цены</h3>
Для расчёта и отображения платежей и полной стоимости заказа на стороне партнёра, следует использовать упрощенную формулу:

`ПЛАТЕЖ  = (СУММА ЗАКАЗА * K) / 3,`

где `K = 1.10` для случаев, когда вылет наступает после окончания договора займа (позднее 60 дней от даты подачи заявки),
<br>и `K = 1.21` для случаев, когда вылет наступает ранее даты окончания договора займа.

<h2 id='auth'>Аутентификация</h2>

Аутентификация производится через токен партнера, передаваемый в заголовке Authorization при каждом запросе:

`Authorization: SuperSecretTokenValue`

```shell
curl "https://dev.moneywall.io/api/partners/payments/init"
  -H "Authorization: SuperSecretTokenValue"
```

<aside class="notice">
Пожалуйста, замените <code>SuperSecretTokenValue</code> на предоставленный вам токен
</aside>

<h2 id='payments'>Платежные сессии</h2>

<h3 id='payment-init'>Начало платежной сессии</h3>

```shell
curl "https://dev.moneywall.io/api/partners/payments/init"
  -X POST
  -H "Content-Type: application/json"
  -H "Authorization: SuperSecretTokenValue"
  -d '{ "phone": "79031232299", "email": "test@test.com", "amount": 10000, "calculated_credit_cost": 12000, "success_callback_url": "http://yoursite.com/callbacks/success", "failure_callback_url": "http://yoursite.com/callbacks/failure", "redirect_url": "http://yoursite.com/order/234", "failure_redirect_url": "http://yoursite.com/order/234/failure", "order_id": "partner_order_id", "web_mode": "standalone", "original_provider_order_id": "original_provider_order_id", "expires_at": "2017-07-01T00:00:00", "product_type": "airline_tickets" }'

```

> Команда выше вернет ответ в JSON следующего вида:

```json
{
  "id": 10,
  "amount": 10000,
  "frame_url": "https://dev.moneywall.io/frame/credit_applications/10/payment_graph"
}
```

Метод предназначен для начала сессии выдачи кредита

**HTTP Запрос**

`POST https://dev.moneywall.io/api/partners/payments/init`

**Схема запроса**

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
phone | String | "79588249424" | Номер телефона клиента | Да
email | String | "user@gmail.com" | Адрес электронной почты клиента | Да
amount | Decimal | 10000 | Сумма заказа клиента на сайте партнера в рублях | Да
success_callback_url | String | "http://yoursite.com/callbacks/success" | Адрес callback-страницы, вызываемой при одобрении заявки | Да
failure_callback_url | String | "http://yoursite.com/callbacks/failure" | Адрес callback-страницы, вызываемой при отклонении заявки | Да
redirect_url | String | "http://yoursite.com/order/AB123-45" | Адрес страницы заказа в системе партнера | Да
failure_redirect_url | String | "http://yoursite.com/order/AB123-45/failed" | Страница для перенаправления в случае неуспеха | Нет
order_id | String | "vpih-234lgh" | Идентификатор сессии со стороны партнера (по нему можно будет [получать статус](#payment-state)) | Да
web_mode | String: "standalone", "iframe" | "standalone" | Режим, в котором будет открыт диалог заполнения кредитной заявки (по умолчанию: standalone)| Нет
original_provider_order_id | String | "AB123-45" | Идентификатор заказа в системе партнера | Нет
expires_at | DateTime | "2017-07-01T15:00:00+03:00" | Таймлимит для заказа | Нет
product_type | String | "airline_tickets"/"railway_tickets"/"accommodation" | Тип товара из заказа | Да
product | Object | См. [схему объекта product](#product-schema)| Детали товара из заказа | Да
local_passport | Object | См. [схему объекта local_passport](#local-passport-schema) | Общегражданский паспорт клиента | Нет
address | Object | См. [схему объекта address](#address-schema) | Адрес регистрации клиента | Нет
international_passport | Object | См. [схему объекта international_passport](#international-passport-schema) | Загранпаспорт клиента | Нет
meta | Object | См. [схему объекта meta](#meta-schema) | Дополнительная информация | Нет

> Пожалуйста, обратите внимание, что параметры с типом DateTime следует передавать в формате ISO8601 с часовым поясом. Рекомендуемый формат даты: `"%Y-%m-%dT%H:%M:%S%:z"`

<h3 id='product-add'>Добавление выписанных билетов к заказу</a></h3>

```shell
curl -X POST   \
 -H 'Authorization: SuperSecretTokenValue' \
 -F 'order_id=partner_order_id' \
 -F 'product[product_type]=airline_tickets' \
 -F 'product[product_items][][ticket_number]=123' \
 -F 'product[product_items][][ticket_receipt]=@path/to/file/test_pdf.pdf' \
https://dev.moneywall.io/api/partners/payments/product
```

Метод предназначен для сохранения номеров билетов и файлов маршрутных квитанций в кредитной заявке
**HTTP Запрос**

`POST https://dev.moneywall.io/api/partners/payments/product`

**URL параметры**

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
order_id | String | "vpih-234lgh" | Идентификатор платежной сессии, переданный в метод начала платежной сессии | Да
transaction_letter | File | @path/to/file/test_pdf.pdf | Транзакционное письмо | Нет
product | Object | См. [схему объекта product](#product) | Фактически выписанные билеты | Да

<h3 id='payment-state'>Статус платёжной сессии</a></h3>

```shell
curl "https://dev.moneywall.io/api/partners/payments/state?order_id=partner_order_id"
  -H "Authorization: SuperSecretTokenValue"
```

> Команда выше вернет ответ в JSON следующего вида:

```json
{
  "order_id": "partner_order_id",
  "state": "processing"
}
```

Метод предназначен для получения статуса платежной сессии

<aside class="notice">Метод возвращает статус кредитной заявки в системе Moneywall</aside>

**HTTP Запрос**

`GET https://dev.moneywall.io/api/partners/payments/state`

**URL параметры**

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
order_id | String | "vpih-234lgh" | Идентификатор платежной сессии, переданный в метод начала платежной сессии | Да

**Возможные статусы**

Описание логики отмены и закрытия договора при возвратах см. в [разделе Возврат](#payment-refund)

Код&nbsp;статуса | Описание
------------|----------
 processing | Платеж обрабатывается
 cancelled  | Платежная сессия отменена
 authorized | Первоначальный платеж оплачен, кредитный договор открыт
 voided     | Деньги за первоначальный платеж возвращены клиенту, кредитный договор отменен
 refunded   | Произведен полный возврат по кредитному договору
 charged    | Денежные средства за первоначальный платеж списаны, возврат по договору возможен только через досрочное погашение и закрытие договора


<h3 id='payment-states-array'>Список статусов платёжных сессий за период</a></h3>

```shell
curl "https://dev.moneywall.io/api/partners/payments/states?from=2019-01-01T23:35:14+0300&till=2019-01-13T00:02:13+0300"
  -H "Authorization: token"
```

> Команда выше вернет ответ в JSON следующего вида:

```json
[
  {
    "order_id":"partner_order_id_1",
    "state":"processing"
  },
  {
    "order_id":"partner_order_id_2",
    "state":"cancelled"
  }
]
```

Метод предназначен для получения статусов платёжных сессий за период

<aside class="notice">Метод возвращает статусы кредитных заявки в системе Moneywall за период</aside>

**HTTP Запрос**

`GET https://dev.moneywall.io/api/partners/payments/states`

**URL параметры**

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
from | DateTime | "2019-01-01T23:35:14+0300" | Время начала периода | Да
till | DateTime | "2019-01-13T00:02:13+0300" | Время окончания периода | Да

<h3 id='payment-refund'>Возврат</h3>

```shell
curl "https://dev.moneywall.io/api/partners/payments/refund"
  -X POST
  -H "Content-Type: application/json"
  -H "Authorization: SuperSecretTokenValue"
  -d '{ "amount": 100, "order_id": "partner_order_id" }'

```

> Команда выше вернет следущий ответ:

```
HTTP статус `200`, в случае если запрос обработан. Любой другой статус говорит о том, что запрос обработан не был.
```

Метод предназначен для проведение частичного или полного возврата средств по заказу

Частичные возвраты влекут за собой учет средств в пользу задолженности по договору.
Полные возвраты влекут за собой закрытие кредитного договора.

**Логика переходов между статусами договора**

Договор считается окончательно открытым и подтвержденным через 24 часа после момента его первоначального открытия.<br/>
До 24 часов с момента первоначального открытия договора, его можно отменить.

Соответственно:

- При полном возврате денежных средств по договору в течение 24 часов после открытия договора, происходит отмена договора и возврат денег на карту клиента путем войдирования платежной сессии;
- При полном возврате денежных средств по договору позднее 24 часов после открытия договора, оплата идет в счет погашения.

**HTTP Запрос**

`POST https://dev.moneywall.io/api/partners/payments/refund`

**URL параметры**

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
order_id | String | "vpih-234lgh" | Идентификатор платежной сессии, переданный в метод [начала платежной сессии](#payment-init) | Да
amount | Decimal | 100 | Сумма возврата в рублях | Да
currency | String | "RUB" | Код валюты в формате ISO-4217 | Нет
document_numbers | Array[String] | ["987654321", "678912345"] | Номера выпущенных документов на билеты/услуги, по которым запрашивается возврат в рамках заказа| Нет

<h2 id='entity-schemes'>Объекты и сущности</h2>
<h4 id='product-schema-airline'>Объект <code>product</code> при <code>product_type == 'airline_tickets'</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
cabin_type          | String | "economy" ("business", "premium") | Класс обслуживания | Нет
validating_airline  | String | "SU" | Валидирующая авиакомпания | Да
segments            | Array | См. [схему объекта segment](#segment-schema-airline) | Сегменты | Да
passengers          | Array | См. [схему объекта passenger](#passenger) | Пассажиры | Да

<h4 id='product-schema-railway'>Объект <code>product</code> при <code>product_type == 'railway_tickets'</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
segments            | Array | См. [схему объекта segment](#segment-schema-railway) | Сегменты | Да
passengers          | Array | См. [схему объекта passenger](#passenger) | Пассажиры | Да

<h4 id='product-schema-accommodation'>Объект <code>product</code> при <code>product_type == 'accommodation'</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
accommodations  | Array | См. [схему объекта accommodation](#accommodation-schema) | Варианты размещения | Да
guests          | Array | См. [схему объекта guest](#guest-schema) | Гости | Да


<h4 id='passenger-schema'>Объект <code>passenger</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
first_name      | String | "Ivan" | Имя | Да
middle_name     | String | "Aleksandrovich" | Отчество | Нет
last_name       | String | "Ivanon" | Фамилия | Да
document_type   | String | "international_passport" ("local_passport", "birth_certificate") | Тип документа | Да
document_number | String | "349287349" | Номер документа | Да
birth_date      | String | "1987-07-18" | Дата рождения | Да

<h4 id='guest-schema'>Объект <code>guest</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
first_name      | String | "Ivan" | Имя | Да
middle_name     | String | "Aleksandrovich" | Отчество | Нет
last_name       | String | "Ivanon" | Фамилия | Да
birth_date      | String | "1987-07-18" | Дата рождения | Да

<h4 id='segment-schema-airline'>Объект <code>segment</code> при <code>product_type == 'airline_tickets'</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
pnr   | String | 'P52DKC' | PNR сегмента | Да
flights   | Array | См. [схему объекта flight](#flight-schema) | Перелеты | Да

<h4 id='segment-schema-railway'>Объект <code>segment</code> при <code>product_type == 'railway_tickets'</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
routes   | Array | См. [схему объекта route](#route-schema) | Сегменты пути | Да

<h4 id='flight-schema'>Объект <code>flight</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
cabin_type    | String | "economy" ("business", "premium") | Класс обслуживания | Нет
aircraft      | String | "737" | Модель самолета | Да
departure_dt  | DateTime | "2016-12-10T08:00:00+03:00" | Время вылета  | Да
arrival_dt    | DateTime | "2018-12-10T10:00:00+03:00" | Время прилета | Да
book_code     | String | "Y" | Код бронирования | Да
orig          | String | "SVO" | Аэропорт вылета | Да
dest          | String | "VIE" | Аэропорт прилета | Да
flight_number | String | "606" | Номер рейса | Да
marketing_ac  | String | "SU" | Маркетинговый перевозчик | Да
operating_ac  | String | "SU" | Оперирующий перевозчик | Да

<h4 id='route-schema'>Объект <code>route</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
service_class | String | "ЗЛ" | Класс обслуживания | Да
departure_dt  | DateTime | "2016-12-10T08:00:00+03:00" | Дата и время отправления  | Да
from_station  | String | "МОСКВА ОКТЯБРЬСКАЯ" | Станция отправления | Да
to_station    | String | "САНКТ-ПЕТЕРБУРГ-ГЛАВН." | Станция прибытия | Да
train_number  | String | "606" | Номер поезда | Да
car_number    | String | "6" | Номер вагона | Да

<h4 id='accommodation-schema'>Объект <code>accommodation</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
type | String | "hotel" | Тип размещения | Да
price  | Decimal | 100 | Полная стоимость размещения | Да
name  | String | "Royal Resort" | Название места размещения | Да
country  | String | "France" | Страна размещения | Да
city  | String | "Nice" | Город размещения | Да
room_type  | String | "Single" | Категория размещения | Да
checkin_date  | Date | "2020-01-10" | Дата заезда | Да
checkout_date | Date | "2020-01-15" | Дата выезда | Да


<h4 id='local-passport-schema'>Объект общегражданского паспорта <code>local_passport</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
birth_date | Date | "1987-07-18" | Дата рождения | Да
birth_place | String | "Россия, г. Москва, ул. Ленина, д. 5" | Место рождения | Нет
issue_date | Date | "2001-01-01" | Дата выдачи паспорта | Да
number | String | "4444333222" | Серия и номер паспорта | Да
last_name | String | "Иванов" | Фамилия | Да
first_name | String | "Иван" | Имя | Да
middle_name | String | "Александрович" | Отчество | Да
sex | String: "female", "male" | "female" | Пол по паспорту | Да
issuing_authority | String | "ОВД Района Калитники" | Кем выдан | Да
authority_code | String | "663402" | Код подразделения | Да

<h4 id='address-schema'>Объект адреса <code>address</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
city | String | "Москва" | Город | Да
street | String | "ул. Китобойцев" | Улица | Да
building | String | "3" | Дом | Да
housing | String | "1" | Строение | Нет
apartment | String | "616" | Квартира | Нет
postcode | String | "111123" | Почтовый индекс | Да


<h4 id='international-passport-schema'>Объект загранпаспорта <code>international_passport</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
country_code | String | "RU" | Код страны | Да
birth_date | Date | "1987-07-18" | Дата рождения | Да
expiration_date | Date | "2020-01-01" | Срок истечения | Да
number | String | "723902034" | Номер | Да
first_name | String | "Ivanov" | Имя | Да
last_name | String | "Ivan" | Фамилия | Да

<h4 id='meta-schema'>Объект <code>meta</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
client_orders_count | Integer | 14 | Количество успешных заказов клиента | Нет
client_login | Boolean | true/false | Статус клиента (залогинен или нет) | Нет
orders_count | Integer | 3 | Сумма заказов клиента у партнера | Нет
book_code     | String | "Y" | Код бронирования | Нет
original_provider_order_id | String | "AB123-45" | Идентификатор заказа в системе партнера | Нет

<h4 id='product-schema'>Объект <code>product</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
product_type | String | "airline_tickets", "railway_tickets" | Тип билетов | Да
product_items | Array | См. [схему объекта product_items](#product-items) | Список билетов и маршрутных квитанций | Да

<h4 id='product-item-schema'>Объект <code>product_items</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
ticket_number | String | "421-288879826" | Номер билета | Да
ticket_receipt | File | @path/to/file/test_pdf.pdf | Файл маршрутной квитанции (pdf, doc, docx). Не больше 8MB | Да


<h2 id='payment-cards'>Тестовые карты</h2>

Номер карты  | CVV | Срок действия | Результат
--------- | --------- | --------- | ---------
4111111111111112 | 123 | 2021/12 | успешный платеж без 3DS с опциональным CVV
4111111111100031 | 123 | 2021/12 | успешный платеж без 3DS с опциональным CVV
4111111111100023 | 123 | 2021/12 | успешный платеж без 3DS с обязательным CVV
5486732058864471 | 123 | 2021/12 | успешный платеж с 3DS
4111111111111111 | 123 | 2021/12 | успешный платеж с 3DS
4111111111111114 | 123 | 2021/12 | неуспешный платеж с 3DS
4111111111111115 | 123 | 2021/12 | неуспешный платеж с 3DS
7000000000000007 | 521 | 2021/12 | неуспешный платеж («недостаточно средств»)
8000000000000008 | 521 | 2021/12 | неуспешный платеж («неверный срок действия карты»)
7600000000000006 | 521 | 2021/12 | неуспешный платеж («номер карты в черном списке»)
1234561999999999 | 374 | 2021/12 | неуспешный платеж («несуществующая карта»)
4111101000000046 | 123 | 2021/12 | неуспешный платеж («недостаточно средств») во время блокировки (при превышении суммы в 100 рублей (Amount=10001))
4100401111100062 | 123 | 2021/12 | Таймаут 40 секунд во время блокировки
4100401111100724 | 123 | 2021/12 | Таймаут 40 секунд во время разблокировки
4100401111100328 | 123 | 2021/12 | Таймаут 40 секунд во время списания
4100401111103025 | 123 | 2021/12 | Таймаут 40 секунд во время возврата

<h1 id='v1-eng'> [ENG] API v1 </h1>

<h2 id='introduction-eng'>General Information</h2>

Moneywall is an online credit service, which can be built into partner checkout pages and allow customers to buy a product on credit.

An API works on HTTPS protocol, all data is presented in JSON format.

Base URL –  `https://dev.moneywall.io/api/partners`

A simple scenario of application submitting and processing:

1.  Method invocation [Start of a payment session](#payment-init-eng)
2.  Output of a frame window on the link from search results
3.  User input
4.  Redirecting user to URL from `redirect_url`
5.  Waiting for a decision on the application
6.  Invocation of `success_callback_url`  or  `failure_callback_url`  when the application is approved or rejected, respectively

<h3 id='calc-eng'>Price Calculation Formula</h3>

To calculate and display payments and the total cost of the order on partner side, a simplified formula is used
`payment = (order sum * K) / 3,`

where  `K = 1.10`  if the flight is scheduled after the credit agreement expires (later than 60 days from the date of application),
and  `K = 1.21` if the flight is scheduled before the credit agreement expires

<h2 id='auth-eng'>Authentification</h2>

Authentification occurs through partner token, passed in Authorization on each request:

`Authorization: SuperSecretTokenValue`

```shell
curl "https://dev.moneywall.io/api/partners/payments/init"
 -H "Authorization: SuperSecretTokenValue"
```

<aside class="notice">
Please replace `SuperSecretTokenValue` with the token given to you
</aside>

<h2 id='payments-eng'>Payment Sessions</h2>

<h3 id='payments-init-eng'>Start of a Payment Session</h3>

```shell
curl "https://dev.moneywall.io/api/partners/payments/init"
 -X POST
 -H "Content-Type: application/json"
 -H "Authorization: SuperSecretTokenValue"
 -d '{ "phone": "79031232299", "email": "test@test.com", "amount": 10000, "calculated_credit_cost": 12000, "success_callback_url": "http://yoursite.com/callbacks/success", "failure_callback_url": "http://yoursite.com/callbacks/failure", "redirect_url": "http://yoursite.com/order/234", "order_id": "partner_order_id", "web_mode": "standalone", "original_provider_order_id": "original_provider_order_id", "expires_at": "2017-07-01T00:00:00", "product_type": "airline_tickets" }'
```

> The above call returns a response in JSON of the following type

```json
{
 "id": 10,
 "amount": 10000,
 "frame_url": "https://dev.moneywall.io/frame/credit_applications/10/payment_graph"
}
```

The method is used to start start a credit application procedure

**HTTP Request**

`POST https://dev.moneywall.io/api/partners/payments/init`

**Query schema**

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
phone | String | "79588249424" | Customer's telephone number | Yes
email | String | "user@gmail.com" | Customer's e-mail address | Yes
amount | Decimal | 10000 | The sum of the customer's order on partner website in rubles | Yes
success_callback_url | String | "http://yoursite.com/callbacks/success" | URL of callback-page, called when the application is approved | Yes
failure_callback_url | String | "http://yoursite.com/callbacks/failure" | URL of callback-page, called when the application is rejected | Yes | redirect_url | String | "http://yoursite.com/order/AB123-45" | URL of order page in partner system | Yes
order_id | String | "vpih-234lgh" | Session ID on partner side (which will allow to [receive status](#payment-state-eng)) | Yes
web_mode | String: "standalone", "iframe" | "standalone" | The mode in which the application form will be opened (default: standalone) | No | original_provider_order_id | String
"AB123-45" | Order ID in partner system | No | expires_at | DateTime | "2017-07-01T15:00:00+03:00" | Timelimit for an order | No
product_type | String | "airline_tickets"/"railway_tickets" | The type of product, to buy which a credit is granted | product | Object | See  [product schema](#product-schema-eng) | The details of product, to buy which a credit is granted | Yes, if the type of product is passed
local_passport | Object | See  [local_passport schema](#local-passport-schema-eng) | Customer's state ID card | No | address | Object | See  [address schema](#address-schema-eng) | Customer's address | No
international_passport | Object | See  [international_passport schema](#international-passport-schema-eng) | Customer's international passport | No | meta | Object | See  [meta schema](#meta-schema-eng) | Additional information | No

> Please, note that parameters of DateTime type should be passed in ISO8601 format with the time zone. The recommended date format:  `"%Y-%m-%dT%H:%M:%S%:z"`

<h3 id='product-add-eng'>Adding issued tickets to the order</h3>

```shell
curl -X POST   \
-H 'Authorization: SuperSecretTokenValue' \
-F 'order_id=partner_order_id' \
-F 'product[product_type]=airline_tickets' \
-F 'product[product_items][][ticket_number]=123' \
-F 'product[product_items][][ticket_receipt]=@path/to/file/test_pdf.pdf' \
https://dev.moneywall.io/api/partners/payments/product
```

This action adds ticket numbers and ticket receipts into credit application

**HTTP Request**

`POST https://dev.moneywall.io/api/partners/payments/product`

**URL parameters**
Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
order_id | String | "vpih-234lgh" | Payment session ID, passed to the method of starting a payment session | Yes
transaction_letter | File | @path/to/file/test_pdf.pdf | Transaction letter | No
product | Object | See [product schema](#product-eng) | Issued tickets | Yes


<h3 id='payments-state-eng'>Payment Session Status</h3>

```shell
curl "https://dev.moneywall.io/api/partners/payments/state?order_id=partner_order_id"
 -H "Authorization: SuperSecretTokenValue"
```

> The action above returns a response in JSON of the following type

```json
{
 "order_id": "partner_order_id",
 "state": "processing"
}
```

<aside class="notice">The method returns a credit application status in Moneywall system</aside>

**HTTP Request**

`GET https://dev.moneywall.io/api/partners/payments/state`

**URL parameters**

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
order_id | String | "vpih-234lgh" | Payment session id, passed to the method of starting a payment session | Yes

**Statuses list**

For a description of the cancellation and closing flow for returns, see [refunds](#payment-refund-eng).

Status&nbsp;code | Description
------------|----------
processing | Payment processing
cancelled | Payment session cancelled
authorized | The first payment is made, the credit agreement is concluded
voided | The first payment is refunded to the customer, the credit agreement is cancelled
refunded | The full refund under the agreement is made
charged | The first payment is charged, refund is possible only by early refund and agreement closing

<h3 id='payments-states-array-eng'>List of payment sessions statuses for a period</h3>

```shell
curl "https://dev.moneywall.io/api/partners/payments/states?from=2019-01-01T23:35:14+0300&till=2019-01-13T00:02:13+0300"
 -H "Authorization: token"

```

> The action returns a response in JSON of the following type:

```json
[
 {
   "order_id":"partner_order_id_1",
   "state":"processing"
 },
 {
   "order_id":"partner_order_id_2",
   "state":"cancelled"
 }
]

```

The method returns credit application statuses in Moneywall system for a period

**HTTP Request**

`GET https://dev.moneywall.io/api/partners/payments/states`

**URL parameters**

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
from | DateTime | "2019-01-01T23:35:14+0300" | Time of period start | Yes
till | DateTime | "2019-01-13T00:02:13+0300" | Time of period end | Yes

<h3 id='payment-refund-eng'>Refund</h3>

```
curl "https://dev.moneywall.io/api/partners/payments/refund"
 -X POST
 -H "Content-Type: application/json"
 -H "Authorization: SuperSecretTokenValue"
 -d '{ "amount": 100, "order_id": "partner_order_id" }'


```

> The action above returns the following response:

```
HTTP status `200`, if the request was processed. Any other status means that the request was not processed.

```

The is used for making a partial or full refund for order

A partial refund causes the accounting of funds in favour of the debt for a loan.
A full refund causes closing of a loan.

**Transitions between agreement statuses**

An agreement is considered to be finally opened and confirmed 24 hours after its initial opening.<br/>
An agreement can be canceled within 24 hours after its initial opening.

Consequently:

-   In case of a full refund within 24 hours after the agreement opening, the agreement is canceled and the money is returned to the client's card by voiding of the payment session;
-   In case of a full refund later than 24 hours after the agreement opening, the payment is considered as a refund.

**HTTP Request**

`POST https://dev.moneywall.io/api/partners/payments/refund`

**URL parameters**

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
order_id | String | "vpih-234lgh" | Payments session ID, passed to the method [start of a payment session](#payment-init-eng) | Yes
amount | Decimal | 100 | Repayment in rubles | Yes
currency | String | 'RUB' | Currency code in ISO-4217 format | No
document_number | String | '987654321' | The number of the document issued for the ticket / service for which the refund is requested within the order | No

<h2 id='entity-schemas-eng'>Objects and Entities</h2>

<h4 id='product-schema-airline-eng'>Schema of <code>product</code> when <code>product_type == 'airline_tickets'</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
cabin_type | String | "economy" ("business", "premium") | Class | Yes
validating_airline | String | "SU" | Validating carrier | Yes
segments | Array | See  [segment schema](#segment-schema-eng) | Segments | Yes
passengers | Array | See  [passenger schema](#passenger-eng) | Passengers | Yes

<h4 id='product-schema-railway-eng'>Schema of <code>product</code> when <code>product_type == 'railway_tickets'</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
segments | Array | See  [segment schema](#segment-schema-eng) | Segments | Yes
passengers | Array | See  [passenger schema](#passenger-eng) | Passengers | Yes

<h4 id='product-schema-accommodation-eng'>Schema of <code>product</code> when <code>product_type == 'accomodation'</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
accommodations  | Array | См. [accommodation schema](#accommodation-schema) | Accomodation options | Yes
guests          | Array | См. [guest schema](#guest-schema) | Guests | Yes

<h4 id='passenger-schema-eng'>Schema of <code>passenger</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
first_name | String | "Ivan" | Name | Yes
middle_name | String | "Aleksandrovich" | Patronymic | No
last_name | String | "Ivanov" | Surname | Yes
document_type | String | "international_passport" ("local_passport", "birth_certificate") | Document type | Yes
document_number | String | "349287349" | Document number | Yes
birth_date | String | "1987-07-18" | Date of birth | Yes

<h4 id='guest-schema-eng'>Schema of <code>guest</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
first_name      | String | "Ivan" | First name | Yes
middle_name     | String | "Aleksandrovich" | Patronymic | No
last_name       | String | "Ivanon" | Last name | Yes
birth_date      | String | "1987-07-18" | Date of birth | Yes

<h4 id='segment-schema-airline-eng'>Schema of <code>segment</code> when <code>product_type == 'airline_tickets'</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
pnr   | String | 'P52DKC' | PNR of segment | Yes
flights | Array | See [flight schema](#flight-schema-eng) | Routes | Yes
routes | Array | See  [route schema](#route-schema-eng) | Routes | Yes

<h4 id='segment-schema-railway-eng'>Schema of <code>segment</code> when <code>product_type == 'railway_tickets'</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
routes | Array | See  [route schema](#route-schema-eng) | Routes | Yes

<h4 id='flight-schema-eng'>Schema of <code>flight</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
cabin_type | String | "economy" ("business", "premium") | Class | No
aircraft | String | "737" | Aircraft | Yes
departure_dt | DateTime | "2016-12-10T08:00:00+03:00" | Departure date and time | Yes
arrival_dt | DateTime | "2018-12-10T10:00:00+03:00" | Arrival date and time | Yes
book_code | String | "Y" | Booking code | Yes
orig | String | "SVO" | Departure airport | Yes
dest | String | "VIE" | Arrival airport | Yes
flight_number | String | "606" | Flight | Yes
marketing_ac | String | "SU" | Marketing carrier | Yes
operating_ac | String | "SU" | Operating carrier | Yes

<h4 id='route-schema-eng'>Schema of <code>route</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
service_class | String | "ЗЛ" | Class | Yes
departure_dt | DateTime | "2016-12-10T08:00:00+03:00" | Departure date and time | Yes
from_station | String | "MOSKVA OKTYABRSKAYA" | From | Yes
to_station | String | "SANKT-PETERBURG-GLAVNII" | To | Yes
train_number | String | "606" | Train | Yes
car_number | String | "6" | Coach | Yes

<h4 id='accommodation-schema-eng'>Schema of <code>accommodation</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
type | String | "hotel" | Accomodation type | Yes
price  | Decimal | 100 | Total cost | Yes
name  | String | "Royal Resort" | Accomodation name | Yes
country  | String | "France" | Country | Yes
city  | String | "Nice" | City | Yes
room_type  | String | "Single" | Room type | Yes
checkin_date  | Date | "2020-01-10" | Date of checkin | Yes
checkout_date | Date | "2020-01-15" | Date of checkout | Yes

<h4 id='local-passport-schema-eng'>Schema of state ID card <code>local_passport</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
birth_date | Date | "1987-07-18" | Date of birth | Yes
birth_place | String | "Lenin St. 5, Moscow, Russia" | Place of birth | No
issue_date | Date | "2001-01-01" | Date of issue | Yes
number | String | "4444333222" | Series and number of ID card | Yes
last_name | String | "Ivanov" | Surname | Yes
first_name | String | "Ivan" | Name | Yes
middle_name | String | "Aleksandrovich" | Patronymic | Yes
sex | String: "female", "male" | "female" | Sex in ID card | Yes
issuing_authority | String | "OVD Rayona Kalitniki" | Issuing authority | Yes
authority_code | String | "663402" | Authority code | Yes

<h4 id='address-schema-eng'>Schema of <code>address</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
city | String | "Moscow" | City | Yes
street | String | "Kitoboytsev St." | Street | Yes
building | String | "3" | Number | Yes
housing | String | "1" | Building | No
apartment | String | "616" | Flat | No
postcode | String | "111123" | Postcode | Yes

<h4 id='international-passport-schema-eng'>Schema of <code>international_passport</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
country_code | String | "RU" | Country code | Yes
birth_date | Date | "1987-07-18" | Date of birth | Yes
expiration_date | Date | "2020-01-01" | Expiration date | Yes
number | String | "723902034" | Number | Yes
first_name | String | "Ivanov" | Name | Yes
last_name | String | "Ivan" | Surname | Yes

<h4 id='meta-schema-eng'>Schema <code>meta</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
client_orders_count | Integer | 14 | The number of customer's successful orders | No
client_login | Boolean | true/false | Customer's status (signed in or not) | No
orders_count | Integer | 3 | The sum of customer's orders at partner | No
book_code | String | "Y" | Booking code | No
original_provider_order_id | String | "AB123-45" | Order ID in partner system | No

<h4 id='product-schema-eng'>Schema of <code>product</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
product_type | String | "airline_tickets", "railway_tickets" | Type of ticket | Yes
product_items | Array | See [schema of product_items](#product-items-eng) | A list of tickets and ticket receipts | Yes

<h4 id='product-item-schema-eng'>Schema of <code>product_items</code></h4>

Parameter | Type | Example | Description | Mandatory
--------- | --------- | --------- | --------- | ---------
ticket_number | String | "421-288879826" | Number of ticket | Yes
ticket_receipt | File | @path/to/file/test_pdf.pdf | Ticket receipt file (pdf, doc, docx). Not exceeding 8MB | Yes


<h2 id='payment-cards-eng'>Test cards</h2>

Card number | CVV | Valid through | Result
--------- | --------- | --------- | ---------
4111111111111112 | 123 | 2021/12 | Successful payment without 3DS with optional CVV
4111111111100031 | 123 | 2021/12 | Successful payment without 3DS with optional CVV
4111111111100023 | 123 | 2021/12 | Successful payment without 3DS with optional CVV
5486732058864471 | 123 | 2021/12 | Successful payment with 3DS
4111111111111111 | 123 | 2021/12 | Successful payment with 3DS
4111111111111114 | 123 | 2021/12 | Unsuccessful payment with 3DS
4111111111111115 | 123 | 2021/12 | Unsuccessful payment with 3DS
7000000000000007 | 521 | 2021/08 | Unsuccessful payment («not sufficient funds»)
8000000000000008 | 521 | 2021/09 | Unsuccessful payment («invalid expiration date»)
7600000000000006 | 521 | 2021/08 | Unsuccessful payment («card number on blacklist»)
1234561999999999 | 374 | 2021/12 | Unsuccessful payment («invalid card number»)
4111101000000046 | 123 | 2021/12 | Unsuccessful payment («not sufficient funds») during authorization (upon exceeding the amount of 100 rubles (Amount=10001))
4100401111100062 | 123 | 2021/12 | Timeout 40 seconds during authorization
4100401111100724 | 123 | 2021/12 | Timeout 40 seconds during void
4100401111100328 | 123 | 2021/12 | Timeout 40 seconds during charge
4100401111103025 | 123 | 2021/12 | Timeout 40 seconds during refund

<h1 id='v2'> API v2 </h1>

Ниже представлен проект документации по новой версии API

<h2 id='entities-v2'>Системные сущности</h2>

* [Кредитная Заявка](#credit-application-entity-v2)

* [Кредитный Договор](#credit-agreement-entity-v2)

* [Электронная подпись](#credit-signature-entity-v2)

* [Фотография Документа](#document-photo-entity-v2)

* [Профиль социальной сети](#social-profile-entity-v2)

* [Первоначальный платеж](#initial-payment-entity-v2)

* [Банковская карта](#credit-card-entity-v2)

* [Решение по заявке](#decision-entity-v2)

* [Текущий статус заявки](#credit-application-state-entity-v2)

* [График платежей](#payment-graph-entity-v2)

* [Очередной платеж](#scheduled-payment-entity-v2)

* [Попытка списания](#payment-attempt-entity-v2)

* [Состоявшийся платеж](#charged-payment-entity-v2)

* [Список возвратов](#refunds-entity-v2)

* [Возврат](#refund-entity-v2)

* [Ввод договора в действие](#activation-entity-v2)

* [Товар](#product-entity-v2)


<h3 id='credit-application-entity-v2'> Кредитная заявка </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
phone | String | "79588249424" | Номер телефона клиента
email | String | "user@gmail.com" | Адрес электронной почты клиента
amount | Decimal | 10000 | Сумма заказа клиента на сайте партнера в рублях
success_callback_url | String | "http://yoursite.com/callbacks/success" | Адрес callback-страницы, вызываемой при одобрении заявки
failure_callback_url | String | "http://yoursite.com/callbacks/failure" | Адрес callback-страницы, вызываемой при отклонении заявки
redirect_url | String | "http://yoursite.com/order/AB123-45" | Адрес страницы заказа в системе партнера
order_id | String | "vpih-234lgh" | Идентификатор заказа со стороны партнера
original_provider_order_id | String | "AB123-45" | Идентификатор заказа в системе партнера
expires_at | DateTime | "2017-09-01T15:00:00+03:00" | Таймлимит для заказа
meta | Object | См. [схему сущности meta](#meta-entity-v2) | Дополнительная информация

> Пожалуйста, обратите внимание, что параметры с типом DateTime следует передавать в формате ISO8601 с часовым поясом. Рекомендуемый формат даты: `"%Y-%m-%dT%H:%M:%S%:z"`

<h3 id='credit-agreement-entity-v2'> Кредитный договор </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID договора
body_amount | Decimal | 10000 | Сумма тела займа
interest_amount | Decimal | 2000 | Сумма процентов займа
body_remainder_amount | Decimal | 9000 | Текущая задолженность по телу
interest_remainder_amount | Decimal | 1000 | Текущая задолженность по процентам
state | String | 'open' | Статус договора
credit_application_id | Integer | ID кредитной заявки


<h3 id='credit-signature-entity-v2'> Электронная подпись </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID подписи
code | String | '0312' | Код
active | Boolean | true | Статус активности
expires_at | Datetime | "2019-07-01T13:00:00+03:00" | Срок активности

<h3 id='local-passport-entity-v2'> Общегражданский паспорт </h3>

Параметр | Тип | Пример | Описание
--------- | --------- | --------- | ---------
birth_date | Date | "1987-07-18" | Дата рождения
issue_date | Date | "2001-01-01" | Дата выдачи паспорта
number | String | "4444333222" | Серия и номер паспорта
last_name | String | "Иванов" | Фамилия
first_name | String | "Иван" | Имя
middle_name | String | "Александрович" | Отчество
sex | String: "female", "male" | "female" | Пол по паспорту
issuing_authority | String | "ОВД Района Калитники" | Кем выдан
authority_code | String | "663402" | Код подразделения

<h3 id='address-entity-v2'> Адрес регистрации </h3>

Параметр | Тип | Пример | Описание
--------- | --------- | --------- | ---------
city | String | "Москва" | Город
street | String | "ул. Китобойцев" | Улица
building | String | "3" | Дом
housing | String | "1" | Строение
apartment | String | "616" | Квартира
postcode | String | "111123" | Почтовый индекс


<h3 id='international-passport-entity-v2'> Заграничный паспорт </h3>

Параметр | Тип | Пример | Описание
--------- | --------- | --------- | ---------
country_code | String | "RU" | Код страны
birth_date | Date | "1987-07-18" | Дата рождения
expiration_date | Date | "2020-01-01" | Срок истечения
number | String | "723902034" | Номер
first_name | String | "Ivanov" | Имя
last_name | String | "Ivan" | Фамилия

<h3 id='document-photo-entity-v2'> Фотография документа </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID фотографии
doument_id | Integer | 1 | ID документа
document_type | String | 'local_passport' | Тип документа
image | File | @path/to/file/local_passport.jpg | Файл фотографии


<h3 id='social-profile-entity-v2'> Профиль социальной сети </h3>

** В разработке **

<h3 id='initial-payment-entity-v2'> Первоначальный платеж </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID платежа
amount | Decimal | 1000 | Сумма

<h3 id='credit-card-entity-v2'> Банковская карта </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID карты
number | String | '411111******1112' | Маскированный номер карты
expiration_date | String | '12/23' | Срок действия карты

<h3 id='decision-entity-v2'> Решение по заявке </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID решения
approved | Boolean | false | Решение по заявке

<h3 id='scheduled-payment-entity-v2'> Очередной платеж </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID платежа
amount | Decimal | 1000 | Размер платежа
state | String | 'awaiting_charge' | Статус платежа
payment_date | Date | '2019-06-06' | Запланированная дата платежа
paid_at | Datetime | '2019-06-08 15:00:00 +03:00' | Фактическая дата оплаты платежа
penalty | Boolean | true | Флаг выхода платежа на просрочку
penalty_amount | Decimal | 50 | Размер штрафа


<h3 id='payment-attempt-entity-v2'> Попытка списания </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
scheduled_payment_id | Integer | 1 | ID платежа подлежащегь списанию

<h3 id='charged-payment-entity-v2'> Состоявшийся платеж </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID платежа
amount | Decimal | 1000 | Размер платежа
paid_at | Datetime | '2019-06-08 15:00:00 +03:00' | Фактическая дата оплаты платежа

<h3 id='refund-entity-v2'> Возврат </h3>

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID платежа
amount | Decimal | 1000 | Сумма возврата

<h3 id='product-entity-v2'> Товар </h3>
Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID продукта
product_type | String | 'airline_ticket'
product_items | Array | См раздел "Товар" | Список товаров

<h3 id='additional-entities-v2'>Дополнительные сущности</h3>

<h4 id='meta-entity-v2'><code>meta</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
client_orders_count | Integer | 14 | Количество успешных заказов клиента | Нет
client_login | Boolean | true/false | Статус клиента (залогинен или нет) | Нет
orders_count | Integer | 3 | Сумма заказов клиента у партнера | Нет
book_code     | String | "Y" | Код бронирования | Нет



Базовый URL: `https://dev.moneywall.io/api/v2/`

<h2 id='auth-v2'> Аутентификация </h2>

  **TBD**

<h2 id='credit-application-v2'> Кредитная заявка </h2>
Создание заявки

**HTTP Запрос**

`POST https://dev.moneywall.io/api/v2/credit_applications`

**Схема запроса**

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
phone | String | "79588249424" | Номер телефона клиента | Да
email | String | "user@gmail.com" | Адрес электронной почты клиента | Да
amount | Decimal | 10000 | Сумма заказа в рублях | Да
success_callback_url | String | "http://yoursite.com/callbacks/success" | Адрес callback-страницы, вызываемой при одобрении заявки | Да
failure_callback_url | String | "http://yoursite.com/callbacks/failure" | Адрес callback-страницы, вызываемой при отклонении заявки | Да
redirect_url | String | "http://yoursite.com/order/AB123-45" | Адрес страницы заказа в системе партнера | Да
order_id | String | "vpih-234lgh" | Идентификатор сессии со стороны партнера (по нему можно будет [получать статус](#payment-state)) | Да
original_provider_order_id | String | "AB123-45" | Идентификатор заказа в системе партнера | Нет
expires_at | DateTime | "2017-07-01T15:00:00+03:00" | Таймлимит для заказа | Нет
product_type | String | "airline_tickets" | Тип товара, под который выдается кредит| Нет
product | Object | См. [схему объекта product](#product-schema)| Детали товара, под который выдается кредит | Да, если передан тип товара
local_passport | Object | См. [схему объекта local_passport](#local-passport-schema) | Общегражданский паспорт клиента | Нет
address | Object | См. [схему объекта address](#address-schema) | Адрес регистрации клиента | Нет
international_passport | Object | См. [схему объекта international_passport](#international-passport-schema) | Загранпаспорт клиента | Нет
meta | Object | См. [схему объекта meta](#meta-schema) | Дополнительная информация | Нет

Получение полной информации по заявке
**HTTP Запрос**
`GET https://dev.moneywall.io/api/v2/credit_applications/:id`

Поле | Тип | Пример | Описание
--------- | --------- | --------- | ---------
id | Integer | 1 | ID заявки
amount | Decimal | 10000 | Сумма займа
remaining_amount | Decimal | 3000 | Оставшаяся сумма по договору к оплате
local_passport | LocalPassport | Объект сущности LocalPassport | Объект общегражданского паспорта
international_passport | InternationalPassport | Объект сущности InternationalPassport | Объект загранпаспорта
address | Address | Объект сущности Address | Объект адреса
document_photos | Array | Массив сущностей DocumentPhoto | Массив фотографий документов
product | Product | Объект сущности Product | Объект товара
payment_graph | Array | Массив объектов Payment | График платажей



<h3 id='credit-application-v2-signature'> Электронная подпись </h3>
  `POST#credit_applications/:id/verification_code`

  Получение кода подписи по заданному каналу

#### Код подписи
  Метод выполняется в контексте [Электронной подписи](#credit-application-v2-signature)

  `POST#credit_applications/:id/verification_code/sign`

  Ввод полученного кода


### Документы
  `POST#credit_applications/:id/document`

  Передача данных документов с типом

### Фотографии документов
  `POST#credit_applications/:id/document_photo`

  Передача фото документов

### Соцсети
  `POST#credit_applications/:id/social_profile`

  ???????

### Первоначальный платеж
  `POST#credit_applications/:id/initial_payment`

  Попытка списания платежа с переданной карты

### Банковская карта
  `POST#credit_applications/:id/credit_payment`

  Создание банковской карты

### Решение по заявке
  `GET#credit_applications/:id/decision`

  Получение решения по заявке

### Степень заполненности заявки
  `GET#credit_applications/:id/verify`

  Получение списка недостающих данных

<h2 id='credit-agreement-v2'> Договор </h2>
  `GET#credit_agreements/:id`

  Получение договора со всеми вложенными сущностями

### График платежей
  `GET#credit_agreements/:id/payment_schedule`

  Получение актуального списка платежей

### Очередной платеж
  `GET#credit_agreements/:id/credit_payments/:id`

  Информация о платеже

  `POST#credit_agreements/:id/credit_payments/:id/charge`

  Попытка оплаты платежа

### Попытка списания
  `POST#credit_agreements/:id/payments/pay`

  Попытка списания платежа (id, сумма, мета)

### Состоявшийся платеж
  `POST#credit_agreements/:id/payments/paid`

  Уведомление об успешно списанном платеже с информацией
            (тип платежа, id, сумма, мета)

### Возвраты
  `POST#credit_agreements/:id/refund`

  Запрос на проведение возврата

  `GET#credit_agreements/:id/refunds`

  Получение списка возвратов

#### Возврат
  `GET#credit_agreements/:id/refunds/:id`

  Получение статуса возврата

### Банковская карта
  `POST#credit_agreements/:id/credit_card`

  Поздание банковской карты (у нас привязывается к клиенту)

### Ввод договора в действие (?)
  `POST#credit_agreements/:id/service_provided`

  Передача данных о предоставленной услуге

<h2 id='cash-flow-v2'> Движения Денежных Средств </h2>
  `GET#cash_flow`

  Получение списка платежей и возвратов по фильтру

## Webhooks
### Принятие решения по заявке
### Изменение графика платежей
