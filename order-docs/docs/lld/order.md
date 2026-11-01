# LLD — Order Service

> Nguồn: `docs/api/order.md` · `docs/db/order.md` · `docs/test/order.md` · HLD `EcommercePlatform-v4(6).excalidraw` · Penpot `New File 1.penpot.zip` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`, chỉ có `README.md`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 `orderdb` · Kafka outbox · REST với Checkout/Inventory/Payment/Shipment

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

Order-Commerce cũ được tách làm 3 service ngày 2026-11-01: `cart-service` (giỏ), `checkout-service` (tính tiền + voucher) và `order-service` (file này: đơn + invoice).

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Order aggregate và state machine; tạo đơn từ `checkout-service`; buyer order read/cancel; seller order read/fulfill/bulk-accept/export/cancel; invoice; payment projection; event `order.events.v1`/`invoice.events.v1`; xử lý event Payment/Inventory/Shipment. |
| Database | MySQL 8.4 `orderdb`: `orders`, `order_items`, `order_vouchers`, `invoices`, `order_status_history`, `voucher_recalc_requests`, `processed_events`, `outbox_events`, `idempotency_keys`. |
| Payment | Payment-Wallet sở hữu giao dịch. `order-service` là **internal caller duy nhất** của Payment (`InternalService.ORDER_COMMERCE` trong code Payment): tạo payment, refund. Lưu payment projection. |
| Inventory | Inventory sở hữu stock. `checkout-service` reserve; `order-service` **commit/release** khi đơn đã tồn tại (đúng chốt 2026-09-18 "Order là bên gọi commit/release"). |
| Shipment | Shipment sở hữu carrier/tracking. `order-service` gọi carrier-quotes, tạo và huỷ shipment; lưu shipment projection. |
| Checkout | `checkout-service` sở hữu tiền và voucher; `order-service` lưu đúng số tiền nhận được, không tự tính, và gọi `recalculate` khi huỷ đơn con. |
| Không thuộc service | Giỏ, tính tiền, voucher, Auth/KYC, product CRUD, inventory ledger, payment gateway, carrier, rating/message/notification. |

### 1.2 Boundary

```text
mfe-buyer/mfe-seller ──Gateway /orders/**, /seller/orders/**──► order-service
checkout-service ──REST POST /internal/orders──────────────────► order-service
                                 order-service
                                 ├─ MySQL orderdb
                                 ├─ REST Payment-Wallet: POST /api/v1/payments, /payments/{id}/refunds
                                 ├─ REST Inventory: reservations/{id}/commit|release
                                 ├─ REST Shipment: carrier-quotes, shipments, shipments/{id}/cancel
                                 ├─ REST checkout-service: /internal/checkout-groups/{id}/recalculate
                                 ├─ Kafka outbox → order.events.v1, invoice.events.v1
                                 └─ Kafka consume ← payment.events.v1, shipment.events.v1, inventory.events.v1, user.events.v1
```

- Order snapshot bất biến sau khi tạo, trừ điều chỉnh tiền từ recalculate (có audit).
- Topic, tên event và payload giữ **nguyên** như thời Order-Commerce: Notification, Shipment, Rating-Comment, Message, Inventory, Payment không phải đổi gì.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Order | Order + Invoice | Gom trong `order-service`. |
| Buyer Orders | List/detail/status/cancel/invoice | Buyer scope từ JWT; cancel trước shipping. |
| Seller Orders | List/detail/fulfill/cancel/invoice/export/bulk-accept | Shop scope; seller không đổi payment/inventory trực tiếp. |
| Payment | VNPAY QR hoặc COD | `order-service` tạo payment ngay khi tạo đơn. |

## 2. Cấu trúc bên trong

⏳ Repo chưa có code; cây package dưới đây là thiết kế. Repo hiện có tên `order-commerce-service` (chỉ README); đề xuất đổi tên thành `order-service`.

```text
com.taca.order
├── order/                # OrderController, SellerOrderController, OrderStateMachine, query
├── creation/             # InternalOrderController: POST /internal/orders
├── cancellation/         # cancel buyer/seller/system, VoucherRecalcJob
├── fulfillment/          # accept/ship, carrier façade, bulk-accept, export
├── invoice/              # invoice number/metadata/PDF reference
├── integrations/         # PaymentClient, InventoryClient, ShipmentClient, CheckoutClient
├── consumer/             # payment/shipment/inventory/user event handlers + processed_events
├── outbox/               # transactional outbox → order.events.v1, invoice.events.v1
├── security/             # actor/shop ownership, internal token, idempotency
└── observability/        # JSON log/trace/metric contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `InternalOrderController` | Tạo đơn từ checkout (§3.2) | `X-Internal-Service-Token`; Idempotency-Key = `checkout_group_id`. |
| `OrderController` | Buyer list/detail/status/cancel/invoice | Chỉ own orders. |
| `SellerOrderController` | Shop list/detail/fulfill/cancel/invoice/export/carriers | Filter theo `X-User-Shop-Scope`. |
| `OrderStateMachine` | `PENDING_PAYMENT/CONFIRMED/PROCESSING/SHIPPED/DELIVERED/CANCELLED` | Không bypass transition. |
| `PaymentClient` | Create/refund | Request đúng `CreatePaymentRequest` của Payment (đã có code). |
| `InventoryClient` | Commit/release | Idempotent; lỗi không đánh dấu đơn sai trạng thái. |
| `VoucherRecalcJob` | Gọi `checkout-service` recalculate sau huỷ | Retry tới khi thành công; key `cancel-{order_id}`. |
| `OutboxPublisher` | Publish event | Retry 3/backoff 2s rồi DLQ `order-service.outbox.dlq.v1`. |

### 2.1 Chuẩn observability dùng chung

- Log JSON stdout với field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST propagate `traceparent`/`tracestate` và `X-Request-ID`; Kafka headers giữ `traceparent`, `request_id`, `event_id`.
- Không log Authorization, token nội bộ, full address/phone/email, payment credential, VNPAY signature hoặc full order payload.
- Chỉ log `order_id`, `checkout_group_id`, `shop_id`, `sku_id` theo allowlist; metric label không dùng `user_id/order_id`.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MySQL, Kafka.
- Error envelope `{error:{code,message,details,trace_id}}`.

## 3. Luồng xử lý

### 3.1 Order lifecycle

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
| `→ PENDING_PAYMENT` | `POST /internal/orders` từ `checkout-service` (VNPAY) | Order tạo với phương thức trả trước; reservation `RESERVED`, chưa commit. |
| `→ CONFIRMED` | `POST /internal/orders` từ `checkout-service` (COD) | Order tạo với `method=COD`; commit reservation ngay; payment projection `PENDING_COD`. |
| `PENDING_PAYMENT → CONFIRMED` | Payment event | `payment.succeeded` từ Payment-Wallet; commit reservation; payment projection `SUCCESS`. |
| `PENDING_PAYMENT → CANCELLED` | System/buyer timeout hoặc `payment.expired` | **Chỉ áp dụng cho đơn trả trước.** Release reservation; `PAYMENT_TIMEOUT`. Đơn COD không bao giờ ở trạng thái này nên không bị job này chạm tới. |
| `CONFIRMED → PROCESSING` | Seller/system | Seller accept đơn (hàng đợi "Chờ xác nhận" gồm **cả** đơn VNPAY đã trả và đơn COD). |
| `PROCESSING → SHIPPED` | Seller/Shipment | Shipment created/tracking valid. |
| `SHIPPED → DELIVERED` | Shipment event (`shipment.delivered`) | Carrier confirms delivered. Với COD, đây là điểm kích hoạt capture tiền ở Payment-Wallet — **không** đổi `OrderStatus` thêm lần nữa. |
| `SHIPPED` (giữ nguyên) | Shipment event (`shipment.failed`) | Carrier báo giao thất bại: order giữ `SHIPPED`, ghi timeline; refund/return do Payment/Support workflow xử lý (không tự huỷ/hiển thị mới). |
| `CONFIRMED/PROCESSING → CANCELLED` | Buyer/seller/system theo policy | Buyer chỉ cancel trước shipping. VNPAY: release reservation đã commit + refund. COD: release/hoàn kho, không refund vì chưa thu tiền. |
| `CANCELLED/DELIVERED` | Không reopen | Tạo refund/return workflow ở Payment/Support nếu cần. |

Ràng buộc bổ sung:

- `PAID` **không còn là một `OrderStatus`**. Trước đây trạng thái này vừa mang nghĩa "đã thu tiền" vừa mang nghĩa "được phép giao", làm COD không có đường đi. Nghĩa "đã thu tiền" nay đọc ở `payment_status = SUCCESS`; nghĩa "được phép giao" là `CONFIRMED`.
- Event `order.paid` vẫn giữ nguyên ngữ nghĩa **tiền đã về** và vẫn phát khi `payment.succeeded` — với VNPAY là lúc vào `CONFIRMED`, với COD là sau `DELIVERED`. Consumer nào cần "đơn đã chốt, chuẩn bị giao" phải dùng `order.confirmed`, không dùng `order.paid`.
- Không service nào được suy ra quyền fulfillment từ `payment_status`; chỉ đọc `OrderStatus`.

### 3.2 Tạo đơn từ checkout

`POST /internal/orders` (`docs/api/order.md` §3.7), gọi bởi `checkout-service` sau khi đã reserve kho (`checkout-docs/docs/lld/checkout.md` §3.3).

1. Kiểm `X-Internal-Service-Token` và `Idempotency-Key` (= `checkout_group_id`). Key đã xử lý → trả response cũ.
2. MySQL transaction: tạo `orders` (dùng `order_id` do Checkout sinh), `order_items`, `order_vouchers`, `invoices` (`PENDING`), `order_status_history`, outbox `order.created` (+ `order.confirmed` nếu COD).
   - VNPAY: `status=PENDING_PAYMENT`, `payment_status=PENDING`.
   - COD: `status=CONFIRMED`, `payment_status=PENDING_COD`.
3. Sau commit, gọi Payment-Wallet `POST /api/v1/payments` (Idempotency-Key = `checkout_group_id`) với đúng shape `CreatePaymentRequest` đã có code:

   ```json
   {
     "checkout_group_id": "group-01912f90",
     "buyer_user_id": "user-01912f10",
     "method": "VNPAY",
     "amount": 700000,
     "currency": "VND",
     "orders": [
       { "order_id": "order-01912f91", "shop_id": "shop-01912f31", "amount": 388889, "shipping_fee": 0 },
       { "order_id": "order-01912f92", "shop_id": "shop-01912f32", "amount": 311111, "shipping_fee": 0 }
     ]
   }
   ```

   `orders[].amount = grand_total` của đơn con, `orders[].shipping_fee = shipping_fee − freeship_discount`, `amount = Σ`. Checkout đã bảo đảm `amount > 0` và `shipping_fee < amount` (`checkout-docs/docs/lld/checkout.md` §3.2). Payment lỗi → lưu `payment_id = null`, trả `payment_next_action.type = RETRY_LATER`; job retry tạo payment với cùng key, đơn VNPAY vẫn bị huỷ nếu quá hạn.
4. COD: gọi Inventory `POST /internal/inventory/reservations/{reservationId}/commit` ngay cho reservation **của từng đơn** (`body.order_id` = đơn đó). Lỗi → retry nền (idempotent); đơn vẫn `CONFIRMED`.
5. Trả `orders[]` (id, `order_number`, status, payment status), `payment_next_action`, `reservation_expires_at`.

### 3.3 Huỷ đơn và tính lại voucher

1. Buyer (`PATCH /orders/{id}/cancel`), seller (`POST /seller/orders/{id}/cancel`) hoặc hệ thống (`payment.expired`, timeout) huỷ theo state machine §3.1.
2. MySQL transaction: `status=CANCELLED`, `order_status_history`, outbox `order.cancelled`, tạo `voucher_recalc_requests` (`PENDING`) nếu đơn có `checkout_group_id`.
3. Compensation sau commit: Inventory `release` reservation của chính đơn bị huỷ (mỗi đơn một reservation, không ảnh hưởng đơn khác trong group); VNPAY đã capture → Payment `POST /api/v1/payments/{paymentId}/refunds`; shipment còn `CREATED` → Shipment tự huỷ khi nhận `order.cancelled` (consumer đã có code).
4. `VoucherRecalcJob` gọi `checkout-service` `POST /internal/checkout-groups/{checkoutGroupId}/recalculate` (Idempotency-Key `cancel-{order_id}`) với `cancelled_order_ids` và `remaining_orders[]` kèm `payment_captured` (= `payment_status = SUCCESS`).
5. Ghi kết quả cho các đơn còn lại: v1 số tiền **không đổi** (`grand_total`, `amounts`, `order_items`, `order_vouchers` giữ nguyên; `checkout-docs/docs/lld/voucher.md` §3.3). Chỉ lưu `absorbed_amount` và ghi `order_status_history` loại `VOUCHER_ADJUSTED` khi `adjustment ≠ NONE`. Vì vậy `cod_amount` gửi Shipment lúc ship và `amount` Payment-Wallet đã chốt luôn bằng `grand_total`.
6. Checkout down → request giữ `PENDING`, retry backoff; không chặn việc huỷ.

### 3.4 Fulfill và shipment

- `from_address_snapshot` cho cả carrier quote và SHIP: đọc auth-user `GET /api/v1/internal/shops/{shopId}/pickup-address` (`auth-user-docs/docs/api/auth-user.md` §3.52), header `X-Internal-Service-Token`.
- `GET /seller/orders/{orderId}/shipment/carriers`: façade, kiểm đơn thuộc shop, `CONFIRMED` hoặc `PROCESSING`, chưa có shipment, rồi gọi Shipment `POST /api/v1/internal/shipping/carrier-quotes`.
- `ACCEPT`: `CONFIRMED → PROCESSING`.
- `SHIP`: gọi Shipment `POST /api/v1/internal/shipments` với `carrier` seller chọn và `order_code = order_number`; thành công → `SHIPPED`.
- `DELIVERED` chỉ đến từ `shipment.delivered`.

### 3.5 Invoice

- Invoice có số unique, total/tax breakdown snapshot và `pdf_url` nullable; tạo `PENDING` cùng transaction tạo đơn.
- `invoice.issued` phát khi chuyển `ISSUED`. Gọi invoice khi còn `PENDING` → `409 ORDER_INVOICE_NOT_READY`.
- Recalculate v1 không đổi số tiền đơn còn lại nên invoice không phải cập nhật hay phát hành lại (§8 mục 6).

Không dùng distributed transaction giữa MySQL/Inventory/Payment/Checkout; dùng idempotency, outbox, compensation và reconcile.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| `PAGE_SIZE_DEFAULT` | 20 | — | Max 100. |
| `ORDER_NUMBER_MAX_LENGTH` | 32 | — | Unique; cũng là `order_code` gửi Shipment (≤32). |
| `ORDER_EXPORT_MAX_ROWS` | 10000 | — | Vượt trả `400 ORDER_EXPORT_TOO_LARGE`. |
| `ORDER_EXPORT_URL_TTL` | 30 phút | — | Signed download URL. |
| `SELLER_BULK_ACCEPT_MAX_ORDERS` | 100 | — | `POST /seller/orders/bulk-accept`. |
| `PAYMENT_TIMEOUT` | 15 phút | ⏳ chưa có | Job huỷ đơn `PENDING_PAYMENT`; bằng `INVENTORY_RESERVATION_TTL`. |
| `OUTBOX_RETRY_COUNT` | 3 | — | Backoff 2s rồi DLQ `order-service.outbox.dlq.v1`. |
| `INTEGRATION_TIMEOUT` | 5s | ⏳ chưa có | Payment/Inventory/Shipment/Checkout. |
| `IDEMPOTENCY_RETENTION` | 24h | — | Create/cancel/fulfill/bulk-accept. |
| `AUDIT_RETENTION` | 365 ngày | — | — |
| `ORDER_INTERNAL_TOKEN` | — | ⏳ chưa có | Token `checkout-service` dùng khi gọi `/internal/**`. |
| `CHECKOUT_SERVICE_BASE_URL` | — | ⏳ chưa có | Địa chỉ nội bộ. |
| `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_CONSUMER_GROUP_ID` | — | ⏳ chưa có | Convention Java ở `CONTRACT-MATRIX.md` §8. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `OrderStatus` | `PENDING_PAYMENT`, `CONFIRMED`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. Trạng thái fulfillment; `PENDING_PAYMENT` chỉ cho phương thức trả trước (VNPAY). |
| `PaymentStatusProjection` | `PENDING`, `PENDING_COD`, `SUCCESS`, `FAILED`, `EXPIRED`, `REFUNDED`, `PARTIALLY_REFUNDED`. `EXPIRED` → đơn `PENDING_PAYMENT` chuyển `CANCELLED`. `PARTIALLY_REFUNDED` không đổi `OrderStatus`. |
| `InvoiceStatus` | `PENDING`, `ISSUED`, `VOIDED`. |
| `CancellationReason` | `BUYER_REQUEST`, `SELLER_REQUEST`, `PAYMENT_TIMEOUT`, `STOCK_FAILURE`, `SYSTEM`. |
| `HistoryEntryType` | `STATUS_CHANGED`, `AMOUNT_ADJUSTED`, `DELIVERY_FAILED`. |

Order state là source of truth của `order-service`; payment/shipment/inventory statuses chỉ là projection, client không tự set.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `order.events.v1` | `order.created` | order/group, buyer, shop, items snapshot, total, payment method |
| `order.events.v1` | `order.confirmed` | order/group, `order_code`, `display_name`, `buyer` (`user_id`, `email`), shop, payment method, confirmed_at |
| `order.events.v1` | `order.paid` | order, `order_code`, `amount`, `buyer` (`user_id`, `email` có thể `null`), payment reference, paid_at |
| `order.events.v1` | `order.processing` | order, shop |
| `order.events.v1` | `order.shipped` | order, shipment reference |
| `order.events.v1` | `order.delivered` | order snapshot + `buyer` (`user_id`, `email`) + `items[]` (`product_id`, `sku_id`) + `delivered_at` — `buyer`/`items[]` là bắt buộc để Rating-Comment dựng eligibility |
| `order.events.v1` | `order.cancelled` | order, `order_code`, `buyer` (`user_id`, `email`), reason, cancelled_at, compensation action |
| `invoice.events.v1` | `invoice.issued` | invoice/order/number, `order_code`, `buyer` (`user_id`, `email`), total, `invoice_url` |

Phân biệt bắt buộc giữa hai event dễ nhầm:

| Event | Nghĩa | VNPAY phát lúc | COD phát lúc | Dùng cho |
|---|---|---|---|---|
| `order.confirmed` | Đơn đã chốt, **được phép giao** | Khi nhận `payment.succeeded` | Ngay tại checkout | Fulfillment, email xác nhận đơn, hàng đợi seller |
| `order.paid` | **Tiền đã về** | Cùng lúc `order.confirmed` | Sau `DELIVERED`, khi capture tiền mặt | Đối soát, ledger, settlement |

Với VNPAY hai event trùng thời điểm; với COD chúng cách nhau cả vòng đời giao hàng. Consumer nào cần "đơn sẵn sàng giao" mà dùng `order.paid` sẽ bỏ sót toàn bộ đơn COD.

### 6.2 Payload chi tiết — `order.confirmed` / `order.paid` / `order.cancelled` / `invoice.issued` / `order.delivered`

Field `buyer` **đã chốt** cho các consumer cần recipient (đặc biệt Notification Service —
xem `notification-docs/docs/lld/notification.md` §6.2) — **gồm** `order.delivered`: `buyer.user_id` bắt buộc, `buyer.email`
optional (`null` cho phép ở event chỉ phát `IN_APP`, ví dụ `order.paid`). Field khác trong `payload`
lấy tên đúng theo response `GET /orders/{orderId}` (`docs/api/order.md` §3.1) và bảng `invoices` (`docs/db/order.md` §3.4).

**Field cho template Notification (bắt buộc, top-level trong `payload`).** Notification chỉ lấy key
**top-level** của `payload` có trong placeholder template (`TemplateService.pickAllowedFields`,
template seed ở `notification-service/src/infrastructure/persistence/seed.ts`), không đổi tên hay đọc
field lồng. Order chưa có code nên phát đúng tên placeholder mà Notification đang dùng:

| Event | Template | Field top-level bắt buộc | Lấy từ |
|---|---|---|---|
| `order.confirmed` | `order-success-v1` | `order_code`, `display_name` | `order_code` = `orders.order_number`; `display_name` = `recipient_name` trong `address_snapshot` |
| `order.paid` | `payment-received-v1` | `order_code`, `amount` | `amount` = số tiền đã thu cho đơn này (VND, integer) = `orders.grand_total` |
| `order.cancelled` | `order-cancelled-v1` | `order_code` | `orders.order_number` |
| `invoice.issued` | `invoice-issued-v1` | `order_code`, `invoice_url` | `invoice_url` = `invoices.pdf_url` (tên field trong event là `invoice_url`, không phải `pdf_url`) |

`order_number` vẫn giữ trong payload cho consumer khác; `order_code` mang **cùng giá trị**, chỉ để khớp template.

`order.confirmed`:

```json
{
  "order_id": "order-01912f91",
  "order_number": "TC-20260830-0001",
  "order_code": "TC-20260830-0001",
  "display_name": "Nguyễn Văn A",
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
  "order_number": "TC-20260830-0001",
  "order_code": "TC-20260830-0001",
  "amount": 1094000,
  "buyer": { "user_id": "user-01912f10", "email": null },
  "payment_reference": "payment-01912f95",
  "paid_at": "2026-08-30T09:05:00Z"
}
```

`order.cancelled`:

```json
{
  "order_id": "order-01912f91",
  "order_number": "TC-20260830-0001",
  "order_code": "TC-20260830-0001",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "reason": "Hết hàng",
  "cancelled_at": "2026-08-30T10:00:00Z"
}
```

> `compensation action` (nhắc ở bảng §6.1) chưa có schema field cụ thể — phụ thuộc seller
> cancel/refund policy chưa chốt (xem §8 mục 5), nên chưa đưa vào ví dụ JSON để tránh set field
> giả không đúng thực tế.

`invoice.issued`:

```json
{
  "invoice_id": "invoice-01912fb0",
  "order_id": "order-01912f91",
  "order_code": "TC-20260830-0001",
  "invoice_number": "INV-20260830-0001",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "total_amount": 1094000,
  "invoice_url": "https://cdn.taca.vn/invoices/invoice-01912fb0.pdf",
  "issued_at": "2026-08-30T09:10:00Z"
}
```

`order.delivered`:

```json
{
  "event_id": "01912f99-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "event_type": "order.delivered",
  "occurred_at": "2026-08-31T10:20:00Z",
  "aggregate_type": "ORDER",
  "aggregate_id": "order-01912f91",
  "payload": {
    "order_id": "order-01912f91",
    "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
    "items": [
      { "product_id": "product-01912f31", "sku_id": "sku-01912f33" },
      { "product_id": "product-01912f41", "sku_id": "sku-01912f43" }
    ],
    "delivered_at": "2026-08-31T10:20:00Z"
  }
}
```

### 6.3 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Payment-Wallet | `payment.succeeded`, `payment.failed`, `payment.refunded` | Update payment projection, transition order, commit/release Inventory. |
| Payment-Wallet | `payment.expired` | Order ở `PENDING_PAYMENT` chuyển sang `CANCELLED` (payment_status projection = `EXPIRED`); không hoàn kho nếu Inventory tự xử lý expire riêng qua `inventory.reservation.expired`. |
| Inventory | `inventory.reservation.created`, `inventory.reservation.committed`, `inventory.reservation.released`, `inventory.reservation.expired` | Reconcile order intent/reservation; không tự sửa quantity. Tên event lấy đúng catalog của Inventory — **không có** `reservation.confirmed`/`reservation.rejected`/`stock_committed`: reserve thành công/thất bại là kết quả **đồng bộ** của `POST /internal/inventory/reservations`, event chỉ dùng để reconcile khi mất response. |
| Shipment | `shipment.created`, `shipment.status_changed` | Update shipment projection/order status. |
| Shipment | `shipment.delivered` | Order chuyển `SHIPPED → DELIVERED`; cập nhật `delivered_at`. |
| Shipment | `shipment.failed` | Ghi nhận delivery failed vào timeline/shipment projection; **giữ** `OrderStatus = SHIPPED` (enum không có FAILED) — xử lý tiếp qua refund/return workflow ở Payment/Support, không tự đổi order status. |
| Auth User | `user.status_changed` (mock policy) | Chặn hành động account mới nếu suspended; không đổi order history. |

> **Lệch với code producer (2026-10-31, xem `CONTRACT-MATRIX.md`):** Payment-Wallet (`payment-wallet-service@main`) hiện chỉ route `payment.created` và `wallet.allocated` ra Kafka; `payment.succeeded/failed/expired/refunded` chỉ nằm trong outbox. Payload Payment-Wallet là record camelCase phẳng (`eventId`, `occurredAt`, `paymentId`…), **không** có envelope `event_type`/`payload` như các topic khác. Format này là contract đã triển khai: consumer của `order-service` phải parse đúng format record của Payment-Wallet (`payment-wallet-docs/docs/lld/payment-wallet.md` §6.1, `CONTRACT-MATRIX.md` §8.4).

## 7. Mã lỗi

⏳ Chưa có code: mọi mã dưới đây là thiết kế. Mã `CART_*` và `CHECKOUT_*` (trước là `ORDER_CART_*`, `ORDER_VOUCHER_*`…) chuyển sang service tương ứng.

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `ORDER_INVALID_INPUT` | 400 | Payload/query sai. |
| `ORDER_UNAUTHENTICATED` | 401 | Thiếu login hoặc token nội bộ. |
| `ORDER_FORBIDDEN` | 403 | Không thuộc buyer/shop scope. |
| `ORDER_NOT_FOUND` | 404 | Đơn không tồn tại hoặc không thuộc actor (anti-enumeration). |
| `ORDER_STATE_INVALID` | 409 | Transition không hợp lệ. |
| `ORDER_CANCEL_NOT_ALLOWED` | 409 | Đã shipping hoặc policy không cho huỷ. |
| `ORDER_VERSION_CONFLICT` | 409 | `version` trong body cũ. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Key dùng lại với payload khác. |
| `ORDER_EXPORT_TOO_LARGE` | 400 | Vượt `ORDER_EXPORT_MAX_ROWS`. |
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice chưa issued. |
| `ORDER_DEPENDENCY_UNAVAILABLE` | 503 | Shipment/Inventory down ở thao tác đồng bộ (fulfill, carriers). |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |
| `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` | 400 | Pass-through nguyên trạng từ Shipment (`docs/api/order.md` §3.4). |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Đã chốt (2026-11-01): `order-service` gồm order + invoice; nhận tạo đơn qua REST từ Checkout; gọi Checkout khi huỷ đơn con. | — | Đã đóng |
| 2 | `order-service` là caller `ORDER_COMMERCE` duy nhất của Payment-Wallet; transport service-auth của Payment (`InternalCallerContext`) vẫn là contract mở. | Khi chốt transport, cấu hình cho `order-service`. | Architecture |
| 3 | Repo `order-commerce-service` giữ làm repo của `order-service` (đề xuất đổi tên). | Header doc cập nhật khi đổi tên. | Tech lead |
| 4 | Một checkout nhiều shop split thành đơn con theo shop. | Ảnh hưởng voucher, shipment, invoice. | Product owner |
| 5 | Buyer huỷ được trước shipping; seller cancel policy cần rule chi tiết. Return/dispute **ngoài v1** — service `dispute` (v1.1); nếu cần sớm làm module trong `order-service` (quyết định 2026-09-18). | Ảnh hưởng refund/stock release. | Product/Finance |
| 6 | Đã chốt (2026-11-01, đóng S11): recalculate không đổi số tiền đơn còn lại vì Payment-Wallet không sửa được `amount`; mã không còn đủ điều kiện thì sàn chịu (`absorbed_amount`). Hạn chế còn lại: Payment-Wallet chưa có API huỷ `PaymentOrder` COD của **đơn bị huỷ**, nên dòng đó nằm ở `PENDING_COD` (không có `shipment.delivered`). | Đối soát Payment thấy dòng COD treo của đơn đã huỷ; không lệch tiền vì chưa capture. | Payment owner |
| 7 | Thuế: snapshot theo danh mục từ Checkout; tax authority/provider chưa chốt; invoice PDF provider chưa chốt. | Ảnh hưởng invoice compliance, `pdf_url`. | Finance/DevOps |
