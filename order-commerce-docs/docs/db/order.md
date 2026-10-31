# Database — Order-Commerce Service · Order

> Nguồn: `docs/lld/order.md` · `docs/api/order.md` · `docs/test/order.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Baseline: MySQL 8.4/InnoDB `orderdb` + Redis cart

Phần **Order** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/db/order-commerce.md`.

## 1. Quy ước chung

Theo `docs/db/order-commerce.md` §1 (UUIDv7 `BINARY(16)`, tiền `BIGINT` VND, `DATETIME(6)` UTC, scope theo auth context, không FK sang DB service khác).

## 2. Quan hệ

Xem ER ở `docs/db/order-commerce.md` §2.

## 3. Chi tiết bảng

### 3.1 `orders`

| Cột | Type | Null | Ràng buộc |
|---|---|---:|---|
| `id` | BINARY(16) | N | PK UUIDv7. |
| `order_number` | VARCHAR(32) | N | Unique, display. |
| `checkout_group_id` | BINARY(16) | N | Liên kết orders split multi-shop. |
| `buyer_user_id` | BINARY(16) | N | Auth reference. |
| `shop_id` | BINARY(16) | N | Shop snapshot reference. |
| `status` | VARCHAR(32) | N | OrderStatus enum. |
| `subtotal/discount/shipping_fee/tax/grand_total` | BIGINT | N | `≥0`, tổng phải reconcile. |
| `payment_method/status` | VARCHAR(32) | N | VNPAY/COD + projection. |
| `address_snapshot` | JSON | N | Snapshot cần thiết; không log raw. |
| `reservation_id` | BINARY(16) | Y | Inventory reference. |
| `created_at/placed_at/updated_at` | DATETIME(6) | N | UTC. |
| `version` | BIGINT | N | Optimistic mutation. |

### 3.2 `order_items`

`id`, `order_id`, `product_id`, `sku_id`, `shop_id`, `title_snapshot`, `seller_sku_snapshot`, `unit_price`, `quantity`, `line_total`, `tax_rate_bps`, `tax_amount`, `discount_allocated`, `created_at`. `unit_price` integer VND; quantity 1–999; line_total phải bằng unit_price × quantity trong range.

| Field thuế/giảm giá | Kiểu | Ý nghĩa |
|---|---|---|
| `tax_rate_bps` | INT | Thuế suất VAT snapshot từ category của product tại lúc checkout (10% = `1000`). Bất biến sau khi đặt — đổi thuế suất danh mục không hồi tố. Xem `docs/lld/checkout.md` §6.2. |
| `tax_amount` | BIGINT | Phần VAT **đã nằm trong** `line_total` (giá gồm VAT), tách ra để ghi hoá đơn: `line_net × rate_bps / (10000 + rate_bps)`. Không cộng thêm vào `grand_total`. |
| `discount_allocated` | BIGINT | Tổng giảm giá (shop + platform) phân bổ về dòng này. Cần để tính `tax_amount` đúng và để hoàn tiền theo dòng khi huỷ một phần. |

### 3.3 `invoices`

`id`, `order_id` unique, `invoice_number` unique, `status` (`PENDING/ISSUED/VOIDED`), `total_amount`, `tax_breakdown` JSON, `pdf_url` nullable, `issued_at`, timestamps. Không lưu payment credential.

## 4. Index

| Bảng | Index |
|---|---|
| `orders` | unique `order_number`; `(buyer_user_id, created_at)`; `(shop_id,status,created_at)`; `checkout_group_id`; `(status,updated_at)` |
| `order_items` | `(order_id)`; `(sku_id,created_at)`; `(product_id,created_at)` |
| `invoices` | unique `order_id`; unique `invoice_number`; `(status,issued_at)` |

## 5. Enum và rules

Order transition, total reconciliation (`orders.discount = Σ order_vouchers.discount_amount`) và one buyer/one shop scope enforce ở service transaction; schema enum chỉ là lớp đầu. Enum ở `docs/lld/order.md` §5.

## 6. Migration và seed

Thứ tự migration và seed chung ở `docs/db/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/db/order-commerce.md` §7.
