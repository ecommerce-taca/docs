# LLD — Payment-Wallet Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 · VNPAY sandbox adapter · COD · Kafka outbox

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
| Penpot Admin — Fees/Taxes, Finance, Seller settlement, Settlement batches | Back-office tài chính sàn | Phục vụ bằng route `/api/v1/admin/**` **trên chính service này**, gác `FINANCE_OPS` + step-up 2FA cho mutation. Không tách microservice admin (xem `System_Overview.md` §6.3). Fee/tax là **config versioned** (effective-dated, append-only); settlement là **read + ops action** trên số dư đã ghi, không tạo tiền mới. |

## 2. Cấu trúc bên trong

### 2.1 Module Spring Boot

```text
com.taca.payment
├── payment/              # intent/transaction/state machine
├── provider/vnpay/       # sign/verify QR, callback adapter
├── cod/                  # cash-on-delivery projection/confirmation
├── allocation/           # order split, commission, tax, seller net
├── wallet/               # double-entry ledger and balance
├── payout/               # withdrawal request and bank snapshot
├── refund/               # refund intent/reconciliation
├── admin/                # admin finance back-office: fee/tax config, settlement, finance summary
├── outbox/               # event publisher
├── security/             # signature, role, idempotency, secret handling
└── observability/        # shared log/trace/metric/health contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `PaymentController` | Create/detail payment | Amount lấy local order contract; idempotency bắt buộc. |
| `VnpayWebhookController` | Nhận callback | Verify signature, provider event ID, amount/order match; không dùng JWT. |
| `PaymentStateMachine` | Pending/success/failed/refunded | Không cho webhook tự nhảy state trái transition. |
| `AllocationService` | Split gross/commission/tax/net theo shop | Integer VND; rounding policy deterministic. |
| `LedgerService` | Debit/credit double-entry | Mỗi posting cân bằng tổng debit=credit; append-only. |
| `PayoutService` | Seller withdraw | KYC/settlement gate từ Auth projection; idempotent bank transfer adapter. |
| `RevenueReportService` | Tổng hợp `GET /seller/revenue` theo range/granularity | Read-only aggregate trên `payment_allocations`/`ledger_entries`; không tạo ledger; range ≤ 366 ngày; số liệu theo rate versioned tại thời điểm allocation. |
| `RefundService` | Refund order/payment | Không refund vượt captured amount; audit reason. |
| `AdminFinanceController` | `/admin/fees`, `/admin/taxes`, `/admin/settlements/**`, `/admin/finance/summary`, `/admin/payments/reconciliation` | `FINANCE_OPS` permission; mutation (tạo version fee/tax, retry batch) yêu cầu step-up 2FA; mọi action ghi `audit_logs`. |
| `FeeTaxConfigService` | Đọc/tạo version config commission + tax | Config **effective-dated, append-only**; không sửa/xóa version cũ; allocation luôn dùng version có hiệu lực tại thời điểm tạo allocation (khớp `AllocationService`/`RevenueReportService`). |
| `SettlementService` | Batch chuyển `pending_balance → available_balance` sau cửa sổ hoàn/đối soát | Read view theo batch/shop; ops action `retry` cho batch `FAILED` (idempotent); không tạo ledger ngoài posting release đã định nghĩa. Trigger batch (scheduled vs event) — xem §8. |
| `OutboxPublisher` | Phát payment/wallet events | Transactional outbox, retry 3/backoff 2s/DLQ. |

### 2.2 Chuẩn observability dùng chung

- Structured JSON stdout, field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- Propagate W3C `traceparent`/`tracestate`, `X-Request-ID`; Kafka headers giữ `traceparent`, `request_id`, `event_id`.
- Không log card/bank credential, VNPAY secret/signature/raw callback, access token, full address, email/phone hoặc full payment payload.
- Metrics OpenTelemetry/Prometheus-compatible, label bounded (`method`, `provider`, `result`, `reason`), không dùng `payment_id/order_id/user_id` làm label.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MySQL/Kafka/VNPAY config/secret availability theo policy.
- API error `{error:{code,message,details,trace_id}}`; audit có actor/reason nhưng không secret.

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
- **`PENDING_COD` không chặn fulfillment.** Order-Commerce cho đơn COD vào `CONFIRMED` ngay tại checkout và không chờ event nào từ Payment-Wallet để giao hàng (xem `order-commerce-docs/docs/lld/order-commerce.md` §3.4). Payment-Wallet **không** phát `payment.succeeded` tại thời điểm đặt đơn COD, và Order-Commerce **không** chờ event đó.
- Capture: khi nhận `shipment.delivered` (hoặc collection event tương đương từ carrier adapter), Payment transition `PENDING_COD → SUCCESS`, post ledger và phát `payment.succeeded`. Đây là **sau** khi order đã `DELIVERED`, nên event này chỉ phục vụ đối soát/settlement, không mở luồng giao hàng.
- `shipment.failed` hoặc order cancelled trước khi giao: `PENDING_COD → FAILED`/`CANCELLED` tuỳ policy, không post ledger, không tạo allocation.
- COD failure/cancel không tạo seller payout; exact cash collection event cần Shipment/Finance contract.

> Thứ tự thời gian của hai phương thức khác nhau, ledger phải chịu được cả hai:
> **VNPAY** — capture → allocation → order `CONFIRMED` → giao hàng.
> **COD** — order `CONFIRMED` → giao hàng → capture → allocation.
> Hệ quả: với COD, `wallet.allocated` và cửa sổ giữ tiền của seller bắt đầu tính từ lúc giao thành công, không phải lúc đặt đơn.

### 3.4 Wallet/allocation/payout/refund

- Sau payment captured, split gross theo order items/shop, commission và tax bằng integer rounding policy.
- Ledger posting atomic: buyer clearing/payment account, platform revenue/tax payable, seller pending wallet.
- Payout chỉ dùng available seller balance, reserve amount trước khi gọi bank adapter; duplicate payout key không double debit.
- Refund tạo refund intent; reverse allocation/ledger theo amount không vượt captured và trạng thái policy.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `PAYMENT_INTENT_TTL` | 15 phút | Align Inventory reservation. |
| `VNPAY_REQUEST_TIMEOUT` | 5s | Sandbox adapter. |
| `WEBHOOK_MAX_SKEW` | 10 phút | Reject callback quá cũ theo provider timestamp. |
| `VNPAY_RETRY_MAX` | 1 | Không retry callback mutation; provider event idempotent. |
| `VND_MIN/MAX` | 1/999999999999 | Integer. |
| `COMMISSION_RATE_MIN/MAX` | 0/100% | Exact platform rate config phải versioned. |
| `PAYOUT_MIN_AMOUNT` | 1000 VND | Baseline, cần finance confirm. |
| `REVENUE_EXPORT_MAX_RANGE` | 366 ngày | Cùng giới hạn với `GET /seller/revenue`. |
| `EXPORT_URL_TTL` | 30 phút | `GET /seller/revenue/export`, `GET /admin/finance/summary/export`. |
| `PAYOUT_RETENTION` | 365 ngày | Bank snapshot masked/encrypted. |
| `OUTBOX_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `IDEMPOTENCY_RETENTION` | 24h | Payment/payout/refund command. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | DB `DATETIME(6)`. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `PaymentMethod` | `VNPAY`, `COD`. |
| `PaymentStatus` | `PENDING`, `PENDING_COD`, `SUCCESS`, `FAILED`, `EXPIRED`, `REFUNDED`, `PARTIALLY_REFUNDED`. `PENDING` dùng cho phương thức trả trước đang chờ provider; `PENDING_COD` dùng cho COD chờ thu tiền khi giao. |
| `WalletStatus` | `ACTIVE`, `FROZEN`, `CLOSED`. |
| `LedgerEntryType` | `DEBIT`, `CREDIT`. |
| `PayoutStatus` | `REQUESTED`, `PROCESSING`, `SUCCESS`, `FAILED`, `CANCELLED`. |
| `RefundStatus` | `REQUESTED`, `PROCESSING`, `SUCCESS`, `FAILED`, `CANCELLED`. |
| `Provider` | `VNPAY`, `COD`, `MOCK`. |

Payment `PENDING → SUCCESS/FAILED/EXPIRED`; `SUCCESS → PARTIALLY_REFUNDED/REFUNDED`; terminal state không reopen.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `payment.events.v1` | `payment.created` | payment/order/amount/method/status |
| `payment.events.v1` | `payment.succeeded` | payment/order/provider ref/paid_at |
| `payment.events.v1` | `payment.failed/expired` | payment/reason |
| `payment.events.v1` | `payment.refunded` | payment/order/refund amount |
| `wallet.events.v1` | `wallet.allocated` | order/shop/gross/commission/tax/net |
| `wallet.events.v1` | `payout.succeeded/failed` | payout/shop/amount/status |

### 6.2 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Order-Commerce | `order.created`, `order.cancelled` | Create payment intent/release pending payment/refund policy. |
| Shipment | `shipment.delivered`, `shipment.failed` | COD collection confirmation hoặc hold. |
| Auth User | `shop.kyc.approved`, `shop.kyc.expired` | Local payout/withdraw gate projection (KYC phải `APPROVED` mới cho payout). |
| Auth User | `shop.status_changed` | Shop `SUSPENDED`/`CLOSED` → khoá payout/withdraw. Không có event `shop.kyc.suspended`; đình chỉ shop đến qua `shop.status_changed`. |

### 6.3 Mock contract — VNPAY webhook

```json
{
  "provider_event_id":"vnp-event-1",
  "payment_id":"payment-1",
  "order_id":"order-1",
  "amount":249000,
  "currency":"VND",
  "provider_status":"SUCCESS",
  "transaction_ref":"vnp-txn-1",
  "paid_at":"2026-08-30T09:01:00Z",
  "signature":"opaque-provider-signature"
}
```

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `PAYMENT_INVALID_INPUT` | 400 | Amount/method/request sai. |
| `PAYMENT_UNAUTHENTICATED` | 401 | Thiếu auth. |
| `PAYMENT_FORBIDDEN` | 403 | Sai buyer/shop/admin scope. |
| `PAYMENT_NOT_FOUND` | 404 | Payment không tồn tại. |
| `PAYMENT_AMOUNT_MISMATCH` | 409 | Amount khác Order snapshot. |
| `PAYMENT_STATE_INVALID` | 409 | Transition sai. |
| `PAYMENT_PROVIDER_UNAVAILABLE` | 503 | VNPAY/bank adapter down. |
| `PAYMENT_WEBHOOK_INVALID` | 400 | Signature/payload sai. |
| `PAYMENT_WEBHOOK_REPLAYED` | 200/409 | Provider event đã xử lý. |
| `WALLET_INSUFFICIENT_BALANCE` | 409 | Không đủ available balance. |
| `WALLET_FROZEN` | 403 | Wallet bị freeze. |
| `PAYOUT_NOT_ALLOWED` | 403/409 | KYC/state/min amount không đạt. |
| `REFUND_AMOUNT_INVALID` | 400/409 | Refund vượt captured/invalid. |
| `PAYMENT_IDEMPOTENCY_CONFLICT` | 409 | Key khác request hash. |
| `PAYMENT_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | VNPAY sandbox được dùng trong v1; merchant credentials thật do secret manager cấp. | Ảnh hưởng adapter/signature/reconciliation. | Finance/Security |
| 2 | COD success được xác nhận bởi shipment/order delivered/collected event. | Ảnh hưởng seller settlement. | Order/Shipment owner |
| 3 | Commission/tax rate có version nhưng exact rate/rounding chưa chốt. Admin `PUT /admin/fees` / `/admin/taxes` tạo version effective-dated; giá trị khởi tạo do Finance seed. | Ảnh hưởng ledger/invoice. | Finance |
| 4 | Payout bank provider và SLA chưa có HLD contract. | Cần mock adapter và retry/reconciliation job. | Finance/DevOps |
| 5 | Refund initiation do Order/Admin gọi Payment; **return/dispute workflow ngoài v1** — sẽ do service `dispute` (v1.1) điều phối, gọi cùng contract `POST /payments/{id}/refunds`. Trong v1 refund chỉ khởi tạo thủ công qua `/admin/payments` hoặc Order. | Ảnh hưởng API/permission. | Product/Finance |
| 6 | Phạm vi Admin/back-office đã chốt (`System_Overview.md` §6.3): Fees/Taxes, Finance, Seller settlement, Settlement batches phục vụ qua `/api/v1/admin/**` **trên service này**, không tách microservice. `dispute`/`campaign` là service v1.1. | Nếu tách service admin gộp phải chuyển ownership fee/tax/settlement. | Architecture + Finance |
| 7 | Cơ chế kích hoạt settlement batch (scheduled job theo cửa sổ hoàn tiền vs. event `order.completed`) và độ dài cửa sổ chưa chốt. Admin screen chỉ giám sát + `retry` batch `FAILED`. | Ảnh hưởng thời điểm `pending → available` và SLA payout. | Finance + Order owner |
