# Test Plan — Cart Service

> Nguồn: `docs/lld/cart.md` · `docs/api/cart.md` · `docs/db/cart.md` · Code: chưa có repo (`ecommerce-taca/cart-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc, Testcontainers (MongoDB 8)

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers MongoDB. |
| Dependency doubles | Product batch (active/inactive/timeout), Inventory availability. |
| Data | Seed ở `docs/db/cart.md` §6. |
| Actors | Buyer owner, buyer khác, thiếu JWT, internal caller có/không token. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Cart | Add/update/delete, giỏ rỗng, SKU bị tắt | mfe-buyer hiển thị đúng; quantity 1–999; cart không giữ stock. |
| Lưu lâu dài | Đăng xuất/đổi thiết bị, quay lại sau nhiều tuần | Giỏ còn nguyên. |
| Sau đặt hàng | Đặt 2 trong 3 item | Chỉ còn item chưa đặt trong giỏ. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| CART-API-01 | `GET /cart` buyer có/không giỏ | `200`; chưa có giỏ thì `items=[]`, `cart_id=null`. |
| CART-API-02 | `POST /cart/items` valid/invalid/SKU trùng/dòng 101 | `201` cộng dồn quantity; `400`; `409 CART_TOO_MANY_ITEMS`. Không tin `user_id` body. |
| CART-API-03 | `PUT/DELETE /cart/items/{itemId}` item của buyer khác | `404 CART_ITEM_NOT_FOUND`. |
| CART-API-04 | `GET /cart` khi Product timeout | `200` với snapshot cũ, không lỗi. |
| CART-API-05 | `GET /internal/carts/{id}` thiếu/sai token | `401 CART_UNAUTHENTICATED`. |
| CART-API-06 | `GET /internal/carts/{id}?item_ids=` có item không tồn tại | `200`, item đó nằm trong `missing_item_ids`. |
| CART-API-07 | `checked-out` G1 → G2 → gọi lại G1 | Không xoá item nào ngoài `item_ids` của G1; lần gọi lại trả `removed_item_ids=[]`, `200`. |
| CART-API-08 | Không có TTL | Document không bị xoá sau 30+ ngày (kiểm không có TTL index). |
| CART-API-09 | `POST /cart/items` gửi 2 lần cùng `Idempotency-Key` | Quantity chỉ cộng 1 lần, lần 2 trả cùng response; cùng key khác body → `409 CART_IDEMPOTENCY_CONFLICT`. |
| CART-API-10 | `GET /internal/carts/{id}` không có `item_ids` | Trả mọi item trong giỏ; giỏ chưa có → `items=[]`. |

## 4. Unit test

⏳ Repo chưa có test class; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `CartService` | Quantity bounds, gộp SKU trùng, cap 999, tối đa 100 dòng, scope buyer, không có reservation semantics. |
| `InternalTokenFilter` | Constant-time compare, thiếu/sai token. |
| `Observability` | Field log bắt buộc, không log token/nội dung giỏ. |

## 5. Contract/security/resilience

- Contract Cart→Product: batch ≤100, timeout dùng snapshot cũ.
- Contract Checkout→Cart: shape §3.5/§3.6 khớp `checkout-docs/docs/lld/checkout.md` §3.3.
- Security: IDOR item/cart, token nội bộ, Gateway không expose `/internal/**`.
- Load: `GET /cart` lúc sale; update đồng thời cùng giỏ → `CART_VERSION_CONFLICT`, không mất item.

## 6. Tiêu chí pass/phát hành

Mọi case §3 pass; không mất item dưới ghi đồng thời; không log nội dung giỏ; `/internal/**` không truy cập được qua Gateway.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | ID test đổi từ `O-API-01..03` (Order-Commerce) sang `CART-API-*`. | Trace với test cũ qua bảng này. | QA |
