# API Spec — Payment-Wallet Service

> Nguồn: `docs/lld/payment-wallet.md` · `docs/db/payment-wallet.md` · ADR-001…004 trong service repo · Code: `ecommerce-taca/payment-wallet-service` @ `main` (`859b7b1`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — payment/refund, seller wallet/ledger/revenue, admin finance, health/Prometheus đã có code; `GET /seller/revenue/export`, `GET /seller/payouts` chưa có, `POST /seller/payouts` bị chặn theo ADR-003.
> Base path: `/api/v1` (business/admin) · health và `/prometheus` ở `/` · VNPAY sandbox + COD

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Buyer/seller/admin JWT qua Gateway. Payment-Wallet đọc actor context do Gateway inject. Internal caller `ORDER_COMMERCE` (enum `InternalService` trong code; từ 2026-11-01 là `order-service`, xem `order-docs/docs/lld/order.md` §1.1) được kiểm tra bằng `InternalCallerPolicy`; transport production để populate `InternalCallerContext` vẫn là contract mở, không tự suy diễn header service-auth mới. |
| Actor context | Đọc `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope` do Gateway inject. |
| Seller scope | Seller Finance suy ra shop từ actor context. Một seller actor phải có đúng một shop trong `shopScope`; seller APIs không nhận `shop_id` từ query/body để chọn shop. |
| Request ID | `X-Request-ID` bắt buộc ở các business/admin HTTP APIs hiện đã expose theo contract tương ứng. Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate` được validate, tạo/continue và propagate qua REST, outbox và Kafka metadata. |
| Time/money | ISO-8601 UTC; integer VND, không dùng FLOAT cho tiền. |
| Idempotency | Create Payment và Request Refund dùng `Idempotency-Key`; provider webhook dedupe theo provider event. |
| Response | Success business API: `{data,meta:{request_id}}`; error: `{error:{code,message},meta:{request_id}}`. |
| Pagination | API có phân trang dùng `page` từ 1, `size` mặc định 20 tối đa 100; meta có thể trả `page,size,total,total_pages`. |
| Redaction | Không trả/log token, VNPAY signature/secret, raw provider payload, bank/card credential hoặc PII nhạy cảm. |
| Operational endpoints | Health và Prometheus là operational surfaces, không dùng base path `/api/v1`. |

### 1.1 Phạm vi đã triển khai

Phase 6 triển khai và kiểm thử:

- `POST /api/v1/payments`
- `GET /api/v1/payments/{paymentId}`
- `POST /api/v1/payments/webhook`
- `POST /api/v1/payments/{paymentId}/refunds`

Phase 7 triển khai và kiểm thử:

- `GET /api/v1/seller/wallet`
- `GET /api/v1/seller/wallet/ledger`
- `GET /api/v1/seller/revenue`

Các target endpoint seller chưa được runtime expose:

- `GET /api/v1/seller/revenue/export`
- `POST /api/v1/seller/payouts`
- `GET /api/v1/seller/payouts`

`POST /seller/payouts` bị production-gate vì các contract payout quan trọng vẫn OPEN theo ADR-003.

Phase 8 triển khai Admin Finance runtime APIs:

- reconciliation;
- fee configuration;
- tax configuration;
- settlement read/retry;
- finance summary/export.

Phase 9 triển khai operational surfaces:

- `GET /health/live`
- `GET /health/liveness`
- `GET /health/ready`
- `GET /health/readiness`
- `GET /prometheus`

Phase 9 không tạo `/api/v1/admin/operations`.

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `POST /payments` | Trusted internal caller `ORDER_COMMERCE` (`order-service`) | Tạo payment intent VNPAY/COD | ✅ |
| 2 | `GET /payments/{paymentId}` | Buyer owner, internal `ORDER_COMMERCE` (`order-service`), `FINANCE_OPS` | Xem payment detail | ✅ |
| 3 | `POST /payments/webhook` | VNPAY provider, không JWT | Reconcile callback | ✅ |
| 4 | `POST /payments/{paymentId}/refunds` | Internal `ORDER_COMMERCE` (`order-service`) hoặc `FINANCE_OPS` | Tạo refund intent | ✅ |
| 5 | `GET /seller/wallet` | Seller | Xem available/pending balance | ✅ |
| 6 | `GET /seller/wallet/ledger` | Seller | Xem ledger | ✅ |
| 7 | `GET /seller/revenue` | Seller | Báo cáo doanh thu | ✅ |
| 7a | `GET /seller/revenue/export` | Seller | Xuất báo cáo doanh thu | ⏳ Chưa có controller |
| 8 | `POST /seller/payouts` | Seller + step-up | Yêu cầu rút tiền | ⏳ Bị chặn (ADR-003) |
| 9 | `GET /seller/payouts` | Seller | Xem payout history | ⏳ Chưa có controller |
| 10 | `GET /admin/payments/reconciliation` | `FINANCE_OPS` | Reconcile provider/payment/ledger | ✅ |
| 11 | `GET /admin/fees` | `FINANCE_OPS` | Danh sách version commission | ✅ |
| 12 | `PUT /admin/fees` | `FINANCE_OPS` + step-up | Tạo commission version mới | ✅ |
| 13 | `GET /admin/taxes` | `FINANCE_OPS` | Danh sách version thuế | ✅ |
| 14 | `PUT /admin/taxes` | `FINANCE_OPS` + step-up | Tạo tax version mới | ✅ |
| 15 | `GET /admin/settlements` | `FINANCE_OPS` | Danh sách settlement batch | ✅ |
| 16 | `GET /admin/settlements/{batchId}` | `FINANCE_OPS` | Chi tiết settlement batch | ✅ |
| 17 | `POST /admin/settlements/{batchId}/retry` | `FINANCE_OPS` + step-up | Retry batch `FAILED` | ✅ |
| 18 | `GET /admin/finance/summary` | `FINANCE_OPS` | Tổng hợp tài chính sàn | ✅ |
| 18a | `GET /admin/finance/summary/export` | `FINANCE_OPS` | Xuất báo cáo tài chính | ✅ |
| 19 | `GET /health/live` | Ops | Liveness | ✅ |
| 19a | `GET /health/liveness` | Ops | Conventional liveness probe | ✅ |
| 20 | `GET /health/ready` | Ops | Readiness | ✅ |
| 20a | `GET /health/readiness` | Ops | Conventional readiness probe | ✅ |
| 21 | `GET /prometheus` | Ops/monitoring | Metrics Prometheus | ✅ |
| 22 | `GET /admin/finance/summary/exports/{token}` | Token tạm do 18a cấp (method không tự kiểm actor; Gateway chỉ route admin) | Tải file CSV do 18a sinh, hết hạn sau 30 phút | ✅ |

> `POST /payments` và `POST /payments/{paymentId}/refunds` không dùng prefix `/internal/**`. Service tự enforce caller. Không được tự tạo một header service-auth mới như `X-Internal-Service` khi contract transport chưa được chốt.

> Health/Prometheus endpoints là operational surfaces ở management base path `/`; chúng không thuộc business base path `/api/v1`.

## 3. Chi tiết endpoint

### 3.1 `POST /payments`

Header bắt buộc:

- `Idempotency-Key`
- `X-Request-ID`

Authorization:

- chỉ trusted internal caller `ORDER_COMMERCE` (`order-service`).

Request contract runtime hiện tại:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `checkout_group_id` | UUID | Có | Một payment đại diện cho một checkout group |
| `buyer_user_id` | UUID | Có | Buyer của checkout |
| `method` | enum | Có | `VNPAY` hoặc `COD` |
| `amount` | integer | Có | > 0 |
| `currency` | string | Có | Chỉ `"VND"` |
| `orders` | array | Có | Ít nhất một child order |
| `orders[].order_id` | UUID | Có | ID order |
| `orders[].shop_id` | UUID | Có | Shop sở hữu order |
| `orders[].amount` | integer | Có | Grand total của child order |
| `orders[].shipping_fee` | integer | Có | `>= 0` và `< amount` |

Ràng buộc:

- tổng `orders[].amount` phải bằng `amount`;
- `merchandise_amount` không phải request field, được suy ra bằng `amount - shipping_fee`;
- VNPAY có `expires_at` do service tính;
- COD có `expires_at = null`.

> Theo ADR-002, `orders[]` trong request hiện tại là implementation debt. Canonical contract tương lai sẽ dùng authoritative Order snapshot thay vì tin allocation do caller gửi. Phase 6 không tự triển khai `OrderSnapshotPort` khi upstream contract chưa hoàn tất.

Request example:

```json
{
  "checkout_group_id": "01912f90-7a1b-7c12-9c55-8b1c34a6d921",
  "buyer_user_id": "01912f80-7a1b-7c12-9c55-8b1c34a6d921",
  "method": "VNPAY",
  "amount": 1094000,
  "currency": "VND",
  "orders": [
    {
      "order_id": "01912f91-7a1b-7c12-9c55-8b1c34a6d921",
      "shop_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
      "amount": 1094000,
      "shipping_fee": 20000
    }
  ]
}
```

Response `201 Created`:

```json
{
  "data": {
    "payment_id": "01912f95-7a1b-7c12-9c55-8b1c34a6d921",
    "checkout_group_id": "01912f90-7a1b-7c12-9c55-8b1c34a6d921",
    "status": "PENDING",
    "method": "VNPAY",
    "amount": 1094000,
    "currency": "VND",
    "payment_url": "https://sandbox.vnpayment.vn/paymentv2/vpcpay.html?...",
    "expires_at": "2026-10-10T10:15:00Z"
  },
  "meta": {
    "request_id": "01912fa6-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

COD trả `status: "PENDING_COD"` và không có `payment_url`/`expires_at`.

Tạo payment URL không đồng nghĩa thanh toán thành công. Payment chỉ chuyển state tài chính khi workflow tương ứng được xác nhận.

### 3.2 `GET /payments/{paymentId}`

Header bắt buộc:

- `X-Request-ID`

Authorization hiện tại:

- Internal caller `ORDER_COMMERCE` (`order-service`): allow;
- `FINANCE_OPS`: allow;
- buyer: chỉ buyer sở hữu payment;
- seller/seller staff: generic payment detail bị từ chối.

Seller không được lọc trực tiếp từ generic payment response vì response chứa payment-level totals của checkout có thể gồm nhiều shop.

Response `200 OK`:

```json
{
  "data": {
    "payment_id": "01912f95-7a1b-7c12-9c55-8b1c34a6d921",
    "checkout_group_id": "01912f90-7a1b-7c12-9c55-8b1c34a6d921",
    "status": "SUCCESS",
    "method": "VNPAY",
    "amount": 1094000,
    "currency": "VND",
    "captured_amount": 1094000,
    "refunded_amount": 0,
    "expires_at": "2026-10-10T10:15:00Z",
    "paid_at": "2026-10-10T10:03:12Z",
    "orders": [
      {
        "order_id": "01912f91-7a1b-7c12-9c55-8b1c34a6d921",
        "shop_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
        "merchandise_amount": 1074000,
        "shipping_fee": 20000,
        "amount": 1094000
      }
    ]
  },
  "meta": {
    "request_id": "01912fa7-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

`expires_at` hoặc `paid_at` có thể vắng mặt khi `null`.

Response không expose:

- `buyer_user_id`;
- raw provider payload;
- provider signature/secret;
- unmasked provider reference;
- `created_at` khi application result hiện chưa cung cấp field này.

Errors chính:

- malformed `paymentId` → `400 PAYMENT_INVALID_INPUT`;
- payment không tồn tại → `404 PAYMENT_NOT_FOUND`;
- chưa xác thực → `401 PAYMENT_UNAUTHENTICATED`;
- không đủ quyền → `403 PAYMENT_FORBIDDEN`.

### 3.3 `POST /payments/webhook`

Không JWT.

Request runtime hiện tại:

```json
{
  "provider_event_id": "vnpay-event-001",
  "provider_transaction_ref": "vnpay-txn-001",
  "response_code": "00",
  "transaction_status": "00",
  "amount": 1094000,
  "currency": "VND",
  "payload_hash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "signed_payload": {
    "vnp_TxnRef": "vnpay-txn-001",
    "vnp_Amount": "109400000",
    "vnp_ResponseCode": "00",
    "vnp_TransactionStatus": "00",
    "vnp_SecureHash": "<signature>"
  }
}
```

Response `200 OK`:

```json
{
  "data": {
    "payment_id": "01912f95-7a1b-7c12-9c55-8b1c34a6d921",
    "payment_status": "SUCCESS",
    "action": "APPLIED"
  },
  "meta": {
    "request_id": "01912fa7-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Duplicate event trả `action: "DUPLICATE"` và không tạo financial side effect lần hai.

Quy tắc:

- verify VNPAY signature;
- dedupe `provider_event_id`;
- tra payment qua payment attempt/provider transaction ref;
- webhook amount/currency phải khớp intent nội bộ;
- success hợp lệ cập nhật payment, allocation, ledger, wallet, provider event và outbox trong transaction;
- duplicate callback không ghi ledger/allocation/outbox lần hai;
- không expose hoặc log raw secret/signature.

Known open item:

- authoritative VNPAY IP allowlist policy chưa được chốt đầy đủ;
- không tự hard-code policy production nếu chưa có nguồn chính thức.

Errors chính:

- signature sai → `400 INVALID_VNPAY_SIGNATURE`;
- payment attempt không tồn tại → `404 PAYMENT_ATTEMPT_NOT_FOUND`;
- amount mismatch → `409 PAYMENT_AMOUNT_MISMATCH`.

### 3.4 Refund

`POST /payments/{paymentId}/refunds`

Headers bắt buộc:

- `Idempotency-Key`
- `X-Request-ID`

Authorized callers:

- trusted internal caller `ORDER_COMMERCE` (`order-service`);
- authenticated actor có role `FINANCE_OPS`.

Buyer và seller client không gọi trực tiếp endpoint refund này.

Request:

```json
{
  "amount": 50000,
  "reason": "BUYER_CANCELLED"
}
```

Ràng buộc:

- `amount > 0`;
- `reason` bắt buộc, tối đa 500 ký tự;
- HTTP caller không truyền `currency`; refund hiện dùng `VND`;
- contract runtime hiện tại là payment-level refund, không nhận `order_id`;
- tổng `refunded + pending refund + requested refund` không vượt `captured_amount`;
- payment phải ở state cho phép refund.

Response `202 Accepted`:

```json
{
  "data": {
    "refund_id": "01912fa8-7a1b-7c12-9c55-8b1c34a6d921",
    "payment_id": "01912f95-7a1b-7c12-9c55-8b1c34a6d921",
    "amount": 50000,
    "currency": "VND",
    "status": "REQUESTED"
  },
  "meta": {
    "request_id": "01912fa9-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Tạo refund request không đồng nghĩa refund đã thành công.

State payment chỉ chuyển:

- `PARTIALLY_REFUNDED`, hoặc
- `REFUNDED`

sau khi refund-result workflow xác nhận thành công.

Idempotency behavior:

- cùng `Idempotency-Key` + cùng request → trả lại cùng refund result, không tạo refund/outbox thứ hai;
- cùng `Idempotency-Key` + payload khác → `409 PAYMENT_IDEMPOTENCY_CONFLICT`;
- request cùng key đang `PROCESSING` → `409 REQUEST_ALREADY_PROCESSING`.

Errors chính:

- payment không tồn tại → `404 PAYMENT_NOT_FOUND`;
- payment state không cho refund → `409 REFUND_STATE_INVALID`;
- refund vượt captured/pending limit → `409 REFUND_AMOUNT_INVALID`;
- chưa xác thực → `401 PAYMENT_UNAUTHENTICATED`;
- không đủ quyền → `403 PAYMENT_FORBIDDEN`.

### 3.5 Seller wallet/ledger/revenue

Phase 7 runtime hiện implement:

- `GET /seller/wallet`;
- `GET /seller/wallet/ledger`;
- `GET /seller/revenue`.

Seller authorization:

- actor phải được xác thực;
- actor phải có role `SELLER`;
- actor phải có đúng một shop trong `shopScope`;
- shop không được lấy từ query/body của client;
- seller finance query luôn scope theo shop của actor.

#### 3.5.1 `GET /seller/wallet`

Header bắt buộc:

- `X-Request-ID`

Response `200 OK`:

```json
{
  "data": {
    "wallet_id": "01912fb5-7a1b-7c12-9c55-8b1c34a6d921",
    "shop_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
    "available_balance": 12500000,
    "pending_balance": 3200000,
    "currency": "VND",
    "status": "ACTIVE",
    "as_of": "2026-08-31T04:00:00Z"
  },
  "meta": {
    "request_id": "01912fb6-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

`pending_balance` là tiền đã ghi nhận nhưng chưa qua settlement.

`status` ∈:

- `ACTIVE`
- `FROZEN`
- `CLOSED`

`as_of` phản ánh thời điểm cập nhật wallet projection.

Errors chính:

- thiếu `X-Request-ID` → `400 MISSING_REQUIRED_HEADER`;
- chưa xác thực → `401 PAYMENT_UNAUTHENTICATED`;
- actor không phải seller hoặc shop scope không đúng một shop → `403 PAYMENT_FORBIDDEN`;
- wallet không tồn tại → `404 WALLET_NOT_FOUND`.

#### 3.5.2 `GET /seller/wallet/ledger?page=&size=&from=&to=&type=`

Header bắt buộc:

- `X-Request-ID`

Query:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `page` | integer | Không | Mặc định `1`, tối thiểu `1` |
| `size` | integer | Không | Mặc định `20`, tối đa `100` |
| `from` | ISO-8601 Instant | Không | Lower bound |
| `to` | ISO-8601 Instant | Không | Upper bound |
| `type` | enum | Không | `DEBIT` hoặc `CREDIT` |

Response `200 OK`:

```json
{
  "data": [
    {
      "entry_id": "01912fb7-7a1b-7c12-9c55-8b1c34a6d921",
      "posting_id": "01912fb8-7a1b-7c12-9c55-8b1c34a6d921",
      "entry_type": "CREDIT",
      "amount": 1062000,
      "balance_after": 12500000,
      "reference": {
        "type": "PAYMENT",
        "id": "01912f95-7a1b-7c12-9c55-8b1c34a6d921"
      },
      "description": "Seller revenue",
      "created_at": "2026-08-30T09:03:12Z"
    }
  ],
  "meta": {
    "request_id": "01912fb9-7a1b-7c12-9c55-8b1c34a6d921",
    "page": 1,
    "size": 20,
    "total": 340,
    "total_pages": 17
  }
}
```

Ledger append-only, không có endpoint sửa/xóa.

Seller chỉ nhìn thấy ledger entries thuộc chính shop trong actor scope.

Không có entry trong range không phải lỗi; API trả:

```json
{
  "data": [],
  "meta": {
    "request_id": "...",
    "page": 1,
    "size": 20,
    "total": 0,
    "total_pages": 0
  }
}
```

Errors chính:

- page/size không hợp lệ → `400 INVALID_REQUEST`;
- `from`/`to` malformed hoặc range không hợp lệ → `400 INVALID_REQUEST`;
- `type` khác `DEBIT`/`CREDIT` → `400 INVALID_REQUEST`;
- chưa xác thực → `401 PAYMENT_UNAUTHENTICATED`;
- seller/shop scope không hợp lệ → `403 PAYMENT_FORBIDDEN`.

#### 3.5.3 `GET /seller/revenue?from=&to=&granularity=DAY|WEEK|MONTH`

Header bắt buộc:

- `X-Request-ID`

Query:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `from` | ISO-8601 date | Có | Ngày bắt đầu |
| `to` | ISO-8601 date | Có | Ngày kết thúc |
| `granularity` | enum | Có | `DAY`, `WEEK`, `MONTH` |

Response `200 OK`:

```json
{
  "data": {
    "range": {
      "from": "2026-08-01",
      "to": "2026-08-31",
      "granularity": "DAY"
    },
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
      {
        "period": "2026-08-01",
        "gross": 4200000,
        "commission": 294000,
        "tax": 42000,
        "net": 3864000,
        "order_count": 12
      }
    ]
  },
  "meta": {
    "request_id": "01912fc0-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Revenue behavior:

- payment allocations được aggregate theo seller shop;
- refund allocations được aggregate vào `summary.refunded`;
- dữ liệu của shop khác không xuất hiện;
- supported granularity: `DAY`, `WEEK`, `MONTH`;
- range không có dữ liệu trả summary bằng `0` và `buckets: []`.

Errors chính:

- thiếu `from`, `to` hoặc `granularity` → `400 MISSING_REQUIRED_PARAMETER`;
- malformed date → `400 INVALID_REQUEST`;
- reversed/range không hợp lệ → `400 INVALID_REQUEST`;
- unsupported granularity → `400 INVALID_REQUEST`;
- chưa xác thực → `401 PAYMENT_UNAUTHENTICATED`;
- seller/shop scope không hợp lệ → `403 PAYMENT_FORBIDDEN`.

#### 3.5.4 `GET /seller/revenue/export?from=&to=&format=xlsx|csv`

Target contract:

```json
{
  "data": {
    "export_url": "https://storage.example/signed-download/revenue-shop-01912f31-202608.xlsx",
    "format": "xlsx",
    "row_count": 31,
    "generated_at": "2026-08-31T04:00:00Z",
    "expires_at": "2026-08-31T04:30:00Z"
  },
  "meta": {
    "request_id": "01912fc3-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Runtime status:

- target contract được giữ trong tài liệu;
- endpoint chưa được implement/expose trong Phase 7 hiện tại;
- không fabricate `export_url`;
- không coi local temp file là production signed-download contract.

### 3.6 Seller payout

#### 3.6.1 `POST /seller/payouts`

> Target contract. Production exposure hiện bị gate bởi ADR-003.

Headers mục tiêu:

- `Idempotency-Key`
- `X-MFA-Step-Up`
- `X-Request-ID`

Body:

```json
{
  "amount": 5000000,
  "bank_account_id": "bank_account-01912fc0",
  "reason": "Rút doanh thu tháng 8"
}
```

Response target:

```json
{
  "data": {
    "payout_id": "01912fc1-7a1b-7c12-9c55-8b1c34a6d921",
    "shop_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
    "amount": 5000000,
    "currency": "VND",
    "status": "REQUESTED",
    "bank_account_masked": "VCB ****3021",
    "requested_at": "2026-08-31T04:10:00Z"
  },
  "meta": {
    "request_id": "01912fc2-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Runtime status:

- payout domain/application/persistence foundation tồn tại;
- production `RequestPayoutUseCase` bean không được đăng ký;
- `SellerPayoutController` không được expose;
- `PayoutContractGateTest` bảo vệ các invariant này.

Các contract OPEN trong ADR-003:

- shop payout-gate projection;
- authoritative source/ownership của `bank_account_id`;
- payout notification recipient.

Cho đến khi các contract này được resolve:

- không fabricate KYC/shop status;
- không fabricate bank-account ownership;
- không dùng `shop_id` thay cho `owner_user_id`;
- không expose production seller payout request endpoint.

#### 3.6.2 `GET /seller/payouts`

Endpoint nằm trong target API inventory nhưng runtime chưa implement.

Không tự suy diễn response shape, pagination hoặc filtering behavior nếu chưa có contract authoritative tương ứng.

### 3.7 Reconciliation and operational endpoints

#### 3.7.1 `GET /admin/payments/reconciliation`

Authorization:

- `FINANCE_OPS`.

The endpoint is read-only.

Example response:

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
    }
  ],
  "meta": {
    "request_id": "01912fa3-7a1b-7c12-9c55-8b1c34a6d921",
    "page": 1,
    "size": 20,
    "total": 1,
    "total_pages": 1,
    "mismatch_count": 0
  }
}
```

`match:false` chỉ phục vụ điều tra.

Reconciliation không tự sửa financial state.

#### 3.7.2 Health

Canonical liveness:

`GET /health/live`

Conventional Spring liveness:

`GET /health/liveness`

Liveness không phụ thuộc vào DB, Kafka, VNPAY hoặc outbox backlog.

Canonical readiness:

`GET /health/ready`

Conventional Spring readiness:

`GET /health/readiness`

Readiness hiện gồm:

- `readinessState`;
- DB;
- Kafka readiness;
- outbox backlog;
- VNPAY configuration readiness.

Dependency failure có thể làm readiness trả HTTP `503` trong khi liveness vẫn
`UP`.

Health details không expose secret.

#### 3.7.3 `GET /prometheus`

Prometheus-compatible operational scrape endpoint.

Runtime Actuator exposure chỉ gồm:

- `health`;
- `info`;
- `prometheus`.

Các endpoint sau không được expose mặc định:

- `/metrics`;
- `/env`;
- `/configprops`;
- `/beans`;
- `/heapdump`;
- `/threaddump`;
- `/loggers`.

`/prometheus` là operational endpoint, không phải business API.

Production deployment nên giới hạn access endpoint này cho monitoring
infrastructure phù hợp.

Phase 9 không tạo custom controller để expose Prometheus.

Phase 9 cũng không tạo:

`/api/v1/admin/operations`

hoặc:

`/api/v1/admin/operations/outbox`

### 3.8 Admin finance back-office (`FINANCE_OPS`)

> Phase 8 runtime contract.

Các nhóm API runtime:

- reconciliation;
- fee configuration;
- tax configuration;
- settlement batch read/retry;
- finance summary/export.

Fee/Tax config là effective-dated và append-only.

Settlement không được production-wire chỉ dựa vào việc allocation chưa xuất hiện trong `settlement_lines`.

Authoritative settlement eligibility vẫn OPEN theo ADR-003.

#### 3.8.1 `GET /admin/finance/summary`

`GET /admin/finance/summary`

Header bắt buộc:

- `X-Request-ID`

Authorization:

- `FINANCE_OPS`.

Response runtime:

```json
{
  "data": {
    "captured_amount": 1000000,
    "refunded_amount": 100000,
    "gross_amount": 900000,
    "commission_amount": 90000,
    "tax_amount": 45000,
    "seller_net_amount": 765000,
    "available_balance": 500000,
    "pending_balance": 265000,
    "completed_payout_amount": 200000,
    "as_of": "2026-10-23T10:00:00Z"
  },
  "meta": {
    "request_id": "req-finance-summary-001"
  }
}
```

#### 3.8.2 `PUT /admin/taxes`

Tạo một tax configuration version mới.

Headers bắt buộc:

- `X-Request-ID`

Authorization:

- `FINANCE_OPS`;
- MFA step-up bắt buộc.

Request:

```json
{
  "scope": "PLATFORM",
  "category_id": null,
  "rate_bps": 100,
  "effective_from": "2026-11-01T00:00:00Z",
  "note": "Platform tax November 2026"
}
```

#### 3.8.3 `POST /admin/settlements/{batchId}/retry`

Retries an existing `FAILED` settlement batch.

Headers:

- `X-Request-ID`

Authorization:

- `FINANCE_OPS`;
- MFA step-up required.

The endpoint has no request body.

Rules:

- only `FAILED` batches may be retried;
- the existing batch, items and settlement lines are reused;
- completed items are never processed again;
- no new settlement candidates are queried;
- no new settlement batch is created;
- retry is transactional.

Response `200 OK`:

```json
{
  "data": {
    "batch_id": "019...",
    "status": "COMPLETED",
    "retried_item_count": 2
  },
  "meta": {
    "request_id": "req-settlement-retry-001"
  }
}
```

#### 3.8.4 `GET /admin/finance/summary/export`

Target export contract for the current admin finance summary.

Authorization:

- `FINANCE_OPS`.

Export format:

- CSV encoded as UTF-8;
- the artifact contains one snapshot row corresponding to the current
  admin finance summary;
- columns are:
  `captured_amount`,
  `refunded_amount`,
  `gross_amount`,
  `commission_amount`,
  `tax_amount`,
  `seller_net_amount`,
  `available_balance`,
  `pending_balance`,
  `completed_payout_amount`,
  `as_of`.

Download contract:

- the endpoint returns a temporary download URL rather than the CSV bytes;
- `EXPORT_URL_TTL` is 30 minutes;
- storage provider details are not part of the API contract;
- object-storage credentials and internal object keys must not be exposed.

Runtime status:

- CSV UTF-8 export is implemented;
- the endpoint requires `FINANCE_OPS`;
- exports are stored temporarily behind an opaque capability token;
- temporary download URLs expire after 30 minutes;
- expired or unknown URLs return `FINANCE_EXPORT_NOT_FOUND`;
- storage provider details are not exposed by the API.

## 4. Mã lỗi chung

### 4.1 Mã lỗi code đang trả

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `PAYMENT_INVALID_INPUT` | 400 | Payment/refund path/header input không hợp lệ |
| `VALIDATION_ERROR` | 400 | Bean Validation thất bại |
| `MISSING_REQUIRED_HEADER` | 400 | Thiếu required header |
| `MISSING_REQUIRED_PARAMETER` | 400 | Thiếu required query parameter |
| `MALFORMED_REQUEST_BODY` | 400 | JSON/request body malformed |
| `INVALID_REQUEST` | 400 | Illegal argument/request fallback |
| `INVALID_VNPAY_SIGNATURE` | 400 | VNPAY signature không hợp lệ |
| `PAYMENT_UNAUTHENTICATED` | 401 | Không có actor/internal caller hợp lệ |
| `AUTH_MFA_REQUIRED` | 401 | Yêu cầu MFA step-up nhưng chưa có |
| `PAYMENT_FORBIDDEN` | 403 | Caller đã xác thực nhưng không đủ quyền |
| `PAYMENT_NOT_FOUND` | 404 | Không tìm thấy payment |
| `PAYMENT_ATTEMPT_NOT_FOUND` | 404 | Không tìm thấy payment attempt cho provider transaction |
| `REFUND_NOT_FOUND` | 404 | Không tìm thấy refund |
| `WALLET_NOT_FOUND` | 404 | Không tìm thấy seller wallet |
| `SETTLEMENT_BATCH_NOT_FOUND` | 404 | Settlement batch không tồn tại |
| `FINANCE_EXPORT_NOT_FOUND` | 404 | Finance export không tồn tại hoặc đã hết hạn |
| `RESOURCE_NOT_FOUND` | 404 | Resource/endpoint không tồn tại hoặc không được expose |
| `PAYMENT_AMOUNT_MISMATCH` | 409 | Provider/payment amount mismatch |
| `PAYMENT_IDEMPOTENCY_CONFLICT` | 409 | Idempotency key được reuse với request khác |
| `REQUEST_ALREADY_PROCESSING` | 409 | Idempotent request cùng key đang xử lý |
| `REFUND_AMOUNT_INVALID` | 409 | Refund vượt captured/pending limit |
| `REFUND_STATE_INVALID` | 409 | Payment state hiện tại không cho phép refund |
| `REFUND_AMOUNT_MISMATCH` | 409 | Refund result amount không khớp refund đã ghi |
| `SETTLEMENT_BATCH_INVALID_STATE` | 409 | Settlement action/retry không hợp lệ với state hiện tại |
| `INTERNAL_ERROR` | 500 | Unexpected server error; không expose internal detail |

### 4.2 Mã lỗi dự phòng (⏳ chưa có endpoint dùng)

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `WALLET_INSUFFICIENT_BALANCE` | 409 | Không đủ available balance |
| `WALLET_FROZEN` | 403 | Wallet frozen |
| `PAYOUT_NOT_ALLOWED` | 403/409 | Payout không thỏa policy |
| `FEE_CONFIG_INVALID` | 409 | Fee/tax config không hợp lệ |

Các code ở mục này không đồng nghĩa endpoint tương ứng đã được runtime expose.

## 5. Giả định & câu hỏi mở

### 5.1 Nợ contract đã biết (theo code + ADR)

| # | Nội dung | Trạng thái / ảnh hưởng |
|---:|---|---|
| 1 | Create-payment canonical Order snapshot | ADR-002 đã Accepted nhưng runtime vẫn dùng `orders[]`; đây là implementation debt. |
| 2 | Internal service-auth transport | `InternalCallerPolicy` đã có nhưng production transport để populate `InternalCallerContext` chưa được chốt; không fabricate service header. |
| 3 | VNPAY IP policy | Signature verification đã có; authoritative allowlist policy vẫn cần nguồn chính thức. |
| 4 | Provider refund transport/worker | Refund HTTP/application foundation đã có; authoritative provider refund transport vẫn là contract riêng. |
| 5 | Buyer notification recipient | `buyer.email` chưa có authoritative source theo ADR-003; không fabricate email. |
| 6 | Seller revenue export | Target contract tồn tại nhưng runtime endpoint chưa được implement. |
| 7 | Seller payout gate projection | KYC/shop-status projection chưa được chốt theo ADR-003. |
| 8 | Bank account source | Ownership/source của `bank_account_id` chưa được chốt theo ADR-003. |
| 9 | Payout notification recipient | `owner_user_id` và `owner_email` chưa có authoritative source theo ADR-003. |
| 10 | Seller payout history | Target endpoint có trong API inventory nhưng runtime chưa implement. |
| 11 | Settlement trigger/eligibility | Hold window, eligible timestamp và upstream trigger để tạo settlement batch mới vẫn OPEN theo ADR-003. Production `SettlementCandidatePort` và `RunSettlementUseCase` tiếp tục không được wire. Admin retry semantics cho persisted `FAILED` batch đã được resolve và runtime endpoint đã implement. |
| 12 | Admin finance summary export storage scaling | Runtime hiện dùng temporary in-process storage. Multi-instance deployment có thể thay bằng shared object storage mà không thay đổi application/API contract. |
| 13 | External tracing backend | Phase 9 implement W3C trace propagation nhưng chưa yêu cầu Tempo/Zipkin/Jaeger/OpenTelemetry Collector exporter. |
| 14 | Monitoring deployment topology | `/prometheus` runtime endpoint đã có; production scrape network/access policy thuộc deployment hardening. |

### 5.2 Giả định nghiệp vụ

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | VNPAY sandbox v1; production merchant config chưa có. | Cần đổi provider/security contract. | Finance/Security |
| 2 | COD success dựa Shipment/Order collection event (ADR-004). | Ảnh hưởng settlement. | Order/Shipment |
| 3 | Payout provider chưa chốt; `POST /seller/payouts` chưa expose. | Seller chưa rút tiền được qua API. | Finance/DevOps |
| 4 | Return/dispute workflow ngoài v1: refund chỉ khởi tạo bởi `order-service` hoặc `FINANCE_OPS`. | Cần thêm permission/state khi bật dispute. | Product/Finance |
| 5 | Event payout (`payout.requested/succeeded/failed`) trong code **không** có `owner_user_id`/`owner_email` (xem LLD §6.1). Notification chưa gửi được mail payout cho seller. | Notification thiếu người nhận. | Payment-Wallet + Notification |
| 6 | Admin back-office (Fees/Taxes, Finance, Settlement) nằm trên service này dưới `/api/v1/admin/**`, không tách microservice admin. | Đổi ownership phải đổi contract allocation. | Architecture + Finance |
| 7 | `PUT /admin/fees`/`/admin/taxes` không chặn `effective_from` trong quá khứ; `FEE_CONFIG_INVALID` chưa được trả. | Có thể tạo version hồi tố. | Finance |
