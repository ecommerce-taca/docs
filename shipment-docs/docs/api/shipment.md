# API Spec — Shipment Service

> Nguồn: `docs/lld/shipment.md` · `docs/db/shipment.md` · Code: `ecommerce-taca/shipment-service` @ `develop` (`a375a41`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — REST, webhook, outbox, reconciliation đã có code; chỉ adapter `MOCK` chạy thật, `GHN` fail-closed, `SPX`/`J&T` chưa có adapter.
> Base path: `/api/v1` (health ở `/health/*`, không có prefix) · Port mặc định `8086`

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth nội bộ | Mọi path chứa `/internal/` bắt buộc `X-Internal-Service-Token` khớp `SHIPMENT_INTERNAL_TOKEN` (so sánh constant-time); sai/thiếu → `401 SHIPMENT_UNAUTHENTICATED`. Path `/internal/ops/**` thêm `X-Operator-Token` (khớp `SHIPMENT_OPS_TOKEN`), `X-Operator-ID` (≤128 ký tự) và `X-Operator-Roles` chứa `SUPER_ADMIN`, `ADMIN` hoặc `SHIPMENT_COMMAND_REPLAY`; sai → `403 SHIPMENT_FORBIDDEN`. |
| Actor context | Path `/orders/**` và `/seller/**` bắt buộc `X-User-ID` + `X-User-Roles` do Gateway inject (khi `SHIPMENT_ENFORCE_GATEWAY_HEADERS=true`, mặc định). Buyer route cần role `BUYER`, `SUPER_ADMIN` hoặc `ADMIN`; seller route cần `SELLER`, `SELLER_STAFF`, `SHOP_STAFF`, `SUPER_ADMIN` hoặc `ADMIN` (`SELLER_STAFF`/`SUPER_ADMIN` là tên role auth-user — shipment-service#12; `SHOP_STAFF`/`ADMIN` giữ để tương thích). Role so khớp không phân biệt hoa thường, chấp nhận tiền tố `ROLE_`. |
| Webhook | `/webhooks/shipping/**` không JWT, không token nội bộ; xác thực bằng IP allowlist + HMAC (§3.7). |
| Request ID/trace | Đọc `X-Request-ID`; thiếu thì service sinh UUID. `trace_id` trong lỗi lấy từ `X-Trace-ID` (≤128, `[A-Za-z0-9._-]`), nếu không có thì `X-Request-ID` (≤64), nếu không có thì UUID mới. |
| Timestamp | ISO-8601 UTC (Jackson `time-zone: UTC`). |
| Idempotency | `Idempotency-Key` bắt buộc cho create và cancel, tối đa 128 ký tự, giữ 24h (bảng `idempotency_keys`, khoá `(idempotency_key, operation)`). Hash request là SHA-256 của JSON đã sort key. |
| Response | `{ "data": …, "meta": { "request_id": … } }`. Jackson `default-property-inclusion: non_null` — field `null` có thể bị bỏ khỏi body. |
| Error | `{ "error": { "code", "message", "details", "trace_id" } }`. |
| ID | `shipment_id` dạng `shp-<uuidv7>`; path `{shipmentId}` chấp nhận có hoặc không tiền tố `shp-`. |
| Pagination | Không có endpoint phân trang. |
| Log | JSON; không log address, phone, signature, raw webhook body, token. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /api/v1/orders/{orderId}/shipment` | Buyer (chủ đơn) | Tracking shipment của đơn mình. | ✅ |
| 2 | `GET /api/v1/seller/orders/{orderId}/shipment` | Seller/Shop staff/Admin | Tracking theo shop scope. | ✅ |
| 3 | `POST /api/v1/internal/shipping/quote` | Order (internal) | Ước tính phí ship cho checkout, carrier mặc định `GHN`. | ✅ |
| 4 | `POST /api/v1/internal/shipping/carrier-quotes` | Order (internal) | So sánh phí/ETA `GHN`/`SPX`/`J&T` cho seller. | ✅ |
| 5 | `POST /api/v1/internal/shipments` | Order (internal) | Tạo shipment. | ✅ |
| 6 | `POST /api/v1/internal/shipments/{shipmentId}/cancel` | Order (internal) | Huỷ trước pickup. | ✅ |
| 7 | `POST /api/v1/webhooks/shipping/{carrier}` | Carrier | Nhận status webhook. | ✅ (chữ ký HMAC chung, chưa theo spec carrier thật) |
| 8 | `POST /api/v1/internal/ops/shipment-create-commands/{commandId}/replay` | Operator | Replay command Kafka ở trạng thái `DEAD`. | ✅ |
| 9 | `POST /api/v1/internal/ops/order-cancellations/{orderId}/replay` | Operator | Replay cancellation ở trạng thái `NEEDS_OPERATOR`. | ✅ |
| 10 | `GET /health/live` | Ops | Liveness. | ✅ |
| 11 | `GET /health/ready` | Ops | Readiness. | ✅ |

Ngoài ra Spring Actuator expose `/actuator/health`, `/actuator/info`, `/actuator/metrics`, `/actuator/prometheus` (có probes).

`GET /seller/orders/{orderId}/shipment/carriers` **không** thuộc Shipment: đó là façade của Order-Commerce (Order kiểm order `CONFIRMED`, chưa có shipment, đúng seller) rồi gọi endpoint #4. Shipment không tự đọc order context từ ID.

## 3. Chi tiết endpoint

### 3.1 Tracking read (#1, #2)

Hai route trả cùng shape. Shipment tìm theo `order_id`.

- Buyer: `X-User-ID` phải bằng `buyer_user_id` của shipment, kể cả khi role là `ADMIN`; khác → `403 SHIPMENT_FORBIDDEN`.
- Seller: `X-User-Shop-Scope` (danh sách phân tách bằng `,` `;` khoảng trắng, chấp nhận dạng `["shop-1"]`) phải chứa `shop_id` của shipment, **hoặc** `X-User-Permissions` chứa `shipment:read:any`; nếu không → `403 SHIPMENT_FORBIDDEN`.
- Chưa có shipment → `404 SHIPMENT_NOT_FOUND`.

Response `200`:

```json
{
  "data": {
    "shipment_id": "shp-01912fb0-7a1b-7c12-9c55-8b1c34a6d921",
    "order_id": "order-01912f91",
    "shop_id": "shop-01912f31",
    "carrier": "MOCK",
    "tracking_code": "MOCK123456",
    "status": "IN_TRANSIT",
    "shipping_fee": 20000,
    "checkout_shipping_fee": 32000,
    "carrier_quoted_fee": 20000,
    "carrier_quote_delta": -12000,
    "currency": "VND",
    "estimated_delivery": { "min_date": "2026-09-01", "max_date": "2026-09-03" },
    "version": 3,
    "created_at": "2026-08-30T09:05:00Z",
    "timeline": [
      { "status": "CREATED", "occurred_at": "2026-08-30T09:05:00Z", "source": "SYSTEM", "note": "Đã tạo vận đơn" },
      { "status": "IN_TRANSIT", "occurred_at": "2026-08-31T08:00:00Z", "source": "CARRIER", "note": "Đang trung chuyển" }
    ]
  },
  "meta": { "request_id": "01912fb1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Ý nghĩa field phí:

| Field | Ý nghĩa |
|---|---|
| `checkout_shipping_fee` | Phí ship Order đã chốt lúc checkout (lấy nguyên giá trị `shipping_fee` trong request create). Dùng cho đối soát thương mại. |
| `carrier_quoted_fee` | Phí carrier báo lúc create; `null`/vắng nếu adapter không trả phí. |
| `carrier_quote_delta` | `carrier_quoted_fee - checkout_shipping_fee`, chỉ có khi cả hai đều có. |
| `carrier_actual_fee` | Chỉ có khi có dữ liệu phí thực tế đã xác thực; hiện chưa adapter nào cung cấp nên luôn vắng. |
| `shipping_fee` | **Deprecated** — phí hiệu lực kiểu cũ (= quote nếu có, ngược lại = phí request). Không dùng cho thanh toán/hoàn tiền. |

`timeline[].source` ∈ `SYSTEM`, `CARRIER`, `RECONCILIATION`. Response không trả address snapshot, số điện thoại hay payload carrier gốc.

### 3.2 `POST /internal/shipping/quote` (#3)

Order gửi snapshot địa chỉ và item nó đang sở hữu (POST body thay cho GET query cũ, để không đưa địa chỉ/mảng item lên URL).

```json
{
  "from_shop_id": "shop-01912f31",
  "from_address_snapshot": { "line1": "Kho A, 15 Lê Lợi", "ward": "Bến Thành", "district": "Quận 1", "province": "TP.HCM" },
  "to_address_snapshot": { "line1": "12 Nguyễn Huệ", "ward": "Bến Nghé", "district": "Quận 1", "province": "TP.HCM" },
  "items": [{ "sku_id": "sku-01912f33", "product_id": "prd-1", "title": "Sạc Anker 65W", "quantity": 2, "weight_gram": 300 }],
  "carrier": "MOCK"
}
```

| Field | Ràng buộc |
|---|---|
| `from_shop_id` | Bắt buộc. |
| `from_address_snapshot`, `to_address_snapshot` | Bắt buộc, JSON object, mỗi cái ≤ 16 KiB UTF-8. |
| `items` | 1–100 dòng. Mỗi dòng: `sku_id` (bắt buộc, ≤80), `product_id` (≤80), `title` (bắt buộc, ≤255), `quantity` ≥1, `weight_gram`/`length_cm`/`width_cm`/`height_cm` ≥0 (tuỳ chọn). |
| `carrier` | Tuỳ chọn, `GHN` \| `SPX` \| `J&T` (cũng nhận `J_AND_T`, `J_AND_T_EXPRESS`) \| `MOCK`; mặc định `GHN`. |

Response `200`:

```json
{
  "data": { "fee": 20000, "currency": "VND", "carrier": "MOCK", "estimate": { "min_days": 1, "max_days": 3 }, "quote_expires_at": "2026-08-30T09:30:00Z" },
  "meta": { "request_id": "01912fb2-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

- `fee` có thể `null` khi adapter không có giá — `null` nghĩa là chưa biết, không phải 0.
- Không tạo shipment, không phát quote ID; create sẽ tự quote lại từ snapshot của chính nó.
- `quote_expires_at` = now + `quote-ttl-minutes` (30 phút) cho carrier thật; với `MOCK` lấy expiry do adapter trả (30 phút).
- `MOCK` bị từ chối (`503 SHIPMENT_CARRIER_UNAVAILABLE`) khi `SHIPMENT_MOCK_ENABLED=false` hoặc `APP_ENV` không thuộc `local`/`test`/`staging`.
- `GHN` hiện fail-closed → `503 SHIPMENT_CARRIER_UNAVAILABLE`; `SPX`/`J&T` chưa có adapter → `503 SHIPMENT_CARRIER_UNAVAILABLE`.

### 3.3 `POST /internal/shipping/carrier-quotes` (#4)

Order-Commerce kiểm order `CONFIRMED`, chưa có shipment và thuộc seller, rồi gọi endpoint này.

```json
{
  "order_id": "order-01912f91",
  "shop_id": "shop-01912f31",
  "from_address_snapshot": { "line1": "Kho A" },
  "to_address_snapshot": { "line1": "12 Nguyễn Huệ" },
  "items": [{ "sku_id": "sku-01912f33", "title": "Sạc Anker 65W", "quantity": 2, "weight_gram": 300 }]
}
```

Response `200` (luôn đủ 3 carrier theo thứ tự `GHN`, `SPX`, `J&T`; `MOCK` không bao giờ xuất hiện):

```json
{
  "data": {
    "order_id": "order-01912f91",
    "carriers": [
      { "carrier": "GHN", "available": false, "unavailable_reason": "CARRIER_UNAVAILABLE" },
      { "carrier": "SPX", "available": false, "unavailable_reason": "ADAPTER_NOT_CONFIGURED" },
      { "carrier": "J&T", "available": false, "unavailable_reason": "ADAPTER_NOT_CONFIGURED" }
    ],
    "quote_expires_at": "2026-08-30T09:30:00Z"
  },
  "meta": { "request_id": "01912fb4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Carrier khả dụng trả `{ "carrier", "available": true, "fee", "currency", "estimate": { "min_days", "max_days" } }`. `unavailable_reason` ∈ `ADAPTER_NOT_CONFIGURED` (chưa có adapter), `CARRIER_UNAVAILABLE` (adapter ném `SHIPMENT_CARRIER_UNAVAILABLE` hoặc quote `available=false` không lý do), `QUOTE_FAILED` (lỗi khác), hoặc lý do adapter trả. Lỗi từng carrier không làm fail cả request. Ví dụ trên là kết quả thực tế với code hiện tại (GHN tắt, SPX/J&T chưa có adapter).

### 3.4 `POST /internal/shipments` (#5)

Headers: `Idempotency-Key` bắt buộc (≤128). REST create là hướng tích hợp đã chọn với Order-Commerce; Kafka `shipment.commands.v1` chỉ là đề xuất opt-in (xem LLD §6).

```json
{
  "order_id": "order-01912f91",
  "order_code": "TC-20260830-0001",
  "checkout_group_id": "checkout-01912f90",
  "shop_id": "shop-01912f31",
  "buyer_user_id": "usr-01912f10",
  "buyer_email": "buyer@example.com",
  "carrier": "MOCK",
  "from_address_snapshot": { "contact_name": "Anker Official", "phone": "0900000000", "line1": "Kho A, 15 Lê Lợi", "ward": "Bến Thành", "district": "Quận 1", "province": "TP.HCM" },
  "to_address_snapshot": { "contact_name": "Nguyễn Văn A", "phone": "0912345678", "line1": "12 Nguyễn Huệ", "ward": "Bến Nghé", "district": "Quận 1", "province": "TP.HCM" },
  "items": [{ "sku_id": "sku-01912f33", "product_id": "prd-1", "title": "Sạc Anker 65W", "quantity": 2, "weight_gram": 300 }],
  "cod_amount": 0,
  "shipping_fee": 32000,
  "currency": "VND",
  "payment_method": "VNPAY"
}
```

| Field | Ràng buộc |
|---|---|
| `order_id`, `shop_id`, `buyer_user_id` | Bắt buộc, ≤80. |
| `checkout_group_id` | Tuỳ chọn, ≤80. |
| `order_code` | Tuỳ chọn, ≤32 — mã đơn hiển thị (`orders.order_number`). Shipment lưu và phát lại ở `payload.order_code` của mọi event để Notification render `{{order_code}}`. Order-Commerce **phải** gửi. (shipment-service#12) |
| `buyer_email` | Bắt buộc, email hợp lệ, ≤254 (Notification cần cho email `shipment.delivered`/`shipment.failed`). |
| `carrier` | Bắt buộc; giá trị như §3.2. Carrier lạ → `400 SHIPMENT_INVALID_INPUT`. |
| `from_address_snapshot`, `to_address_snapshot` | Bắt buộc, JSON object ≤16 KiB. |
| `items` | 1–100 dòng, cấu trúc như §3.2. |
| `cod_amount` | ≥0; `>0` chỉ hợp lệ khi `payment_method` = `COD` (không phân biệt hoa thường). |
| `shipping_fee` | ≥0. Đây là phí checkout Order đã chốt; lưu nguyên vào `checkout_shipping_fee`. Request **không** nhận field `checkout_shipping_fee`. |
| `currency` | Bắt buộc, 3 chữ cái in hoa (`[A-Z]{3}`). Không quy đổi tỉ giá. |
| `payment_method` | Tuỳ chọn; chỉ dùng để chặn COD sai. |

Xử lý:

1. Cùng key + cùng hash → trả kết quả cũ: `200` nếu đã `COMPLETED`, `202` nếu còn chờ đối soát. Cùng key + hash khác (kể cả khác phí/currency) → `409 SHIPMENT_IDEMPOTENCY_CONFLICT`.
2. Đã có shipment cho `(order_id, shop_id)` → trả shipment đó, `200` (hoặc `202` nếu đang `PENDING_RECONCILIATION`); không tạo mới.
3. `MOCK` ngoài `local`/`test`/`staging` hoặc bị tắt → `400 SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER`.
4. Quote lại với carrier đã chọn: không có quote hoặc phí âm → `502 SHIPMENT_CARRIER_INVALID_QUOTE`; quote của carrier khác → `502 SHIPMENT_CARRIER_MISMATCH`; quote `available=false` → `400 SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER`; quote đã hết hạn → `502 SHIPMENT_CARRIER_QUOTE_EXPIRED`; currency khác → `400 SHIPMENT_CURRENCY_MISMATCH`. Phí 0 hợp lệ.
5. Ghi shipment `PENDING_RECONCILIATION` + items + idempotency `PROCESSING` trong một transaction **trước** khi gọi carrier.
6. Gọi carrier create. Thành công → shipment `CREATED`, timeline `CREATED`, outbox `shipment.created`, response `201`.
7. Carrier timeout, lỗi 5xx/408/409/425/429 hoặc lỗi không xác định → giữ `PENDING_RECONCILIATION`, ghi timeline + outbox `shipment.status_changed` (reason `CARRIER_OUTCOME_AMBIGUOUS`), response `202`. Không retry create mù; job đối soát tra theo idempotency key (LLD §3.4). Lỗi carrier xác định khác → xoá bản ghi giữ chỗ và trả lỗi gốc.

Response `201` (cũng là shape cho `200`/`202`):

```json
{
  "data": {
    "shipment_id": "shp-01912fb0-7a1b-7c12-9c55-8b1c34a6d921",
    "order_id": "order-01912f91",
    "shop_id": "shop-01912f31",
    "carrier": "MOCK",
    "tracking_code": "MOCK123456",
    "status": "CREATED",
    "shipping_fee": 20000,
    "checkout_shipping_fee": 32000,
    "carrier_quoted_fee": 20000,
    "carrier_quote_delta": -12000,
    "currency": "VND",
    "estimated_delivery": { "min_date": "2026-08-31", "max_date": "2026-09-02" },
    "version": 2,
    "created_at": "2026-08-30T09:05:00Z"
  },
  "meta": { "request_id": "01912fb3-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Response create không có `timeline` và không có `external_id`. Ràng buộc DB unique `(order_id, shop_id)` chỉ tương đương "một shipment mỗi Shop Order" nếu mỗi Shop Order có đúng một `shop_id` bất biến (câu hỏi mở #2).

### 3.5 `POST /internal/shipments/{shipmentId}/cancel` (#6)

Headers: `Idempotency-Key` bắt buộc (≤128). Body:

```json
{ "reason": "ORDER_CANCELLED", "version": 2 }
```

`reason` bắt buộc, ≤255; `version` ≥0 và phải bằng version hiện tại. Response `200`:

```json
{ "data": { "shipment_id": "shp-01912fb0-7a1b-7c12-9c55-8b1c34a6d921", "status": "CANCELLED", "reason": "ORDER_CANCELLED", "version": 3 }, "meta": { "request_id": "01912fbd-7a1b-7c12-9c55-8b1c34a6d921" } }
```

- Chỉ huỷ được khi status = `CREATED`; khác → `409 SHIPMENT_STATE_INVALID`. Version lệch → `409 SHIPMENT_STATE_INVALID`.
- ID không parse được hoặc không tồn tại → `404 SHIPMENT_NOT_FOUND`.
- Gọi carrier cancel trước, sau đó transaction đổi state + timeline + outbox `shipment.cancelled`.
- Cùng key + cùng body → trả shipment hiện tại; khác body → `409 SHIPMENT_IDEMPOTENCY_CONFLICT`.
- Không chạm refund/payment.

### 3.6 Ops replay (#8, #9)

Cần token nội bộ + operator token + `X-Operator-ID` + role (§1). Không có body.

- `POST /internal/ops/shipment-create-commands/{commandId}/replay`: chỉ replay command ở trạng thái `DEAD`. Response `200 { "data": { "command_id", "status": "RETRY_WAIT" } }`; trạng thái khác → `409` (Spring `ResponseStatusException`).
- `POST /internal/ops/order-cancellations/{orderId}/replay`: chỉ replay intent ở `NEEDS_OPERATOR`. Response `200 { "data": { "order_id", "status": "RETRY_WAIT" } }`; khác → `409`.

Replay giữ nguyên idempotency key gốc và được ghi vào `shipment_command_replay_audit`.

### 3.7 `POST /webhooks/shipping/{carrier}` (#7)

Không JWT. Yêu cầu:

- IP nguồn (`request.getRemoteAddr()`) thuộc `SHIPMENT_WEBHOOK_ALLOWED_IPS` (mặc định `127.0.0.1,::1`).
- Header `X-Carrier-Signature` = hex HMAC-SHA256 của raw body (chấp nhận tiền tố `sha256=`), secret theo carrier: `SHIPMENT_WEBHOOK_GHN_SECRET`, `SHIPMENT_WEBHOOK_SPX_SECRET`, `SHIPMENT_WEBHOOK_JT_SECRET`, `SHIPMENT_WEBHOOK_MOCK_SECRET`. Secret rỗng → luôn từ chối. Đây là contract tạm cho test, chưa phải chữ ký thật của GHN.

Body:

```json
{ "external_event_id": "mock-evt-88213", "tracking_code": "MOCK123456", "status": "delivered", "occurred_at": "2026-09-01T04:22:10Z", "note": "Giao thành công" }
```

`external_event_id` ≤128, `tracking_code` ≤64, `status` ≤80, `occurred_at` bắt buộc, `note` ≤500. Chữ ký không nằm trong body.

Response luôn `200` khi đã xác thực:

| Trường hợp | `data` |
|---|---|
| Áp dụng thành công | `{ "accepted": true, "duplicate": false }` |
| Trùng `external_event_id` cùng payload hash | `{ "accepted": true, "duplicate": true }` |
| Không tìm thấy tracking, status lạ, event cũ hơn event mới nhất, transition không hợp lệ | `{ "accepted": true, "ignored": true }` (log lưu `REJECTED`/`UNKNOWN_STATUS`/`IGNORED_OLD`) |

Carrier lạ, IP/chữ ký sai, body không parse được, thiếu field, hoặc trùng `external_event_id` nhưng **khác** payload hash → `400 SHIPMENT_WEBHOOK_INVALID`, không đổi state.

Bảng map status carrier (không phân biệt hoa thường, khoảng trắng → `_`):

| Carrier status | Canonical |
|---|---|
| `created`, `ready_to_ship`, `picked` | `CREATED` |
| `picked_up`, `picked-up`, `pickedup`, `pickup` | `PICKED_UP` |
| `in_transit`, `transporting`, `delivering`, `delivery_in_progress` | `IN_TRANSIT` |
| `delivered`, `delivery_success`, `success` | `DELIVERED` |
| `failed`, `delivery_failed`, `exception`, `returning` | `FAILED` |
| `cancelled`, `canceled` | `CANCELLED` |

Chỉ webhook đã xác thực mới set được `DELIVERED`. Status lặp lại trùng state hiện tại vẫn ghi timeline nhưng không phát event mới. `note` được làm sạch (bỏ xuống dòng, che `phone|address|token|secret: …`).

### 3.8 Health (#10, #11)

- `GET /health/live` → `200 {"status":"UP"}`.
- `GET /health/ready` → `200 {"status":"UP","database":"UP","kafka_topic_configured":"true"}` khi `SELECT 1` thành công và `shipment.kafka.shipment-events-topic` có giá trị; `503 {"status":"DOWN","database":"DOWN"}` khi DB lỗi; `503 {"status":"DOWN","reason":"KAFKA_TOPIC_NOT_CONFIGURED"}` khi thiếu topic. Chưa probe broker Kafka thật.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `SHIPMENT_INVALID_INPUT` | 400 | Validation DTO, JSON lỗi, address/item/currency/COD sai, carrier không hỗ trợ, idempotency key thiếu/quá dài. |
| `SHIPMENT_CURRENCY_MISMATCH` | 400 | Currency quote khác currency request. |
| `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` | 400 | Carrier không khả dụng cho shipment này (quote `available=false`, hoặc `MOCK` bị tắt khi create). |
| `SHIPMENT_WEBHOOK_INVALID` | 400 | Webhook sai IP/chữ ký/payload, hoặc replay khác payload. |
| `SHIPMENT_UNAUTHENTICATED` | 401 | Thiếu/sai token nội bộ, thiếu actor header. |
| `SHIPMENT_FORBIDDEN` | 403 | Sai role, sai buyer, sai shop scope, thiếu operator context. |
| `SHIPMENT_NOT_FOUND` | 404 | Shipment không tồn tại. |
| `SHIPMENT_STATE_INVALID` | 409 | Cancel sai state hoặc version lệch. |
| `SHIPMENT_ALREADY_EXISTS` | 409 | Vi phạm unique `(order_id, shop_id)` khi ghi DB đồng thời. |
| `SHIPMENT_TRACKING_CONFLICT` | 409 | Vi phạm unique `(carrier, tracking_code)`. |
| `SHIPMENT_IDEMPOTENCY_CONFLICT` | 409 | Cùng `Idempotency-Key`, khác payload. |
| `SHIPMENT_CARRIER_INVALID_QUOTE` | 502 | Carrier không trả quote, phí âm, hoặc carrier không hỗ trợ. |
| `SHIPMENT_CARRIER_MISMATCH` | 502 | Quote trả về của carrier khác. |
| `SHIPMENT_CARRIER_QUOTE_EXPIRED` | 502 | Carrier trả quote đã hết hạn. |
| `SHIPMENT_CARRIER_UNAVAILABLE` | 503 | Adapter chưa cấu hình/tắt (GHN, SPX, J&T, MOCK bị tắt khi quote). |
| `SHIPMENT_CARRIER_TIMEOUT` | 504 | Carrier timeout (ở create sẽ chuyển thành `202 PENDING_RECONCILIATION`). |
| `SHIPMENT_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại hoặc constraint DB khác. |

Mã `SHIPMENT_WEBHOOK_REPLAYED` trong thiết kế cũ **không tồn tại** trong code: webhook trùng trả `200` với `duplicate: true`.

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Seller chọn 1 trong 3 carrier thật (`GHN`/`SPX`/`J&T`); hiện `GHN` fail-closed (`GHN_ENABLED=false`, contract chưa code), `SPX`/`J&T` chưa có adapter, chỉ `MOCK` chạy. | Không thể tạo shipment thật ở production. | Shipment/DevOps |
| 2 | Unique `(order_id, shop_id)` = một shipment mỗi Shop Order chỉ khi Shop Order có đúng một `shop_id` bất biến. | Có thể trùng shipment nếu Order đổi mô hình. | Order owner |
| 3 | Chữ ký webhook hiện là HMAC-SHA256 chung; chữ ký/IP thật của từng carrier chưa chốt. `getRemoteAddr()` phía sau proxy cần cấu hình forwarded header. | Webhook thật bị từ chối hoặc kém an toàn. | Security |
| 4 | Token nội bộ `X-Internal-Service-Token` là giả định của service; chưa đồng bộ chính sách mTLS/token chung. | Phải đổi khi Gateway/platform chốt. | Platform |
| 5 | Buyer route so `X-User-ID` với `buyer_user_id` cả khi role `ADMIN` — admin không xem được qua route buyer. | Admin cần route riêng nếu muốn xem. | Product |
| 6 | Return/exchange, multi-package chưa thuộc v1. | Cần state/fee/refund workflow mới. | Product/Finance |
