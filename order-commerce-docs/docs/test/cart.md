# Test Plan — Order-Commerce Service · Cart

> Nguồn: `docs/lld/cart.md` · `docs/api/cart.md` · `docs/db/cart.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers (MySQL 8.4, Redis, Kafka)

Phần **Cart** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/test/order-commerce.md`.

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

⏳ Repo chưa có test class nào; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `CartService` | TTL, quantity bounds, user scope, no reservation semantics. |

## 5. Contract/security/resilience

- Security: cart scope theo token, không tin `user_id` trong body; item user khác trả `404` (anti-enumeration).

## 6. Tiêu chí pass/phát hành

Theo `docs/test/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/test/order-commerce.md` §7.
