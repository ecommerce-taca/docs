# API Spec — Payment-Wallet Service

> Nguồn: `docs/lld/payment-wallet.md` · `docs/db/payment-wallet.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · VNPAY sandbox + COD · Payment secrets không đi qua client

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Buyer/seller/admin JWT qua Gateway; internal Order/Payment callback dùng service auth. |
| Request ID | `X-Request-ID` tối đa 64, Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate` REST/Kafka; error có `trace_id`. |
| Time/money | ISO-8601 UTC; integer VND, không FLOAT. |
| Idempotency | Payment/payout/refund command và webhook provider event bắt buộc dedupe. |
| Response | `{data,meta:{request_id}}`; error `{error:{code,message,details,trace_id}}`. |
| Log | JSON field chuẩn `timestamp,level,service,env,version,event,trace_id,span_id,request_id,route,method,status_code,duration_ms`. |
| Redaction | Không log token, VNPAY signature/raw payload, bank/card credential, full address/PII. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `POST /payments` | Internal Order/buyer flow | Tạo payment intent VNPAY/COD. |
| 2 | `GET /payments/{paymentId}` | Buyer/internal | Xem payment status. |
| 3 | `POST /payments/webhook` | VNPAY provider | Reconcile callback, không JWT. |
| 4 | `POST /payments/{paymentId}/refunds` | Internal/admin | Tạo refund intent. |
| 5 | `GET /seller/wallet` | Seller | Xem available/pending balance. |
| 6 | `GET /seller/wallet/ledger` | Seller | Xem ledger summary. |
| 7 | `GET /seller/revenue` | Seller | Báo cáo doanh thu theo khoảng thời gian (HLD #38). |
| 8 | `POST /seller/payouts` | Seller + step-up | Yêu cầu rút tiền. |
| 9 | `GET /seller/payouts` | Seller | Xem payout history. |
| 10 | `GET /admin/payments/reconciliation` | `FINANCE_OPS` | Reconcile provider/payment/ledger. |
| 11 | `GET /admin/fees` | `FINANCE_OPS` | Danh sách version config commission (hiện hành + lịch sử). |
| 12 | `PUT /admin/fees` | `FINANCE_OPS` + 2FA | Tạo version commission mới, effective-dated. |
| 13 | `GET /admin/taxes` | `FINANCE_OPS` | Danh sách version config thuế. |
| 14 | `PUT /admin/taxes` | `FINANCE_OPS` + 2FA | Tạo version thuế mới, effective-dated. |
| 15 | `GET /admin/settlements` | `FINANCE_OPS` | Danh sách settlement batch (period/status/tổng gross/commission/tax/net). |
| 16 | `GET /admin/settlements/{batchId}` | `FINANCE_OPS` | Chi tiết batch + breakdown theo shop. |
| 17 | `POST /admin/settlements/{batchId}/retry` | `FINANCE_OPS` + 2FA | Retry batch `FAILED` (idempotent). |
| 18 | `GET /admin/finance/summary` | `FINANCE_OPS` | Tổng hợp tài chính sàn read-only (GMV, commission income, tax, refund, payout volume). |
| 19 | `GET /health/live` | Ops | Liveness. |
| 20 | `GET /health/ready` | Ops | Readiness. |

## 3. Chi tiết endpoint

### 3.1 `POST /payments`

Internal request `{order_id,checkout_group_id,buyer_user_id,amount,currency:"VND",method:"VNPAY"|"COD",expires_at}`; client không được tự truyền trusted shop allocation. Header `Idempotency-Key` bắt buộc. Response `201` VNPAY trả `payment_id,status:PENDING,payment_url/qr_payload,expires_at`; COD trả `PENDING_COD`, không gọi provider.

Amount phải match Order contract; mismatch `409 PAYMENT_AMOUNT_MISMATCH`. Tạo URL không đồng nghĩa success.

### 3.2 `GET /payments/{paymentId}`

Buyer chỉ xem payment của own order; seller chỉ xem allocation liên quan shop; admin/finance xem theo permission. Trả status, method, amount, provider reference masked, timestamps, không trả signature/secret.

### 3.3 `POST /payments/webhook`

Không JWT; xác thực VNPAY signature/IP policy, unique provider event ID, amount/order/payment match. Callback success transactionally đổi payment + ledger + outbox `payment.succeeded`; duplicate ACK an toàn. Sai signature/amount không đổi state.

### 3.4 Refund

`POST /payments/{paymentId}/refunds` body `{amount,reason,order_id}` + Idempotency-Key. Không vượt captured amount; response `202`/`200` theo provider. Payment state thành `PARTIALLY_REFUNDED` hoặc `REFUNDED` sau confirmation.

### 3.5 Seller wallet/ledger/revenue

- `GET /seller/wallet`: trả `available_balance`, `pending_balance`, currency, wallet status, `as_of`.
- `GET /seller/wallet/ledger`: page/size/date/type; chỉ ledger của shop, PII/payment credential masked.
- `GET /seller/revenue?from=&to=&granularity=DAY|WEEK|MONTH`: báo cáo tổng hợp **read-only** trên `payment_allocations`/`ledger_entries` của shop (đáp ứng HLD #38 `/seller/revenue?range=`). Response:

```json
{
  "data": {
    "range": { "from": "2026-08-01", "to": "2026-08-31", "granularity": "DAY" },
    "currency": "VND",
    "summary": {
      "gross": 125000000,
      "commission": 8750000,
      "tax": 1250000,
      "net": 115000000,
      "refunded": 2000000,
      "order_count": 340
    },
    "buckets": [
      { "period": "2026-08-01", "gross": 4200000, "commission": 294000, "tax": 42000, "net": 3864000, "order_count": 12 }
    ]
  },
  "meta": { "request_id": "01912fc0-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Ràng buộc: chỉ tổng hợp từ dữ liệu đã ghi (không tạo ledger mới); `from..to` tối đa 366 ngày/request; số liệu là snapshot allocation/ledger đã captured, không phản ánh payout. Đây là **báo cáo**, không phải nghiệp vụ tiền mới.

### 3.6 `POST /seller/payouts`

Body `{amount,bank_account_id,reason?}` + Idempotency-Key + step-up. Kiểm KYC projection approved, wallet active, available balance và minimum amount; reserve/debit wallet trước provider adapter. Response `202 REQUESTED`.

### 3.7 Reconciliation/health

Admin reconciliation filter provider/status/date, trả mismatch counts không raw secret. `/health/live` process-only; `/health/ready` MySQL/Kafka/config/VNPAY secret availability.

### 3.8 Admin finance back-office (`FINANCE_OPS`)

Phục vụ các màn Penpot Admin *Fees/Taxes*, *Finance*, *Seller settlement*, *Settlement batches*. Không có microservice admin riêng (xem `System_Overview.md` §6.3); Gateway coarse-gate role admin, service này enforce `FINANCE_OPS` + step-up 2FA cho mutation. Mọi mutation ghi `audit_logs` (actor/reason).

**Fee/Tax config — `GET/PUT /admin/fees`, `GET/PUT /admin/taxes`**

- Config là **effective-dated, append-only**: `PUT` tạo version mới `{scope: PLATFORM|CATEGORY, category_id?, rate_bps, effective_from, note}`; không sửa/xóa version cũ.
- `AllocationService` luôn chọn version có `effective_from` ≤ thời điểm tạo allocation; đổi rate **không** hồi tố allocation/ledger đã ghi.
- `PUT` với `effective_from` trong quá khứ → `409 FEE_CONFIG_INVALID`.

```json
// GET /admin/fees
{ "data": [
  { "version_id": "01J...", "scope": "PLATFORM", "category_id": null, "rate_bps": 700, "effective_from": "2026-09-01T00:00:00Z", "note": "Q4 baseline", "created_by": "usr_...", "created_at": "2026-08-30T10:00:00Z" }
], "meta": { "request_id": "01J..." } }
```

**Settlement — `GET /admin/settlements`, `GET /admin/settlements/{batchId}`, `POST /admin/settlements/{batchId}/retry`**

- Batch tổng hợp việc chuyển `pending_balance → available_balance` sau cửa sổ hoàn tiền/đối soát; **không tạo tiền mới**, chỉ posting release đã định nghĩa trong ledger.
- `GET` list filter `period`/`status`; detail trả breakdown theo shop (`gross`, `commission`, `tax`, `net`, `released_amount`, `held_amount`).
- `retry` chỉ hợp lệ khi batch `FAILED`; idempotent theo `batchId`; state khác → `409 SETTLEMENT_BATCH_INVALID_STATE`.
- Hold/release settlement theo rủi ro (liên quan dispute) **ngoài v1** — thuộc service `dispute` (v1.1).

**Finance summary — `GET /admin/finance/summary?from=&to=`**

Read-only aggregate toàn sàn trên `payment_allocations`/`ledger_entries`/`payouts`: `gmv`, `commission_income`, `tax_collected`, `refunded`, `payout_volume`, `wallet_float`. `from..to` ≤ 366 ngày. Không phải nghiệp vụ tiền mới.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `PAYMENT_INVALID_INPUT` | 400 | Amount/method/request sai. |
| `PAYMENT_UNAUTHENTICATED` | 401 | Thiếu auth. |
| `PAYMENT_FORBIDDEN` | 403 | Sai scope. |
| `PAYMENT_NOT_FOUND` | 404 | Không tìm thấy payment. |
| `PAYMENT_AMOUNT_MISMATCH` | 409 | Amount không khớp Order. |
| `PAYMENT_STATE_INVALID` | 409 | Transition sai. |
| `PAYMENT_PROVIDER_UNAVAILABLE` | 503 | VNPAY/bank down. |
| `PAYMENT_WEBHOOK_INVALID` | 400 | Signature/payload invalid. |
| `PAYMENT_WEBHOOK_REPLAYED` | 200/409 | Event đã xử lý. |
| `WALLET_INSUFFICIENT_BALANCE` | 409 | Không đủ balance. |
| `WALLET_FROZEN` | 403 | Wallet frozen. |
| `PAYOUT_NOT_ALLOWED` | 403/409 | KYC/state/min amount. |
| `REFUND_AMOUNT_INVALID` | 400/409 | Refund vượt captured. |
| `FEE_CONFIG_INVALID` | 409 | Version fee/tax có `effective_from` quá khứ hoặc overlap không hợp lệ. |
| `SETTLEMENT_BATCH_INVALID_STATE` | 409 | Action settlement không hợp lệ với state batch. |
| `PAYMENT_IDEMPOTENCY_CONFLICT` | 409 | Key khác payload. |
| `PAYMENT_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | VNPAY sandbox v1; production merchant config chưa có. | Cần đổi provider/security contract. | Finance/Security |
| 2 | COD success dựa Shipment/Order collection event. | Ảnh hưởng settlement. | Order/Shipment |
| 3 | Commission/tax/rounding chưa chốt rate. | Ảnh hưởng ledger/API totals. | Finance |
| 4 | Payout provider chưa chốt. | Tạm mock adapter/reconciliation. | Finance/DevOps |
| 5 | Return/dispute workflow **ngoài v1**: refund chỉ khởi tạo thủ công (Order/`/admin/payments`). v1.1 service `dispute` sẽ điều phối và gọi cùng contract refund. | Cần thêm permission/state khi bật dispute. | Product/Finance |
| 6 | `GET /seller/revenue` là báo cáo read-only tổng hợp `payment_allocations`/`ledger_entries` (HLD #38); commission/tax dùng đúng rate đã versioned tại thời điểm allocation. | Nếu rate/rounding chưa chốt, số tổng hợp phải khớp rate versioned, không tính lại. | Finance |
| 7 | Admin back-office (Fees/Taxes, Finance, Settlement) phục vụ qua `/api/v1/admin/**` trên service này, `FINANCE_OPS` + 2FA; **không** tách microservice admin (`System_Overview.md` §6.3). Fee/tax là config effective-dated append-only; settlement là read + `retry`. | Nếu chuyển ownership fee/tax/settlement sang service khác phải đổi contract allocation. | Architecture + Finance |
| 8 | Trigger settlement batch (scheduled theo cửa sổ hoàn tiền vs event `order.completed`) và độ dài cửa sổ chưa chốt. | Ảnh hưởng thời điểm `pending → available` và SLA payout. | Finance + Order owner |
