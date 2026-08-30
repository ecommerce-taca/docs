# LLD — Shipment Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack baseline: Java 25 · Spring Boot · MySQL 8.4 · GHN + MOCK carrier adapter · Kafka consumer (Event listening)

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Shipment per shop-order, shipping quote, create shipment, tracking code/status timeline, carrier webhook và delivery event. |
| Database | MySQL 8.4 `shipmentdb`; carrier webhook log idempotent. |
| Carrier | GHN adapter + MOCK adapter; carrier status map về canonical state. |
| Order | Order-Commerce sở hữu order state; Shipment chỉ phát shipping status/result. |
| Payment | Payment-Wallet sở hữu payment; Shipment không tự capture/refund. |
| Không thuộc service | Product/cart/order price, inventory, payment, address source, rating/review, carrier credentials ownership. |
| Tách shipment | Mỗi `shop_id` trong checkout group tạo một shipment riêng. |

### 1.2 Boundary

```text
Order-Commerce ──quote/create/cancel──► Shipment
Shipment ──GHN/MOCK API──► Carrier
Carrier webhook ──signature/idempotency──► Shipment
Shipment ──Event publishing──► Order/Payment/Notification
Buyer/Seller ──Gateway──► tracking/status read
```

- Destination/source address snapshot được nhận từ Order; Shipment không query trực tiếp UserDB.
- Carrier webhook không có JWT theo HLD; xác thực bằng signature/IP allowlist/secret adapter.
- Tracking code và external event ID unique; webhook duplicate phải ACK idempotent.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Shipment | Shipment DB, carrier/tracking/status | Canonical shipment aggregate + carrier adapter. |
| Buyer order | Xem tracking/status | Public scoped tracking read qua Order/Gateway. |
| Seller order | Create shipment/tracking | Seller fulfill gọi create/label/track theo shop scope. |
| Shipping flow | Create shipment → tracking → delivered | Event-driven status update, webhook reconciliation. |

## 2. Cấu trúc bên trong

### 2.1 Module Spring Boot

```text
com.taca.shipment
├── shipment/             # aggregate and state machine
├── quote/                # shipping fee estimate
├── carrier/              # GHN/MOCK adapters and status mapping
├── tracking/             # status timeline
├── webhook/              # signature/idempotency/event log
├── event/                # Kafka publisher & consumer
├── security/             # buyer/seller scope and webhook auth
└── observability/        # shared telemetry contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `ShipmentController` | Internal create/quote và buyer/seller tracking read | Scope từ auth/order context. |
| `CarrierAdapter` | GHN/MOCK quote/create/cancel/track | Không leak carrier secret; timeout bounded. |
| `StatusMapper` | Map carrier → canonical status | Unknown status không tự đánh dấu delivered. |
| `WebhookController` | Receive carrier callback | Verify signature, external event unique, append timeline. |
| `ShipmentStateMachine` | Created/picked/in_transit/delivered/failed/cancelled | Không cho client set arbitrary state. |
| `EventPublisher` | Emit shipment events | Retry 3/backoff 2s/DLQ. |

### 2.2 Chuẩn observability dùng chung

- Structured JSON stdout field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST/Kafka propagate W3C `traceparent`/`tracestate`, `X-Request-ID`; webhook tự tạo request ID nếu carrier không gửi.
- Không log carrier token/secret/signature, full address/phone, access token hoặc raw webhook payload; chỉ log external event ID/hash và safe status.
- Metrics bounded theo `carrier`, `operation`, `canonical_status`, `result`; không label bằng order/shipment/user ID.
- `/health/live` process-only; `/health/ready` MySQL/Kafka/carrier config; carrier live call không giữ readiness nếu policy không yêu cầu.
- Error envelope `{error:{code,message,details,trace_id}}`; audit reason/actor không raw PII.

## 3. Luồng xử lý

### 3.1 Shipping quote

1. Order gửi shop origin, buyer destination snapshot, item weight/dimensions và carrier preference.
2. Validate address/weight bounds; chọn GHN hoặc MOCK adapter.
3. Trả fee/estimated delivery/currency; quote có expiry và không tạo shipment.

### 3.2 Create shipment

1. Order/Seller gửi shop-order ID, snapshots, carrier và idempotency key.
2. Kiểm tra order shipment chưa tồn tại và seller scope.
3. Gọi carrier create; lưu request reference/response safe, tracking code.
4. Transaction ghi shipment `CREATED`, first timeline event, publish `shipment.created`.
5. Duplicate request trả shipment đã tạo; carrier timeout tạo `PENDING_RECONCILIATION`, không tạo duplicate blind retry.

### 3.3 Carrier webhook/tracking

```text
Webhook → verify carrier auth → unique external_event_id
  → map status → validate transition
  → transaction shipment + shipment_events + outbox
  → ACK
```

Unknown/out-of-order status được lưu diagnostic nhưng không làm state lùi; delivered cần carrier evidence hợp lệ.

### 3.4 Cancel/fail

- Cancel chỉ trước carrier pickup và theo Order policy; gọi carrier cancel idempotent.
- Failed delivery phát event để Order/Notification xử lý; Shipment không tự refund.
- Carrier unavailable không giả tạo tracking/delivered; retry job/reconciliation bounded.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `PAGE_SIZE_DEFAULT` | 20 | Max 100. |
| `CARRIER_REQUEST_TIMEOUT` | 5s | GHN/MOCK call. |
| `CARRIER_RETRY_MAX` | 1 | Chỉ retry safe quote; create dùng idempotency/reconcile. |
| `WEBHOOK_MAX_SKEW` | 15 phút | Carrier timestamp. |
| `TRACKING_CODE_MAX_LENGTH` | 64 | Normalize/unique. |
| `SHIPMENT_ITEM_MAX` | 100 | Per shipment. |
| `ADDRESS_SNAPSHOT_MAX_BYTES` | 16 KiB | Validate request, redact logs. |
| `OUTBOX_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `IDEMPOTENCY_RETENTION` | 24h | Create/cancel command. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | DB UTC. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `Carrier` | `GHN`, `MOCK`. |
| `ShipmentStatus` | `CREATED`, `PICKED_UP`, `IN_TRANSIT`, `DELIVERED`, `FAILED`, `CANCELLED`, `PENDING_RECONCILIATION`. |
| `CarrierEventStatus` | `RECEIVED`, `APPLIED`, `IGNORED_OLD`, `REJECTED`. |
| `QuoteStatus` | `VALID`, `EXPIRED`, `FAILED`. |

Canonical transition: `CREATED → PICKED_UP → IN_TRANSIT → DELIVERED`; failure/cancel theo carrier evidence và policy. Không client tự set `DELIVERED`.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `shipment.events.v1` | `shipment.created` | shipment/order/shop/carrier/tracking |
| `shipment.events.v1` | `shipment.status_changed` | old/new status, source, occurred_at |
| `shipment.events.v1` | `shipment.delivered` | shipment/order/delivered_at |
| `shipment.events.v1` | `shipment.failed/cancelled` | shipment/reason/source |

### 6.2 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Order-Commerce | `order.created`, `order.cancelled` | Create/cancel shipment workflow; không đổi order state trực tiếp. |
| Payment-Wallet | Không cần payment event trong baseline | Shipment không capture payment. |

### 6.3 Mock contract — carrier webhook

```json
{
  "carrier":"GHN",
  "external_event_id":"ghn-event-1",
  "tracking_code":"GHN123456",
  "carrier_status":"delivering",
  "occurred_at":"2026-08-30T10:00:00Z",
  "signature":"opaque-carrier-signature",
  "payload_hash":"sha256:..."
}
```

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `SHIPMENT_INVALID_INPUT` | 400 | Address/item/carrier sai. |
| `SHIPMENT_UNAUTHENTICATED` | 401 | Thiếu auth. |
| `SHIPMENT_FORBIDDEN` | 403 | Sai buyer/shop scope. |
| `SHIPMENT_NOT_FOUND` | 404 | Shipment/tracking không tồn tại. |
| `SHIPMENT_STATE_INVALID` | 409 | Transition/cancel sai. |
| `SHIPMENT_ALREADY_EXISTS` | 409 | Shop-order đã có shipment. |
| `SHIPMENT_CARRIER_UNAVAILABLE` | 503 | GHN/MOCK adapter unavailable. |
| `SHIPMENT_CARRIER_TIMEOUT` | 504 | Carrier timeout. |
| `SHIPMENT_WEBHOOK_INVALID` | 400 | Signature/payload sai. |
| `SHIPMENT_WEBHOOK_REPLAYED` | 200/409 | Event đã xử lý. |
| `SHIPMENT_TRACKING_CONFLICT` | 409 | Tracking code collision. |
| `SHIPMENT_IDEMPOTENCY_CONFLICT` | 409 | Key khác payload. |
| `SHIPMENT_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | GHN sandbox + MOCK adapter là baseline; production credentials/contract chưa có. | Ảnh hưởng signature/status/label. | Shipment/DevOps |
| 2 | Mỗi shop-order một shipment. | Ảnh hưởng split, seller view và invoice/shipping fee. | Order owner |
| 3 | Address snapshot do Order cung cấp; Shipment không gọi Auth User. | Ảnh hưởng privacy/address update. | Security/Order owner |
| 4 | Carrier webhook signature/IP policy chưa chốt. | Cần threat model và contract test trước production. | Security |
| 5 | Return/exchange và multi-package chưa thuộc v1. | Cần aggregate/linkage mới nếu bật. | Product owner |
