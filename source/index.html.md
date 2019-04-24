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

Базовый URL: `https://dev.moneywall.io/api/v2/`

<h2 id='auth-v2'> Аутентификация </h2>

  **TBD**

<h2 id='credit-application-v2'> Кредитная заявка </h2>
  `POST#credit_applications`

  Создание заявки

  `GET#credit_applications/:id`

  Получение полной информации по заявке

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



