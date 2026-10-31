# Database — Order-Commerce Service · Cart

> Nguồn: `docs/lld/cart.md` · `docs/api/cart.md` · `docs/test/cart.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Baseline: MySQL 8.4/InnoDB `orderdb` + Redis cart

Phần **Cart** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/db/order-commerce.md`.

## 1. Quy ước chung

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

`CartStatus` ở `docs/lld/cart.md` §5. Redis không có migration SQL.

## 6. Migration và seed

Thứ tự migration và seed chung ở `docs/db/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/db/order-commerce.md` §7.
