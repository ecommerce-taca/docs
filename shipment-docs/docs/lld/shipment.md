# LLD — Shipment Service

> Nguồn: `HighLevelDesign/HLD.md` · `docs/api/shipment.md` · `docs/db/shipment.md` · Code: `ecommerce-taca/shipment-service` @ `develop` (`a375a41`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — luồng shipment/webhook/outbox/reconciliation đã có code; adapter carrier thật chưa xong; Kafka consumer và publisher tắt mặc định.
> Tech stack: Java 25 · Spring Boot · Spring Data JPA + Flyway · MySQL 8.4 · Spring Kafka (outbox publisher + consumer opt-in)

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Shipment mỗi Shop Order, quote phí ship, create/cancel shipment, tracking timeline, carrier webhook, đối soát create mơ hồ, phát lifecycle event. |
| Database | MySQL `shipmentdb` (xem `docs/db/shipment.md`). |
| Carrier | Interface `CarrierAdapter`; có `MockCarrierAdapter` (chạy thật, tắt bằng `SHIPMENT_MOCK_ENABLED=false`) và `GhnCarrierAdapter` (fail-closed). `SPX`, `J&T` chưa có adapter. |
| Order | Order-Commerce sở hữu order state, seller authorization, address/item, quyết định bắt đầu fulfill. Shipment không đọc `orderdb`. |
| Payment | Shipment không capture/refund. |
| Không thuộc service | Giá sản phẩm/cart/order, inventory, payment, nguồn address, review. |

### 1.2 Boundary

```text
Order-Commerce ──REST quote / carrier-quotes / create / cancel──► Shipment
Order-Commerce ──Kafka order.cancelled (opt-in)──────────────────► Shipment
Shipment ──CarrierAdapter (MOCK; GHN fail-closed)────────────────► Carrier
Carrier ──POST /webhooks/shipping/{carrier} (IP + HMAC)──────────► Shipment
Shipment ──outbox → shipment.events.v1 (publisher opt-in)───────► Order / Notification
Buyer/Seller ──Gateway (actor headers)──► tracking read
```

`GET /seller/orders/{orderId}/shipment/carriers` là façade của Order-Commerce, gọi `POST /internal/shipping/carrier-quotes`.

## 2. Cấu trúc bên trong

### 2.1 Package

```text
com.taca.shipment
├── carrier/
│   ├── domain/        # CarrierAdapter, CarrierRegistry, ShippingQuote, CarrierCreateResult, CarrierTimeoutException
│   ├── ghn/           # GhnCarrierAdapter (fail-closed)
│   └── mock/          # MockCarrierAdapter
├── common/
│   ├── api/           # ApiResponse {data, meta.request_id}
│   ├── error/         # GlobalExceptionHandler, ShipmentException
│   └── id/            # ShipmentIds (shp-<uuid>), UuidV7
├── event/
│   ├── EventContext, EventEnvelopeFactory
│   └── consumer/      # OrderCancelledConsumer, ShipmentCreateCommandConsumer, *Inbox, *Processor, *ReplayController
├── health/            # HealthController (/health/live, /health/ready)
├── outbox/            # OutboxService, OutboxClaimService, OutboxPublisher, KafkaProducerSafetyGuard
├── quote/             # QuoteController, QuoteService
├── security/          # TrustedContextFilter
├── shipment/
│   ├── api/           # ShipmentController + request DTO
│   ├── domain/        # ShipmentCommandService, ShipmentStateMachine, ShipmentStatus, Carrier, ShipmentReconciliationJob
│   └── persistence/   # JPA entity + repository
└── webhook/           # CarrierWebhookController, ShipmentWebhookService, WebhookSignatureVerifier, log entity
```

| Thành phần | Trách nhiệm |
|---|---|
| `TrustedContextFilter` | Kiểm `X-Internal-Service-Token` cho `/internal/**`, operator token/role cho `/internal/ops/**`, actor header + role cho `/orders/**`, `/seller/**`. Bỏ qua health, webhook, path ngoài `/api/v1/`. |
| `ShipmentCommandService` | Create (idempotent, reserve-before-call), cancel, áp status carrier, build view và payload event. |
| `QuoteService` | Quote một carrier, so sánh 3 carrier. |
| `ShipmentWebhookService` | Verify IP/HMAC, dedupe, map status, chặn event cũ/transition sai. |
| `ShipmentReconciliationJob` | Tra carrier theo idempotency key cho shipment `PENDING_RECONCILIATION`. |
| `OutboxPublisher` | Claim + publish outbox lên Kafka, retry, DLQ. |

### 2.2 Observability

- Log JSON; không log address/phone/signature/raw webhook/token. Webhook chỉ lưu payload hash.
- `trace_id` lỗi lấy từ `X-Trace-ID` → `X-Request-ID` → UUID. Header W3C trace **chưa** được propagate qua Kafka.
- Actuator expose `health,info,metrics,prometheus`.

## 3. Luồng xử lý

### 3.1 Quote

1. Validate snapshot là JSON object ≤16 KiB.
2. Chọn carrier (mặc định `GHN`). `MOCK` chỉ khi mock bật và `APP_ENV` ∈ `local/test/staging`.
3. Gọi `adapter.quote`; gắn `quote_expires_at` = now + 30 phút (trừ `MOCK` giữ expiry của adapter).
4. `compare` lặp `GHN`, `SPX`, `J&T`: chưa có adapter → `ADAPTER_NOT_CONFIGURED`; lỗi → `CARRIER_UNAVAILABLE`/`QUOTE_FAILED`. Không fail cả request.

MOCK fee = `20000 + floor(tổng gram / 1000) × 3000` VND, ETA 1–3 ngày.

### 3.2 Create

```text
validate (currency, address ≤16KiB, items 1..100, COD chỉ khi payment_method=COD)
→ idempotency lookup (CREATE): hash khác → 409; có shipment → 200/202
→ đã có (order_id, shop_id) → 200/202
→ quote lại với carrier đã chọn → kiểm available / carrier / expiry / fee ≥ 0 / currency
→ TX1: insert shipments(PENDING_RECONCILIATION) + items + idempotency(PROCESSING)
→ adapter.create(key, order_id, carrierRequest)
   ├─ ok → TX2: CREATED + timeline + outbox shipment.created + idempotency COMPLETED → 201
   ├─ timeout / 5xx / 408 / 409 / 425 / 429 / lỗi lạ → timeline + outbox shipment.status_changed
   │     (reason CARRIER_OUTCOME_AMBIGUOUS) + idempotency PENDING_RECONCILIATION → 202
   └─ lỗi xác định khác → xoá reservation → trả lỗi
```

`carrierRequest` gửi carrier: `order_id`, `shop_id`, `idempotency_key = "shipment-create:<order_id>:<shop_id>"`, `currency`, `checkout_shipping_fee`, `carrier_quoted_fee`, snapshot, items, `cod_amount`.

### 3.3 Cancel

Kiểm idempotency (CANCEL) → version khớp → status `CREATED` → `adapter.cancel` (ngoài TX) → TX: re-check version, `CANCELLED`, timeline, outbox `shipment.cancelled` (payload có `reason`), idempotency COMPLETED.

### 3.4 Đối soát `PENDING_RECONCILIATION`

`ShipmentReconciliationJob.reconcile` chạy mỗi `SHIPMENT_RECONCILIATION_DELAY_MS` (60s), lấy tối đa 100 shipment cũ nhất. Nếu carrier tìm thấy theo `create_idempotency_key` → `CREATED`, timeline source `RECONCILIATION`, outbox `shipment.created` (payload rút gọn, **không** có field phí, thêm `reconciled: true`), đánh dấu command liên quan hoàn tất, idempotency COMPLETED. Không tìm thấy → giữ nguyên chờ lần sau (job không tự chuyển `CANCELLED`/`FAILED`).

### 3.5 Webhook

```text
carrier hợp lệ? → IP allowlist + HMAC → parse body → validate field
→ log trùng (carrier, external_event_id)? hash khác → 400, hash giống → duplicate
→ ghi log RECEIVED → tìm shipment theo (tracking_code, carrier)
→ map status → event cũ hơn event mới nhất? → state machine cho phép?
→ applyCarrierStatus: shipment + timeline + outbox (nếu state đổi) → log PROCESSED
```

Event phát theo state mới: `DELIVERED` → `shipment.delivered`, `FAILED` → `shipment.failed`, `CANCELLED` → `shipment.cancelled`, còn lại → `shipment.status_changed`.

### 3.6 Outbox

- Claim từng record bằng `SELECT … FOR UPDATE SKIP LOCKED` trong transaction ngắn, lease 60s (`SHIPMENT_OUTBOX_LEASE_SECONDS`), publish ngoài transaction, cập nhật được fence bằng `lease_token`.
- Thứ tự theo aggregate `created_at`/ID: record sau bị giữ khi record trước còn `PENDING`/`DLQ_PENDING` (không phải causal order tuyệt đối).
- Tối đa 3 lần gửi topic chính (backoff 2s, 4s) → `DLQ_PENDING` → tối đa 3 lần gửi DLQ → `DEAD` (giữ trong DB).
- Publisher chỉ chạy khi `SHIPMENT_KAFKA_PUBLISHER_ENABLED=true`; poll mỗi `OUTBOX_POLL_DELAY_MS` (1000ms). `KafkaProducerSafetyGuard` kiểm `acks=all`, idempotence, retries > 0, in-flight ≤ 5, `request.timeout < delivery.timeout < send-timeout < lease`.
- Service không tự tạo topic.

### 3.7 Kafka consumer (opt-in)

- `order.cancelled` trên `order.events.v1` (`SHIPMENT_KAFKA_CONSUMER_ENABLED=true`): record ≤256 KiB được lưu durable vào `order_cancellation_intents` trước khi ack; worker lease 120s, poll 250ms, chỉ huỷ khi shipment còn huỷ được; đang chờ đối soát thì `WAITING_RECONCILIATION`; hết retry → `NEEDS_OPERATOR` (replay qua ops API). Intent đến trước create trở thành tombstone chặn create.
- `shipment.commands.v1` (`SHIPMENT_CREATE_CONSUMER_ENABLED=true`): **đề xuất**, không phải contract create đã chọn. Lưu `shipment_create_commands` rồi worker gọi `ShipmentCommandService.create`; 5 lần retry có jitter → `DEAD` + `shipment_command_dlq`; record lỗi schema → DLT `shipment.consumer.dlq.v1` (Spring Kafka, 3 retry).

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---|---|---|
| Port | `8086` | `SERVER_PORT` | |
| DB URL | `jdbc:mysql://localhost:15406/shipmentdb` | `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | |
| Môi trường | `local` | `APP_ENV` | `MOCK` chỉ chạy ở `local/test/staging`. |
| Internal token | `local-dev-token-change-me` | `SHIPMENT_INTERNAL_TOKEN` | Bắt buộc đổi ngoài local. |
| Ops token | rỗng (ops API luôn 403) | `SHIPMENT_OPS_TOKEN` | |
| Enforce gateway headers | `true` | `SHIPMENT_ENFORCE_GATEWAY_HEADERS` | |
| Carrier request timeout | 5000 ms | `shipment.carrier.request-timeout-ms` | |
| Quote TTL | 30 phút | `shipment.carrier.quote-ttl-minutes` | |
| Mock enabled | `true` | `SHIPMENT_MOCK_ENABLED` | |
| GHN | tắt | `GHN_ENABLED`, `GHN_BASE_URL`, `GHN_TOKEN`, `GHN_SHOP_ID` | Bật cũng vẫn fail-closed vì contract chưa code. |
| Webhook secrets | MOCK: `local-mock-webhook-secret`, còn lại rỗng | `SHIPMENT_WEBHOOK_{GHN,SPX,JT,MOCK}_SECRET` | |
| Webhook IP allowlist | `127.0.0.1,::1` | `SHIPMENT_WEBHOOK_ALLOWED_IPS` | |
| Events topic | `shipment.events.v1` | `SHIPMENT_EVENTS_TOPIC` | |
| Outbox DLQ topic | `shipment.events.dlq.v1` | `SHIPMENT_DLQ_TOPIC` | |
| Publisher enabled | `false` | `SHIPMENT_KAFKA_PUBLISHER_ENABLED` | |
| Kafka send timeout | 15000 ms | `SHIPMENT_KAFKA_SEND_TIMEOUT_MS` | Producer request 5000 ms, delivery 10000 ms. |
| Outbox lease | 60 s | `SHIPMENT_OUTBOX_LEASE_SECONDS` | |
| Outbox poll | 1000 ms | `OUTBOX_POLL_DELAY_MS` | |
| Order cancel consumer | `false`, topic `order.events.v1` | `SHIPMENT_KAFKA_CONSUMER_ENABLED`, `ORDER_EVENTS_TOPIC` | |
| Create command consumer | `false`, topic `shipment.commands.v1`, group `shipment-service-create-v1` | `SHIPMENT_CREATE_CONSUMER_ENABLED`, `SHIPMENT_CREATE_COMMANDS_TOPIC`, `SHIPMENT_CREATE_CONSUMER_GROUP_ID` | |
| Consumer DLT | `shipment.consumer.dlq.v1`, 3 retry | `SHIPMENT_CONSUMER_DLQ_TOPIC`, `SHIPMENT_CONSUMER_RETRY_COUNT` | |
| Command worker | poll 250 ms, lease 120 s | `SHIPMENT_COMMAND_POLL_DELAY_MS`, `SHIPMENT_COMMAND_LEASE_SECONDS` | |
| Reconciliation | 60000 ms, batch 100 | `SHIPMENT_RECONCILIATION_DELAY_MS` | |
| Idempotency retention | 24h | — | |
| Item / snapshot limit | 100 dòng / 16 KiB | — | |
| Kafka message limit | 256 KiB | — | Consumer. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `Carrier` | `GHN`, `SPX`, `J_AND_T` (API: `J&T`), `MOCK`. |
| `ShipmentStatus` | `CREATED`, `PICKED_UP`, `IN_TRANSIT`, `DELIVERED`, `FAILED`, `CANCELLED`, `PENDING_RECONCILIATION`. |
| Timeline `source` | `SYSTEM`, `CARRIER`, `RECONCILIATION`. |
| Webhook log `status` | `RECEIVED`, `PROCESSED`, `REJECTED`, `UNKNOWN_STATUS`, `IGNORED_OLD`. |

Transition cho phép (`ShipmentStateMachine`; cùng state luôn hợp lệ):

| Từ | Sang |
|---|---|
| `PENDING_RECONCILIATION` | `CREATED`, `CANCELLED`, `FAILED` |
| `CREATED` | `PICKED_UP`, `IN_TRANSIT`, `CANCELLED`, `FAILED` |
| `PICKED_UP` | `IN_TRANSIT`, `DELIVERED`, `FAILED` |
| `IN_TRANSIT` | `DELIVERED`, `FAILED` |
| `FAILED` | `IN_TRANSIT` (carrier giao lại) |
| `DELIVERED`, `CANCELLED` | — (terminal) |

Cancel qua API chỉ khi `CREATED`. `NOT_CREATED` là giá trị ảo phía Order-Commerce, không thuộc enum này.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra — topic `shipment.events.v1`, key = `shipment_id`

Envelope (`EventEnvelopeFactory`): `event_id`, `schema_version` = 1, `event_type`, `occurred_at`, `aggregate_type` = `SHIPMENT`, `aggregate_id`, `correlation_id`?, `causation_id`?, `payload`.

| Event | Khi nào | Trạng thái |
|---|---|---|
| `shipment.created` | Carrier xác nhận create (REST) hoặc đối soát thành công. | ✅ |
| `shipment.status_changed` | State đổi sang `PICKED_UP`/`IN_TRANSIT`/`CREATED`, hoặc create mơ hồ → `PENDING_RECONCILIATION`. | ✅ |
| `shipment.delivered` | Webhook → `DELIVERED`. | ✅ |
| `shipment.failed` | Webhook → `FAILED` (chỉ nghĩa giao thất bại). | ✅ |
| `shipment.cancelled` | Cancel API hoặc webhook → `CANCELLED`. Chưa có consumer nào khai báo dùng. | ✅ |
| `shipment.creation_pending`, `shipment.creation_failed` | Chỉ từ create-command worker (đề xuất); `creation_failed` có `aggregate_type=SHIPMENT_COMMAND`, key = `command_id`. | 🟡 opt-in |

Payload chung: `shipment_id`, `order_id`, `shop_id`, `carrier`, `tracking_code`, `status`, `old_status`, `new_status`, `source`, `occurred_at`, `shipping_fee` (deprecated), `checkout_shipping_fee`, `carrier_quoted_fee`, `carrier_quote_delta`?, `carrier_actual_fee`?, `currency`, `reason`?. `shipment.delivered` thêm `delivered_at`; `shipment.delivered` và `shipment.failed` thêm `buyer: { user_id, email }` cho Notification. Ngoại lệ: `shipment.created` từ đối soát chỉ có `shipment_id, order_id, shop_id, carrier, tracking_code, status, new_status, reconciled` — consumer phải chịu được việc thiếu field phí.

Delivery at-least-once; consumer dedupe theo `event_id`.

### 6.2 Event lắng nghe

| Nguồn | Topic / event | Xử lý | Trạng thái |
|---|---|---|---|
| Order-Commerce | `order.events.v1` / `order.cancelled` | Huỷ shipment nếu còn `CREATED`; tombstone nếu chưa có shipment. Không tạo shipment từ `order.created`. | 🟡 opt-in |
| Order-Commerce (đề xuất) | `shipment.commands.v1` / `shipment.create` | Tạo shipment qua inbox. Không phải contract đã chọn: create chính thức là REST `POST /internal/shipments`, Order gọi khi seller xác nhận chuẩn bị hàng (`order-commerce-docs/docs/api/order.md` §3.4). | 🟡 opt-in |

## 7. Mã lỗi

Xem bảng đầy đủ ở `docs/api/shipment.md` §4. Tóm tắt nhóm: input (`SHIPMENT_INVALID_INPUT`, `SHIPMENT_CURRENCY_MISMATCH`, `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER`, `SHIPMENT_WEBHOOK_INVALID`) · auth (`SHIPMENT_UNAUTHENTICATED`, `SHIPMENT_FORBIDDEN`) · state (`SHIPMENT_NOT_FOUND`, `SHIPMENT_STATE_INVALID`, `SHIPMENT_ALREADY_EXISTS`, `SHIPMENT_TRACKING_CONFLICT`, `SHIPMENT_IDEMPOTENCY_CONFLICT`) · carrier (`SHIPMENT_CARRIER_INVALID_QUOTE`, `SHIPMENT_CARRIER_MISMATCH`, `SHIPMENT_CARRIER_QUOTE_EXPIRED`, `SHIPMENT_CARRIER_UNAVAILABLE`, `SHIPMENT_CARRIER_TIMEOUT`) · `SHIPMENT_INTERNAL_ERROR`.

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | GHN adapter chưa code contract thật; SPX/J&T chưa có adapter. | Không ship thật được. | Shipment/DevOps |
| 2 | Event `shipment.created` từ đối soát thiếu field phí. | Consumer đọc phí bị null. | Shipment (cần fix code) |
| 3 | Đối soát chỉ xử lý trường hợp carrier tìm thấy; không có timeout chuyển `FAILED`/`CANCELLED`. | Shipment kẹt `PENDING_RECONCILIATION`. | Shipment |
| 4 | Kafka command create và result event chỉ là đề xuất. | Không được coi là contract. | Order-Commerce/Platform |
| 5 | W3C trace chưa propagate qua Kafka. | Mất trace xuyên service. | Platform |
| 6 | Chữ ký webhook/IP thật của carrier chưa chốt. | Bảo mật webhook. | Security |
