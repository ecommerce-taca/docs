# API Spec — Shipment Service

> Nguồn: `docs/lld/shipment.md` · `docs/db/shipment.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · GHN + MOCK · Webhook carrier không dùng JWT nhưng bắt buộc signature policy

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Buyer/seller/admin JWT qua Gateway; internal Order service token/mTLS. |
| Request ID/trace | `X-Request-ID`, W3C `traceparent`/`tracestate`; webhook tự tạo ID nếu thiếu. |
| Timestamp | ISO-8601 UTC. |
| Idempotency | Create/cancel/webhook external event bắt buộc dedupe. |
| Response/error | `{data,meta}` / `{error:{code,message,details,trace_id}}`. |
| Log | JSON schema chuẩn auth-user; không raw address/carrier secret/webhook. |
| Scope | Buyer own order; seller own shop; admin permission. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /orders/{orderId}/shipment` | Buyer | Tracking shipment own order. |
| 2 | `GET /seller/orders/{orderId}/shipment` | Seller | Shop tracking. |
| 3 | `GET /internal/shipping/quote` | Order/internal | Shipping fee estimate. |
| 4 | `POST /internal/shipments` | Order/Seller internal | Create shipment. |
| 5 | `POST /internal/shipments/{shipmentId}/cancel` | Order/Seller internal | Cancel before pickup. |
| 6 | `POST /webhooks/shipping/{carrier}` | Carrier | Receive status webhook. |
| 7 | `GET /health/live` | Ops | Liveness. |
| 8 | `GET /health/ready` | Ops | Readiness. |

## 3. Chi tiết endpoint

### 3.1 Tracking read

`GET /orders/{orderId}/shipment` trả shipment status, carrier, tracking code, safe timeline, shipping fee và estimated delivery. Buyer không thấy shipment của user khác. Seller route filter shop scope.

### 3.2 `GET /internal/shipping/quote`

Query/body gồm shop origin, destination snapshot, items weight/dimensions, carrier `GHN|MOCK`. Response `200` `{fee,currency:"VND",estimate,quote_expires_at}`. Không tạo shipment; address max 16 KiB và không log raw.

### 3.3 `POST /internal/shipments`

Headers Idempotency-Key. Request `{order_id,shop_id,buyer_user_id,carrier,from_address_snapshot,to_address_snapshot,items[]}`. Response `201` shipment `CREATED`, tracking code và external ID. Mỗi shop-order một shipment; duplicate trả shipment cũ. Carrier timeout không retry blind create; state `PENDING_RECONCILIATION`.

### 3.4 Cancel

`POST /internal/shipments/{id}/cancel` body `{reason,version}`; chỉ trước `PICKED_UP`, gọi carrier cancel idempotently, state `CANCELLED` khi carrier confirmation/policy đạt. Không refund/payment trực tiếp.

### 3.5 `POST /webhooks/shipping/{carrier}`

Không JWT; verify carrier signature/IP/secret, unique `external_event_id`, map status và transaction timeline/event publishing. Status cũ lưu ignored diagnostic, không lùi state; không mfe-buyer/mfe-seller set delivered.

### 3.6 Health

`/health/live` `200` nếu process bind; `/health/ready` `200` khi DB/Kafka/config ready, `503` nếu unavailable. Không expose debug/carrier secret.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `SHIPMENT_INVALID_INPUT` | 400 | Address/item/carrier sai. |
| `SHIPMENT_UNAUTHENTICATED` | 401 | Thiếu auth. |
| `SHIPMENT_FORBIDDEN` | 403 | Sai scope. |
| `SHIPMENT_NOT_FOUND` | 404 | Shipment/tracking không tồn tại. |
| `SHIPMENT_STATE_INVALID` | 409 | Transition/cancel sai. |
| `SHIPMENT_ALREADY_EXISTS` | 409 | Shop-order đã có shipment. |
| `SHIPMENT_CARRIER_UNAVAILABLE` | 503 | Carrier down. |
| `SHIPMENT_CARRIER_TIMEOUT` | 504 | Carrier timeout. |
| `SHIPMENT_WEBHOOK_INVALID` | 400 | Signature/payload sai. |
| `SHIPMENT_WEBHOOK_REPLAYED` | 200/409 | Event đã xử lý. |
| `SHIPMENT_TRACKING_CONFLICT` | 409 | Tracking collision. |
| `SHIPMENT_IDEMPOTENCY_CONFLICT` | 409 | Key khác payload. |
| `SHIPMENT_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | GHN sandbox + MOCK adapter baseline. | Ảnh hưởng carrier API/signature. | Shipment/DevOps |
| 2 | Một shop-order một shipment; multi-package chưa v1. | Cần package API nếu bật. | Order owner |
| 3 | Carrier webhook auth policy chưa chốt. | Cần threat model/contract test. | Security |
| 4 | Return/exchange chưa thuộc v1. | Cần state/fee/refund workflow. | Product/Finance |
