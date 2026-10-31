# Database — Order-Commerce Service · Voucher

> Nguồn: `docs/lld/voucher.md` · `docs/api/voucher.md` · `docs/test/voucher.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Baseline: MySQL 8.4/InnoDB `orderdb` + Redis cart

Phần **Voucher** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/db/order-commerce.md`.

## 1. Quy ước

Theo `docs/db/order-commerce.md` §1 (UUIDv7 `BINARY(16)`, tiền `BIGINT` VND, `DATETIME(6)` UTC, scope theo auth context, không FK sang DB service khác).

## 2. Quan hệ

Xem ER ở `docs/db/order-commerce.md` §2.

## 3. Chi tiết bảng

### 3.1 `vouchers`, `order_vouchers` và `voucher_redemptions`

| Bảng | Field chính | Rules |
|---|---|---|
| `vouchers` | id, code, **scope**, shop_id nullable, discount_type/value, **max_discount_amount**, min_order, usage_limit, used_count, starts_at, expires_at, status, version | Code normalized unique; `PERCENT` 1–100; `FIXED`/`FREESHIP` VND range. |
| `order_vouchers` | id, order_id, voucher_id, scope, code_snapshot, discount_amount, created_at | **Bảng mới.** Một order có tối đa 3 dòng (1 mỗi `scope`). `discount_amount` là phần **đã phân bổ về đúng order con này**, không phải giá trị gốc của voucher. Unique `(order_id, scope)` — chặn cứng việc cộng dồn 2 mã cùng scope ở tầng DB. |
| `voucher_redemptions` | id, voucher_id, order_id, user_id, redeemed_at, status | Baseline unique `(voucher_id,user_id)`; atomic usage increment; policy có thể nới theo campaign sau khi chốt. |

Chi tiết field mới của `vouchers`:

| Field | Kiểu | N/Y | Ràng buộc |
|---|---|:--:|---|
| `scope` | VARCHAR(16) | N | `PLATFORM` \| `SHOP` \| `FREESHIP` (bổ sung `FREESHIP` so với v cũ). `scope` vừa xác định phạm vi áp dụng vừa là **slot cộng dồn**: tối đa 1 mã mỗi `scope` cho một lần checkout. `SHOP` bắt buộc có `shop_id`; `PLATFORM`/`FREESHIP` bắt buộc `shop_id IS NULL`. |
| `discount_type` | VARCHAR(16) | N | `PERCENT` \| `FIXED` \| `FREESHIP`. `scope=FREESHIP` chỉ nhận `discount_type` `PERCENT` hoặc `FREESHIP`; scope khác không được dùng `FREESHIP`. |
| `max_discount_amount` | BIGINT | Y | **Trần giảm giá**, chỉ có nghĩa với `PERCENT`. `NULL` = không trần. Đây là thứ hiện thực hoá "Giảm 8% tối đa 1 triệu" trên Penpot — thiếu cột này thì voucher 8% áp lên đơn 100 triệu sẽ giảm 8 triệu. |

> **Vì sao cần `order_vouchers` riêng thay vì cột `voucher_id` trên `orders`:** mô hình 3 slot cho phép một order mang nhiều voucher, và mỗi voucher góp một `discount_amount` khác nhau vào cùng order. Nhét vào một cột thì không ghi được hoá đơn chi tiết, không hoàn đúng tiền khi huỷ một phần, và không đối soát được voucher nào tốn ngân sách bao nhiêu. Cột `orders.discount` vẫn giữ, mang tổng — phải luôn bằng `SUM(order_vouchers.discount_amount)` của order đó.

## 4. Index

| Bảng | Index |
|---|---|
| `vouchers` | unique normalized `code`; `(scope,status,starts_at,expires_at)`; `(shop_id,status,starts_at,expires_at)`; `(status,expires_at)` |
| `order_vouchers` | unique `(order_id,scope)`; `(voucher_id,created_at)` cho báo cáo ngân sách voucher |
| `voucher_redemptions` | unique `(voucher_id,user_id,order_id)`; `(order_id)`; `(user_id,redeemed_at)` |

## 5. Enum và rules

Voucher usage increment và redemption idempotent, enforce ở service transaction. Enum ở `docs/lld/voucher.md` §5.

## 6. Migration và seed

Thứ tự migration và seed chung ở `docs/db/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/db/order-commerce.md` §7.
