# LLD — Payment-Wallet Service

> Nguồn: `docs/api/payment-wallet.md` · `docs/db/payment-wallet.md` · ADR-001…004 trong service repo · Code: `ecommerce-taca/payment-wallet-service` @ `main` (`859b7b1`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — payment/refund/webhook/COD capture, allocation/ledger, admin finance đã có code; payout chỉ có domain/service (chưa có API); outbox chỉ publish `payment.created` và `wallet.allocated`.
> Tech stack: Java 25 · Spring Boot · MySQL 8.4 (Flyway `V001`–`V021`) · Kafka (transactional outbox) · VNPAY sandbox adapter · COD · port local `8086`

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Payment intent/transaction, VNPAY QR callback, COD payment state, marketplace allocation, commission/tax, seller wallet, payout và refund orchestration. |
| Source of truth | Payment, wallet balance và double-entry ledger thuộc service này. |
| Order | Order-Commerce sở hữu order lifecycle; Payment chỉ nhận order/payment intent và phát payment result. |
| Inventory | Inventory sở hữu reserve/commit/release; Payment không mutate stock. |
| Shipment | Shipment sở hữu delivery state; payment không tự đánh dấu delivered. |
| Không thuộc service | User/KYC decision, product price source, cart/order content, inventory ledger, carrier tracking. |
| Provider | VNPAY sandbox ở v1 qua adapter; COD không gọi payment gateway. |

### 1.2 Boundary

```text
Order-Commerce ──create/confirm/refund──► Payment-Wallet
Payment-Wallet ──VNPAY adapter──► VNPAY sandbox
VNPAY webhook ──signature/idempotency──► Payment-Wallet
Payment-Wallet ──Kafka outbox──► Order/Inventory/Notification
Seller ──wallet/payout──► Payment-Wallet
```

- Payment không tin amount từ webhook; amount/order/payment ID phải match local intent.
- Webhook là at-least-once và có thể out-of-order; provider event ID unique.
- Wallet dùng double-entry ledger; không update balance bằng phép cộng không có ledger entry.
- Secrets VNPAY không log, không commit source, đọc từ secret manager/environment.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Payment | VNPAY QR hoặc COD, payment DB | Payment intent + provider transaction + reconciliation. |
| Seller finance | Revenue, commission/tax, withdraw | Allocation/ledger/wallet/payout. `GET /seller/revenue` (HLD #38) là báo cáo read-only tổng hợp allocation/ledger theo khoảng thời gian — Seller Center Finance (Doanh thu tháng/Revenue chart/Fee breakdown/Tải báo cáo). |
| Buyer checkout | Payment pending/confirmed | Order gọi create payment sau order intent. |
| Payment screens | Create QR, pending confirmation, webhook auto-confirm | Provider adapter + polling/detail + webhook. |
| Penpot Admin — Fees/Taxes, Finance, Seller settlement, Settlement batches | Back-office tài chính sàn | Phục vụ bằng route `/api/v1/admin/**` **trên chính service này**, gác `FINANCE_OPS` + step-up 2FA cho mutation. Không tách microservice admin (quyết định admin-scope đã chốt 2026-09-18). Fee/tax là **config versioned** (effective-dated, append-only); settlement là **read + ops action** trên số dư đã ghi, không tạo tiền mới. |

## 2. Cấu trúc bên trong

### 2.1 Module Spring Boot

Cây package thật (`com.taca.paymentwallet`, hexagonal):

```text
com.taca.paymentwallet
├── presentation/rest/          # PaymentController, seller/*, admin/*, GlobalRestExceptionHandler
├── application/
│   ├── port/in, port/out       # use case + outbound port
│   ├── service/                # CreatePaymentService, ProcessVnpayWebhookService, ProcessCodPaymentService,
│   │                           # RequestRefundService, ProcessRefundResultService, RequestPayoutService,
│   │                           # ProcessPayoutResultService, RunSettlementService, PublishOutboxService, ...
│   ├── gateway/vnpay, payment, refund, payout, settlement, fee, export, idempotency, inbox, outbox, security
├── domain/                     # payment, wallet, refund, payout, settlement, finance, valueobject, event
└── infrastructure/
    ├── persistence/            # JPA entity/repository/adapter, audit, projection
    ├── messaging/kafka/        # KafkaTopicRouter, KafkaOutboxMessagePublisherAdapter, OutboxPublisherScheduler,
    │                           # OutboxCleanupScheduler, KafkaDeadLetterPublisherAdapter, shipment/ShipmentEventListener
    ├── vnpay/, crypto/, export/, health/, observability/, config/, id/, time/, transaction/, serialization/
```

| Thành phần (code) | Trách nhiệm | Ghi chú |
|---|---|---|
| `PaymentController` | `POST /payments`, `GET /payments/{id}`, `POST /payments/webhook`, `POST /payments/{id}/refunds` | Caller internal qua `InternalCallerPolicy`. |
| `ProcessVnpayWebhookService` | Verify chữ ký, dedupe provider event, cập nhật payment/allocation/ledger/outbox trong một transaction | |
| `ShipmentEventListener` → `ProcessCodPaymentService` | Capture/fail COD theo `shipment.delivered`/`shipment.failed` | Inbox dedupe theo `event_id`. |
| `SellerWalletController`, `SellerLedgerController`, `SellerRevenueController` | Seller finance read API | Shop lấy từ actor `shopScope` (đúng 1 shop). |
| `Admin*Controller` (Fee, Tax, Settlement, FinanceSummary, PaymentReconciliation) | Admin finance | `FINANCE_OPS`; mutation cần step-up. |
| `RequestPayoutService`, `ProcessPayoutResultService` | Domain payout | ⏳ Chưa có controller (ADR-003). |
| `RunSettlementService` | Tạo settlement batch | ⏳ `SettlementCandidatePort` chưa wire production; chỉ có API retry. |
| `OutboxPublisherScheduler`, `OutboxCleanupScheduler` | Publish outbox, dọn bản ghi cũ | Bật/tắt bằng property (§4). |

### 2.2 Chuẩn observability dùng chung

- Structured JSON stdout, field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- Propagate W3C `traceparent`/`tracestate`, `X-Request-ID`; Kafka headers giữ `traceparent`, `request_id`, `event_id`.
- Không log card/bank credential, VNPAY secret/signature/raw callback, access token, full address, email/phone hoặc full payment payload.
- Metrics OpenTelemetry/Prometheus-compatible, label bounded (`method`, `provider`, `result`, `reason`), không dùng `payment_id/order_id/user_id` làm label.
- `/health/live`, `/health/liveness` chỉ `livenessState`; `/health/ready`, `/health/readiness` gồm `readinessState`, `db`, `kafkaReadiness`, `outboxBacklog`, `vnpayReadiness`; `/prometheus` cho metrics. Không nằm dưới `/api/v1`.
- API error `{error:{code,message},meta:{request_id}}` (`ApiErrorResponse`); audit có actor/reason nhưng không secret.

## 3. Luồng xử lý

### 3.1 Create VNPAY payment

1. Order gửi order/payment intent với amount, currency, buyer, idempotency key.
2. Payment load local order payment allocation/reference; reject amount mismatch.
3. Tạo `payments(PENDING)`, sign VNPAY request bằng secret từ secret manager.
4. Trả QR/payment URL; không mark success khi chỉ tạo URL.
5. VNPAY callback/return xử lý riêng; callback server-to-server là nguồn confirmation ưu tiên.

### 3.2 Verify webhook

```text
Webhook → parse allowlist fields → verify checksum/signature
  → unique provider_event_id/idempotency
  → match payment_id/order_id/amount/currency
  → transaction payment state + ledger allocation + outbox
  → ACK provider
```

Callback duplicate trả ACK an toàn; callback sai signature/amount không đổi state và ghi security audit/metric.

### 3.3 COD

- Create payment với method `COD` tạo `PENDING_COD`; không gọi VNPAY.
- **`PENDING_COD` không chặn fulfillment.** Order-Commerce cho đơn COD vào `CONFIRMED` ngay tại checkout và không chờ event nào từ Payment-Wallet để giao hàng (xem `order-commerce-docs/docs/lld/order.md` §3.1). Payment-Wallet **không** phát `payment.succeeded` tại thời điểm đặt đơn COD, và Order-Commerce **không** chờ event đó.
- Capture: khi nhận `shipment.delivered` (hoặc collection event tương đương từ carrier adapter), Payment transition `PENDING_COD → SUCCESS`, post ledger và phát `payment.succeeded`. Đây là **sau** khi order đã `DELIVERED`, nên event này chỉ phục vụ đối soát/settlement, không mở luồng giao hàng. **Amount capture = `grand_total` của order (gồm `shipping_fee`)**: carrier thu hộ cả tiền hàng lẫn phí ship; Payment hạch toán phần phí ship vào shipment payable (xem §3.4), không credit seller phần phí.
- `shipment.failed` hoặc order cancelled trước khi giao: `PENDING_COD → FAILED` (enum PaymentStatus **không có** `CANCELLED` — chốt dùng `FAILED`), không post ledger, không tạo allocation.
- COD failure/cancel không tạo seller payout; exact cash collection event cần Shipment/Finance contract.

> Thứ tự thời gian của hai phương thức khác nhau, ledger phải chịu được cả hai:
> **VNPAY** — capture → allocation → order `CONFIRMED` → giao hàng.
> **COD** — order `CONFIRMED` → giao hàng → capture → allocation.
> Hệ quả: với COD, `wallet.allocated` và cửa sổ giữ tiền của seller bắt đầu tính từ lúc giao thành công, không phải lúc đặt đơn.

### 3.4 Wallet/allocation/payout/refund

- Sau payment captured, split gross theo order items/shop (`gross` **không gồm** phí ship — phí ship hạch toán riêng, xem dòng dưới), commission và tax bằng integer rounding policy.
- Ledger posting atomic: buyer clearing/payment account, platform revenue/tax payable, seller pending wallet, shipment payable (phí ship — posting riêng từ buyer clearing sang shipment payable, không đi qua allocation của shop).
- Payout chỉ dùng available seller balance, reserve amount trước khi gọi bank adapter; duplicate payout key không double debit.
- Refund tạo refund intent; reverse allocation/ledger theo amount không vượt captured và trạng thái policy.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---|---|---|
| VNPAY payment TTL | 15 phút | hằng `CreatePaymentService.VNPAY_PAYMENT_TTL` | COD không có `expires_at`. |
| Export URL TTL | 30 phút | hằng `ExportAdminFinanceSummaryService.EXPORT_URL_TTL` | Lưu tạm in-process. |
| VNPAY | — | `VNPAY_TMN_CODE`, `VNPAY_HASH_SECRET`, `VNPAY_PAYMENT_URL` (sandbox), `VNPAY_RETURN_URL` | Thiếu secret → `vnpayReadiness` DOWN. |
| Topic | `payment.events.v1`, `wallet.events.v1`, `shipment.events.v1`, `payment-wallet.outbox.dlq.v1` | `app.kafka.topics.*` | |
| Kafka | `localhost:19092`, group `payment-wallet-service` | `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_CONSUMER_GROUP_ID` | Producer `acks=all`, idempotent. |
| Topic provisioning guard | `false` | `KAFKA_TOPIC_PROVISIONING_GUARD_ENABLED` | |
| Outbox publisher | `enabled=false` (profile `local`: `true`), batch 50, max-retries 3, backoff 2s, poll 1000 ms, send-timeout 35000 ms | `app.outbox.publisher.*` | Hết retry → DLQ. |
| Outbox health | max-lag 5m | `app.outbox.health.max-lag` | |
| Outbox cleanup | `enabled=false` (local: `true`), retention 7d, batch 500, interval 1h | `app.outbox.cleanup.*` | |
| Port | `8086` (profile local) | `server.port` | Trùng port local của Shipment; cần tách khi chạy chung. |
| DB local | `localhost:15405/paymentdb` | `spring.datasource.*` | `ddl-auto=validate`, Flyway bật. |
| Tiền | integer VND, `currency` chỉ `VND` | — | |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `PaymentMethod` | `VNPAY`, `COD`. |
| `PaymentStatus` | `PENDING`, `PENDING_COD`, `SUCCESS`, `FAILED`, `EXPIRED`, `REFUNDED`, `PARTIALLY_REFUNDED`. `PENDING` dùng cho phương thức trả trước đang chờ provider; `PENDING_COD` dùng cho COD chờ thu tiền khi giao. |
| `WalletStatus` | `ACTIVE`, `FROZEN`, `CLOSED`. |
| `LedgerEntryType` | `DEBIT`, `CREDIT`. |
| `PayoutStatus` | `REQUESTED`, `PROCESSING`, `SUCCESS`, `FAILED`, `CANCELLED`. |
| `RefundStatus` | `REQUESTED`, `PROCESSING`, `SUCCESS`, `FAILED`, `CANCELLED`. |
| `SettlementBatchStatus` | `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`. |
| `FeeTaxScope` | `PLATFORM`, `CATEGORY`. |
| `SettlementBatchItemStatus` | `PENDING`, `COMPLETED`, `FAILED`. |
| `PaymentAttemptStatus` | `PENDING`, `SUCCESS`, `FAILED`, `EXPIRED`. |
| `PaymentOrderCodStatus` | `PENDING`, `CAPTURED`, `FAILED`. |
| `LedgerAccountType` | `VNPAY_CLEARING`, `COD_CLEARING`, `PLATFORM_COMMISSION`, `TAX_PAYABLE`, `SHIPMENT_PAYABLE`, `SELLER_PENDING`, `SELLER_AVAILABLE`, `PAYOUT_CLEARING`, `REFUND_CLEARING`. |

Payment `PENDING → SUCCESS/FAILED/EXPIRED`; `PENDING_COD → SUCCESS` (capture sau delivered) `→ FAILED` (shipment.failed/order cancelled trước giao); `SUCCESS → PARTIALLY_REFUNDED/REFUNDED`; terminal state không reopen.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

Mọi domain event được ghi vào `outbox_events` trong cùng transaction. Payload là bản serialize JSON của record domain (field camelCase như `eventId`, `occurredAt`, `paymentId`…); header outbox mang `event_id`, `request_id`, `traceparent`, `tracestate`. Kafka key = `aggregate_id`. Value Kafka chính là record đó — **không** có envelope `event_id`/`event_type`/`occurred_at`/`payload` (snake_case) như auth-user, shipment, product-catalog; consumer chuẩn (Notification `DomainEventSchema`, Order) sẽ không parse được. Xem `CONTRACT-MATRIX.md` §2.

**Chỉ hai event được `KafkaTopicRouter` route ra Kafka.** Các event còn lại nằm trong outbox, không có topic, chưa consumer nào nhận được.

| Event | Payload (field record) | Topic | Trạng thái |
|---|---|---|---|
| `payment.created` | `eventId, occurredAt, paymentId, checkoutGroupId, orderIds, amount, method, status` | `payment.events.v1` | ✅ |
| `wallet.allocated` | `eventId, occurredAt, walletId, orderId, shopId, grossAmount, commissionAmount, taxAmount, sellerNetAmount` | `wallet.events.v1` | ✅ |
| `payment.succeeded` | `eventId, occurredAt, paymentId, capturedAmount` | — | 🟡 Chỉ ghi outbox, chưa route |
| `payment.failed` | `eventId, occurredAt, paymentId, failureCode` | — | 🟡 Chỉ ghi outbox |
| `payment.expired` | `eventId, occurredAt, paymentId` | — | 🟡 Chỉ ghi outbox; chưa có job tự expire |
| `payment.refunded` | `eventId, occurredAt, paymentId, refundAmount, newStatus` | — | 🟡 Chỉ ghi outbox |
| `refund.requested` / `refund.succeeded` / `refund.failed` | `eventId, occurredAt, refundId, paymentId` + `amount` (requested/succeeded) hoặc `failureCode` (failed) | — | 🟡 Chỉ ghi outbox |
| `payout.requested` / `payout.succeeded` / `payout.failed` | `eventId, occurredAt, payoutId, shopId, amount`; succeeded thêm `status, providerReference`; failed thêm `status, failureCode` | — | 🟡 Chỉ ghi outbox; chưa có API payout |
| `settlement.batch.completed` | `eventId, occurredAt, settlementBatchId, totalReleased` | — | 🟡 Chỉ ghi outbox |

> **Không có recipient trong payload.** `payment.*` không có `buyer.user_id`/`buyer.email`; `payout.*` không có `owner_user_id`/`owner_email`. Thiết kế cũ (quyết định 2026-09-18) yêu cầu các field này cho template EMAIL của Notification, nhưng code chưa có nguồn authoritative (ADR-003). Đây là khoảng trống code, xem §8.

### 6.2 Event lắng nghe

| Nguồn | Topic | Event | Xử lý | Trạng thái |
|---|---|---|---|---|
| Shipment | `shipment.events.v1` | `shipment.delivered` | `PENDING_COD → SUCCESS`, allocation + ledger + `wallet.allocated` | ✅ (`ShipmentEventListener`, inbox dedupe) |
| Shipment | `shipment.events.v1` | `shipment.failed` | `PENDING_COD → FAILED`, không post ledger | ✅ |
| Order-Commerce | — | `order.created`, `order.cancelled` | Thiết kế: release/refund policy. Payment hiện được tạo qua `POST /payments`. | ⏳ Không có listener |
| Auth User | — | `shop.kyc.approved`, `shop.kyc.expired`, `shop.status_changed` | Thiết kế: gate payout. | ⏳ Không có listener |

### 6.3 Contract VNPAY webhook (code)

Body thật của `POST /payments/webhook` xem API §3.3: `provider_event_id`, `provider_transaction_ref`, `response_code`, `transaction_status`, `amount`, `currency`, `payload_hash`, `signed_payload{vnp_*}`. Chữ ký sai → `400 INVALID_VNPAY_SIGNATURE`.

## 7. Mã lỗi

Danh sách đầy đủ mã code đang trả và mã dự phòng: xem `docs/api/payment-wallet.md` §4 (nguồn: `GlobalRestExceptionHandler`). Các mã thiết kế cũ `PAYMENT_STATE_INVALID`, `PAYMENT_PROVIDER_UNAVAILABLE`, `PAYMENT_WEBHOOK_INVALID`, `PAYMENT_WEBHOOK_REPLAYED`, `PAYMENT_INTERNAL_ERROR` **không có trong code**; tương ứng code dùng `REFUND_STATE_INVALID`/`SETTLEMENT_BATCH_INVALID_STATE`, `INVALID_VNPAY_SIGNATURE`, `action: "DUPLICATE"` (200) và `INTERNAL_ERROR`.

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | VNPAY sandbox được dùng trong v1; merchant credentials thật do secret manager cấp. | Ảnh hưởng adapter/signature/reconciliation. | Finance/Security |
| 2 | COD success được xác nhận bởi shipment/order delivered/collected event. | Ảnh hưởng seller settlement. | Order/Shipment owner |
| 3 | Commission/tax rate có version nhưng exact rate/rounding chưa chốt. Admin `PUT /admin/fees` / `/admin/taxes` tạo version effective-dated; giá trị khởi tạo do Finance seed. | Ảnh hưởng ledger/invoice. | Finance |
| 4 | Payout bank provider và SLA chưa có HLD contract. | Cần mock adapter và retry/reconciliation job. | Finance/DevOps |
| 5 | Refund initiation do Order/Admin gọi Payment; **return/dispute workflow ngoài v1** — sẽ do service `dispute` (v1.1) điều phối, gọi cùng contract `POST /payments/{paymentId}/refunds`. Trong v1 refund chỉ khởi tạo thủ công qua `/admin/payments` hoặc Order. | Ảnh hưởng API/permission. | Product/Finance |
| 6 | Phạm vi Admin/back-office đã chốt (2026-09-18): Fees/Taxes, Finance, Seller settlement, Settlement batches phục vụ qua `/api/v1/admin/**` **trên service này**, không tách microservice. `dispute`/`campaign` là service v1.1. | Nếu tách service admin gộp phải chuyển ownership fee/tax/settlement. | Architecture + Finance |
| 7 | Cơ chế kích hoạt settlement batch (scheduled job theo cửa sổ hoàn tiền vs. event `order.completed` — **ghi chú: `order.completed` hiện không tồn tại**, order-commerce không phát event này; nếu chọn hướng event phải mở rộng contract order trước) và độ dài cửa sổ chưa chốt. Admin screen chỉ giám sát + `retry` batch `FAILED`. | Ảnh hưởng thời điểm `pending → available` và SLA payout. | Finance + Order owner |
| 8 | Chỉ `payment.created` và `wallet.allocated` được publish ra Kafka; `payment.succeeded/failed/expired/refunded`, `refund.*`, `payout.*`, `settlement.batch.completed` chỉ nằm trong outbox. | Notification/Order không nhận được kết quả thanh toán qua Kafka. | Payment-Wallet |
| 9 | Payload event không có recipient (`buyer.email`, `owner_user_id`, `owner_email`). | Notification không gửi được email payment/payout. | Payment-Wallet + Auth User (ADR-003) |
| 10 | Không có scheduled job tự expire payment VNPAY sau 15 phút. | Payment `PENDING` treo; Inventory reservation không được giải phóng theo payment. | Payment-Wallet |
