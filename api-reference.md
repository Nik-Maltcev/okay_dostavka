---
layout: default
title: OK Delivery API — Reference (v1)
nav_order: 2
---

# OK Delivery API — Reference

**Версия:** v1  
**Формат:** JSON over HTTPS  
**Базовый URL**  
- **Production:** `https://api.ok-delivery.ru/v1`  
- **Sandbox:** `https://sandbox.ok-delivery.ru/v1`

## Аутентификация

Передавайте **Bearer Token** в заголовке:

```http
Authorization: Bearer <access_token>
Content-Type: application/json
Idempotency-Key: <uuid>   # для POST/PUT/PATCH (рекомендуется)
```

Токены выдаёт B2B‑кабинет партнёра или внутренний Auth‑service (OAuth2 Client Credentials).  
Невалидный/просроченный токен → `401 Unauthorized`.

## Версионирование
- По пути (`/v1/...`). Минорные несовместимости — через заголовок `X-API-Warning`.

## Ограничения и пагинация
- **Rate limit:** 60 rps по ключу/организации. Превышение → `429 Too Many Requests`.
- **Пагинация:** `?limit=50&cursor=<opaque-token>` — ответы содержат `next_cursor` (или `null`).

## Стандарт ошибок
```json
{
  "error": "validation_error",
  "message": "Field 'phone' is invalid",
  "details": [{"field":"phone","code":"format"}],
  "request_id": "req_97c6d..."
}
```
Частые коды: `400 validation_error`, `401 unauthorized`, `403 forbidden`, `404 not_found`,  
`409 conflict`, `422 unprocessable_entity`, `429 rate_limited`, `500 internal`.

---

## Словари объектов

### Money
```json
{ "currency":"RUB", "amount": 1299.90 }
```

### Address
```json
{
  "city":"Санкт-Петербург",
  "street":"Лиговский пр.",
  "house":"30",
  "apt":"12",
  "entrance":"3",
  "floor":"5",
  "comment":"Домофон 1234"
}
```

### Customer
```json
{ "id":"cus_71ab", "name":"Иван Иванов", "phone":"+79991234567", "email":"ivan@example.com" }
```

---

## Каталог

### GET /products
Возвращает список товаров с фильтрами.

**Query**
- `q` — строка поиска  
- `category_id` — фильтр категории  
- `limit`, `cursor`

**Response 200**
```json
{
  "items": [
    {
      "id":"sku_apple_01",
      "title":"Яблоки сезонные",
      "unit":"kg",
      "price": {"currency":"RUB","amount":129.90},
      "promo_price": null,
      "in_stock": true
    }
  ],
  "next_cursor": null
}
```

### GET /products/{id}
```json
{
  "id":"sku_apple_01",
  "title":"Яблоки сезонные",
  "description":"Сладкие, фермерские",
  "nutrition": {"cal": 52},
  "price":{"currency":"RUB","amount":129.90},
  "images":["https://cdn.ok.../apple.jpg"],
  "attributes":{"country":"RU","brand":null}
}
```

### GET /inventory
Проверка доступности по локации и времени.

**Query**
- `lat`, `lon` **или** `store_id`  
- `product_ids=sku_1,sku_2`

**Response 200**
```json
{
  "availability": [
    {"product_id":"sku_apple_01","available":true,"qty":120}
  ]
}
```

---

## Слоты доставки

### GET /delivery/slots
Подбор временных интервалов с учётом адреса и корзины.

**Query**
- `lat`,`lon` или `address_id`
- `cart_id`

**Response 200**
```json
{
  "slots":[
    {"slot_id":"2025-10-16T10:00/12:00","price":{"currency":"RUB","amount":149.00},"eta_hours":2}
  ]
}
```

---

## Корзина

### POST /carts
Создаёт корзину.

**Body**
```json
{ "customer_id":"cus_71ab", "store_id":"store_spb_01" }
```

**Response 201**
```json
{ "id":"cart_9f2a", "customer_id":"cus_71ab", "items":[], "total":{"currency":"RUB","amount":0} }
```

### POST /carts/{cart_id}/items
Добавляет/обновляет позицию.

**Body**
```json
{ "product_id":"sku_apple_01", "qty": 2 }
```

**Response 200**
```json
{
  "id":"cart_9f2a",
  "items":[
    {"product_id":"sku_apple_01","qty":2,"price":{"currency":"RUB","amount":129.90},"line_total":{"currency":"RUB","amount":259.80}}
  ],
  "subtotal":{"currency":"RUB","amount":259.80},
  "delivery_fee":{"currency":"RUB","amount":149.00},
  "total":{"currency":"RUB","amount":408.80}
}
```

### DELETE /carts/{cart_id}/items/{product_id}
Удаляет позицию. `204 No Content`.

---

## Заказы

### POST /orders
Создание заказа из корзины.

**Body**
```json
{
  "cart_id":"cart_9f2a",
  "delivery": {
    "address": { "city":"СПб", "street":"Лиговский пр.", "house":"30" },
    "slot_id":"2025-10-16T10:00/12:00"
  },
  "payment": { "method":"card_online", "save_card":false },
  "customer": { "name":"Иван", "phone":"+79991234567" }
}
```

**Response 201**
```json
{
  "id":"ord_53c1",
  "status":"created",
  "total":{"currency":"RUB","amount":408.80},
  "payment":{"status":"pending","payment_id":"pay_77ab"},
  "tracking_url":"https://ok-delivery.ru/track/ord_53c1"
}
```
Статусы: `created` → `confirmed` → `assembling` → `on_the_way` → `delivered` / `canceled`.

### GET /orders/{order_id}
```json
{
  "id":"ord_53c1",
  "status":"assembling",
  "items":[{"product_id":"sku_apple_01","qty":2}],
  "delivery":{"slot_id":"2025-10-16T10:00/12:00","courier":{"name":"Алексей","phone":"+7..." }},
  "history":[{"ts":"2025-10-15T08:01Z","status":"confirmed"}]
}
```

### PATCH /orders/{order_id}/status
(для партнёров, если вебхуки отключены)

**Body**
```json
{ "status":"canceled", "reason":"customer_request" }
```

**Response 200**
```json
{ "id":"ord_53c1","status":"canceled" }
```

---

## Оплата

### POST /payments
Инициирует платёж.

**Body**
```json
{ "order_id":"ord_53c1", "method":"card_online", "return_url":"https://shop.example/success" }
```

**Response 201**
```json
{ "payment_id":"pay_77ab", "status":"pending", "redirect_url":"https://pg.example/3ds" }
```

### GET /payments/{payment_id}
```json
{ "payment_id":"pay_77ab", "status":"succeeded", "amount":{"currency":"RUB","amount":408.80} }
```

### POST /refunds
Возврат средств.

**Body**
```json
{ "payment_id":"pay_77ab", "amount":{"currency":"RUB","amount":129.90}, "reason":"item_unavailable" }
```

**Response 201**
```json
{ "refund_id":"ref_11dd", "status":"processing" }
```

---

## Пользователи

### POST /customers
```json
{ "name":"Иван Иванов","phone":"+79991234567","email":"ivan@example.com" }
```
**201** → `{"id":"cus_71ab", ...}`

### GET /customers/{id}
Профиль клиента + адреса, соглашия.

---

## Вебхуки

**Подписка** в B2B‑кабинете. Подтверждение эндпоинта — `GET` с `challenge`.  
Подписанные уведомления: заголовок `X-Signature: sha256=<hex>`.

События:
- `order.created`
- `order.status_changed`
- `payment.succeeded`
- `refund.created`

**Payload**
```json
{
  "event":"order.status_changed",
  "ts":"2025-10-15T09:10:31Z",
  "data":{"order_id":"ord_53c1","old":"confirmed","new":"assembling"}
}
```
**Ретраи**: экспоненциальная задержка до 24 ч. Успех — любые `2xx`.

---

## Идемпотентность
Передавайте уникальный `Idempotency-Key` для безопасных повторов `POST/PUT/PATCH`.

---

## Чек‑лист интеграции
1. Создать корзину и наполнить товарами.  
2. Получить слоты доставки под адрес.  
3. Создать заказ с выбранным слотом.  
4. Инициировать/подтвердить оплату.  
5. Слушать вебхуки по статусам.  
6. Обрабатывать возвраты при необходимости.
