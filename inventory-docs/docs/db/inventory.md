# Database — Inventory Service

> Nguồn: `docs/lld/inventory.md` · HLD `EcommercePlatform-v4(6).excalidraw` · Cập nhật: `2026-08-30`
> Baseline: MySQL 8.4/InnoDB `inventorydb` · SKU-level, một balance/SKU, atomic reservation

## 1. Quy ước chung

| Quy ước | Quyết định | Ràng buộc |
|---|---|---|
| ID | UUIDv7 API, `BINARY(16)` storage | Cross-service references không FK. |
| Quantity | `BIGINT` integer | `available ≥0`, `reserved ≥0`, max 999999999. |
| Time | `DATETIME(6)` UTC | API/event ISO-8601 UTC. |
| Lock | Optimistic locking (version) | Cập nhật với điều kiện `version` để tránh over-sell. |
| Ledger | Append-only movement | Không update/delete lịch sử. |
| Idempotency | Command key + request hash | Replay giống payload trả result cũ; khác payload conflict. |
| Observability | JSON log/W3C trace/X-Request-ID | Không log full order/address/token/payment. |

## 2. Quan hệ

```mermaid
erDiagram
    INVENTORY_ITEMS ||--o{ STOCK_RESERVATIONS : reserves
    INVENTORY_ITEMS ||--o{ STOCK_MOVEMENTS : records
    STOCK_RESERVATIONS ||--o{ STOCK_MOVEMENTS : causes
    INVENTORY_ITEMS ||--o{ OUTBOX_EVENTS : emits
```

## 3. Chi tiết bảng

### 3.1 `inventory_items`

| Cột | Type | Null | Ràng buộc |
|---|---|---:|---|
| `id` | BINARY(16) | N | PK. |
| `sku_id` | BINARY(16) | N | Unique, Product reference. |
| `product_id` | BINARY(16) | N | Reference để event projection. |
| `shop_id` | BINARY(16) | N | Seller scope. |
| `status` | VARCHAR(16) | N | `ACTIVE/DISABLED/ARCHIVED`. |
| `qty_available` | BIGINT | N | `≥0`. |
| `qty_reserved` | BIGINT | N | `≥0`. |
| `version` | BIGINT | N | Increment mỗi mutation. |
| `created_at/updated_at` | DATETIME(6) | N | UTC. |

### 3.2 `stock_reservations`

| Cột | Type | Ràng buộc |
|---|---|---|
| `id` | BINARY(16) PK | `reservation_id`. |
| `order_intent_id` | BINARY(16) | Order reference. |
| `idempotency_key` | VARCHAR(128) | Unique command scope. |
| `status` | VARCHAR(16) | `RESERVED/COMMITTED/RELEASED/EXPIRED`. |
| `expires_at` | DATETIME(6) | Required for RESERVED. |
| `created_at/updated_at` | DATETIME(6) | UTC. |

`stock_reservation_items(id,reservation_id,sku_id,quantity,created_at)` có unique `(reservation_id,sku_id)` và quantity integer 1–999999999.

### 3.3 `stock_movements`

Append-only: `id`, `sku_id`, `reservation_id` nullable, `order_intent_id` nullable, `delta_available`, `delta_reserved`, `reason`, `source`, `actor_user_id` nullable, `reference_id`, `occurred_at`. Tổng delta phải reconcile balance; không nhận arbitrary negative dẫn tới âm.

### 3.4 `idempotency_keys`, `outbox_events`, `inventory_audits`

- `idempotency_keys`: key, operation, caller_scope, request_hash, response_snapshot, status, expires_at; giữ 24h.
- `outbox_events`: event envelope, payload, occurred_at, published_at, attempt_count, last_error, DLQ timestamp.
- `inventory_audits`: actor/shop, operation, SKU, reason, before/after summary, trace/request ID, occurred_at; không lưu token/raw PII.

## 4. Index

| Bảng | Index |
|---|---|
| `inventory_items` | unique `sku_id`; `(shop_id,status,updated_at)`; `(product_id,status)` |
| `stock_reservations` | `(order_intent_id,status)`; `(status,expires_at)`; unique `(idempotency_key,order_intent_id)` |
| `stock_reservation_items` | unique `(reservation_id,sku_id)`; `(sku_id,reservation_id)` |
| `stock_movements` | `(sku_id,occurred_at)`; `(reservation_id,occurred_at)`; `(reference_id,reason)` |
| `idempotency_keys` | PK `(key,operation,caller_scope)`; `expires_at` |
| `outbox_events` | `(published_at,occurred_at)`; `(aggregate_type,aggregate_id,occurred_at)` |
| `inventory_audits` | `(sku_id,occurred_at)`; `(shop_id,occurred_at)` |

## 5. Enum và rules

| Rule | Giá trị |
|---|---|
| Reservation TTL | 15 phút. |
| Balance invariant | `qty_available ≥ 0`, `qty_reserved ≥ 0`. |
| Reserve | `available -= q`, `reserved += q`, cùng transaction. |
| Commit | `reserved -= q`; không tăng available. |
| Release/Expire | `reserved -= q`, `available += q`. |
| Adjustment | Chỉ thay available, reason/actor bắt buộc. |
| SKU state | Disabled/archived không reserve. |

## 6. Migration và seed

### 6.1 Thứ tự migration

| Thứ tự | Nội dung | Phụ thuộc |
|---:|---|---|
| 001 | Tạo database `inventorydb`, charset/collation, migration metadata | — |
| 002 | Tạo `inventory_items` (unique `sku_id`) | — |
| 003 | Tạo `stock_reservations` | — |
| 004 | Tạo `stock_reservation_items` (unique `(reservation_id,sku_id)`), FK → `stock_reservations` | `stock_reservations` |
| 005 | Tạo `stock_movements` (append-only) | `inventory_items`, `stock_reservations` |
| 006 | Tạo `idempotency_keys` | — |
| 007 | Tạo `outbox_events` | — |
| 008 | Tạo `inventory_audits` | `inventory_items` |
| 009 | Thêm index `(order_intent_id,status)`, `(status,expires_at)` trên `stock_reservations`; `(sku_id,occurred_at)` trên `stock_movements` | Tất cả bảng liên quan |
| 010 | Thêm CHECK constraint `qty_available>=0`, `qty_reserved>=0` trên `inventory_items` | `inventory_items` |
| 011 | Seed fixture cho local/test (§6.2) | Tất cả bảng trên |

Mỗi migration có `up`/`down` cho local/test. Production không hard-delete `stock_movements`/`stock_reservations` — retention policy áp dụng archive job, không phải migration xoá bảng.

### 6.2 Seed tối thiểu cho local/test

| Seed | Giá trị |
|---|---|
| `inventory_items` | 1 SKU `qty_available=10, qty_reserved=0` (bình thường); 1 SKU `qty_available=0` (hết hàng); 1 SKU `status=DISABLED`; 1 SKU `qty_available=5, qty_reserved=5` (đang bị giữ hết). |
| `stock_reservations` | 1 `RESERVED` chưa hết hạn; 1 `RESERVED` đã hết `expires_at` (test job expire); 1 `COMMITTED`; 1 `RELEASED`. |
| `stock_movements` | Đủ movement khớp balance hiện tại của từng `inventory_items` fixture ở trên — dùng để test reconciliation (§3.6 API `/admin/inventory/reconciliation`) ra `MATCHED`. |
| `idempotency_keys` | 1 key đã dùng cho reserve, còn hạn 24h — test replay trả cùng response. |

Không seed dữ liệu liên quan tới order/shop thật; `sku_id`/`order_intent_id` dùng UUID cố định để test khác cross-reference được.

### 6.3 Kiểm tra bắt buộc trước khi chạy migration production

1. Reconcile `qty_available + qty_reserved` khớp `Σ stock_movements.delta` cho từng SKU — migration phải fail nếu lệch, không tự sửa.
2. Test deadlock/lock timeout với 2 instance ghi đồng thời cùng SKU trước khi rollout.
3. Verify `stock_reservation_items` không có `(reservation_id,sku_id)` trùng trước khi tạo unique index.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Một balance/SKU, không multi-warehouse trong v1. | Multi-warehouse cần allocation/location schema/API. | Inventory owner |
| 2 | MySQL 8.4/InnoDB optimistic locking là baseline exactness. | Ảnh hưởng failover/replication/lock semantics. | DevOps |
| 3 | Reserve TTL 15 phút align Order. | Ảnh hưởng abandoned checkout và expiry. | Order owner |
| 4 | Seller adjustment dùng delta có reason; absolute target/cycle count chưa chốt. | Ảnh hưởng audit/reconciliation trên `mfe-seller` / `mfe-admin`. | Seller/Product owner |
| 5 | Retention stock movement/reservation chưa chốt. | Ảnh hưởng storage/compliance/reconciliation window. | Finance/Platform |
