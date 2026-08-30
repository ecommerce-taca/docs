# Database — Shipment Service

> Nguồn: `docs/lld/shipment.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Baseline: MySQL 8.4/InnoDB `shipmentdb`; GHN + MOCK adapter

## 1. Quy ước chung

| Quy ước | Quyết định | Ràng buộc |
|---|---|---|
| ID | UUIDv7 API, `BINARY(16)` | Cross-service order/shop/user IDs không FK. |
| Time | `DATETIME(6)` UTC | API/event ISO-8601. |
| Snapshot | From/to address snapshot từ Order | Redact logs; không query UserDB. |
| Carrier event | `external_event_id` unique | Webhook at-least-once/idempotent. |
| Observability | JSON log/W3C trace/X-Request-ID | Không log secret/full address/raw webhook. |
| Secret | Secret manager | GHN credential không source/log. |

## 2. Quan hệ

```mermaid
erDiagram
    SHIPMENTS ||--o{ SHIPMENT_ITEMS : contains
    SHIPMENTS ||--o{ SHIPMENT_EVENTS : tracks
    SHIPMENTS ||--o{ CARRIER_WEBHOOK_LOGS : receives
```

## 3. Chi tiết bảng

### 3.1 `shipments`

`id`, `order_id`, `checkout_group_id`, `shop_id`, `buyer_user_id`, `carrier`, `tracking_code`, `status`, `from_address_snapshot`, `to_address_snapshot`, `shipping_fee`, `currency`, `external_shipment_id`, `version`, timestamps. Unique one shipment per shop-order baseline.

### 3.2 `shipment_items`

`id`, `shipment_id`, `sku_id`, `product_id`, `title_snapshot`, `quantity`, `weight_grams`, `created_at`. Quantity positive; snapshot immutable.

### 3.3 `shipment_events`

`id`, `shipment_id`, `canonical_status`, `carrier_status`, `description_safe`, `source`, `external_event_id`, `occurred_at`, `received_at`. Unique `(carrier,external_event_id)`; old status lưu diagnostic nhưng không làm state lùi.

### 3.4 `carrier_webhook_logs`, `idempotency_keys`

Webhook log giữ carrier/external ID/payload hash/signature result/status/timestamps; raw body không lưu hoặc encrypt theo policy. Idempotency giữ request hash/response snapshot 24h.

## 4. Index

| Bảng | Index |
|---|---|
| `shipments` | unique `(order_id,shop_id)`; unique `(carrier,tracking_code)`; `(buyer_user_id,created_at)`; `(shop_id,status,created_at)` |
| `shipment_items` | `(shipment_id)`; `(sku_id,created_at)` |
| `shipment_events` | `(shipment_id,occurred_at)`; unique `(carrier,external_event_id)` |
| `carrier_webhook_logs` | unique `(carrier,external_event_id)`; `(status,received_at)` |

## 5. Enum và rules

`ShipmentStatus`: `CREATED/PICKED_UP/IN_TRANSIT/DELIVERED/FAILED/CANCELLED/PENDING_RECONCILIATION`; `Carrier`: `GHN/SPX/J&T/MOCK` (seller chọn tay 1 trong 3 carrier thật); tracking code unique; delivered không do client set.

## 6. Migration và seed

### 6.1 Thứ tự migration

| Thứ tự | Nội dung | Phụ thuộc |
|---:|---|---|
| 001 | Tạo database `shipmentdb`, charset/collation, migration metadata | — |
| 002 | Tạo `shipments` (unique `(order_id,shop_id)`, unique `(carrier,tracking_code)`) | — |
| 003 | Tạo `shipment_items`, FK → `shipments` | `shipments` |
| 004 | Tạo `shipment_events` (unique `(carrier,external_event_id)`) | `shipments` |
| 005 | Tạo `carrier_webhook_logs` (unique `(carrier,external_event_id)`) | — |
| 006 | Tạo `idempotency_keys` | — |
| 007 | Thêm index `(buyer_user_id,created_at)`, `(shop_id,status,created_at)` trên `shipments`; `(shipment_id,occurred_at)` trên `shipment_events` | Tất cả bảng trên |
| 008 | Seed fixture cho local/test (§6.2) | Tất cả bảng trên |

Mỗi migration có `up`/`down` cho local/test. Không hard-delete `shipment_events`/`carrier_webhook_logs` — cleanup theo retention policy (job riêng, không phải migration).

### 6.2 Seed tối thiểu cho local/test

| Seed | Giá trị |
|---|---|
| `shipments` | 1 `CREATED` (carrier `MOCK`); 1 `PICKED_UP`; 1 `IN_TRANSIT`; 1 `DELIVERED` full timeline; 1 `PENDING_RECONCILIATION` (carrier timeout fixture). |
| `shipment_events` | Timeline đủ cho mỗi shipment fixture ở trên, đúng thứ tự `CREATED→PICKED_UP→IN_TRANSIT→DELIVERED`; thêm 1 event `external_event_id` trùng (test webhook idempotent) và 1 event status **cũ hơn** state hiện tại (test không lùi state). |
| `carrier_webhook_logs` | 1 log ACK thành công; 1 log chữ ký sai (`SHIPMENT_WEBHOOK_INVALID` fixture). |
| `idempotency_keys` | 1 key đã dùng cho `POST /internal/shipments`, còn hạn 24h. |

Không seed GHN/SPX/J&T credential thật hoặc địa chỉ buyer/seller thật — dùng snapshot giả lập.

### 6.3 Kiểm tra bắt buộc trước khi chạy migration production

1. Reconcile one-shop-order-one-shipment: verify không có `(order_id,shop_id)` trùng trước khi tạo unique index.
2. Verify `tracking_code` không trùng giữa các carrier trước khi tạo unique `(carrier,tracking_code)`.
3. Verify thứ tự `shipment_events` theo `occurred_at` không có gap bất thường trước go-live.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | GHN sandbox + MOCK baseline. | Ảnh hưởng carrier mapping/signature. | Shipment/DevOps |
| 2 | Một shop-order có một shipment; multi-package chưa có. | Cần child package schema/API nếu bật. | Order owner |
| 3 | Address snapshot nhận từ Order. | Ảnh hưởng privacy/update address. | Security/Order owner |
| 4 | Carrier webhook authentication policy chưa chốt. | Cần signature/IP/secret rotation contract. | Security |
| 5 | Return/exchange chưa thuộc v1. | Cần lifecycle/fee/refund linkage. | Product/Finance |
