# Database — Checkout Service · Voucher

> Nguồn: `docs/lld/voucher.md` · `docs/api/voucher.md` · `docs/test/voucher.md` · Code: chưa có repo (`ecommerce-taca/checkout-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế.
> Baseline: MySQL 8.4/InnoDB · database riêng `checkoutdb`

Phần **Voucher** của Checkout Service. Quy ước chung, ER, migration và seed ở `docs/db/checkout.md`.

## 1. Quy ước chung

Theo `docs/db/checkout.md` §1.

## 2. Quan hệ

Xem ER ở `docs/db/checkout.md` §2.

## 3. Chi tiết bảng

### 3.1 `vouchers`

| Field | Kiểu | N/Y | Ràng buộc |
|---|---|:--:|---|
| `id` | BINARY(16) | N | PK UUIDv7. |
| `code` | VARCHAR(32) | N | Normalize uppercase/trim. |
| `scope` | VARCHAR(16) | N | `PLATFORM` \| `SHOP` \| `FREESHIP`; vừa là phạm vi vừa là **slot cộng dồn**. `SHOP` bắt buộc `shop_id`; `PLATFORM`/`FREESHIP` bắt buộc `shop_id IS NULL`. |
| `shop_id` | BINARY(16) | Y | Shop reference. |
| `title` | VARCHAR(128) | Y | Hiển thị. |
| `discount_type` | VARCHAR(16) | N | `PERCENT` \| `FIXED` \| `FREESHIP`. `scope=FREESHIP` chỉ nhận `PERCENT`/`FREESHIP`; scope khác không dùng `FREESHIP`. |
| `value` | BIGINT | N | `PERCENT` 1–100; còn lại VND. |
| `max_discount_amount` | BIGINT | Y | Trần cho `PERCENT`; `NULL` = không trần ("Giảm 8% tối đa 1 triệu"). |
| `min_order` | BIGINT | N | `≥0`. |
| `usage_limit` | INT | Y | `NULL` = không giới hạn. |
| `used_count` | INT | N | Chỉ đổi cùng transaction với redemption. |
| `starts_at`, `expires_at` | DATETIME(6) | N | UTC. |
| `status` | VARCHAR(16) | N | `VoucherStatus`. |
| `created_by` | BINARY(16) | N | Seller/admin user. |
| `version` | BIGINT | N | Optimistic lock cho PUT/DELETE. |
| `created_at/updated_at` | DATETIME(6) | N | UTC. |

### 3.2 `voucher_redemptions`

| Field | Kiểu | N/Y | Ràng buộc |
|---|---|:--:|---|
| `id` | BINARY(16) | N | PK. |
| `voucher_id` | BINARY(16) | N | FK → `vouchers`. |
| `checkout_group_id` | BINARY(16) | N | FK → `checkout_groups`. |
| `order_id` | BINARY(16) | Y | Có khi `REDEEMED`; mã `SHOP` = đơn của shop, `PLATFORM`/`FREESHIP` = đơn đầu tiên của nhóm (tham chiếu). |
| `user_id` | BINARY(16) | N | Buyer. |
| `scope` | VARCHAR(16) | N | Copy từ voucher. |
| `code_snapshot` | VARCHAR(32) | N | — |
| `discount_amount` | BIGINT | N | Tổng giảm của mã này trên cả nhóm (sau recalculate là số mới). |
| `status` | VARCHAR(16) | N | `HELD`/`REDEEMED`/`RELEASED`. |
| `redeemed_at`, `released_at` | DATETIME(6) | Y | UTC. |

Phần giảm **đã phân bổ về từng đơn con** lưu ở `order_vouchers` của `order-service` (`order-docs/docs/db/order.md` §3.3) và ở `checkout_group_orders` (`docs/db/checkout.md` §3.2).

### 3.3 `voucher_adjustments`

| Field | Kiểu | Ràng buộc |
|---|---|---|
| `id`, `checkout_group_id`, `order_id`, `voucher_id?` | BINARY(16) | Audit một lần tính lại cho một đơn còn lại. |
| `type` | VARCHAR(32) | `AdjustmentType`. |
| `grand_total_before`, `grand_total_after`, `absorbed_amount` | BIGINT | `absorbed_amount > 0` chỉ khi `VOUCHER_ADJUSTMENT_ABSORBED`. |
| `idempotency_key` | VARCHAR(128) | `cancel-{order_id}` của `order-service`. |
| `created_at` | DATETIME(6) | UTC. |

> **Vì sao không còn cột `voucher_id` trên đơn:** mô hình 3 slot cho phép một đơn mang nhiều voucher, mỗi voucher góp một số tiền khác nhau. `order-service` lưu từng dòng ở `order_vouchers`; `orders.discount` luôn bằng tổng `discount_amount` các dòng `PLATFORM`+`SHOP`.

## 4. Index

| Bảng | Index |
|---|---|
| `vouchers` | unique `(scope, shop_id, code)` (normalized; MySQL coi `NULL` khác nhau nên dùng cột sinh `shop_key = COALESCE(shop_id, 0x00)`); `(scope,status,starts_at,expires_at)`; `(shop_id,status,starts_at,expires_at)`; `(status,expires_at)` |
| `voucher_redemptions` | unique `(voucher_id, checkout_group_id)`; `(user_id, voucher_id, status)` cho giới hạn mỗi buyer; `(user_id, redeemed_at)`; `(order_id)` |
| `voucher_adjustments` | `(checkout_group_id, created_at)`; unique `(idempotency_key, order_id)` |

## 5. Enum và rules

Enum ở `docs/lld/voucher.md` §5. Usage increment và redemption idempotent, enforce ở service transaction.

## 6. Migration và seed

Thứ tự migration và seed ở `docs/db/checkout.md` §6.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Bảng `vouchers` **không có** cột audience/segment trong v1. | Bật targeting cần bảng `voucher_audiences`. | Product owner |
| 2 | Trước khi tách, doc ghi vừa "unique `(voucher_id,user_id)`" vừa "unique `(voucher_id,user_id,order_id)`". Nay chốt: unique `(voucher_id, checkout_group_id)` + giới hạn mỗi buyer kiểm ở service (`VOUCHER_PER_USER_LIMIT`). | Nếu muốn chặn cứng ở DB, cần thêm cột sinh. | Architecture |
