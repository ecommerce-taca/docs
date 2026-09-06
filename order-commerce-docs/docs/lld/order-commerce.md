# LLD — Order-Commerce Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 · Redis Cart · Kafka outbox · REST với Product/Inventory/Payment/Shipment/Auth

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Cart, checkout orchestration, order split theo shop, price/address snapshot, voucher, invoice metadata, buyer/seller order read và cancel/fulfill command. |
| Database | MySQL 8.4 `orderdb`; Redis chỉ lưu cart ephemeral/TTL và không là order source of truth. |
| Payment | Payment-Wallet sở hữu payment transaction, VNPAY/COD reconciliation, wallet/commission; Order chỉ lưu payment method/status projection. |
| Inventory | Inventory sở hữu availability/reserve/commit/release/deduct; Order gọi contract và không tự trừ stock. |
| Shipment | Shipment sở hữu carrier/tracking/ship status; Order lưu shipment snapshot/status projection. |
| Product | Product Catalog cung cấp content/giá hiện tại; Order snapshot title/sku/shop/unit price tại checkout. |
| Không thuộc service | Auth/KYC decision, product CRUD, inventory ledger, payment gateway, shipping carrier, rating/message/notification delivery. |

### 1.2 Boundary

```text
mfe-buyer/mfe-seller ──Gateway──► Order-Commerce
                              ├─ MySQL orderdb: orders/items/vouchers/invoices
                              ├─ Redis: cart:{user_id}, TTL
                              ├─ REST Product: validate price/product/sku
                              ├─ REST Inventory: availability/reserve/commit/release
                              ├─ REST Payment: create/reconcile payment
                              └─ Kafka outbox → Notification/Shipment/Payment/Analytics
```

- Order snapshot là bất biến sau place trừ workflow correction/audit; không đọc giá Product để thay đổi đơn cũ.
- Một checkout nhiều shop tạo nhiều order con, liên kết bằng `checkout_group_id`.
- Idempotency bắt buộc cho checkout/cancel/fulfill/voucher redemption; retry không tạo duplicate order.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Order | Cart + Checkout + Invoice + Voucher | Gom trong Order-Commerce. |
| Buyer Cart | Get/add/update/delete item | Redis cart + product validation khi add/checkout. |
| Buyer Checkout | Address, shipping fee, payment method, voucher | Preview rồi place; address snapshot từ Auth User; shipping fee qua Shipment mock/contract. |
| Buyer Orders | List/detail/status/cancel/invoice | Buyer scope từ JWT; cancel trước shipping theo state machine. |
| Seller Orders | List/detail/fulfill/cancel/invoice | Shop scope; seller không đổi buyer/payment/inventory ledger trực tiếp. |
| Payment | VNPAY QR hoặc COD | Payment-Wallet owns provider; Order phát pending payment/paid events. |

## 2. Cấu trúc bên trong

### 2.1 Module Spring Boot

```text
com.taca.order
├── cart/                 # Redis cart command/query
├── checkout/             # preview, price/voucher/shipping validation
├── order/                # aggregate, state machine, buyer/seller query
├── voucher/              # platform/shop voucher and redemption
├── invoice/              # invoice number/metadata/PDF reference
├── integrations/         # Product, Inventory, Payment, Shipment, Auth clients
├── outbox/               # transactional event publisher
├── security/             # actor/shop ownership and idempotency
└── observability/        # shared JSON log/trace/metric contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `CartController` | Cart CRUD | User ID từ auth context; Redis TTL; không tin user_id body. |
| `CheckoutController` | Preview/shipping fee/place | Place idempotency; reserve Inventory trước khi commit order. |
| `OrderController` | Buyer list/detail/status/cancel/invoice | Chỉ own orders; snapshot immutable. |
| `SellerOrderController` | Shop order list/detail/fulfill/cancel/invoice | Filter theo shop; không expose other shop data. |
| `VoucherService` | Validate/apply/redeem platform/shop voucher | Atomic usage limit; unique redemption theo voucher+user/order policy. |
| `OrderStateMachine` | Pending/paid/processing/shipped/delivered/cancelled | Không cho seller/buyer bypass transition. |
| `InventoryClient` | Reserve/commit/release | Timeout/failure không đánh dấu order paid; idempotency required. |
| `OutboxPublisher` | Publish order/invoice/voucher events | Transactional outbox, retry 3/backoff 2s/DLQ. |

### 2.2 Chuẩn observability dùng chung

- Log JSON stdout với field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST propagate `traceparent`/`tracestate` và `X-Request-ID`; Kafka headers giữ `traceparent`, `request_id`, `event_id`.
- Không log Authorization, refresh token, password, full address/phone/email, voucher raw secret, payment credential, VNPAY signature hoặc full cart/order payload.
- Chỉ log `order_id`, `checkout_group_id`, `shop_id`, `sku_id` theo allowlist; metric label không dùng `user_id/order_id`.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MySQL, Redis, Kafka và dependency policy.
- API error envelope `{error:{code,message,details,trace_id}}`; mọi error/audit có request/trace correlation.

## 3. Luồng xử lý

### 3.1 Cart

1. Authenticated user gọi get/add/update/delete cart.
2. Cart lưu Redis key `cart:{user_id}`, item gồm `product_id`, `sku_id`, `shop_id`, quantity và optional price display cache.
3. Add/update validate quantity >0 và Product/Inventory availability snapshot; không coi cart là reservation.
4. Giá/tiêu đề trong cart chỉ là cache hiển thị; Checkout phải re-read Product và tạo price snapshot.
5. Cart expired/empty không tạo order; mfe-buyer nhận `ORDER_CART_EMPTY`.

### 3.2 Checkout preview

```text
Cart → load selected items → Product validate active SKU/price
     → Inventory availability check (read-only)
     → validate voucher/usage/time/scope
     → calculate subtotal/discount/tax/shipping/total
     → return preview with warnings; không reserve, không charge
```

Preview không đảm bảo giá/stock đến lúc place; response ghi `expires_at` ngắn và Checkout place phải tính lại.

### 3.2a Công thức tính tiền *(nguồn sự thật — mọi nơi khác tham chiếu về đây)*

Đây là phép tính quan trọng nhất của hệ thống. Preview, place checkout, invoice và hiển thị FE **phải dùng chung đúng một cài đặt**; lệch một đồng là lệch đối soát.

**Nguyên tắc bất di bất dịch:**

- Mọi số tiền là **số nguyên VND**. Cấm float/double ở mọi bước trung gian.
- Giá sản phẩm **đã bao gồm VAT** (chuẩn bán lẻ VN). Thuế được **tách ra để ghi hoá đơn**, không cộng thêm vào `grand_total`.
- Thứ tự áp dụng là cố định: shop voucher → platform voucher → freeship voucher → tách thuế.

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

Mỗi item mang `tax_rate` **snapshot tại thời điểm checkout**, lấy từ category của sản phẩm (Product Catalog cung cấp, xem §6.4). Vì giá đã gồm VAT, thuế được tách ngược ra:

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


### 3.3 Place checkout

1. Validate JWT, cart ownership, idempotency key và selected items.
2. Re-read Product fields và giá; snapshot `title`, `sku`, `shop`, `unit_price`, `currency=VND`.
3. Validate/redeem voucher trong transaction hoặc reservation protocol; không vượt usage limit.
4. Gọi Inventory `reserve` với `reservation_id`, `order_intent_id`, item SKU/qty và TTL.
5. Nếu reserve fail: rollback order intent/voucher hold, trả `ORDER_STOCK_UNAVAILABLE`.
6. Tạo orders split theo shop + order_items + address snapshot + invoice pending + outbox `order.created` trong MySQL transaction. Trạng thái khởi tạo phụ thuộc payment method — xem bước 7.
7. Rẽ nhánh theo payment method:
   - **VNPAY**: order khởi tạo `PENDING_PAYMENT`, payment projection `PENDING`. Tạo payment intent qua Payment-Wallet sau order commit/idempotency. Reservation giữ nguyên `RESERVED`, chỉ commit khi nhận `payment.succeeded`.
   - **COD**: order khởi tạo thẳng `CONFIRMED` (**không** đi qua `PENDING_PAYMENT`), payment projection `PENDING_COD`. Gọi Inventory `commit` ngay trong luồng checkout vì không có bước chờ tiền; phát `order.confirmed` cùng `order.created`.
8. Trả `order_id(s)`, `checkout_group_id`, payment next action và reservation expiry.

> **Vì sao COD không đi qua `PENDING_PAYMENT`:** với COD tiền chỉ được thu khi giao hàng, nên `payment.succeeded` chỉ tới **sau** `DELIVERED`. Nếu để đơn COD nằm ở `PENDING_PAYMENT` chờ event đó thì đơn không bao giờ tiến được (không tới `PROCESSING`/`SHIPPED` để mà giao), đồng thời bị job timeout huỷ với `PAYMENT_TIMEOUT` và bị nhả kho khi hết `INVENTORY_RESERVATION_TTL`. `PENDING_PAYMENT` chỉ dành cho phương thức trả trước.

Không dùng distributed transaction giữa MySQL/Inventory/Payment; dùng idempotency, saga compensation và event reconciliation.

### 3.4 Order lifecycle

`OrderStatus` là trạng thái **fulfillment**, không phải trạng thái tiền. Trạng thái tiền nằm ở `PaymentStatusProjection` và do Payment-Wallet sở hữu. Hai phương thức thanh toán vào cùng một cửa fulfillment (`CONFIRMED`) nhưng thu tiền ở hai thời điểm khác nhau.

```text
VNPAY:  [checkout] → PENDING_PAYMENT --payment.succeeded--> CONFIRMED → PROCESSING → SHIPPED → DELIVERED
                            │                                                                      (tiền đã thu từ trước)
                            └──timeout──> CANCELLED

COD:    [checkout] ────────────────────────────────────────► CONFIRMED → PROCESSING → SHIPPED → DELIVERED
                                                                                                   │
                                                                          capture tiền mặt ────────┘
                                                                          → payment.succeeded → payment_status=SUCCESS
```

| Transition | Actor | Điều kiện |
|---|---|---|
| `→ PENDING_PAYMENT` | Checkout (VNPAY) | Order tạo với phương thức trả trước; reservation `RESERVED`, chưa commit. |
| `→ CONFIRMED` | Checkout (COD) | Order tạo với `method=COD`; commit reservation ngay; payment projection `PENDING_COD`. |
| `PENDING_PAYMENT → CONFIRMED` | Payment event | `payment.succeeded` từ Payment-Wallet; commit reservation; payment projection `SUCCESS`. |
| `PENDING_PAYMENT → CANCELLED` | System/buyer timeout | **Chỉ áp dụng cho đơn trả trước.** Release reservation; `PAYMENT_TIMEOUT`. Đơn COD không bao giờ ở trạng thái này nên không bị job này chạm tới. |
| `CONFIRMED → PROCESSING` | Seller/system | Seller accept đơn (hàng đợi "Chờ xác nhận" gồm **cả** đơn VNPAY đã trả và đơn COD). |
| `PROCESSING → SHIPPED` | Seller/Shipment | Shipment created/tracking valid. |
| `SHIPPED → DELIVERED` | Shipment event | Carrier confirms delivered. Với COD, đây là điểm kích hoạt capture tiền ở Payment-Wallet — **không** đổi `OrderStatus` thêm lần nữa. |
| `CONFIRMED/PROCESSING → CANCELLED` | Buyer/seller/system theo policy | Buyer chỉ cancel trước shipping. VNPAY: release reservation đã commit + refund. COD: release/hoàn kho, không refund vì chưa thu tiền. |
| `CANCELLED/DELIVERED` | Không reopen | Tạo refund/return workflow ở Payment/Support nếu cần. |

Ràng buộc bổ sung:

- `PAID` **không còn là một `OrderStatus`**. Trước đây trạng thái này vừa mang nghĩa "đã thu tiền" vừa mang nghĩa "được phép giao", làm COD không có đường đi. Nghĩa "đã thu tiền" nay đọc ở `payment_status = SUCCESS`; nghĩa "được phép giao" là `CONFIRMED`.
- Event `order.paid` vẫn giữ nguyên ngữ nghĩa **tiền đã về** và vẫn phát khi `payment.succeeded` — với VNPAY là lúc vào `CONFIRMED`, với COD là sau `DELIVERED`. Consumer nào cần "đơn đã chốt, chuẩn bị giao" phải dùng `order.confirmed`, không dùng `order.paid`.
- Không service nào được suy ra quyền fulfillment từ `payment_status`; chỉ đọc `OrderStatus`.

### 3.5 Voucher và invoice

- Voucher scope `PLATFORM` hoặc `SHOP`; một order item/shop phải kiểm scope trước split.
- **Discovery vs validate là hai đường khác nhau.** `GET /vouchers` liệt kê voucher buyer đang dùng được (HLD #23 "choose ... voucher"), `GET /vouchers/validate` kiểm một mã cụ thể buyer đã biết. Cả hai chỉ **đọc**, không giữ chỗ, không tăng `used_count`; redemption duy nhất xảy ra trong `POST /checkout`. Cùng dùng chung bộ mã lý do không hợp lệ để UI không phải map hai bảng.
- `GET /vouchers` cho phép gọi **không cần đăng nhập** (PDP/Shop voucher strip là màn public); khi không có JWT thì `eligible` trả `null`, chỉ hiển thị điều kiện. Có JWT + `cart_id` mới tính `eligible`/`discount_amount`.
- `GET /vouchers/redemptions` đọc từ `voucher_redemptions` join order để loại redemption của order đã huỷ/hoàn khỏi `total_saved`.
- Voucher v1 chỉ áp theo **mã** (`code`) + điều kiện `min_order`/thời gian/`usage_limit`; ràng buộc `PERCENT` 1–100, `FIXED` VND, `FREESHIP` VND hoặc phần trăm phí ship. Không có audience/targeting (selected users, segment, VIP, followers, import user IDs). Các màn Penpot Seller "Voucher audience / Selected users / Tab followers / Import user IDs / Create targeted voucher" là **ngoài phạm vi v1**; Penpot phần này cần đơn giản hoá về form tạo voucher theo mã. Nếu bật targeting sau này cần bảng `voucher_audiences` + quan hệ follow shop (chưa có trong v1).

#### Quy tắc cộng dồn voucher — mô hình 3 slot

Một lần checkout áp được **tối đa 3 loại voucher cùng lúc**, mỗi loại một mã:

| `scope` | Số mã | Áp lên | Ai tạo |
|---|---|---|---|
| `PLATFORM` | 1 cho cả giỏ | Tiền hàng, sau khi trừ shop voucher | Admin (`/admin/vouchers`) |
| `SHOP` | 1 **mỗi shop** | Tiền hàng của đúng shop đó | Seller (`/seller/vouchers`) |
| `FREESHIP` | 1 cho cả giỏ | Chỉ phí ship | Admin |

Ràng buộc:

- **Không** cộng dồn hai mã cùng `scope`. Gửi 2 mã `PLATFORM` → `400 ORDER_VOUCHER_SLOT_CONFLICT`.
- Giỏ 3 shop có thể dùng 3 mã `SHOP` khác nhau — mỗi mã chỉ ăn vào shop của nó.
- Mã `SHOP` gửi kèm mà giỏ không có shop tương ứng → mã đó bị bỏ qua, trả `warning VOUCHER_NOT_APPLICABLE`, **không** chặn đặt hàng.
- Thứ tự áp dụng cố định (shop → platform → freeship) — xem §3.2a. Đổi thứ tự sẽ ra số khác vì `min_order` của platform tính trên phần đã trừ shop voucher.
- Mỗi mã sinh **một** bản ghi `voucher_redemptions` riêng; một order có nhiều redemption là bình thường.
- Platform voucher (`scope=PLATFORM`) và freeship voucher (`scope=FREESHIP`) do admin tạo/sửa qua `order-commerce` (endpoint admin); seller chỉ tạo `scope=SHOP`.
- Usage increment/redemption idempotent; không giảm `used_count` nếu order không tạo được hoặc đã release theo policy.

#### Huỷ một phần nhóm đơn — tính lại voucher

Khi buyer huỷ **một đơn con** trong nhóm nhiều shop, voucher `PLATFORM`/`FREESHIP` đã phân bổ cho cả nhóm phải được tính lại; nếu không, phần giảm giá của đơn bị huỷ sẽ bốc hơi hoặc bị tính hai lần.

Quy tắc: **giữ voucher, tính lại trên phần còn lại.**

1. Bỏ đơn bị huỷ ra khỏi giỏ, tính lại `base_platform` và `base_ship` trên các đơn còn lại.
2. Kiểm lại `min_order` của từng voucher trên base mới.
3. **Còn đủ điều kiện** → phân bổ lại theo §3.2a. Đơn còn lại có thể được giảm **nhiều hơn** trước (vì cùng số tiền giảm chia cho ít đơn hơn); phần chênh này ghi nhận là điều chỉnh giảm.
4. **Không còn đủ `min_order`** → voucher bị gỡ khỏi nhóm, `used_count` được nhả về, các đơn còn lại mất phần giảm giá đó. Chênh lệch buyer phải trả thêm:
   - Đơn **chưa thu tiền** (COD, hoặc VNPAY chưa capture): cập nhật `grand_total` mới, buyer trả theo số mới.
   - Đơn **đã thu tiền** (VNPAY đã capture): **không** truy thu buyer. Phần chênh do sàn chịu, ghi nhận `VOUCHER_ADJUSTMENT_ABSORBED` trong audit. Truy thu sau khi đã trừ tiền là trải nghiệm không chấp nhận được và dễ thành khiếu nại.
5. Huỷ **toàn bộ** nhóm → nhả tất cả voucher, `used_count` trả về nguyên trạng.

Mọi lần tính lại đều ghi audit kèm `grand_total` trước/sau để đối soát truy được.
- Invoice có số unique, total/tax breakdown snapshot và `pdf_url` nullable; PDF generation/notification có thể async.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `PAGE_SIZE_DEFAULT` | 20 | Max 100. |
| `CART_TTL` | 30 ngày | Redis; cart không reserve stock. |
| `CART_MAX_ITEMS` | 100 | Item line, không phải quantity. |
| `ITEM_QTY_MIN` | 1 | Integer. |
| `ITEM_QTY_MAX` | 999 | Per SKU/order line. |
| `CHECKOUT_PREVIEW_TTL` | 5 phút | Chỉ warning/UX, không giữ stock. |
| `INVENTORY_RESERVATION_TTL` | 15 phút | Exact TTL phải align Inventory. |
| `ORDER_NUMBER_MAX_LENGTH` | 32 | Unique. |
| `VND_MIN/MAX` | 1/999999999999 | Integer; snapshot không float. |
| `OUTBOX_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `INTEGRATION_TIMEOUT` | 5s | Checkout dependency baseline; Payment/Shipment có route override. |
| `IDEMPOTENCY_RETENTION` | 24h | Place/cancel/fulfill response snapshot. |
| `AUDIT_RETENTION` | 365 ngày | Không lưu payment credential/raw address ngoài snapshot nghiệp vụ cần thiết. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | DB DATETIME(6), API/event UTC. |
| `ORDER_EXPORT_MAX_ROWS` | 10000 | Vượt trả `400 ORDER_EXPORT_TOO_LARGE`, seller lọc hẹp hơn. |
| `ORDER_EXPORT_URL_TTL` | 30 phút | Signed download URL hết hạn, gọi lại endpoint để có URL mới. |
| `SELLER_BULK_ACCEPT_MAX_ORDERS` | 100 | `POST /seller/orders/bulk-accept` — tối đa order_id/lần gọi. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `CartStatus` | `ACTIVE`, `EXPIRED`, `CHECKED_OUT`. |
| `OrderStatus` | `PENDING_PAYMENT`, `CONFIRMED`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. Trạng thái fulfillment; `PENDING_PAYMENT` chỉ dùng cho phương thức trả trước (VNPAY). |
| `PaymentStatusProjection` | `PENDING`, `PENDING_COD`, `SUCCESS`, `FAILED`, `REFUNDED`. |
| `VoucherStatus` | `DRAFT`, `ACTIVE`, `INACTIVE`, `EXPIRED`, `ARCHIVED`. |
| `DiscountType` | `PERCENT`, `FIXED`, `FREESHIP`. `FREESHIP` chỉ áp lên phí ship; `PERCENT`/`FIXED` chỉ áp lên tiền hàng. |
| `VoucherScope` | `PLATFORM`, `SHOP`, `FREESHIP`. Vừa là phạm vi áp dụng vừa là **slot cộng dồn**: mỗi checkout tối đa 1 mã mỗi scope (`SHOP` là 1 mã **mỗi shop**). |
| `InvoiceStatus` | `PENDING`, `ISSUED`, `VOIDED`. |
| `CancellationReason` | `BUYER_REQUEST`, `SELLER_REQUEST`, `PAYMENT_TIMEOUT`, `STOCK_FAILURE`, `SYSTEM`. |

Order state là source of truth của Order-Commerce; payment/shipment/inventory statuses được projection và không cho client tự set.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `order.events.v1` | `order.created` | order/group, buyer, shop, items snapshot, total, payment method |
| `order.events.v1` | `order.confirmed` | order/group, shop, payment method, confirmed_at |
| `order.events.v1` | `order.paid` | order, payment reference, paid_at |
| `order.events.v1` | `order.processing` | order, shop |
| `order.events.v1` | `order.shipped` | order, shipment reference |
| `order.events.v1` | `order.delivered` | order, delivered_at |
| `order.events.v1` | `order.cancelled` | order, reason, compensation action |
| `invoice.events.v1` | `invoice.issued` | invoice/order/number/total/pdf reference |
| `voucher.events.v1` | `voucher.redeemed/released` | voucher/order/user/scope |

Phân biệt bắt buộc giữa hai event dễ nhầm:

| Event | Nghĩa | VNPAY phát lúc | COD phát lúc | Dùng cho |
|---|---|---|---|---|
| `order.confirmed` | Đơn đã chốt, **được phép giao** | Khi nhận `payment.succeeded` | Ngay tại checkout | Fulfillment, email xác nhận đơn, hàng đợi seller |
| `order.paid` | **Tiền đã về** | Cùng lúc `order.confirmed` | Sau `DELIVERED`, khi capture tiền mặt | Đối soát, ledger, settlement |

Với VNPAY hai event trùng thời điểm; với COD chúng cách nhau cả vòng đời giao hàng. Consumer nào cần "đơn sẵn sàng giao" mà dùng `order.paid` sẽ bỏ sót toàn bộ đơn COD.

#### 6.1a Payload chi tiết — `order.confirmed` / `order.paid` / `order.cancelled` / `invoice.issued`

Field `buyer` **đã chốt** cho các consumer cần recipient (đặc biệt Notification Service —
xem `notification-docs/docs/lld/notification.md` §6.2): `buyer.user_id` bắt buộc, `buyer.email`
optional (`null` cho phép ở event chỉ phát `IN_APP`, ví dụ `order.paid`). Field khác trong `payload`
lấy tên đúng theo response `GET /orders/{orderId}` (§3.6) và bảng `invoices` (db.md §3.4).

`order.confirmed`:

```json
{
  "order_id": "order-01912f91",
  "order_number": "TC-20260830-0001",
  "checkout_group_id": "group-01912f90",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "shop": { "shop_id": "shop-01912f31", "shop_name": "Anker Official" },
  "payment_method": "VNPAY",
  "confirmed_at": "2026-08-30T09:05:00Z"
}
```

`order.paid` (chỉ phát `IN_APP` — xem bảng phân biệt ở trên — nên `buyer.email` có thể `null`):

```json
{
  "order_id": "order-01912f91",
  "buyer": { "user_id": "user-01912f10", "email": null },
  "payment_reference": "payment-01912f95",
  "paid_at": "2026-08-30T09:05:00Z"
}
```

`order.cancelled`:

```json
{
  "order_id": "order-01912f91",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "reason": "Hết hàng",
  "cancelled_at": "2026-08-30T10:00:00Z"
}
```

> `compensation action` (nhắc ở bảng §6.1) chưa có schema field cụ thể — phụ thuộc seller
> cancel/refund policy chưa chốt (xem §8 mục 7), nên chưa đưa vào ví dụ JSON để tránh set field
> giả không đúng thực tế.

`invoice.issued`:

```json
{
  "invoice_id": "invoice-01912fb0",
  "order_id": "order-01912f91",
  "invoice_number": "INV-20260830-0001",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "total_amount": 1094000,
  "pdf_url": "https://cdn.taca.vn/invoices/invoice-01912fb0.pdf",
  "issued_at": "2026-08-30T09:10:00Z"
}
```

### 6.2 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Payment-Wallet | `payment.succeeded`, `payment.failed`, `payment.refunded` | Update payment projection, transition order, commit/release Inventory. |
| Inventory | `inventory.reservation.created`, `inventory.reservation.committed`, `inventory.reservation.released`, `inventory.reservation.expired` | Reconcile order intent/reservation; không tự sửa quantity. Tên event lấy đúng catalog của Inventory — **không có** `reservation.confirmed`/`reservation.rejected`/`stock_committed`: reserve thành công/thất bại là kết quả **đồng bộ** của `POST /internal/inventory/reservations`, event chỉ dùng để reconcile khi mất response. |
| Shipment | `shipment.created`, `shipment.status_changed` | Update shipment projection/order status. |
| Auth User | `user.status_changed` (mock policy) | Chặn hành động account mới nếu suspended; không đổi order history. |

### 6.3 Contract — Inventory reserve request

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

Response `201`: `{ "data": { "reservation_id", "status": "RESERVED", "expires_at", "items": [...] } }`. Commit/release theo `POST /internal/inventory/reservations/{id}/commit|release`. `expires_at` do Order đặt phải khớp `INVENTORY_RESERVATION_TTL` (15 phút) đã align với Inventory.

### 6.4 Contract — thuế suất từ Product Catalog

Thuế tính **theo danh mục**, mà danh mục do Product Catalog sở hữu. Order-Commerce không tự suy ra thuế suất và không lưu bảng thuế riêng.

Thuế suất đi kèm ngay trong response `GET /products?product_ids=` / `GET /products/{id}` mà Checkout **đã gọi sẵn** để re-read giá — không thêm lượt gọi mạng nào:

```json
{
  "product_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
  "primary_category_id": "01912f20-7a1b-7c12-9c55-8b1c34a6d921",
  "tax_rate_bps": 1000
}
```

| Field | Kiểu | Nghĩa |
|---|---|---|
| `tax_rate_bps` | int | Thuế suất VAT theo basis point (10% = `1000`). Product Catalog **đã giải quyết xong** việc thừa kế theo cây danh mục, trả ra giá trị cuối cùng — Order-Commerce không phải leo cây. |

Quy tắc:

- Checkout **snapshot** `tax_rate_bps` vào từng `order_items` tại thời điểm đặt hàng. Admin đổi thuế suất danh mục về sau **không** hồi tố đơn cũ — hoá đơn đã phát hành phải bất biến.
- Thiếu `tax_rate_bps` trong response (product cũ chưa migrate) → coi như `0`, ghi log cảnh báo, **không** chặn checkout. Chặn đặt hàng vì thiếu cấu hình thuế là đánh đổi sai.
- Giá đã gồm VAT nên thuế **không** cộng thêm vào `grand_total`; xem §3.2a bước 4.

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `ORDER_INVALID_INPUT` | 400 | Payload/query sai. |
| `ORDER_UNAUTHENTICATED` | 401 | Cart/checkout/order cần login. |
| `ORDER_FORBIDDEN` | 403 | Không thuộc buyer/shop scope. |
| `ORDER_CART_EMPTY` | 409 | Cart không có selected item. |
| `ORDER_CART_ITEM_NOT_FOUND` | 404 | Item không tồn tại trong cart. |
| `ORDER_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active. |
| `ORDER_PRICE_CHANGED` | 409 | Giá hiện tại khác preview/cart; cần xác nhận lại. |
| `ORDER_STOCK_UNAVAILABLE` | 409 | Inventory reserve thất bại/quantity không đủ. |
| `ORDER_RESERVATION_EXPIRED` | 409 | Reservation hết TTL. |
| `ORDER_VOUCHER_INVALID` | 400 | Code/scope/time/value không hợp lệ. |
| `ORDER_VOUCHER_LIMIT_REACHED` | 409 | Hết usage/user limit. |
| `ORDER_ADDRESS_INVALID` | 400 | Address snapshot không hợp lệ. |
| `ORDER_STATE_INVALID` | 409 | Transition không hợp lệ. |
| `ORDER_CANCEL_NOT_ALLOWED` | 409 | Đã shipping hoặc policy không cho cancel. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Key dùng lại với payload khác. |
| `ORDER_DEPENDENCY_UNAVAILABLE` | 503 | Product/Inventory/Payment/Shipment timeout/down. |
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice chưa issued. |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Database Order là MySQL 8.4 và backend Spring Boot/Java 25 theo lựa chọn A. | Ảnh hưởng transaction, migration và deployment BOM. | Tech lead |
| 2 | Cart nằm Redis; Order/Invoice/Voucher nằm MySQL. | Nếu muốn cart durable, cần thêm schema và recovery policy. | Architecture |
| 3 | Checkout tạo order sau khi Inventory reserve; không charge/payment trong cùng distributed transaction. | Ảnh hưởng compensation, timeout và payment reconciliation. | Order/Payment/Inventory owner |
| 4 | Một checkout multi-shop split thành order con theo shop. | Ảnh hưởng voucher scope, shipment, seller view và invoice grouping. | Product owner |
| 5 | Shipping fee là mock/integration với Shipment; exact carrier calculation chưa có HLD contract. | Cần thêm quote contract/SLA trước production. | Shipment owner |
| 6 | Payment service tên `payment-wallet`; HLD có chỗ gọi Payment Service. | Cần chuẩn hóa topic/API service name ở platform registry. | Architecture |
| 7 | Buyer cancel được trước shipping; seller cancel policy cần business rule chi tiết. Return/dispute workflow **ngoài v1** — service `dispute` (v1.1) điều phối; nếu cần sớm thì làm module trong `order-commerce` (`System_Overview.md` §6.3). | Ảnh hưởng refund/stock release và dispute. | Product/Finance |
| 8 | Tax calculation baseline là snapshot/tax breakdown từ Order; tax authority/provider chưa chốt. | Ảnh hưởng invoice compliance. | Finance owner |
| 9 | Voucher v1 chỉ theo mã + scope PLATFORM/SHOP + `min_order`/thời gian/`usage_limit`; **không có** audience/targeting/segment/import user IDs. Các màn Penpot Seller voucher targeting là ngoài v1. | Nếu bật targeting cần bảng `voucher_audiences` + quan hệ follow shop; đổi checkout redemption. | Product owner |
| 10 | Đã chốt (`System_Overview.md` §6.3): v1 **không** tách microservice admin. Platform voucher (`scope=PLATFORM`) do `order-commerce` sở hữu; admin console gọi trực tiếp `/api/v1/admin/vouchers/**` của service này (`VOUCHER_MANAGE` + 2FA), Gateway chỉ coarse-gate role admin. | Nếu voucher aggregate chuyển service khác phải đổi event/redemption. | Product owner + Architecture |
