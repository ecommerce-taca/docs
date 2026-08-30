# Test Plan — Payment-Wallet Service

> Nguồn: `docs/lld/payment-wallet.md` · `docs/db/payment-wallet.md` · `docs/api/payment-wallet.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers. |
| Dependency | MySQL 8.4, Kafka, VNPAY sandbox/mock, Order/Auth/Shipment doubles. |
| Payment data | Pending/success/failed/expired VNPAY; COD pending/success; amount mismatch. |
| Ledger data | Balanced double-entry postings, commission/tax/net, payout/refund states. |
| Security | Valid/invalid signature, duplicate/out-of-order webhook, KYC approved/expired/frozen wallet. |
| Telemetry | JSON logs, traceparent, X-Request-ID, metrics; scan secret/PII leakage. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Buyer payment | VNPAY QR pending→success/failed | Không hiển thị success trước callback hợp lệ. |
| COD | Pending→delivered/collection | Payment success đúng event; không gọi VNPAY. |
| Webhook | Duplicate, wrong signature, amount mismatch | ACK/reject đúng; không double post. |
| Seller wallet | Pending/available ledger, payout | Balance/commission/tax rõ; payout cần KYC/step-up. |
| Refund | Partial/full/over captured | Không refund quá amount; state/ledger đúng. |
| UI states | Loading/pending/failed/retry | Retry idempotent, không charge hai lần. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| P-API-01 | `POST /payments` VNPAY valid | `201 PENDING`, QR/URL, no false success. |
| P-API-02 | `POST /payments` COD valid | `201 PENDING_COD`, no provider call. |
| P-API-03 | Amount/method/order mismatch | `400/409 PAYMENT_AMOUNT_MISMATCH`; no payment. |
| P-API-04 | Same idempotency payload/different payload | Same result / `409 PAYMENT_IDEMPOTENCY_CONFLICT`. |
| P-API-05 | `GET /payments/{id}` own/other scope | Correct data or `403 PAYMENT_FORBIDDEN`; no secret. |
| P-API-06 | Webhook valid success | `200`, payment success, balanced ledger/outbox once. |
| P-API-07 | Webhook wrong signature/amount/old timestamp | `400 PAYMENT_WEBHOOK_INVALID`; no state change. |
| P-API-08 | Duplicate provider event | Idempotent ACK; no duplicate ledger/allocation. |
| P-API-09 | Refund partial/full/over amount | Correct state; over amount `REFUND_AMOUNT_INVALID`. |
| P-API-10 | Seller wallet/ledger scope | Own shop only; masked fields/pagination. |
| P-API-10b | `GET /seller/revenue?from=&to=&granularity=` | Summary `gross=commission+tax+net−refunded` reconcile với ledger; chỉ shop của actor; range >366 ngày → `400`; không tạo ledger mới. |
| P-API-11 | Payout KYC not approved/frozen/low balance | `403/409`; no debit/provider call. |
| P-API-12 | Payout retry same idempotency | One payout/debit; same result. |
| P-API-13 | Reconciliation/admin permission | `FINANCE_OPS` sees mismatch summary; non-admin/không đủ permission denied. |
| P-API-13b | `PUT /admin/fees`/`/admin/taxes` tạo version, `effective_from` quá khứ | Version mới append (không sửa bản cũ); quá khứ → `409 FEE_CONFIG_INVALID`; thiếu 2FA → `401 AUTH_MFA_REQUIRED`; allocation cũ giữ nguyên rate. |
| P-API-13c | `GET /admin/settlements` + `{batchId}` + `retry` | List/detail đúng breakdown theo shop; `retry` batch `COMPLETED` → `409 SETTLEMENT_BATCH_INVALID_STATE`; `retry` batch `FAILED` idempotent; non-`FINANCE_OPS` denied. |
| P-API-13d | `GET /admin/finance/summary` | Read-only aggregate khớp tổng `payment_allocations`/`payouts`; range >366 ngày → `400`; không tạo ledger. |
| P-API-14 | Health live/ready | Live independent DB; ready checks dependencies. |

### 3.1 Contract/security/reliability

| ID | Case | Expected |
|---|---|---|
| P-INT-01 | Payment event duplicate/out-of-order | One order transition/ledger posting. |
| P-INT-02 | Kafka down after DB commit | Outbox retry/DLQ; local payment correct. |
| P-SEC-01 | Signature algorithm/key injection | Reject; only configured VNPAY algorithm/key. |
| P-SEC-02 | Log scan | No secret, card/bank credential, callback raw, token, PII. |
| P-RES-01 | DB deadlock/provider timeout | Bounded recovery; no duplicate charge/posting. |

## 4. Unit test

| Module | Cases |
|---|---|
| Provider adapter | Signature, amount/order match, timeout, sandbox response. |
| State machine | Pending/success/fail/expire/refund transitions. |
| Allocation | Gross=commission+tax+net, deterministic rounding. |
| Ledger | Debit=credit, non-negative wallet, immutable entries. |
| Payout/refund | KYC/balance/captured amount, idempotency. |
| Revenue report | Aggregate theo range/granularity, reconcile với allocation/ledger, không tạo posting, giới hạn range. |
| Fee/tax config | Chọn version theo `effective_from` ≤ allocation time; reject `effective_from` quá khứ; append-only. |
| Settlement | Batch tổng = Σ item; item release chỉ tạo posting cân bằng; `retry` chỉ khi `FAILED`; idempotent theo `batchId`. |
| Observability | Required JSON fields, trace propagation, redaction, bounded metrics. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | VNPAY sandbox v1; merchant config mock/secret manager. | Production provider tests bổ sung sau. | Finance/Security |
| 2 | COD confirmation từ Shipment/Order. | Cần golden settlement cases. | Order/Shipment |
| 3 | Commission/tax/rounding rate chưa chốt. | Ảnh hưởng ledger expected totals. | Finance |
| 4 | Payout provider chưa chốt. | Tạm mock/reconciliation. | Finance/DevOps |
