# LLD — Order-Commerce · Checkout

> Thuộc service Order-Commerce — tổng quan, quy ước chung và mã lỗi đầy đủ ở `docs/lld/order-commerce.md` · Cập nhật: `2026-10-31`
> API: `docs/api/checkout.md` · Test: `docs/test/checkout.md` · Checkout không có bảng riêng — ghi vào `orders`/`order_items`/`order_vouchers`/`invoices` (`docs/db/order.md`, `docs/db/voucher.md`)

## 1. Phạm vi

| Mục | Nội dung |
|---|---|
| Trách nhiệm | Shipping fee preview, checkout preview, tính tiền, place checkout (reserve kho, split order theo shop, snapshot giá/địa chỉ/thuế, redeem voucher, tạo invoice pending). |
| Module | `com.taca.order.checkout` — `CheckoutController`; gọi `integrations/` (Product, Inventory, Payment, Shipment, Auth). |
| Ràng buộc | Place idempotency; reserve Inventory trước khi commit order; không distributed transaction — dùng saga compensation. |

## 2. Luồng xử lý

### 2.1 Checkout preview

```text
Cart → load selected items → Product validate active SKU/price
     → Inventory availability check (read-only)
     → validate voucher/usage/time/scope
     → calculate subtotal/discount/tax/shipping/total
     → return preview with warnings; không reserve, không charge
```

Preview không đảm bảo giá/stock đến lúc place; response ghi `expires_at` ngắn và Checkout place phải tính lại.

### 2.2 Công thức tính tiền *(nguồn sự thật — mọi nơi khác tham chiếu về đây)*

Đây là phép tính quan trọng nhất của hệ thống. Preview, place checkout, invoice và hiển thị FE **phải dùng chung đúng một cài đặt**; lệch một đồng là lệch đối soát.

**Nguyên tắc bất di bất dịch:**

- Mọi số tiền là **số nguyên VND**. Cấm float/double ở mọi bước trung gian.
- Giá sản phẩm **đã bao gồm VAT** (chuẩn bán lẻ VN). Thuế được **tách ra để ghi hoá đơn**, không cộng thêm vào `grand_total`.
- Thứ tự áp dụng là cố định: shop voucher → platform voucher → freeship voucher → tách thuế. Quy tắc slot voucher ở `voucher.md` §2.2.

#### Bước 0 — Gom theo shop

Với mỗi shop `s` trong giỏ:

```
subtotal_s  = Σ (unit_price × quantity) của các item thuộc shop s
shipping_s  = phí ship của shop s, lấy từ Shipment quote
```

#### Bước 1 — Voucher SHOP (mỗi shop tối đa 1 mã)

Chỉ áp cho đúng shop sở hữu mã. Điều kiện: `subtotal_s >= voucher.min_order`.

```
PERCENT:  shop_discount_s = min( floor(subtotal_s × value / 100), max_discount_amount )
FIXED:    shop_discount_s = min( value, subtotal_s )
```

`max_discount_amount` là `null` thì không có trần. `shop_discount_s` không bao giờ vượt `subtotal_s`.

#### Bước 2 — Voucher PLATFORM (toàn giỏ, tối đa 1 mã)

Áp trên phần **còn lại sau khi đã trừ shop voucher**:

```
base_platform = Σ (subtotal_s − shop_discount_s)
```

Điều kiện: `base_platform >= voucher.min_order`.

```
PERCENT:  platform_discount = min( floor(base_platform × value / 100), max_discount_amount )
FIXED:    platform_discount = min( value, base_platform )
```

Vì đơn bị tách theo shop, `platform_discount` phải **phân bổ về từng đơn con** — nếu không thì không biết ghi bao nhiêu vào invoice của shop nào, và không hoàn đúng tiền khi huỷ một phần.

Phân bổ **tỉ lệ thuận** theo phần đóng góp của từng shop, dùng **phương pháp phần dư lớn nhất** để tổng khớp tuyệt đối:

```
raw_s       = platform_discount × (subtotal_s − shop_discount_s) / base_platform
alloc_s     = floor(raw_s)
remainder   = platform_discount − Σ alloc_s
```

Cộng thêm 1 VND vào `alloc_s` của các shop có phần thập phân `raw_s − alloc_s` lớn nhất, cho tới khi hết `remainder`. Bằng nhau thì ưu tiên `shop_id` nhỏ hơn (để kết quả **tất định**, chạy lại luôn ra cùng số).

> Bất biến bắt buộc: `Σ alloc_s == platform_discount`. Đây là assertion phải có trong code, không chỉ trong test.

#### Bước 3 — Voucher FREESHIP (toàn giỏ, tối đa 1 mã)

Chỉ áp lên phí ship, **không** đụng tới tiền hàng:

```
base_ship = Σ shipping_s
PERCENT:  freeship_discount = min( floor(base_ship × value / 100), max_discount_amount )
FIXED:    freeship_discount = min( value, base_ship )
```

Phân bổ về từng shop theo tỉ lệ `shipping_s / base_ship`, cùng phương pháp phần dư lớn nhất. `freeship_discount` không bao giờ vượt `base_ship` — không có chuyện ship âm.

#### Bước 4 — Tách thuế theo danh mục

Mỗi item mang `tax_rate` **snapshot tại thời điểm checkout**, lấy từ category của sản phẩm (Product Catalog cung cấp, xem §5.2). Vì giá đã gồm VAT, thuế được tách ngược ra:

```
line_net_i = line_total_i − (phần discount phân bổ về item i)
tax_i      = round_half_up( line_net_i × rate_i / (1 + rate_i) )
tax_s      = Σ tax_i của các item thuộc shop s
```

Discount phân bổ xuống từng item cũng theo tỉ lệ `line_total_i / subtotal_s`, phần dư lớn nhất.

Thuế **không** làm đổi `grand_total`; nó chỉ là dòng breakdown trên hoá đơn ("Đã bao gồm VAT").

#### Bước 5 — Tổng của mỗi đơn con

```
grand_total_s = subtotal_s − shop_discount_s − alloc_s + shipping_s − freeship_alloc_s
```

Bất biến: `grand_total_s >= 0`. Nếu công thức ra số âm thì có voucher cấu hình sai — chặn ở bước validate, không được ghi đơn âm.

#### Ví dụ chuẩn *(dùng làm golden test)*

Giỏ: Shop A 800.000 (2 item: 500.000 + 300.000), Shop B 400.000. Ship 30.000/shop.
Voucher: `APPLE300K` (SHOP A, FIXED 300.000) · `TACA200K` (PLATFORM, FIXED 200.000) · `FREESHIPMAX` (FREESHIP, FIXED 60.000).

| Bước | Shop A | Shop B | Tổng |
|---|---:|---:|---:|
| `subtotal_s` | 800.000 | 400.000 | 1.200.000 |
| Bước 1 — shop discount | −300.000 | 0 | −300.000 |
| Sau bước 1 | 500.000 | 400.000 | 900.000 |
| Bước 2 — platform phân bổ | −111.111 | −88.889 | −200.000 |
| `shipping_s` | 30.000 | 30.000 | 60.000 |
| Bước 3 — freeship phân bổ | −30.000 | −30.000 | −60.000 |
| **`grand_total_s`** | **388.889** | **311.111** | **700.000** |

Kiểm tra bước 2: `raw_A = 200000 × 500000/900000 = 111111,11` → `floor` = 111.111; `raw_B = 88888,89` → `floor` = 88.888. Tổng 199.999, thiếu 1 VND. Phần thập phân của B (0,89) lớn hơn của A (0,11) nên B nhận thêm 1 → 88.889. Tổng khớp 200.000 ✓


### 2.3 Place checkout

1. Validate JWT, cart ownership, idempotency key và selected items.
2. Re-read Product fields và giá; snapshot `title`, `sku`, `shop`, `unit_price`, `currency=VND`.
3. Validate/redeem voucher trong transaction hoặc reservation protocol; không vượt usage limit.
4. Gọi Inventory `reserve` với `reservation_id`, `order_intent_id`, item SKU/qty và TTL.
5. Nếu reserve fail: rollback order intent/voucher hold, trả `ORDER_STOCK_UNAVAILABLE`.
6. Tạo orders split theo shop + order_items + address snapshot + invoice pending + outbox `order.created` trong MySQL transaction. Trạng thái khởi tạo phụ thuộc payment method — xem bước 7 và vòng đời ở `order.md` §2.1.
7. Rẽ nhánh theo payment method:
   - **VNPAY**: order khởi tạo `PENDING_PAYMENT`, payment projection `PENDING`. Tạo payment intent qua Payment-Wallet sau order commit/idempotency. Reservation giữ nguyên `RESERVED`, chỉ commit khi nhận `payment.succeeded`.
   - **COD**: order khởi tạo thẳng `CONFIRMED` (**không** đi qua `PENDING_PAYMENT`), payment projection `PENDING_COD`. Gọi Inventory `commit` ngay trong luồng checkout vì không có bước chờ tiền; phát `order.confirmed` cùng `order.created`.
8. Trả `order_id(s)`, `checkout_group_id`, payment next action và reservation expiry.

> **Vì sao COD không đi qua `PENDING_PAYMENT`:** với COD tiền chỉ được thu khi giao hàng, nên `payment.succeeded` chỉ tới **sau** `DELIVERED`. Nếu để đơn COD nằm ở `PENDING_PAYMENT` chờ event đó thì đơn không bao giờ tiến được (không tới `PROCESSING`/`SHIPPED` để mà giao), đồng thời bị job timeout huỷ với `PAYMENT_TIMEOUT` và bị nhả kho khi hết `INVENTORY_RESERVATION_TTL`. `PENDING_PAYMENT` chỉ dành cho phương thức trả trước.

Không dùng distributed transaction giữa MySQL/Inventory/Payment; dùng idempotency, saga compensation và event reconciliation.

## 3. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `CHECKOUT_PREVIEW_TTL` | 5 phút | Chỉ warning/UX, không giữ stock. |
| `INVENTORY_RESERVATION_TTL` | 15 phút | Exact TTL phải align Inventory. |
| `VND_MIN/MAX` | 1/999999999999 | Integer; snapshot không float. |
| `INTEGRATION_TIMEOUT` | 5s | Checkout dependency baseline; Payment/Shipment có route override. |

## 4. Enum & trạng thái

Checkout không có enum riêng. Trạng thái order/payment khởi tạo dùng `OrderStatus`/`PaymentStatusProjection` (`order.md` §4); voucher dùng `VoucherScope`/`DiscountType` (`voucher.md` §4).

## 5. Contract với service khác

### 5.1 Contract — Inventory reserve request

Contract thật do Inventory sở hữu (`inventory-docs/docs/api/inventory.md` §3.2). Correlation đi ở **header**, không nằm trong body; item chỉ có `sku_id` + `quantity` (Inventory không nhận `product_id`).

```http
POST /internal/inventory/reservations
Idempotency-Key: checkout-intent-01912f81
X-Request-ID: 01912f80-7a1b-7c12-9c55-8b1c34a6d921
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json
```

```json
{
  "order_intent_id": "intent-01912f81",
  "expires_at": "2026-08-30T09:15:00Z",
  "items": [{ "sku_id": "sku-01912f33", "quantity": 2 }]
}
```

Response `201`: `{ "data": { "reservation_id", "status": "RESERVED", "expires_at", "items": [...] } }`. Commit/release theo `POST /internal/inventory/reservations/{reservationId}/commit|release`. `expires_at` do Order đặt phải khớp `INVENTORY_RESERVATION_TTL` (15 phút) đã align với Inventory.

### 5.2 Contract — thuế suất từ Product Catalog

Thuế tính **theo danh mục**, mà danh mục do Product Catalog sở hữu. Order-Commerce không tự suy ra thuế suất và không lưu bảng thuế riêng.

Thuế suất đi kèm ngay trong response `GET /products?product_ids=` / `GET /products/{productId}` mà Checkout **đã gọi sẵn** để re-read giá — không thêm lượt gọi mạng nào:

```json
{
  "product_id": "product-01912f31",
  "primary_category_id": "category-01912f20",
  "tax_rate_bps": 1000
}
```

| Field | Kiểu | Nghĩa |
|---|---|---|
| `tax_rate_bps` | int | Thuế suất VAT theo basis point (10% = `1000`). Product Catalog **đã giải quyết xong** việc thừa kế theo cây danh mục, trả ra giá trị cuối cùng — Order-Commerce không phải leo cây. |

Quy tắc:

- Checkout **snapshot** `tax_rate_bps` vào từng `order_items` tại thời điểm đặt hàng. Admin đổi thuế suất danh mục về sau **không** hồi tố đơn cũ — hoá đơn đã phát hành phải bất biến.
- Thiếu `tax_rate_bps` trong response (product cũ chưa migrate) → coi như `0`, ghi log cảnh báo, **không** chặn checkout. Chặn đặt hàng vì thiếu cấu hình thuế là đánh đổi sai.
- Giá đã gồm VAT nên thuế **không** cộng thêm vào `grand_total`; xem §2.2 bước 4.

Event `order.created`/`order.confirmed` phát ra tại checkout được mô tả ở `order.md` §5.

## 6. Mã lỗi

`ORDER_CART_EMPTY`, `ORDER_PRODUCT_UNAVAILABLE`, `ORDER_PRICE_CHANGED`, `ORDER_STOCK_UNAVAILABLE`, `ORDER_RESERVATION_EXPIRED`, `ORDER_VOUCHER_INVALID`, `ORDER_VOUCHER_SLOT_CONFLICT`, `ORDER_VOUCHER_LIMIT_REACHED`, `ORDER_ADDRESS_INVALID`, `ORDER_IDEMPOTENCY_CONFLICT`, `ORDER_DEPENDENCY_UNAVAILABLE` — định nghĩa ở `order-commerce.md` §7.

## 7. Giả định & câu hỏi mở

Xem `order-commerce.md` §8 mục 3 (reserve trước tạo order, payment tách saga), 4 (split theo shop), 5 (shipping fee mock), 8 (tax provider).
