# Test Plan — Order-Commerce Service

> Nguồn: `docs/lld/order-commerce.md` · `docs/db/order-commerce.md` · `docs/api/order-commerce.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers. |
| DB/cache | MySQL 8.4 InnoDB + Redis TTL 30 ngày; Kafka test broker. |
| Dependency doubles | Product price/status, Inventory availability/reserve/commit/release, Payment, Shipment, Auth. |
| Data | Cart empty/valid/expired, multi-shop cart, product price changed, vouchers platform/shop, orders mọi state, invoice pending/issued. |
| Actors | Buyer owner/other buyer, seller owner/staff/other shop, admin, expired JWT. |
| Observability | Capture JSON logs/traces/metrics; assert `X-Request-ID`, W3C context, Kafka correlation. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Cart | Add/update/delete, empty/expired, SKU disabled | mfe-buyer state đúng; quantity 1–999; cart không giữ stock. |
| Checkout | Preview, price changed, stock unavailable, multi-shop split | Warnings rõ; không tạo order ở preview; place split đúng. |
| Payment | VNPAY pending/success/fail, COD | Payment action đúng; Order không tự giả payment success. |
| Voucher | Platform/shop, expired, limit, wrong scope | Discount/validation đúng; redemption atomic. |
| Orders | Buyer list/detail/status/cancel before/after shipping | Scope đúng; cancel sau shipping bị chặn. |
| Seller | Order list/fulfill/cancel/invoice | Chỉ shop own; không tự set delivered. |
| MFE states | Loading, empty, error, timeout, retry | Idempotent retry không duplicate order. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| O-API-01 | `GET /cart` buyer/empty | `200`, own cart; empty items hợp lệ. |
| O-API-02 | `POST /cart/items` valid/invalid/other user | `201` hoặc `400/403`; không trust user_id body. |
| O-API-03 | `PUT/DELETE /cart/items/{id}` | Update/delete own; wrong scope not found/forbidden. |
| O-API-04 | `GET /vouchers/validate` valid/expired/wrong scope | `200 valid=false` hoặc correct voucher error; preview không redeem. |
| O-API-05 | `GET /checkout/shipping-fee` valid/address other user/dependency down | `200`, `400 ORDER_ADDRESS_INVALID`, hoặc `503`. |
| O-API-06 | `POST /checkout/preview` valid | Total snapshot/warnings; không call reserve/charge. |
| O-API-07 | `POST /checkout` valid VNPAY multi-shop | `201`, group/order(s), reserve IDs, pending payment. |
| O-API-08 | `POST /checkout` stock fail/price changed | `409`, no partial order/reservation/voucher. |
| O-API-09 | `POST /checkout` same idempotency key same payload | Same response/no duplicate. |
| O-API-10 | Same key different payload | `409 ORDER_IDEMPOTENCY_CONFLICT`. |
| O-API-11 | `GET /orders`, detail/status | Buyer only; snapshot immutable; no other buyer data. |
| O-API-12 | `PATCH /orders/{id}/cancel` before shipping | `200 CANCELLED`, Inventory release command/event. |
| O-API-13 | Cancel shipped/delivered/wrong version | `409 ORDER_CANCEL_NOT_ALLOWED/ORDER_STATE_INVALID`. |
| O-API-14 | Invoice pending/issued | `409 ORDER_INVOICE_NOT_READY` or issued metadata. |
| O-API-15 | Seller list/detail own/other shop | Correct shop scope; no cross-tenant leak. |
| O-API-16 | Seller fulfill accept/ship invalid transition | Correct state/tracking validation; Shipment dependency not bypassed. |
| O-API-17 | Seller cancel allowed/disallowed | Policy/compensation/idempotency correct. |
| O-API-18 | Seller voucher CRUD platform attempt | Shop voucher works; PLATFORM creation denied. |
| O-API-19 | `POST/PUT/DELETE /admin/vouchers` với/không `VOUCHER_MANAGE` | Có quyền → tạo/sửa platform voucher; không quyền → `403`. `scope` bị ép `PLATFORM`. |
| O-API-20 | Body voucher gửi kèm field audience/segment | Field lạ bị bỏ qua/`400`; v1 không hỗ trợ targeting. |

### 3.1 Event/saga/reliability

| ID | Case | Expected |
|---|---|---|
| O-INT-01 | Payment success event duplicate/out-of-order | One transition/commit; old event ignored. |
| O-INT-02 | Inventory reserve success then MySQL failure | Compensation/reconcile; no orphan untracked order. |
| O-INT-03 | Order cancel duplicate | One release/redemption release. |
| O-INT-04 | Shipment delivered event | Order `DELIVERED` only valid transition. |
| O-INT-05 | Kafka unavailable | Local transaction + outbox; retry/DLQ, not false success to consumers. |
| O-INT-06 | MySQL deadlock/Redis down | Bounded retry/failure; no duplicate checkout. |

## 4. Unit test

| Module | Cases |
|---|---|
| `CartService` | TTL, quantity bounds, user scope, no reservation semantics. |
| `CheckoutCalculator` | VND integer totals, shipping, tax, discount, overflow/reconciliation. |
| `VoucherService` | Scope/time/usage, percent 1–100, fixed value, idempotent redemption. |
| `OrderStateMachine` | All valid/invalid transitions and cancel-before-shipping rule. |
| `InventoryClient` | Timeout/error mapping, idempotency, reserve/commit/release command contract. |
| `OrderMapper` | Product/address/shop/price immutable snapshots, redaction. |
| `Observability` | Required JSON fields, trace propagation, no token/payment/address/order raw logs. |

## 5. Contract/security/resilience

- Contract Product→Order: active SKU, current integer VND price, unavailable/price-changed errors.
- Contract Order→Inventory: all-or-nothing reserve, reservation TTL, commit/release idempotency.
- Contract Order→Payment/Shipment: event/status transition and compensation.
- Security: IDOR buyer/seller, forged shop/user IDs, replay/idempotency, voucher abuse, raw address/payment log scan.
- Load: cart/preview/list read load; checkout concurrency same SKU must not create excess reservations/orders.

## 6. Tiêu chí pass/phát hành

Không duplicate order/payment/reservation dưới retry/concurrency; mọi state transition, money reconciliation, scope/RBAC, outbox/replay, trace/log redaction pass; no test relies on Product as inventory ledger.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | MySQL 8.4 thay cho Postgres theo lựa chọn A. | Cần đổi Testcontainers/SQL dialect nếu architecture đổi. | Tech lead |
| 2 | Checkout reserve trước tạo order và Payment tách saga. | Cần điều chỉnh compensation test nếu flow đổi. | Order/Payment/Inventory |
| 3 | Voucher stacking/tax/carrier quote policy chưa đầy đủ. | Cần bổ sung golden total cases. | Product/Finance |
| 3b | Voucher v1 chỉ theo mã (không audience/targeting); platform voucher qua `/admin/vouchers`. | Nếu bật targeting cần thêm test audience/segment. | Product owner |
| 4 | PDF invoice provider chưa chốt. | Tạm test metadata/async state, chưa visual PDF. | Finance/DevOps |
