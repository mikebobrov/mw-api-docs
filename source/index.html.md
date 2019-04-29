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
  4. Перенаправление пользователя на адресу из `redirect_url`
  5. Ожидание решения по заявке
  6. Вызов `success_callback_url` или `failure_callback_url` при одобрении или отклонении заявки соответственно

<h3 id='formula'>Формула рассчета цены</h3>
Для расчёта и отображения платежей и полной стоимости заказа на стороне партнёра, следует использовать упрощенную формулу:

`ПЛАТЕЖ  = (СУММА ЗАКАЗА * K) / 3,`

где `K = 1.10` для случаев, когда вылет наступает после окончания договора займа (больше, чем 64 дня от даты открытия договора),
<br>и `K = 1.20` для случаев, когда вылет наступает ранее даты окончания договора займа.

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
  -d '{ "phone": "79031232299", "email": "test@test.com", "amount": 10000, "calculated_credit_cost": 12000, "success_callback_url": "http://yoursite.com/callbacks/success", "failure_callback_url": "http://yoursite.com/callbacks/failure", "redirect_url": "http://yoursite.com/order/234", "order_id": "partner_order_id", "web_mode": "standalone", "original_provider_order_id": "original_provider_order_id", "expires_at": "2017-07-01T00:00:00", "product_type": "airline_tickets" }'

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
order_id | String | "vpih-234lgh" | Идентификатор сессии со стороны партнера (по нему можно будет [получать статус](#payment-state)) | Да
web_mode | String: "standalone", "iframe" | "standalone" | Режим, в котором будет открыт диалог заполнения кредитной заявки (по умолчанию: standalone)| Нет
original_provider_order_id | String | "AB123-45" | Идентификатор заказа в системе партнера | Нет
expires_at | DateTime | "2017-07-01T15:00:00+03:00" | Таймлимит для заказа | Нет
product_type | String | "airline_tickets" | Тип товара, под который выдается кредит| Нет
product | Object | См. [схему объекта product](#product-schema)| Детали товара, под который выдается кредит | Да, если передан тип товара
local_passport | Object | См. [схему объекта local_passport](#local-passport-schema) | Общегражданский паспорт клиента | Нет
address | Object | См. [схему объекта address](#address-schema) | Адрес регистрации клиента | Нет
international_passport | Object | См. [схему объекта international_passport](#international-passport-schema) | Загранпаспорт клиента | Нет
meta | Object | См. [схему объекта meta](#meta-schema) | Дополнительная информация | Нет

> Пожалуйста, обратите внимание, что параметры с типом DateTime следует передавать в формате ISO8601 с часовым поясом. Рекомендуемый формат даты: `"%Y-%m-%dT%H:%M:%S%:z"`

<h4 id='product-schema'>Схема объекта <code>product</code> при <code>product_type == 'airline_tickets'</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
cabin_type          | String | "economy" ("business", "premium") | Класс обслуживания | Да
validating_airline  | String | "SU" | Валидирующая авиакомпания | Да
segments            | Array | См. [схему объекта segment](#segment-schema) | Сегменты | Да
passengers          | Array | См. [схему объекта passenger](#passenger) | Пассажиры | Да

<h4 id='passenger-schema'>Схема объекта <code>passenger</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
first_name      | String | "Ivan" | Имя | Да
middle_name     | String | "Aleksandrovich" | Отчество | Нет
last_name       | String | "Ivanon" | Фамилия | Да
document_type   | String | "international_passport" ("local_passport", "birth_certificate") | Тип документа | Да
document_number | String | "349287349" | Номер документа | Да
birth_date      | String | "1987-07-18" | Дата рождения | Да

<h4 id='segment-schema'>Схема объекта <code>segment</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
flights   | Array | См. [схему объекта flight](#flight-schema) | Перелеты | Да

<h4 id='flight-schema'>Схема объекта <code>flight</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
aircraft      | String | "737" | Модель самолета | Да
departure_dt  | DateTime | "2016-12-10T08:00:00+03:00" | Время вылета  | Да
arrival_dt    | DateTime | "2018-12-10T10:00:00+03:00" | Время прилета | Да
book_code     | String | "Y" | Код бронирования | Да
orig          | String | "SVO" | Аэропорт вылета | Да
dest          | String | "VIE" | Аэропорт прилета | Да
flight_number | String | "606" | Номер рейса | Да
marketing_ac  | String | "SU" | Маркетинговый перевозчик | Да
operating_ac  | String | "SU" | Оперирующий перевозчик | Да


<h4 id='local-passport-schema'>Схема объекта общегражданского паспорта <code>local_passport</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
birth_date | Date | "1987-07-18" | Дата рождения | Да
issue_date | Date | "2001-01-01" | Дата выдачи паспорта | Да
number | String | "4444333222" | Серия и номер паспорта | Да
last_name | String | "Иванов" | Фамилия | Да
first_name | String | "Иван" | Имя | Да
middle_name | String | "Александрович" | Отчество | Да
sex | String: "female", "male" | "female" | Пол по паспорту | Да
issuing_authority | String | "ОВД Района Калитники" | Кем выдан | Да
authority_code | String | "663402" | Код подразделения | Да

<h4 id='address-schema'>Схема объекта адреса <code>address</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
city | String | "Москва" | Город | Да
street | String | "ул. Китобойцев" | Улица | Да
building | String | "3" | Дом | Да
housing | String | "1" | Строение | Нет
apartment | String | "616" | Квартира | Нет
postcode | String | "111123" | Почтовый индекс | Да


<h4 id='international-passport-schema'>Схема объекта загранпаспорта <code>international_passport</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
country_code | String | "RU" | Код страны | Да
birth_date | Date | "1987-07-18" | Дата рождения | Да
expiration_date | Date | "2020-01-01" | Срок истечения | Да
number | String | "723902034" | Номер | Да
first_name | String | "Ivanov" | Имя | Да
last_name | String | "Ivan" | Фамилия | Да

<h4 id='meta-schema'>Схема объекта <code>meta</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
client_orders_count | Integer | 14 | Количество успешных заказов клиента | Нет
client_login | Boolean | true/false | Статус клиента (залогинен или нет) | Нет
orders_count | Integer | 3 | Сумма заказов клиента у партнера | Нет
book_code     | String | "Y" | Код бронирования | Нет
original_provider_order_id | String | "AB123-45" | Идентификатор заказа в системе партнера | Нет
<h3 id='product'>Добавление выписанных билетов к заказу</a></h3>

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

<h4 id='product'>Схема объекта <code>product</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
product_type | String | "airline-tickets" | Тип билетов | Да
product_items | Array | См. [схему объекта product_items](#product-items) | Список билетов и маршрутных квитанций | Да

<h4 id='product-items'>Схема объекта <code>product_items</code></h4>

Параметр | Тип | Пример | Описание | Обязательно
--------- | --------- | --------- | --------- | ---------
ticket_number | String | "421-288879826" | Номер билета | Да
ticket_receipt | File | @path/to/file/test_pdf.pdf | Файл маршрутной квитанции (pdf, doc, docx). Не больше 8MB | Да

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



