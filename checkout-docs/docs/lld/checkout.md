# LLD — Checkout Service

> Nguồn: `docs/api/checkout.md` · `docs/db/checkout.md` · `docs/test/checkout.md` · HLD `EcommercePlatform-v4(6).excalidraw` · Penpot `New File 1.penpot.zip` · Code: chưa có repo (`ecommerce-taca/checkout-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 `checkoutdb` · Kafka outbox · REST với Cart/Order/Product/Inventory/Shipment/Auth

Tài liệu này là **tổng quan** của Checkout Service và phần checkout. Phần voucher nằm ở `voucher.md` cùng thư mục; mỗi file cùng tên ở `docs/api/`, `docs/db/`, `docs/test/` là phần API/DB/test tương ứng.

| Tài liệu | Phạm vi |
|---|---|
| `checkout.md` | Ranh giới service, quy ước chung, shipping fee, preview, công thức tính tiền, place checkout (saga), tính lại khi huỷ đơn, mã lỗi. |
| `voucher.md` | Voucher buyer/seller/admin, mô hình 3 slot, giữ/chốt/nhả lượt dùng, quy tắc huỷ một phần. |

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

Order-Commerce cũ được tách làm 3 service ngày 2026-11-01: `cart-service` (giỏ), `checkout-service` (file này, gồm voucher) và `order-service` (đơn + invoice).

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Shipping fee preview, checkout preview, **công thức tính tiền** (nguồn sự thật), place checkout: đọc giỏ, snapshot giá/thuế/địa chỉ, giữ lượt voucher, reserve kho, gọi `order-service` tạo đơn; voucher CRUD/discovery/redemption; tính lại voucher khi huỷ đơn con. |
| Database | MySQL 8.4 `checkoutdb`: `checkout_groups`, `checkout_group_orders`, `vouchers`, `voucher_redemptions`, `voucher_adjustments`, `outbox_events`, `idempotency_keys`. |
| Cart | `cart-service` sở hữu giỏ; Checkout đọc qua `GET /internal/carts/{buyerUserId}` và báo xoá item qua `checked-out`. |
| Order | `order-service` sở hữu order, invoice, payment projection và **mọi lời gọi Payment-Wallet**; Checkout gọi `POST /internal/orders` (REST đồng bộ, quyết định 2026-11-01). |
| Inventory | Checkout chỉ **reserve** (và release khi saga thất bại trước khi có đơn). Commit/release sau khi đơn tồn tại do `order-service` gọi. |
| Shipment | Checkout gọi `POST /api/v1/internal/shipping/quote` để lấy phí ship. Tạo shipment do `order-service`. |
| Product | Product Catalog cung cấp giá hiện tại, `tax_rate_bps`; Checkout snapshot tại lúc place. |
| Không thuộc service | Order lifecycle, invoice, payment, fulfill, giỏ hàng, inventory ledger. |

### 1.2 Boundary

```text
mfe-buyer/mfe-seller/admin ──Gateway──► checkout-service
                                   ├─ MySQL checkoutdb: checkout groups, vouchers, redemptions, outbox
                                   ├─ REST cart-service: đọc giỏ / checked-out
                                   ├─ REST Product Catalog: GET /products/{id} (giá SKU, SKU active, tax_rate_bps)
                                   ├─ REST Auth User: địa chỉ buyer
                                   ├─ REST Shipment: shipping quote
                                   ├─ REST Inventory: availability / reserve / release (trước khi có đơn)
                                   ├─ REST order-service: POST /internal/orders
                                   └─ Kafka outbox → voucher.events.v1
order-service ──REST POST /internal/checkout-groups/{id}/recalculate──► checkout-service
```

- Một checkout nhiều shop tạo nhiều order con trong `order-service`, liên kết bằng `checkout_group_id` (Checkout sinh ID này).
- Idempotency bắt buộc cho place checkout, voucher mutation và recalculate; retry không tạo duplicate đơn hay trừ lượt voucher hai lần.
- Checkout **không** gọi Payment-Wallet: Payment-Wallet (đã có code) chỉ tin một internal caller `InternalService.ORDER_COMMERCE`, và vai trò này thuộc `order-service`. Vì vậy Payment không phải sửa.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Order | Checkout + Voucher | Gom trong `checkout-service`; voucher đi chung checkout vì dính chặt công thức tiền (quyết định 2026-11-01). |
| Buyer Checkout | Address, shipping fee, payment method, voucher | Preview rồi place; address từ Auth User; shipping fee qua Shipment quote. |
| Payment | VNPAY QR hoặc COD | Checkout truyền `payment_method` sang `order-service`; `order-service` tạo payment ở Payment-Wallet. |
| Admin voucher | Platform/freeship voucher | `/admin/vouchers/**` của service này (`VOUCHER_MANAGE` + 2FA). |

## 2. Cấu trúc bên trong

⏳ Repo chưa có code; cây package dưới đây là thiết kế.

```text
com.taca.checkout
├── checkout/             # CheckoutController, CheckoutSaga, preview
├── pricing/              # CheckoutCalculator, TaxExtractor, phân bổ phần dư lớn nhất
├── voucher/              # VoucherService, VoucherScopeResolver, seller/admin CRUD (voucher.md)
├── recalculation/        # InternalRecalculateController, tính lại khi huỷ đơn con
├── integrations/         # CartClient, OrderClient, ProductClient, InventoryClient, ShipmentClient, AuthUserClient
├── outbox/               # transactional outbox → voucher.events.v1
├── security/             # actor/shop scope, internal token, idempotency
└── observability/        # JSON log/trace/metric contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `CheckoutController` | Shipping fee, preview, place | Place bắt buộc `Idempotency-Key`. |
| `CheckoutCalculator` | Công thức §3.2 | Cài đặt **duy nhất**; preview, place và recalculate dùng chung. |
| `CheckoutSaga` | Các bước §3.3 và compensation | Không distributed transaction; mỗi bước idempotent. |
| `VoucherService` | Validate/hold/redeem/release | Atomic usage limit (xem `voucher.md`). |
| `InternalRecalculateController` | `POST /internal/checkout-groups/{id}/recalculate` | Chỉ `order-service` gọi (`X-Internal-Service-Token`). |
| `OrderClient` | `POST /internal/orders`, `GET /internal/orders?checkout_group_id=` | Idempotency-Key = `checkout_group_id`. |
| `OutboxPublisher` | `voucher.events.v1` | Retry 3, backoff 2s, rồi DLQ `checkout-service.outbox.dlq.v1`. |

### 2.1 Chuẩn observability dùng chung

- Log JSON stdout với field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST propagate `traceparent`/`tracestate` và `X-Request-ID`; Kafka headers giữ `traceparent`, `request_id`, `event_id`.
- Không log Authorization, token nội bộ, full address/phone/email, voucher secret hoặc toàn bộ payload giỏ/đơn.
- Chỉ log `checkout_group_id`, `order_id`, `shop_id`, `sku_id`, `voucher_id` theo allowlist; metric label không dùng `user_id/order_id`.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MySQL, Kafka.
- Error envelope `{error:{code,message,details,trace_id}}`.

## 3. Luồng xử lý

### 3.1 Checkout preview

```text
cart-service GET /internal/carts/{buyer}?item_ids=
     → Product validate active SKU/giá/tax_rate_bps
     → Inventory availability (read-only)
     → Shipment quote theo shop
     → validate voucher (không giữ lượt)
     → CheckoutCalculator (§3.2)
     → trả preview + warnings; không reserve, không ghi DB
```

Preview không đảm bảo giá/stock đến lúc place; response ghi `expires_at` ngắn và place phải tính lại.

### 3.2 Công thức tính tiền *(nguồn sự thật — mọi nơi khác tham chiếu về đây)*

Đây là phép tính quan trọng nhất của hệ thống. Preview, place checkout, invoice và hiển thị FE **phải dùng chung đúng một cài đặt**; lệch một đồng là lệch đối soát.

**Nguyên tắc bất di bất dịch:**

- Mọi số tiền là **số nguyên VND**. Cấm float/double ở mọi bước trung gian.
- Giá sản phẩm **đã bao gồm VAT** (chuẩn bán lẻ VN). Thuế được **tách ra để ghi hoá đơn**, không cộng thêm vào `grand_total`.
- Thứ tự áp dụng là cố định: shop voucher → platform voucher → freeship voucher → tách thuế. Quy tắc slot voucher ở `voucher.md` §3.2.

#### Bước 0 — Gom theo shop

Với mỗi shop `s` trong giỏ:

```
subtotal_s  = Σ (unit_price × quantity) của các item thuộc shop s
shipping_s  = phí ship của shop s, lấy từ Shipment quote
```

#### Bước 1 — Voucher SHOP (mỗi shop tối đa 1 mã)

Chỉ áp cho đúng shop sở hữu mã. Điều kiện: `subtotal_s >= voucher.min_order`.

```
PERCENT:  shop_discount_s = min( floor(subtotal_s × value / 100), max_discount_amount, subtotal_s − 1 )
FIXED:    shop_discount_s = min( value, subtotal_s − 1 )
```

`max_discount_amount` là `null` thì không có trần. `shop_discount_s` không bao giờ vượt `subtotal_s − 1` (giữ tối thiểu 1 VND tiền hàng cho Payment, xem Bước 5).

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
FREESHIP: freeship_discount = min( value, base_ship )   # discount_type=FREESHIP, value VND
```

Phân bổ về từng shop theo tỉ lệ `shipping_s / base_ship`, cùng phương pháp phần dư lớn nhất. `freeship_discount` không bao giờ vượt `base_ship` — không có chuyện ship âm.

#### Bước 4 — Tách thuế theo danh mục

Mỗi item mang `tax_rate` **snapshot tại thời điểm checkout**, lấy từ category của sản phẩm (Product Catalog cung cấp, xem §6.3). Vì giá đã gồm VAT, thuế được tách ngược ra:

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

**Ràng buộc từ Payment-Wallet (đã có code, `CreatePaymentOrderRequest`).** `POST /api/v1/payments` bắt buộc `amount > 0`, mỗi `orders[].amount > 0` và `0 <= orders[].shipping_fee < orders[].amount` (tức tiền hàng của mỗi đơn con `>= 1` VND). Checkout tính số tiền để luôn thoả điều kiện này; `order-service` gửi đúng các số đó sang Payment (`order-docs/docs/lld/order.md` §3.2), không đổi Payment:

```
goods_s    = subtotal_s − shop_discount_s − alloc_s        # tiền hàng sau giảm của đơn con
ship_net_s = shipping_s − freeship_alloc_s
grand_total_s = goods_s + ship_net_s

order-service gửi Payment: orders[].amount = grand_total_s, orders[].shipping_fee = ship_net_s, amount = Σ grand_total_s
```

- Bước 1 trần `shop_discount_s` ở `subtotal_s − 1` (không phải `subtotal_s`).
- Sau phân bổ Bước 2, nếu `alloc_s > subtotal_s − shop_discount_s − 1` thì hạ `alloc_s` về đúng mức đó và giảm `platform_discount` tương ứng (phần bị cắt không phân bổ lại). Kết quả: `goods_s >= 1` với mọi đơn con, nên `grand_total_s >= 1` và `ship_net_s < grand_total_s` luôn đúng.
- Voucher FREESHIP không cần trần thêm: `ship_net_s >= 0` và tiền hàng đã `>= 1`.
- Không có đơn con 0 đồng; checkout không bao giờ gửi `amount = 0` sang Payment.

Ví dụ chuẩn bên dưới không chạm trần nên số không đổi.

#### Ví dụ chuẩn *(dùng làm golden test)*

Giỏ: Shop A 800.000 (2 item: 500.000 + 300.000), Shop B 400.000. Ship 30.000/shop.
Voucher: `APPLE300K` (SHOP A, FIXED 300.000) · `TACA200K` (PLATFORM, FIXED 200.000) · `FREESHIPMAX` (scope `FREESHIP`, `discount_type=FREESHIP`, value 60.000).

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


### 3.3 Place checkout (saga)

```text
1. Validate JWT, Idempotency-Key, payment_method, address (Auth User), slot voucher.
2. cart-service: đọc item đã chọn (thiếu hết → CHECKOUT_CART_EMPTY).
3. Product: GET /products/{productId} cho từng product (§6.4) → giá SKU, SKU active, tax_rate_bps → snapshot.
4. Shipment quote theo shop → CheckoutCalculator (§3.2).
5. MySQL tx: tạo checkout_groups(PENDING) + checkout_group_orders (order_id sinh trước, UUIDv7)
            + giữ lượt voucher (voucher_redemptions HELD, used_count += 1).
6. Inventory reserve MỖI ĐƠN CON một lần (order_intent_id = order_id,
      Idempotency-Key = reserve-{order_id}), lưu reservation_id vào checkout_group_orders.
      một đơn fail → release các reservation đã giữ, nhả voucher (RELEASED),
      group FAILED, trả CHECKOUT_STOCK_UNAVAILABLE.
7. order-service POST /internal/orders (Idempotency-Key = checkout_group_id):
      gửi snapshot đầy đủ + reservation_id của từng đơn con + payment_method.
      order-service tạo đơn, invoice PENDING, gọi Payment-Wallet, COD thì commit kho.
      lỗi 4xx → release mọi reservation của group + nhả voucher, group FAILED.
      timeout/5xx → retry cùng key; vẫn không rõ → group RECONCILING (§3.5).
8. MySQL tx: group ORDERS_CREATED, redemption HELD → REDEEMED (gắn order_id), outbox voucher.redeemed.
9. cart-service checked-out (async qua outbox job, retry; lỗi không ảnh hưởng đơn).
10. Trả order(s), checkout_group_id, payment_next_action (lấy từ response order-service).
```

Trạng thái khởi tạo của đơn theo payment method do `order-service` quyết định (`order-docs/docs/lld/order.md` §3.1):

- **VNPAY**: đơn `PENDING_PAYMENT`, payment `PENDING`; reservation giữ `RESERVED`, `order-service` commit khi nhận `payment.succeeded`.
- **COD**: đơn `CONFIRMED` ngay, payment `PENDING_COD`; `order-service` commit reservation ngay trong lời gọi `POST /internal/orders`.

> **Vì sao COD không đi qua `PENDING_PAYMENT`:** với COD tiền chỉ thu khi giao hàng, nên `payment.succeeded` chỉ tới **sau** `DELIVERED`. Nếu để đơn COD ở `PENDING_PAYMENT` thì đơn không bao giờ tới `PROCESSING`/`SHIPPED`, bị job timeout huỷ và bị nhả kho khi hết `INVENTORY_RESERVATION_TTL`.

Payment-Wallet lỗi khi `order-service` tạo payment: đơn đã tạo vẫn tồn tại ở `PENDING_PAYMENT` (VNPAY) và sẽ bị job timeout huỷ nếu không thanh toán được; `order-service` trả `payment_next_action.type = "RETRY_LATER"`, Checkout trả `201` nguyên trạng cho FE. Xem `order-docs/docs/api/order.md` §3.7.

### 3.4 Tính lại voucher khi huỷ đơn con

`order-service` gọi `POST /internal/checkout-groups/{checkoutGroupId}/recalculate` sau khi đã huỷ đơn (`order-docs/docs/lld/order.md` §3.3). Checkout:

1. Đánh dấu các `checkout_group_orders` bị huỷ là `CANCELLED`.
2. Áp quy tắc ở `voucher.md` §3.3 bằng **cùng** `CheckoutCalculator` §3.2 trên các đơn còn lại.
3. Cập nhật `voucher_redemptions` (nhả lượt nếu voucher bị gỡ), ghi `voucher_adjustments` (trước/sau), outbox `voucher.released` khi có nhả.
4. Trả số tiền mới cho từng đơn còn lại; `order-service` áp vào đơn.

Idempotent theo header `Idempotency-Key` (`order-service` dùng `cancel-{order_id}`): gọi lặp trả cùng kết quả, không nhả `used_count` hai lần.

### 3.5 Reconcile group không rõ kết quả

Job quét `checkout_groups` ở `RECONCILING` quá 1 phút và **gọi lại `POST /internal/orders` với cùng `Idempotency-Key` và cùng body** (body dựng lại từ `checkout_group_orders` + `checkout_group_items`). `order-service` idempotent theo key nên:

- Đơn đã tạo → trả lại đúng response đã lưu (đủ `payment_next_action`, `payment_id`, `reservation_expires_at`). Checkout lưu response này vào `idempotency_keys.response_snapshot` của `PLACE_CHECKOUT` rồi hoàn tất bước 8–9 của §3.3. Buyer retry `POST /checkout` sau đó nhận đúng response này.
- Đơn chưa tạo → `order-service` tạo bình thường (vẫn còn trong TTL reservation).
- Vẫn timeout/5xx và đã quá `INVENTORY_RESERVATION_TTL` → hỏi `GET /internal/orders?checkout_group_id=`; `orders=[]` thì release mọi reservation của group (idempotent), nhả voucher, group `FAILED`.

Trong lúc group còn `RECONCILING`, buyer retry `POST /checkout` cùng key nhận `409 CHECKOUT_IN_PROGRESS` (FE chờ vài giây rồi hỏi lại).

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| `CHECKOUT_PREVIEW_TTL` | 5 phút | — | Chỉ warning/UX, không giữ stock. |
| `INVENTORY_RESERVATION_TTL` | 15 phút | ⏳ chưa có | Phải khớp Inventory. |
| `VND_MIN/MAX` | 1/999999999999 | — | Integer; không float. |
| `INTEGRATION_TIMEOUT` | 5s | ⏳ chưa có | Baseline cho Cart/Product/Inventory/Shipment/Auth. |
| `PRODUCT_DETAIL_CONCURRENCY` | 10 | ⏳ chưa có | Số lời gọi `GET /products/{productId}` song song (§6.4). |
| `ORDER_CREATE_TIMEOUT` | 8s | ⏳ chưa có | `order-service` gọi tiếp Payment; Gateway timeout route checkout là 10s. |
| `PAGE_SIZE_DEFAULT` | 20 | — | Max 100. |
| `OUTBOX_RETRY_COUNT` | 3 | — | Backoff 2s rồi DLQ `checkout-service.outbox.dlq.v1`. |
| `IDEMPOTENCY_RETENTION` | 24h | — | Place/voucher mutation/recalculate. |
| `RECONCILE_AFTER` | 1 phút | — | §3.5. |
| `CHECKOUT_INTERNAL_TOKEN` | — | ⏳ chưa có | Token `order-service` dùng khi gọi `/internal/**`. |
| `CART_SERVICE_BASE_URL`, `ORDER_SERVICE_BASE_URL` | — | ⏳ chưa có | Địa chỉ nội bộ. |
| `KAFKA_BOOTSTRAP_SERVERS` | — | ⏳ chưa có | Theo convention Java ở `CONTRACT-MATRIX.md` §8. |

Hằng số voucher ở `voucher.md` §4.

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `CheckoutGroupStatus` | `PENDING` (đã giữ voucher, đang reserve/tạo đơn), `ORDERS_CREATED`, `RECONCILING`, `FAILED`. |
| `GroupOrderStatus` | `ACTIVE`, `CANCELLED` (trên `checkout_group_orders`). |
| `PaymentMethod` | `VNPAY`, `COD` (khớp regex Payment-Wallet `VNPAY|COD`). |

`VoucherStatus`, `DiscountType`, `VoucherScope`, `RedemptionStatus` ở `voucher.md` §5. `OrderStatus`/`PaymentStatusProjection` thuộc `order-service`.

## 6. Event phát ra / lắng nghe

| Topic / nguồn | Hướng | Chi tiết |
|---|---|---|
| `voucher.events.v1` | Phát ra | `voucher.md` §6 |
| Cart (REST) | Gọi ra | `cart-docs/docs/api/cart.md` §3.5, §3.6 |
| Order (REST, đồng bộ) | Gọi ra | §6.2 |
| Inventory reserve/release (REST) | Gọi ra | §6.1 |
| Thuế suất Product Catalog (REST) | Gọi ra | §6.3 |
| Giá/trạng thái SKU từ Product Catalog (REST) | Gọi ra | §6.4 |
| Shipment quote (REST) | Gọi ra | `shipment-docs/docs/api/shipment.md` §3.2 |

Checkout không consume Kafka. Mọi event đi qua transactional outbox.

### 6.1 Contract — Inventory reserve request

Contract do Inventory sở hữu (`inventory-docs/docs/api/inventory.md` §3.2). Correlation ở **header**; item chỉ có `sku_id` + `quantity`.

```http
POST /internal/inventory/reservations
Idempotency-Key: reserve-order-01912f91
X-Internal-Service-Token: <token>
X-Request-ID: 01912f80-7a1b-7c12-9c55-8b1c34a6d921
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json
```

```json
{
  "order_intent_id": "order-01912f91",
  "expires_at": "2026-08-30T09:15:00Z",
  "items": [{ "sku_id": "sku-01912f33", "quantity": 2 }]
}
```

Response `201`: `{ "data": { "reservation_id", "status": "RESERVED", "expires_at", "items": [...] } }`. `order_intent_id` là `order_id` của đơn con: **mỗi đơn con một reservation**, vì `release` của Inventory nhả cả reservation chứ không chọn được item, và `commit` gắn đúng một `order_id`. Nhờ vậy huỷ đơn shop A không nhả kho của shop B. Checkout chỉ gọi `release` khi saga thất bại **trước** khi có đơn; sau đó `reservation_id` đi theo từng đơn và `order-service` gọi commit/release.

### 6.2 Contract — tạo đơn ở order-service

`POST /internal/orders` (`order-docs/docs/api/order.md` §3.7). Checkout gửi toàn bộ snapshot đã tính: `checkout_group_id`, `buyer`, `payment_method`, `address_snapshot`, và mỗi đơn con: `order_id`, `reservation_id`, `shop`, `items[]` (`order_item_id` do Checkout sinh trước, giá, quantity, `tax_rate_bps`, `tax_amount`, `discount_allocated`), `amounts` (subtotal, shop/platform/freeship discount, shipping, tax, grand_total), `applied_vouchers[]`. `order-service` **không** tính lại tiền; nó lưu đúng số Checkout gửi.

### 6.3 Contract — thuế suất từ Product Catalog

Thuế tính **theo danh mục**, mà danh mục do Product Catalog sở hữu. checkout-service không tự suy ra thuế suất và không lưu bảng thuế riêng.

Thuế suất đi kèm ngay trong response `GET /products/{productId}` mà Checkout **đã gọi sẵn** để re-read giá SKU (§6.4) — không thêm lượt gọi mạng nào:

```json
{
  "product_id": "product-01912f31",
  "primary_category_id": "category-01912f20",
  "tax_rate_bps": 1000
}
```

| Field | Kiểu | Nghĩa |
|---|---|---|
| `tax_rate_bps` | int | Thuế suất VAT theo basis point (10% = `1000`). Product Catalog **đã giải quyết xong** việc thừa kế theo cây danh mục, trả ra giá trị cuối cùng — checkout-service không phải leo cây. |

Quy tắc:

- Checkout **snapshot** `tax_rate_bps` vào từng item gửi sang `order-service` (lưu ở `order_items`) tại thời điểm đặt hàng. Admin đổi thuế suất danh mục về sau **không** hồi tố đơn cũ — hoá đơn đã phát hành phải bất biến.
- Thiếu `tax_rate_bps` trong response (product cũ chưa migrate) → coi như `0`, ghi log cảnh báo, **không** chặn checkout. Chặn đặt hàng vì thiếu cấu hình thuế là đánh đổi sai.
- Giá đã gồm VAT nên thuế **không** cộng thêm vào `grand_total`; xem §3.2 bước 4.

### 6.4 Contract — giá và trạng thái SKU từ Product Catalog

Batch `GET /products?product_ids=` của Catalog chỉ trả **thẻ sản phẩm**, không có `skus[]` (đã kiểm code `ProductCardDto`, `Product-Catalog-Service@develop` `6572ece`). Giá ở thẻ là giá của product, không phải giá riêng từng SKU, nên **không dùng để chốt tiền**.

Checkout gọi `GET /products/{productId}` (`product-catalog-docs/docs/api/product-catalog.md` §3.1, đã có code) cho **mỗi `product_id` khác nhau** trong các item được chọn, song song tối đa `PRODUCT_DETAIL_CONCURRENCY`, không cache ở bước place. Không đổi contract Catalog.

| Field Checkout cần | Lấy từ `ProductDetail` | Ghi chú |
|---|---|---|
| Product còn bán | `status = ACTIVE`; `404 PRODUCT_NOT_FOUND` = không còn public | Không đạt → `409 CHECKOUT_PRODUCT_UNAVAILABLE`. |
| SKU còn bán | `skus[]` có `sku_id` của item | Catalog chỉ trả SKU active; thiếu SKU → `409 CHECKOUT_PRODUCT_UNAVAILABLE`. |
| Đơn giá | `skus[].price.sale_price` | Catalog đã áp `price_override` của SKU (code `CatalogQueryService`); `≤ 0` → coi như SKU không bán được. Khác `unit_price_snapshot` của cart → warning/`409 CHECKOUT_PRICE_CHANGED` như hiện tại. |
| Nhãn biến thể | `skus[].attributes` (ghép `label` từ `attributes[]` của product) | Thành `variant_label_snapshot`. |
| Mã SKU seller | `skus[].seller_sku` | Thành `seller_sku_snapshot`. |
| Thuế suất | `tax_rate_bps` | §6.3. |

Endpoint batch theo SKU ở Catalog là tối ưu để sau, không chặn v1.


## 7. Mã lỗi

⏳ Chưa có code: mọi mã dưới đây là thiết kế. Đổi từ prefix `ORDER_` sang `CHECKOUT_` khi tách service (2026-11-01).

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `CHECKOUT_INVALID_INPUT` | 400 | Payload/query sai. |
| `CHECKOUT_UNAUTHENTICATED` | 401 | Thiếu login hoặc token nội bộ. |
| `CHECKOUT_FORBIDDEN` | 403 | Không thuộc buyer/shop scope, thiếu permission. |
| `CHECKOUT_CART_EMPTY` | 409 | Không còn item hợp lệ được chọn. |
| `CHECKOUT_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active. |
| `CHECKOUT_PRICE_CHANGED` | 409 | Giá hiện tại khác preview; cần xác nhận lại. |
| `CHECKOUT_STOCK_UNAVAILABLE` | 409 | Inventory reserve thất bại. |
| `CHECKOUT_RESERVATION_EXPIRED` | 409 | Reservation hết TTL trước khi tạo đơn. |
| `CHECKOUT_VOUCHER_INVALID` | 400 | Code/scope/time/value không hợp lệ. |
| `CHECKOUT_VOUCHER_SLOT_CONFLICT` | 400 | Hai mã cùng `scope`. |
| `CHECKOUT_VOUCHER_CODE_EXISTS` | 409 | Trùng `code` trong cùng shop/platform. |
| `CHECKOUT_VOUCHER_CODE_LOCKED` | 409 | Đổi `code` khi voucher đã có redemption. |
| `CHECKOUT_VOUCHER_LIMIT_REACHED` | 409 | Hết lượt dùng. |
| `CHECKOUT_ADDRESS_INVALID` | 400 | Địa chỉ không thuộc buyer/sai. |
| `CHECKOUT_GROUP_NOT_FOUND` | 404 | Recalculate với group không tồn tại. |
| `CHECKOUT_IN_PROGRESS` | 409 | Retry `POST /checkout` khi group còn `RECONCILING` (§3.5). |
| `CHECKOUT_IDEMPOTENCY_CONFLICT` | 409 | Key dùng lại với payload khác. |
| `CHECKOUT_DEPENDENCY_UNAVAILABLE` | 503 | Cart/Product/Inventory/Shipment/Order/Auth timeout hoặc down. |
| `CHECKOUT_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

Mã lỗi của service phụ thuộc không pass-through ra buyer, trừ khi ghi rõ ở endpoint.

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Đã chốt (2026-11-01): voucher ở chung `checkout-service`; Checkout gọi `order-service` bằng REST đồng bộ; `order-service` gọi lại Checkout khi huỷ đơn con. | — | Đã đóng |
| 2 | Checkout không gọi Payment-Wallet; `order-service` là internal caller `ORDER_COMMERCE` duy nhất của Payment (code Payment hiện chỉ có enum này). | Nếu sau này Checkout cần gọi Payment, Payment phải thêm `InternalService` mới. | Architecture |
| 3 | Reserve kho trước khi tạo đơn; không distributed transaction, dùng saga + reconcile §3.5. | Ảnh hưởng compensation/timeout. | Order/Inventory owner |
| 4 | Multi-shop checkout split thành đơn con theo shop. | Ảnh hưởng voucher scope, shipment, invoice. | Product owner |
| 5 | Shipping fee dùng Shipment quote (carrier mặc định `GHN`, chỉ `MOCK` chạy end-to-end). | Phí thật cần adapter carrier. | Shipment owner |
| 6 | Thuế lấy theo danh mục từ Product Catalog; tax authority/provider chưa chốt. | Ảnh hưởng hoá đơn. | Finance owner |
| 7 | Repo `checkout-service` chưa tạo. | Header cập nhật khi có code. | Tech lead |
| 8 | Đơn VNPAY đã tạo nhưng Payment lỗi trả `RETRY_LATER`; chưa có endpoint cho buyer tạo lại payment. | Buyer phải đặt lại nếu đơn bị huỷ do timeout. | Product/Payment owner |
