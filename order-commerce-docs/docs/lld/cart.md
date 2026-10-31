# LLD — Order-Commerce Service · Cart

> Nguồn: `docs/api/cart.md` · `docs/db/cart.md` · `docs/test/cart.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 · Redis Cart · Kafka outbox · REST với Product/Inventory/Payment/Shipment/Auth

Phần **Cart** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/lld/order-commerce.md`.

## 1. Phạm vi

| Mục | Nội dung |
|---|---|
| Trách nhiệm | Giỏ hàng của buyer: get/add/update/delete item. |
| Module | `com.taca.order.cart` — `CartController` (xem `order-commerce.md` §2.1). |
| Lưu trữ | Redis key `cart:{user_id}`, TTL; không phải order source of truth và **không** là reservation kho. |
| Không thuộc phạm vi | Giữ stock (Inventory reserve chỉ xảy ra ở `POST /checkout` — `checkout.md` §3.3), giá chốt (Checkout snapshot lại). |

## 2. Cấu trúc bên trong

Package `com.taca.order.cart` trong cây module ở `order-commerce.md` §2.1. ⏳ Repo chưa có code nên chưa có cây package thật.

## 3. Luồng xử lý

1. Authenticated user gọi get/add/update/delete cart.
2. Cart lưu Redis key `cart:{user_id}`, item gồm `product_id`, `sku_id`, `shop_id`, quantity và optional price display cache.
3. Add/update validate quantity >0 và Product/Inventory availability snapshot; không coi cart là reservation.
4. Giá/tiêu đề trong cart chỉ là cache hiển thị; Checkout phải re-read Product và tạo price snapshot.
5. Cart expired/empty không tạo order; mfe-buyer nhận `ORDER_CART_EMPTY`.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| `CART_TTL` | 30 ngày | ⏳ chưa có | Redis; cart không reserve stock. |
| `CART_MAX_ITEMS` | 100 | ⏳ chưa có | Item line, không phải quantity. |
| `ITEM_QTY_MIN` | 1 | ⏳ chưa có | Integer. |
| `ITEM_QTY_MAX` | 999 | ⏳ chưa có | Per SKU/order line. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `CartStatus` | `ACTIVE`, `EXPIRED`, `CHECKED_OUT`. |

## 6. Event phát ra / lắng nghe

Cart không phát và không lắng nghe event. Add/update gọi Product để validate SKU active và đọc availability snapshot từ Inventory (read-only).

## 7. Mã lỗi

⏳ Chưa có code. `ORDER_INVALID_INPUT`, `ORDER_UNAUTHENTICATED`, `ORDER_CART_EMPTY`, `ORDER_CART_ITEM_NOT_FOUND`, `ORDER_PRODUCT_UNAVAILABLE` — định nghĩa ở `order-commerce.md` §7.

## 8. Giả định & câu hỏi mở

Xem `order-commerce.md` §8 mục 2 (cart nằm Redis, không durable).
