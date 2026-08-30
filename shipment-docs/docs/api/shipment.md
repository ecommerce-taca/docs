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
| 2a | `GET /seller/orders/{orderId}/shipment/carriers` | Seller | So sánh phí/ETA từng carrier trước khi chọn. |
| 3 | `GET /internal/shipping/quote` | Order/internal | Shipping fee estimate (buyer checkout, carrier mặc định). |
| 4 | `POST /internal/shipments` | Order/Seller internal | Create shipment. |
| 5 | `POST /internal/shipments/{shipmentId}/cancel` | Order/Seller internal | Cancel before pickup. |
| 6 | `POST /webhooks/shipping/{carrier}` | Carrier | Receive status webhook. |
| 7 | `GET /health/live` | Ops | Liveness. |
| 8 | `GET /health/ready` | Ops | Readiness. |

## 3. Chi tiết endpoint

### 3.1 Tracking read

`GET /orders/{orderId}/shipment` (buyer) và `GET /seller/orders/{orderId}/shipment` (seller) dùng chung response. Buyer không thấy shipment của user khác; seller route filter theo shop scope trong token.

```json
{
  "data": {
    "shipment_id": "shp-01912fb0",
    "order_id": "order-01912f91",
    "shop_id": "shop-01912f31",
    "carrier": "GHN",
    "tracking_code": "GHN123456789",
    "status": "IN_TRANSIT",
    "shipping_fee": 32000,
    "currency": "VND",
    "estimated_delivery": { "min_date": "2026-09-01", "max_date": "2026-09-03" },
    "timeline": [
      { "status": "CREATED",    "occurred_at": "2026-08-30T09:05:00Z", "source": "SYSTEM",  "note": null },
      { "status": "PICKED_UP",  "occurred_at": "2026-08-31T02:10:00Z", "source": "CARRIER", "note": "Đã lấy hàng" },
      { "status": "IN_TRANSIT", "occurred_at": "2026-08-31T08:00:00Z", "source": "CARRIER", "note": "Đang trung chuyển" }
    ],
    "version": 3,
    "created_at": "2026-08-30T09:05:00Z"
  },
  "meta": { "request_id": "01912fb1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`timeline` chỉ chứa trạng thái an toàn — **không** trả địa chỉ thô, số điện thoại tài xế, hay payload carrier gốc. Đơn chưa tạo vận đơn → `404 SHIPMENT_NOT_FOUND` (không trả object rỗng).

### 3.1a `GET /seller/orders/{orderId}/shipment/carriers` — so sánh carrier

**Seller tự chọn carrier** khi chuẩn bị hàng (Penpot: `Overlay / Prepare shipment`, `Field / Carrier` với `GHN Express`/`SPX Express`/`J&T Express`). Endpoint này phục vụ đúng màn đó — trả phí + ETA của **từng** carrier khả dụng để seller so sánh trước khi bấm "Xác nhận & chuẩn bị".

Chỉ gọi được khi order đã `CONFIRMED` và **chưa có shipment** (`shipment.status = NOT_CREATED`); order đã có shipment → `409 SHIPMENT_ALREADY_EXISTS`, seller xem quote cũ qua §3.1.

Response `200`:

```json
{
  "data": {
    "order_id": "order-01912f91",
    "carriers": [
      { "carrier": "GHN",  "available": true,  "fee": 32000, "estimate": { "min_days": 2, "max_days": 4 } },
      { "carrier": "SPX",  "available": true,  "fee": 28000, "estimate": { "min_days": 3, "max_days": 5 } },
      { "carrier": "J&T",  "available": false, "fee": null,  "estimate": null, "unavailable_reason": "REGION_NOT_COVERED" }
    ],
    "quote_expires_at": "2026-08-30T09:30:00Z"
  },
  "meta": { "request_id": "01912fb4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`available:false` khi carrier không phủ khu vực giao/lấy hàng hoặc adapter đang lỗi — FE disable option đó, không ẩn hẳn (để seller hiểu vì sao không chọn được). Quote hết hạn không chặn tạo shipment — giống `GET /internal/shipping/quote`, phí thật chốt tại lúc tạo. `MOCK` không xuất hiện trong danh sách này (chỉ dùng nội bộ cho test/staging).

### 3.2 `GET /internal/shipping/quote`

| Query | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `from_shop_id` | string | Có | — |
| `to_address_id` | string | Có | Snapshot địa chỉ ≤ 16 KiB |
| `items` | array | Có | `{sku_id, quantity, weight_gram?, length_cm?, width_cm?, height_cm?}`, ≤100 dòng |
| `carrier` | enum | Không | `GHN` \| `SPX` \| `J&T` \| `MOCK`, mặc định `GHN` |

```json
{
  "data": {
    "fee": 32000,
    "currency": "VND",
    "carrier": "GHN",
    "estimate": { "min_days": 2, "max_days": 4 },
    "quote_expires_at": "2026-08-30T09:30:00Z"
  },
  "meta": { "request_id": "01912fb2-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Không tạo shipment. Không log địa chỉ thô. Quote hết hạn không chặn tạo shipment — phí thật chốt tại `POST /internal/shipments`.

Endpoint này phục vụ **buyer checkout** (Order-Commerce gọi để hiển thị phí ship trước khi đặt hàng — hệ thống tự dùng carrier mặc định `GHN`, buyer không chọn carrier). Việc **seller chọn carrier lúc chuẩn bị hàng** dùng endpoint riêng — xem §3.1a.

### 3.3 `POST /internal/shipments`

Headers: `Idempotency-Key` bắt buộc.

```json
{
  "order_id": "order-01912f91",
  "shop_id": "shop-01912f31",
  "buyer_user_id": "usr-01912f10",
  "carrier": "GHN",
  "from_address_snapshot": {
    "contact_name": "Anker Official", "phone": "0900000000",
    "line1": "Kho A, 15 Lê Lợi", "ward": "Bến Thành", "district": "Quận 1", "province": "TP.HCM"
  },
  "to_address_snapshot": {
    "contact_name": "Nguyễn Văn A", "phone": "0912345678",
    "line1": "12 Nguyễn Huệ", "ward": "Bến Nghé", "district": "Quận 1", "province": "TP.HCM"
  },
  "items": [{ "sku_id": "sku-01912f33", "title": "Sạc Anker 65W", "quantity": 2, "weight_gram": 300 }],
  "cod_amount": 0,
  "shipping_fee": 32000
}
```

Response `201`:

```json
{
  "data": {
    "shipment_id": "shp-01912fb0",
    "order_id": "order-01912f91",
    "carrier": "GHN",
    "tracking_code": "GHN123456789",
    "external_id": "5e2f1a9c",
    "status": "CREATED",
    "shipping_fee": 32000,
    "version": 1
  },
  "meta": { "request_id": "01912fb3-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Mỗi shop-order **một** shipment; gọi trùng trả shipment cũ (`200`, không tạo mới). `cod_amount > 0` chỉ hợp lệ khi order dùng COD. Carrier timeout → **không** retry create một cách mù quáng: shipment vào `PENDING_RECONCILIATION`, job đối soát sẽ tra theo `Idempotency-Key`/`order_id`.

`carrier` trong body này là carrier **seller đã chọn** ở §3.1a, do Order-Commerce truyền xuống nguyên văn khi seller bấm "Xác nhận & chuẩn bị" (`PATCH /seller/orders/{id}/fulfill` action=`SHIP` — xem `order-commerce-docs/docs/api/order-commerce.md` §3.7). Shipment **không** tự chọn carrier thay seller; carrier không nằm trong danh sách khả dụng của order đó (theo §3.1a tại thời điểm gọi) → `400 SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER`.

### 3.4 Cancel

`POST /internal/shipments/{shipmentId}/cancel` body `{ "reason": "BUYER_CANCELLED", "version": 1 }`. Response `200`:

```json
{ "data": { "shipment_id": "shp-01912fb0", "status": "CANCELLED", "reason": "BUYER_CANCELLED", "version": 2 }, "meta": { "request_id": "01912fbd-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Chỉ hợp lệ trước `PICKED_UP` → sau đó `409 SHIPMENT_STATE_INVALID`. `version` lệch → `409`. Gọi carrier cancel idempotently; state về `CANCELLED` khi carrier xác nhận hoặc theo policy timeout. **Không** chạm refund/payment — đó là việc của Order/Payment.

### 3.5 `POST /webhooks/shipping/{carrier}`

Không JWT. Bắt buộc: verify chữ ký carrier + IP allowlist, và `external_event_id` unique.

```json
{
  "external_event_id": "ghn-evt-88213",
  "tracking_code": "GHN123456789",
  "status": "DELIVERED",
  "occurred_at": "2026-09-01T04:22:10Z",
  "note": "Giao thành công",
  "signature": "<carrier-signature>"
}
```

Response `200 {"data":{"accepted":true}}`. Quy tắc:
- Event trùng `external_event_id` → `200` idempotent, không xử lý lại.
- Status **cũ hơn** state hiện tại → ghi log diagnostic `ignored`, **không lùi state**.
- Chữ ký sai → `400 SHIPMENT_WEBHOOK_INVALID`, không đổi state.
- Chỉ carrier được set `DELIVERED`; buyer/seller UI **không** có đường set trạng thái này.
- Map status carrier → state nội bộ qua bảng ánh xạ trong LLD; status lạ → lưu raw đã redact + cảnh báo, không tự đoán.

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
| `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` | 400 | Carrier không phủ khu vực/không khả dụng cho order này tại thời điểm tạo. |
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
| 1 | ~~GHN sandbox + MOCK adapter baseline~~ → **Đã chốt**: seller chọn tay 1 trong 3 carrier thật (`GHN`/`SPX`/`J&T`) qua `GET /seller/orders/{orderId}/shipment/carriers` (§3.1a); `MOCK` chỉ dùng nội bộ test/staging. Cần: hợp đồng/API key thật với SPX và J&T trước go-live — hiện chỉ có adapter GHN sandbox + MOCK, **2 carrier còn lại chưa có adapter thật**. | Nếu thiếu adapter SPX/J&T lúc go-live, seller chọn được nhưng tạo shipment fail runtime. | Shipment/DevOps |
| 2 | Một shop-order một shipment; multi-package chưa v1. | Cần package API nếu bật. | Order owner |
| 3 | Carrier webhook auth policy chưa chốt. | Cần threat model/contract test. | Security |
| 4 | Return/exchange chưa thuộc v1. | Cần state/fee/refund workflow. | Product/Finance |
