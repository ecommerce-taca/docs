# LLD — Order-Commerce · Cart

> Thuộc service Order-Commerce — tổng quan, quy ước chung và mã lỗi đầy đủ ở `docs/lld/order-commerce.md` · Cập nhật: `2026-10-31`
> API: `docs/api/cart.md` · DB: `docs/db/cart.md` · Test: `docs/test/cart.md`

## 1. Phạm vi

| Mục | Nội dung |
|---|---|
| Trách nhiệm | Giỏ hàng của buyer: get/add/update/delete item. |
| Module | `com.taca.order.cart` — `CartController` (xem `order-commerce.md` §2.1). |
| Lưu trữ | Redis key `cart:{user_id}`, TTL; không phải order source of truth và **không** là reservation kho. |
| Không thuộc phạm vi | Giữ stock (Inventory reserve chỉ xảy ra ở `POST /checkout` — `checkout.md` §2.3), giá chốt (Checkout snapshot lại). |

## 2. Luồng xử lý

1. Authenticated user gọi get/add/update/delete cart.
2. Cart lưu Redis key `cart:{user_id}`, item gồm `product_id`, `sku_id`, `shop_id`, quantity và optional price display cache.
3. Add/update validate quantity >0 và Product/Inventory availability snapshot; không coi cart là reservation.
4. Giá/tiêu đề trong cart chỉ là cache hiển thị; Checkout phải re-read Product và tạo price snapshot.
5. Cart expired/empty không tạo order; mfe-buyer nhận `ORDER_CART_EMPTY`.

## 3. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `CART_TTL` | 30 ngày | Redis; cart không reserve stock. |
| `CART_MAX_ITEMS` | 100 | Item line, không phải quantity. |
| `ITEM_QTY_MIN` | 1 | Integer. |
| `ITEM_QTY_MAX` | 999 | Per SKU/order line. |

## 4. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `CartStatus` | `ACTIVE`, `EXPIRED`, `CHECKED_OUT`. |

## 5. Event / contract

Cart không phát và không lắng nghe event. Add/update gọi Product để validate SKU active và đọc availability snapshot từ Inventory (read-only).

## 6. Mã lỗi

`ORDER_INVALID_INPUT`, `ORDER_UNAUTHENTICATED`, `ORDER_CART_EMPTY`, `ORDER_CART_ITEM_NOT_FOUND`, `ORDER_PRODUCT_UNAVAILABLE` — định nghĩa ở `order-commerce.md` §7.

## 7. Giả định & câu hỏi mở

Xem `order-commerce.md` §8 mục 2 (cart nằm Redis, không durable).
