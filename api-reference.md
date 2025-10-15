---
layout: default
title: OK Delivery API — Reference (v1)
nav_order: 2
---
# OK Delivery API — Reference
**Версия:** v1 • JSON over HTTPS
**Production:** `https://api.ok-delivery.ru/v1` • **Sandbox:** `https://sandbox.ok-delivery.ru/v1`
## Auth
`Authorization: Bearer <token>` • `Idempotency-Key: <uuid>`
## Пример ошибок
```json
{"error":"validation_error","message":"Field 'phone' is invalid"}
```
## GET /products
```json
{"items":[{"id":"sku_apple_01","title":"Яблоки"}]}
```
