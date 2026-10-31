# Test Plan — Order-Commerce · Cart

> Thuộc service Order-Commerce — chuẩn bị, contract/security chung và tiêu chí phát hành ở `docs/test/order-commerce.md` · Cập nhật: `2026-10-31`
> LLD: `docs/lld/cart.md` · API: `docs/api/cart.md`

## 1. Chuẩn bị

Theo `docs/test/order-commerce.md` §1.

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Cart | Add/update/delete, empty/expired, SKU disabled | mfe-buyer state đúng; quantity 1–999; cart không giữ stock. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| O-API-01 | `GET /cart` buyer/empty | `200`, own cart; empty items hợp lệ. |
| O-API-02 | `POST /cart/items` valid/invalid/other user | `201` hoặc `400/403`; không trust user_id body. |
| O-API-03 | `PUT/DELETE /cart/items/{itemId}` | Update/delete own; wrong scope not found/forbidden. |


## 4. Unit test

| Module | Cases |
|---|---|
| `CartService` | TTL, quantity bounds, user scope, no reservation semantics. |

## 5. Contract/security/resilience

- Security: cart scope theo token, không tin `user_id` trong body; item user khác trả `404` (anti-enumeration).

## 6. Tiêu chí pass/phát hành

Theo `docs/test/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/test/order-commerce.md` §7.
