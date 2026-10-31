# Database — Order-Commerce · Cart

> Thuộc service Order-Commerce — quy ước chung, ER, migration và seed ở `docs/db/order-commerce.md` · Cập nhật: `2026-10-31`
> LLD: `docs/lld/cart.md` · API: `docs/api/cart.md`

## 1. Quy ước

Theo `docs/db/order-commerce.md` §1 (UUIDv7 `BINARY(16)`, tiền `BIGINT` VND, `DATETIME(6)` UTC, scope theo auth context, không FK sang DB service khác).

## 2. Quan hệ

Xem ER ở `docs/db/order-commerce.md` §2.

## 3. Chi tiết bảng

### 3.1 Redis cart

```text
cart:{buyer_user_id} → {
  status: ACTIVE,
  items: [{item_id, product_id, sku_id, shop_id, quantity, price_display_snapshot}],
  updated_at: ISO-8601 UTC
}
TTL: 30 days
```

Redis không được dùng để quyết định payment/order state hoặc stock reservation.

## 4. Index

Không có index — truy cập trực tiếp theo key `cart:{buyer_user_id}`.

## 5. Enum và rules

`CartStatus` ở `docs/lld/cart.md` §4. Redis không có migration SQL.

## 6. Migration và seed

Thứ tự migration và seed chung ở `docs/db/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/db/order-commerce.md` §7.
