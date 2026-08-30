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
| 7a | `GET /seller/revenue/export` | Seller | Xuất báo cáo doanh thu ra file (.xlsx/.csv). |
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
| 18a | `GET /admin/finance/summary/export` | `FINANCE_OPS` | Xuất báo cáo tài chính sàn ra file (.xlsx/.csv). |
| 19 | `GET /health/live` | Ops | Liveness. |
| 20 | `GET /health/ready` | Ops | Readiness. |

## 3. Chi tiết endpoint

### 3.1 `POST /payments`

Header `Idempotency-Key` bắt buộc. Chỉ gọi từ Order-Commerce (internal scope) — client **không** được tự truyền allocation theo shop.

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `order_id` | string | Có | — |
| `checkout_group_id` | string | Có | Nhóm order multi-shop của cùng lần checkout |
| `buyer_user_id` | string | Có | — |
| `amount` | integer | Có | VND, > 0, **phải khớp** tổng của Order |
| `currency` | string | Có | Cố định `"VND"` |
| `method` | enum | Có | `VNPAY` \| `COD` |
| `expires_at` | string | Có | ISO-8601 UTC |

```json
{
  "data": {
    "payment_id": "payment-01912f95",
    "order_id": "order-01912f91",
    "status": "PENDING",
    "method": "VNPAY",
    "amount": 1094000,
    "currency": "VND",
    "payment_url": "https://sandbox.vnpayment.vn/paymentv2/vpcpay.html?…",
    "qr_payload": "00020101021238…",
    "expires_at": "2026-08-30T09:15:00Z"
  },
  "meta": { "request_id": "01912fa6-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

COD trả `status: "PENDING_COD"`, **không** có `payment_url`/`qr_payload` và không gọi provider. Amount lệch Order → `409 PAYMENT_AMOUNT_MISMATCH`. **Tạo được URL không đồng nghĩa đã thanh toán** — chỉ webhook đã verify mới chuyển `SUCCESS`.

### 3.2 `GET /payments/{paymentId}`

```json
{
  "data": {
    "payment_id": "payment-01912f95",
    "order_id": "order-01912f91",
    "status": "SUCCESS",
    "method": "VNPAY",
    "amount": 1094000,
    "currency": "VND",
    "provider_ref_masked": "VNP****4821",
    "paid_at": "2026-08-30T09:03:12Z",
    "refunded_amount": 0,
    "created_at": "2026-08-30T09:00:05Z"
  },
  "meta": { "request_id": "01912fa7-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Scope: buyer chỉ xem payment của order mình; seller chỉ xem phần allocation của shop mình; admin/finance theo permission. **Không bao giờ** trả signature, secret, hay payload provider thô.

### 3.3 `POST /payments/webhook`

Không JWT. Bắt buộc verify chữ ký VNPAY + IP policy, `provider_event_id` unique, và amount/order/payment khớp bản ghi nội bộ.

```json
{
  "provider_event_id": "vnp-evt-77213",
  "payment_id": "payment-01912f95",
  "order_id": "order-01912f91",
  "amount": 1094000,
  "status": "SUCCESS",
  "provider_ref": "VNP20260830004821",
  "occurred_at": "2026-08-30T09:03:12Z",
  "signature": "<vnpay-signature>"
}
```

Response `200 {"data":{"accepted":true}}`. Quy tắc:
- Success hợp lệ → **một transaction** đổi payment state + ghi ledger + ghi outbox `payment.succeeded`.
- Trùng `provider_event_id` → ACK `200`, **không** ghi ledger lần hai.
- Sai chữ ký → `400 PAYMENT_WEBHOOK_INVALID`, không đổi state.
- Amount lệch → `409 PAYMENT_AMOUNT_MISMATCH`, không đổi state, ghi cảnh báo đối soát.
- **Không tin amount từ webhook** — luôn so với intent nội bộ.

### 3.4 Refund

`POST /payments/{paymentId}/refunds` + `Idempotency-Key`.

```json
{ "amount": 1094000, "reason": "BUYER_CANCELLED", "order_id": "order-01912f91" }
```

```json
{
  "data": {
    "refund_id": "rf-01912fa8",
    "payment_id": "payment-01912f95",
    "amount": 1094000,
    "status": "REQUESTED",
    "payment_status_after": "REFUNDED"
  },
  "meta": { "request_id": "01912fa9-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Tổng refund **không vượt** số đã capture → `409 REFUND_AMOUNT_INVALID`. Refund một phần → payment `PARTIALLY_REFUNDED`; refund hết → `REFUNDED`. State cuối chỉ chốt sau xác nhận provider.

### 3.5 Seller wallet/ledger/revenue

`GET /seller/wallet`:

```json
{
  "data": {
    "wallet_id": "wl-01912fb5",
    "shop_id": "shop-01912f31",
    "available_balance": 12500000,
    "pending_balance": 3200000,
    "currency": "VND",
    "status": "ACTIVE",
    "as_of": "2026-08-31T04:00:00Z"
  },
  "meta": { "request_id": "01912fb6-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`pending_balance` = tiền đã ghi nhận nhưng chưa qua settlement (chưa rút được). `status` ∈ `ACTIVE | FROZEN | CLOSED`; `FROZEN` vẫn xem được số dư nhưng payout trả `403 WALLET_FROZEN`.

`GET /seller/wallet/ledger?page=&size=&from=&to=&type=`:

```json
{
  "data": [
    {
      "entry_id": "le-01912fb7",
      "posting_id": "ps-01912fb8",
      "entry_type": "CREDIT",
      "amount": 1094000,
      "balance_after": 12500000,
      "reference": { "type": "ORDER", "id": "order-01912f91" },
      "description": "Doanh thu đơn TC-20260830-0001",
      "created_at": "2026-08-30T09:03:12Z"
    }
  ],
  "meta": { "request_id": "01912fb9-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 340 }
}
```

Ledger là **append-only** — không có endpoint sửa/xoá. Chỉ trả ledger của shop trong token; PII và thông tin thanh toán luôn masked.
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

`GET /seller/revenue/export?from=&to=&format=xlsx|csv` — phục vụ Penpot `CTA / Tải báo cáo` ở Seller Finance. Cùng bộ query với `GET /seller/revenue` (không có `granularity`, export luôn theo `DAY`). Response `200`:

```json
{
  "data": {
    "export_url": "https://storage.example/signed-download/revenue-shop-01912f31-202608.xlsx",
    "format": "xlsx",
    "row_count": 31,
    "generated_at": "2026-08-31T04:00:00Z",
    "expires_at": "2026-08-31T04:30:00Z"
  },
  "meta": { "request_id": "01912fc3-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Cùng convention "signed URL, không stream qua Gateway" với `product-catalog`/`order-commerce` export. Cột export: `period, gross, commission, tax, net, refunded, order_count`. `from..to` tối đa 366 ngày, giống `GET /seller/revenue`.

### 3.6 `POST /seller/payouts`

Header `Idempotency-Key` + step-up 2FA. Body:

```json
{ "amount": 5000000, "bank_account_id": "ba-01912fc0", "reason": "Rút doanh thu tháng 8" }
```

```json
{
  "data": {
    "payout_id": "po-01912fc1",
    "shop_id": "shop-01912f31",
    "amount": 5000000,
    "currency": "VND",
    "status": "REQUESTED",
    "bank_account_masked": "VCB ****3021",
    "requested_at": "2026-08-31T04:10:00Z"
  },
  "meta": { "request_id": "01912fc2-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Response `202`. Điều kiện (kiểm theo đúng thứ tự này): KYC projection `APPROVED` → wallet `ACTIVE` → `amount` ≤ `available_balance` → `amount` ≥ ngưỡng tối thiểu. Sai điều kiện → `403 PAYOUT_NOT_ALLOWED` / `409 WALLET_INSUFFICIENT_BALANCE`. Wallet bị **debit trước** khi gọi bank adapter (tránh rút trùng); adapter fail thì hoàn lại bằng posting bù, không sửa ngược ledger cũ.

### 3.7 Reconciliation/health

`GET /admin/payments/reconciliation` — query `provider?`, `status?`, `from?`, `to?`, `page`, `size`:

```json
{
  "data": [
    {
      "payment_id": "payment-01912fa1",
      "provider": "VNPAY",
      "local_status": "SUCCESS",
      "provider_status": "success",
      "amount": 1094000,
      "match": true,
      "checked_at": "2026-08-30T09:10:00Z"
    },
    {
      "payment_id": "payment-01912fa2",
      "provider": "VNPAY",
      "local_status": "PENDING",
      "provider_status": "success",
      "amount": 500000,
      "match": false,
      "mismatch_reason": "LOCAL_STALE",
      "checked_at": "2026-08-30T09:10:00Z"
    }
  ],
  "meta": { "request_id": "01912fa3-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 2, "mismatch_count": 1 }
}
```

`match:false` không tự sửa state — chỉ báo cáo cho ops điều tra. Không trả raw provider secret/signature. `/health/live` process-only; `/health/ready` MySQL/Kafka/config/VNPAY secret availability.

### 3.8 Admin finance back-office (`FINANCE_OPS`)

Phục vụ các màn Penpot Admin *Fees/Taxes*, *Finance*, *Seller settlement*, *Settlement batches*. Không có microservice admin riêng (xem `System_Overview.md` §6.3); Gateway coarse-gate role admin, service này enforce `FINANCE_OPS` + step-up 2FA cho mutation. Mọi mutation ghi `audit_logs` (actor/reason).

`GET /admin/finance/summary/export?from=&to=&format=xlsx|csv` — phục vụ Penpot `CTA / Xuất báo cáo` ở Admin Fees/Taxes. Cùng dữ liệu nguồn với `GET /admin/finance/summary`, xuất theo ngày. Response `200` cùng hình dạng với `GET /seller/revenue/export` ở trên (`export_url`/`format`/`row_count`/`generated_at`/`expires_at`). Cột export: `period, gmv, commission_income, tax_collected, refund_amount, payout_volume, shop_count`. Chỉ `FINANCE_OPS`; không có tham số `shop_id` — đây là tổng hợp toàn sàn.

**Fee/Tax config — `GET/PUT /admin/fees`, `GET/PUT /admin/taxes`**

- Config là **effective-dated, append-only**: `PUT` tạo version mới `{scope: PLATFORM|CATEGORY, category_id?, rate_bps, effective_from, note}`; không sửa/xóa version cũ.
- `AllocationService` luôn chọn version có `effective_from` ≤ thời điểm tạo allocation; đổi rate **không** hồi tố allocation/ledger đã ghi.
- `PUT` với `effective_from` trong quá khứ → `409 FEE_CONFIG_INVALID`.

`GET /admin/fees`:

```json
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
