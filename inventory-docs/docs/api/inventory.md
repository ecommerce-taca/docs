# API Spec — Inventory Service

> Nguồn: `docs/lld/inventory.md` · `docs/db/inventory.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · Internal REST cho Order/Product; seller/admin qua Gateway

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Internal service token/mTLS cho Order; seller/admin JWT + shop scope qua Gateway. |
| Request ID | `X-Request-ID` tối đa 64; Gateway/service tạo nếu internal job. |
| Trace | W3C `traceparent`/`tracestate` REST/Kafka; response/error có `trace_id`. |
| Timestamp | ISO-8601 UTC; quantity integer. |
| Idempotency | `Idempotency-Key` bắt buộc reserve/commit/release/adjust; giữ 24h. |
| Atomicity | Mỗi command có transaction + optimistic lock; multi-SKU request all-or-nothing. |
| Response | `{data,meta:{request_id}}`; error chung `{error:{code,message,details,trace_id}}`. |
| Log | JSON chuẩn auth-user/Gateway; không log token/full order/address/payment. |
| Accuracy | Không trả “đã giữ hàng” nếu transaction chưa commit; không partial success. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /internal/inventory/availability` | Order/internal | Đọc availability, không reserve. |
| 2 | `POST /internal/inventory/reservations` | Order/internal | Atomic reserve nhiều SKU. |
| 3 | `POST /internal/inventory/reservations/{id}/commit` | Order/Payment internal | Commit reservation. |
| 4 | `POST /internal/inventory/reservations/{id}/release` | Order/Payment internal | Release reservation. |
| 5 | `GET /seller/inventory` | Seller/staff | Xem stock shop. |
| 6 | `PATCH /seller/inventory/{skuId}/adjust` | Seller | Điều chỉnh available bằng delta. |
| 7 | `GET /admin/inventory/reconciliation` | Admin | Reconcile balance/ledger. |
| 8 | `GET /health/live` | Internal/ops | Liveness. |
| 9 | `GET /health/ready` | Internal/ops | Readiness MySQL/Kafka. |

Không có endpoint cho buyer tự sửa stock hoặc cho Product/Order cập nhật `qty_available` trực tiếp.

## 3. Chi tiết endpoint

### 3.1 `GET /internal/inventory/availability`

Query `sku_ids` tối đa 100. Response `200`:

```json
{"data":[{"sku_id":"sku-1","available_qty":7,"status":"ACTIVE","as_of":"2026-08-30T09:00:00Z"}],"meta":{"request_id":"01912fa0"}}
```

Read-only; quantity có thể thay đổi ngay sau response, caller không được dùng thay reserve.

### 3.2 `POST /internal/inventory/reservations`

Headers: `Idempotency-Key` bắt buộc. Request:

```json
{"order_intent_id":"intent-1","expires_at":"2026-08-30T09:15:00Z","items":[{"sku_id":"sku-1","quantity":2}]}
```

Response `201` gồm `reservation_id`, status `RESERVED`, expires_at và items. Transaction update từng SKU theo thứ tự tăng dần kèm version; thiếu một SKU rollback toàn request. Không gọi Payment/Product trong transaction.

### 3.3 Commit/release

- `POST /internal/inventory/reservations/{id}/commit` body `{order_id,reason?}`; `RESERVED → COMMITTED`, giảm reserved, append movement.
- `POST /internal/inventory/reservations/{id}/release` body `{reason}`; `RESERVED → RELEASED`, trả available, append movement.
- Gọi lại cùng command sau trạng thái hoàn tất trả idempotent result; commit reservation released/expired trả `409 INVENTORY_RESERVATION_STATE_INVALID`.

### 3.4 `GET /seller/inventory`

Query `sku_id?`, `product_id?`, `status?`, `page`, `size`; seller chỉ thấy shop scope. Trả available/reserved, version, updated_at; không cho sửa reserved.

### 3.5 `PATCH /seller/inventory/{skuId}/adjust`

Headers Idempotency-Key. Request `{ "delta_available": 10, "reason": "Nhập thêm hàng", "version": 4 }`. Chỉ adjustment available; nếu delta âm làm quantity <0 trả `409 INVENTORY_ADJUSTMENT_INVALID/INVENTORY_STOCK_INSUFFICIENT`; append movement/audit/outbox atomically.

### 3.6 `GET /admin/inventory/reconciliation`

Query SKU/shop/time; trả balance, aggregate movement, active reservations, difference và `reconciliation_status`. Chỉ admin; không sửa dữ liệu qua GET.

### 3.7 Health

- `/health/live`: `200` nếu process bind, không phụ thuộc DB.
- `/health/ready`: `200` khi MySQL/Kafka/config sẵn sàng; `503` khi dependency fail.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `INVENTORY_INVALID_INPUT` | 400 | Quantity/idempotency sai. |
| `INVENTORY_UNAUTHENTICATED` | 401 | Thiếu auth/service context. |
| `INVENTORY_FORBIDDEN` | 403 | Sai shop/permission. |
| `INVENTORY_SKU_NOT_FOUND` | 404 | SKU chưa register. |
| `INVENTORY_SKU_DISABLED` | 409 | SKU disabled/archived. |
| `INVENTORY_STOCK_INSUFFICIENT` | 409 | Không đủ available. |
| `INVENTORY_RESERVATION_NOT_FOUND` | 404 | Reservation không tồn tại/scope sai. |
| `INVENTORY_RESERVATION_STATE_INVALID` | 409 | Commit/release sai state. |
| `INVENTORY_RESERVATION_EXPIRED` | 409 | Reservation hết TTL. |
| `INVENTORY_IDEMPOTENCY_CONFLICT` | 409 | Key khác payload. |
| `INVENTORY_CONCURRENCY_CONFLICT` | 409 | Version/lock conflict. |
| `INVENTORY_ADJUSTMENT_INVALID` | 400/409 | Adjustment/reason/state sai. |
| `INVENTORY_DEPENDENCY_UNAVAILABLE` | 503 | DB/Kafka/config unavailable. |
| `INVENTORY_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Inventory endpoint internal dùng service token/mTLS; Gateway route seller/admin. | Ảnh hưởng network policy/API Gateway. | Platform/Security |
| 2 | Reserve TTL 15 phút, all-or-nothing nhiều SKU. | Ảnh hưởng checkout UX/abandoned stock. | Order owner |
| 3 | Seller adjustment là signed delta; absolute target/cycle count chưa chốt. | Ảnh hưởng audit/reconciliation. | Seller owner |
| 4 | Multi-warehouse chưa có trong v1. | Nếu thêm kho cần allocation/location API. | Architecture |
| 5 | Payment event/Order command dùng saga; exact source of commit chưa chốt. | Ảnh hưởng duplicate commit/release. | Payment/Order owner |
