# Test Plan — Inventory Service

> Nguồn: `docs/lld/inventory.md` · `docs/db/inventory.md` · `docs/api/inventory.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, Testcontainers MySQL/Kafka. |
| Data | SKU active qty 10/0, disabled SKU, reservations reserved/committed/released/expired, movement ledger. |
| Concurrency | N threads cùng reserve một SKU; multi-SKU requests; deadlock/lock timeout. |
| Actors | Internal Order service, seller own/other shop, admin with/without step-up, anonymous. |
| Observability | JSON logs/metrics/OpenTelemetry; traceparent/X-Request-ID across REST/Kafka. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Seller stock | View/adjust own SKU, negative delta/other shop | Balance đúng; reason/audit bắt buộc; không sửa reserved. |
| Checkout | Reserve enough/insufficient/duplicate retry | Atomic all-or-nothing; không oversell; same key same result. |
| Lifecycle | Commit/release/expire, repeated command | State transition đúng; movement không duplicate. |
| Product projection | SKU created/disabled event | Inventory item register/disable; event old/duplicate safe. |
| Reliability | Kafka down, DB lock/rollback, expiry two instances | Recovery/retry/idempotency; không mất ledger/balance consistency. |
| UI states | Loading/empty/error/stale | `mfe-seller` hiển thị đúng quantity; `mfe-buyer` không có quyền adjust. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| I-API-01 | `GET /internal/inventory/availability` valid/batch 100 | `200` exact current read; no reservation side effect. |
| I-API-02 | Availability >100/invalid SKU | `400 INVENTORY_INVALID_INPUT`/`404 INVENTORY_SKU_NOT_FOUND`. |
| I-API-03 | `POST /internal/inventory/reservations` enough stock | `201 RESERVED`, available--/reserved++, movement/outbox. |
| I-API-04 | Reserve insufficient one of many SKU | `409 INVENTORY_STOCK_INSUFFICIENT`, rollback all SKUs. |
| I-API-05 | Concurrent reserve same SKU | At most available quantity succeeds; no negative balance/oversell. |
| I-API-06 | Reserve same idempotency key same payload | Same reservation result, no second movement. |
| I-API-07 | Same key different payload | `409 INVENTORY_IDEMPOTENCY_CONFLICT`. |
| I-API-08 | Commit reserved/duplicate commit | `200 COMMITTED`; repeat idempotent, no double movement. |
| I-API-09 | Release reserved/duplicate release | `200 RELEASED`; available restored once. |
| I-API-10 | Commit/release expired/finished | `409 INVENTORY_RESERVATION_STATE_INVALID/EXPIRED`. |
| I-API-11 | `GET /seller/inventory` own/other shop | Scope correct; no cross-shop data. |
| I-API-12 | `PATCH /seller/inventory/{skuId}/adjust` valid | Available delta + ledger/audit/outbox atomic. |
| I-API-13 | Adjustment makes negative/wrong version/no reason | `400/409`, no balance change. |
| I-API-14 | Admin reconciliation | Correct balance-vs-ledger difference; read-only. |
| I-API-15 | Health live/ready | Live independent DB; ready checks MySQL/Kafka. |

### 3.1 Event/concurrency/reliability

| ID | Case | Expected |
|---|---|---|
| I-INT-01 | Product `sku.created` duplicate/old status event | One item; old version ignored. |
| I-INT-02 | Expiry job two instances | One release/movement per reservation. |
| I-INT-03 | Kafka failure after DB commit | Outbox pending/retry/DLQ; balance remains correct. |
| I-INT-04 | Transaction rollback after lock/movement attempt | Balance/movement/reservation/outbox rollback together. |
| I-INT-05 | Order cancel reconciliation duplicate | Release idempotent, no negative reserved. |

## 4. Unit test

| Module | Cases |
|---|---|
| `BalanceService` | Non-negative invariant, version increment, overflow, lock retry mapping. |
| `ReservationService` | All-or-nothing multi-SKU, TTL, commit/release/expire state, duplicate commands. |
| `MovementService` | Delta reconciliation, immutable append, reason/source/ref validation. |
| `ExpiryJob` | Bounded batch, two-instance advisory lock, idempotent expiry. |
| `AuthorizationPolicy` | Internal service auth, seller shop scope, admin step-up. |
| `OutboxPublisher` | Retry 3/backoff 2s, DLQ, source event correlation. |
| `Observability` | JSON fields, trace/request propagation, no raw token/order/address/payment logs, bounded metrics. |

## 5. Contract/security/resilience

- Contract Order→Inventory: reserve/commit/release request hash, TTL, atomic response and error mapping.
- Contract Inventory→Product: stock snapshot available/reserved/committed, source version/as_of, display-only warning.
- Security: no public stock mutation, IDOR seller, forged order intent, idempotency replay, admin step-up.
- Database: lock contention, deadlock, rollback, replica/read consistency and ledger reconciliation.
- Performance: concurrent reserve correctness is more important than throughput; measure p95 lock/command latency and reject unbounded batch.

## 6. Tiêu chí pass/phát hành

Zero oversell/negative balance in concurrency suite; exact reserve/commit/release accounting; all idempotency/state/RBAC/ledger/outbox/observability tests pass; no stock mutation outside Inventory.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | SKU-level single balance, no warehouse allocation. | Cần thêm location/concurrency matrix nếu đổi. | Inventory owner |
| 2 | MySQL 8.4/InnoDB optimistic locking là accuracy baseline. | Ảnh hưởng Testcontainers/DB isolation. | DevOps |
| 3 | Reservation TTL 15 phút align Order. | Cần chỉnh expiry/checkout tests nếu đổi. | Order owner |
| 4 | Seller adjustment delta, chưa có cycle-count/approval workflow. | Cần bổ sung test authorization/reconciliation. | Product owner |
| 5 | Payment/Order event ordering dùng saga/idempotency, chưa có formal schema registry. | Cần contract test bổ sung khi schema chốt. | Platform owner |
