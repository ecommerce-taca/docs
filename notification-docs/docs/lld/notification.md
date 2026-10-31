# LLD — Notification Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Code: `ecommerce-taca/notification-service` @ `main` (`6c6d9a2`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — consumer Kafka, delivery Email/In-app + retry/outbox, notification center, preference và admin deliveries đã có code; retention, audit, metrics và một số mapping event/template chưa có.
> Tech stack: Node.js 20 + NestJS 10 · MySQL 8/TypeORM · Kafka (kafkajs) consumer + transactional outbox · Email (SMTP/Nodemailer) + In-app

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Nhận command/domain event từ Kafka, chọn template, gửi Email/In-app, lưu delivery attempt, retry, phát kết quả delivery và phục vụ notification center (list/unread/read/preference). |
| Nguồn dữ liệu | MySQL `notificationdb` sở hữu notification, template, preference, delivery attempt, processed event, delivery outbox. |
| Channel v1 | `EMAIL`, `IN_APP`. Không có SMS/push — command có `channel` khác bị reject ở schema (xem §6.2). |
| Không thuộc service | Order/payment/shipment state, user profile/KYC, message chat, SMS/push provider. Notification không mutate domain state của service khác. |
| Delivery | At-least-once kèm dedupe theo `event_id`/`dedupe_key`; không đảm bảo exactly-once ở SMTP. |
| Privacy | Recipient lưu mã hoá AES-256-GCM + HMAC hash; template data chỉ giữ field có trong placeholder của template; logger tự redact key nhạy cảm. |

### 1.2 Boundary

```text
Order/Invoice/Shipment/Payment/Wallet ──Kafka domain event──► Notification
Auth User / Message / Rating ──Kafka notification.commands.v1──► Notification
Notification ──SMTP (Nodemailer)──► Email server (Mailhog local, Gmail/SMTP thật)
Buyer/Seller/Admin ──Gateway──► REST /api/v1/notifications/**, /api/v1/admin/notifications/**
Notification ──Kafka (outbox relay)──► notification.delivered.v1 / notification.failed.v1
Notification ──Kafka──► notification.events.dlq.v1 (message lỗi vĩnh viễn)
```

- Notification là delivery service, không quyết định order/payment/shipment state.
- Recipient (user ID + email) lấy từ payload event/command; không query UserDB.
- Không có REST endpoint để service khác gửi thông báo; mọi input đi qua Kafka.
- REST API tự verify JWT RS256 qua JWKS của auth-user (`JWT_JWKS_URL`, `JWT_ISSUER`, `JWT_AUDIENCE`); không dựa vào header `X-User-ID` của Gateway.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Notification | `notificationdb` MySQL → EMAIL | MySQL/TypeORM; SMTP adapter qua `EmailGatewayPort`. |
| Order flow | Order success + invoice email | Consume `order.confirmed` → `order-success-v1` (EMAIL + IN_APP) và `invoice.issued` → `invoice-issued-v1` (EMAIL). Không gắn email này vào `order.paid` vì với COD `order.paid` chỉ tới sau khi giao hàng. |
| Shipment flow | Delivered → review prompt | Consume `shipment.delivered` → `shipment-delivered-v1` (IN_APP). |
| UI | Notification center/read state | In-app list, unread count, mark read/read-all, preference. |
| KYC/security | Verification/reset/OTP | Command từ Auth User, category `SECURITY`, không tắt được qua preference. |

## 2. Cấu trúc bên trong

### 2.1 Module NestJS

Code theo ports & adapters: business module chỉ inject abstract port ở `domain/ports/`, adapter thật nằm ở `infrastructure/`.

```text
src/
├── main.ts · app.module.ts   # bootstrap, helmet, ValidationPipe, interceptor/filter toàn cục
├── config/                   # env schema (zod, fail-fast) + AppConfigService
├── common/
│   ├── http/                 # RequestContext (AsyncLocalStorage), request-id/logging middleware, envelope
│   ├── errors/               # ErrorCode catalog, AppException, AllExceptionsFilter
│   ├── logging/              # AppLogger JSON + redact
│   ├── security/             # recipient-crypto (AES-256-GCM), recipient-masking (HMAC/mask)
│   └── id/                   # uuidv7
├── domain/
│   ├── enums/                # Channel, NotificationStatus, AttemptStatus, ReadStatus, ...
│   ├── entities/             # 7 entity TypeORM
│   └── ports/                # repository/gateway/clock/id ports
├── modules/
│   ├── consumer/             # contracts (zod) + NotificationConsumer: dedupe, map, persist
│   ├── template/             # resolve/render (Handlebars), allowlist theo placeholder
│   ├── preference/           # preference + khoá SECURITY
│   ├── dispatcher/           # route EMAIL/IN_APP, áp preference, gọi EmailGatewayPort
│   ├── delivery/             # DeliveryService (claim/retry/finalize), sweeper, outbox relay
│   ├── notification/         # NotificationService (facade repository)
│   ├── in-app/               # NotificationController + InAppService
│   ├── admin/                # AdminController + AdminService (delivery diagnostics)
│   └── health/               # /health/live, /health/ready
└── infrastructure/
    ├── persistence/          # TypeORM repositories, migrations 001–010, seed, data-source
    ├── email/                # SmtpEmailGateway (nodemailer)
    ├── messaging/            # KafkaConsumerAdapter, KafkaEventPublisher
    ├── auth/                 # JwtTokenVerifier (jose + JWKS), AuthGuard, AdminGuard
    └── common/               # SystemClock, UuidV7IdGenerator
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `KafkaConsumerAdapter` | Subscribe command topic + domain event topics, parse JSON, phân loại command/event, commit offset thủ công | `autoCommit=false`; lỗi validate → DLQ rồi commit; lỗi hạ tầng → không commit (redeliver). |
| `NotificationConsumer` | Dedupe, map event → template, lọc data theo allowlist, persist `QUEUED`, ghi `processed_events` | Dispatch chạy fire-and-forget sau persist, không chặn commit offset. |
| `TemplateService` | Resolve bản `PUBLISHED` mới nhất theo key + locale `vi-VN`, render Handlebars | Allowlist = placeholder `{{field}}` trong subject/body; Handlebars HTML-escape mặc định. |
| `DispatcherService` | Route channel, kiểm preference (chỉ EMAIL), giải mã recipient, gửi email | `IN_APP` không gọi provider; `SECURITY` bỏ qua preference. |
| `SmtpEmailGateway` | Gửi qua Nodemailer, phân loại lỗi SMTP | Timeout `SMTP_TIMEOUT_MS` cho connect/greeting/socket. |
| `DeliveryService` | Atomic claim, vòng retry, ghi attempt, finalize SENT/FAILED kèm outbox trong một transaction | Lỗi DB sau khi provider gửi xong không được chuyển `FAILED`. |
| `DeliveryRetryService` | Sweeper định kỳ re-dispatch notification bị kẹt | Claim thật vẫn qua `tryClaimProcessing`. |
| `DeliveryOutboxRelayService` | Publish `delivery_outbox` chưa publish lên Kafka | Publish lỗi → tăng `retry_count`, giữ lại lần sau. |
| `InAppService` | List/unread/read/preference cho user hiện tại | Chỉ notification `IN_APP` của chính user. |
| `AdminService` | Delivery diagnostics, recipient đã mask | Chỉ role admin (§3.6). |

### 2.2 Chuẩn observability dùng chung

- Log JSON một dòng (stdout, `error` ra stderr). Field top-level: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `message`; field khác nằm trong `context` (vd. log `http.request` có `context.route`, `context.method`, `context.status_code`, `context.duration_ms`).
- `X-Request-ID` nhận từ client nếu ≤64 ký tự và khớp `^[A-Za-z0-9._:-]+$`, ngược lại sinh UUIDv7; luôn echo lại ở response header. `trace_id`/`span_id` tách từ W3C `traceparent`.
- Kafka message: đọc header `request_id` (hoặc `x-request-id`) và `traceparent`; `trace_id`/`request_id` được ghi vào `delivery_attempts`.
- `redact()` mask value của key khớp `email|phone|recipient|token|password|secret|authorization|otp|cookie|credential|card|bank|verification` và mask chuỗi trông như email/phone ở bất kỳ đâu. DLQ payload cũng đi qua `redact()`.
- Metrics/OpenTelemetry exporter **chưa implement** ở v1 (xem §8).
- Error envelope `{error:{code,message,details,trace_id}}`; `trace_id` fallback về `request_id` khi không có `traceparent`.

## 3. Luồng xử lý

### 3.1 Consume command/event

1. `KafkaConsumerAdapter` parse JSON. JSON hỏng → DLQ reason `INVALID_JSON`, commit.
2. Phân loại theo nội dung, **không theo topic**: có `command_type` → command; có `event_type` → domain event; không có cả hai → `NOTIFICATION_INVALID_INPUT` → DLQ.
3. Validate bằng zod (`NotificationCommandSchema` / `DomainEventSchema`). Sai schema → DLQ reason `SCHEMA_INVALID`, commit.
4. `processed_events` đã có `event_id` → bỏ qua (idempotent).
5. Command: resolve template theo field `template` của command. Domain event: tra `DOMAIN_EVENT_MAP` (§6.2); event không có trong map → ghi `processed_events` rồi bỏ qua (log `event.ignored`).
6. Domain event: lấy recipient bằng `extractRecipient` (§6.2); thiếu `user_id` → `NOTIFICATION_INVALID_INPUT` → DLQ.
7. Lọc `data` (command) hoặc `payload` (event) — chỉ giữ **key top-level** có trong placeholder của template.
8. Với mỗi channel: tìm notification theo `(recipient_user_id, dedupe_key, channel)`. Có rồi và cùng `source_event_id` → dùng lại; có rồi nhưng khác `source_event_id` → `NOTIFICATION_IDEMPOTENCY_CONFLICT` → DLQ. Chưa có → tạo `QUEUED` (`read_status=UNREAD` nếu `IN_APP`, recipient chỉ giữ khi `EMAIL`).
9. Ghi `processed_events` (`PROCESSED`), sau đó dispatch async các notification vừa tạo, rồi commit offset.
10. Lỗi khác (DB/Kafka) → không commit, Kafka redeliver.

`dedupe_key` của notification: command dùng `dedupe_key` của command; domain event dùng chính `event_id`.

### 3.2 Delivery và retry

1. `tryClaimProcessing`: `UPDATE ... SET status=PROCESSING, processing_started_at=now()` khi `status=QUEUED` hoặc (`PROCESSING` và lease cũ hơn `now - DELIVERY_STALE_AFTER_MS`). Không claim được → dừng.
2. `attempt_no` = `MAX(attempt_no)+1` (đơn điệu qua các lần reclaim). Nếu đã vượt `DELIVERY_RETRY_COUNT` → finalize `FAILED` với attempt `RETRY_EXHAUSTED`, `error_code=NOTIFICATION_DELIVERY_FAILED`.
3. Gọi `DispatcherService.dispatch`:
   - `IN_APP` → `sent` ngay (provider `in_app`), không kiểm preference.
   - `EMAIL` + preference `DISABLED` (trừ category `SECURITY`) → `skipped`.
   - `EMAIL` → giải mã recipient, render, gửi SMTP (provider `smtp`).
4. Kết quả:
   - `sent` → một transaction: `notifications.status=SENT`, `sent_at`, xoá lease + insert attempt `SENT` + insert outbox `notification.delivered`.
   - `skipped` → `status=SKIPPED`, insert attempt `SKIPPED`; **không** ghi outbox.
   - Lỗi SMTP retryable và còn budget → attempt `RETRYABLE_FAILED`, sleep backoff, thử lại.
   - Lỗi SMTP retryable hết budget → finalize `FAILED`, attempt `RETRY_EXHAUSTED` + outbox `notification.failed`.
   - Lỗi SMTP không retryable → finalize `FAILED`, attempt `PERMANENT_FAILED` + outbox `notification.failed`.
   - Lỗi khác (template thiếu, EMAIL không có recipient...) → finalize `FAILED`, attempt `PERMANENT_FAILED`, `error_code=NOTIFICATION_INTERNAL_ERROR`.
5. Nếu transaction finalize lỗi sau khi email đã gửi: giữ `PROCESSING`, sweeper reclaim sau khi lease hết hạn → có thể gửi lại (at-least-once).

Backoff giữa các lần: `min(DELIVERY_RETRY_BACKOFF_MS × 2^(attempt_no−1), 30000) + jitter[0, DELIVERY_RETRY_BACKOFF_MS)` ms — với baseline: ~2s rồi ~4s. Retry chạy in-process (sleep trong worker), không qua Kafka.

Sweeper (`DeliveryRetryService`) chạy mỗi `DELIVERY_SWEEP_INTERVAL_MS`, lấy tối đa 100 notification `QUEUED` có `updated_at` cũ hơn ngưỡng stale hoặc `PROCESSING` có lease hết hạn/null, order `created_at ASC`, rồi gọi lại `dispatch`.

Phân loại lỗi SMTP (`SmtpEmailGateway`):

| Điều kiện | `error_code` | Retryable |
|---|---|---|
| SMTP response 5xx | `SMTP_REJECTED` | Không |
| SMTP response 4xx | `SMTP_TRANSIENT` | Có |
| `ETIMEDOUT` / `ECONNREFUSED` / `ECONNECTION` | `SMTP_TIMEOUT` | Có |
| Lỗi khác | `SMTP_UNKNOWN` | Có |

### 3.3 In-app read/unread

- List chỉ trả notification `channel=IN_APP` của user trong JWT (`sub`); sort `created_at DESC, _id DESC`.
- Title/body được render lúc đọc từ template `PUBLISHED` mới nhất cùng key (không theo `template_version` đã lưu); không có template → `title`/`body` rỗng.
- `mark-read` idempotent; notification không thuộc user hoặc không phải `IN_APP` → `404 NOTIFICATION_NOT_FOUND`.
- `read-all` chỉ update `UNREAD` → `READ` (tuỳ chọn `created_at <= before`), không đụng email.

### 3.4 Preference

- Không có row preference = mặc định `ENABLED`.
- Preference chỉ có hiệu lực với `EMAIL`; `IN_APP` luôn được tạo và hiển thị.
- `SECURITY` không tắt được: `PUT` với `status=DISABLED` cho category `SECURITY` hoặc row `locked=true` → `403 NOTIFICATION_PREFERENCE_LOCKED`. Row mới tự set `locked=true` khi category `SECURITY`.
- Mỗi lần update tăng `version`.

### 3.5 Template/security

- Template key lưu dạng `<name>-v1` (version trong key) đồng thời có cột `version`; consumer luôn lấy bản `PUBLISHED` có `version` cao nhất, locale cố định `vi-VN`.
- Field ngoài placeholder bị loại trước khi lưu `notifications.data`; Handlebars escape HTML chống XSS.
- Recipient email: lưu `recipient_encrypted` (AES-256-GCM, key = SHA-256 của `RECIPIENT_ENCRYPTION_SECRET`, format `base64(iv‖tag‖ciphertext)`) và `recipient_hash` (HMAC-SHA256 trên giá trị đã `trim().toLowerCase()`, secret `RECIPIENT_HASH_SECRET`).

## 4. Hằng số & cấu hình

Env validate bằng zod khi boot (`src/config/env.schema.ts`); biến bắt buộc không có mặc định, thiếu thì app dừng.

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| HTTP port | 8080 | `PORT` | |
| Service name/version | `notification-service` / `0.1.0` | `SERVICE_NAME` / `SERVICE_VERSION` | Ghi vào log. |
| MySQL | — (port 3306) | `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `DB_NAME` | Bắt buộc trừ `DB_PORT`. |
| Kafka brokers | — | `KAFKA_BROKERS` | Bắt buộc, phân tách dấu phẩy. |
| Kafka client/group | `notification-service` | `KAFKA_CLIENT_ID` / `KAFKA_CONSUMER_GROUP_ID` | |
| Command topic | `notification.commands.v1` | `KAFKA_TOPIC_COMMANDS` | |
| Domain event topics | `''` | `KAFKA_TOPIC_DOMAIN_EVENTS` | `.env.example`/compose: `order.events.v1,invoice.events.v1,shipment.events.v1,payment.events.v1,wallet.events.v1`. Topic phải tạo trước (compose có `kafka-init`). |
| Delivered/failed topic | `notification.delivered.v1` / `notification.failed.v1` | `KAFKA_TOPIC_DELIVERED` / `KAFKA_TOPIC_FAILED` | |
| DLQ topic | `notification.events.dlq.v1` | `KAFKA_TOPIC_DLQ` | |
| SMTP host | — | `SMTP_HOST` | Bắt buộc. |
| SMTP port/secure | 1025 / `false` | `SMTP_PORT` / `SMTP_SECURE` | Gmail: 587 + App Password. |
| SMTP auth | `''` | `SMTP_USER` / `SMTP_PASSWORD` | Rỗng = không auth. |
| `EMAIL_SEND_TIMEOUT` | 5000 ms | `SMTP_TIMEOUT_MS` | 100–120000; áp cho connect/greeting/socket. |
| Người gửi | `Taca` / — | `EMAIL_FROM_NAME` / `EMAIL_FROM_ADDRESS` | Address bắt buộc. |
| JWT verify | — | `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_JWKS_URL` | Bắt buộc. |
| Secret recipient | — | `RECIPIENT_HASH_SECRET` / `RECIPIENT_ENCRYPTION_SECRET` | Bắt buộc, ≥32 ký tự. |
| `DELIVERY_RETRY_COUNT` | 3 | `DELIVERY_RETRY_COUNT` | **Tổng** số lần gửi tối đa (gồm lần đầu), 0–10. |
| Backoff base | 2000 ms | `DELIVERY_RETRY_BACKOFF_MS` | 0–60000; exponential cap 30s + jitter. |
| Chu kỳ sweeper | 30000 ms | `DELIVERY_SWEEP_INTERVAL_MS` | |
| Lease/stale | 60000 ms | `DELIVERY_STALE_AFTER_MS` | Lease `PROCESSING` và ngưỡng `QUEUED` bị kẹt. |
| Chu kỳ outbox relay | 5000 ms | `OUTBOX_RELAY_INTERVAL_MS` | |
| `PAGE_SIZE_DEFAULT` / max | 20 / 100 | Hard-code DTO | `page` ≥1. |
| Batch sweeper / relay | 100 | Hard-code | |
| Locale | `vi-VN` | Hard-code | |
| `TEMPLATE_KEY_MAX_LENGTH` | 100 | Schema `templates.key` | |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | TypeORM `timezone: 'Z'` | MySQL `DATETIME(6)`. |
| `IN_APP_RETENTION` | 90 ngày | — | ⏳ Chưa có job. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `Channel` | `EMAIL`, `IN_APP`. |
| `NotificationStatus` | `QUEUED`, `PROCESSING`, `SENT`, `FAILED`, `SKIPPED`, `EXPIRED`. |
| `AttemptStatus` | `STARTED`, `SENT`, `SKIPPED`, `RETRYABLE_FAILED`, `RETRY_EXHAUSTED`, `PERMANENT_FAILED`. |
| `ReadStatus` | `UNREAD`, `READ` (null với `EMAIL`). |
| `PreferenceStatus` | `ENABLED`, `DISABLED`. |
| `NotificationCategory` | `ORDER`, `SHIPMENT`, `PAYMENT`, `REVIEW`, `CONVERSATION`, `SHOP`, `SECURITY`, `MARKETING`. |
| `ReferenceType` | `ORDER`, `SHIPMENT`, `PAYMENT`, `REVIEW`, `CONVERSATION`, `SHOP`. |
| `TemplateStatus` | `DRAFT`, `PUBLISHED`. |
| `ProcessedEventStatus` | `PROCESSED`, `FAILED` (code hiện chỉ ghi `PROCESSED`). |

Chuyển trạng thái notification:

```text
QUEUED ──claim──► PROCESSING ──► SENT | FAILED | SKIPPED
PROCESSING (lease hết hạn) ──reclaim──► PROCESSING
```

- `EXPIRED` là giá trị **reserved**: không luồng nào set ở v1; giữ trong enum để filter admin `status=EXPIRED` hợp lệ.
- `STARTED` reserved: code không ghi attempt `STARTED` (attempt được ghi khi đã có kết quả).
- `RETRY_EXHAUSTED` = lỗi tạm thời nhưng hết budget; `PERMANENT_FAILED` = lỗi không retry được.

## 6. Event phát ra / lắng nghe

### 6.1 Event/command lắng nghe

| Nguồn | Topic | Event/command | Template/channel | Trạng thái |
|---|---|---|---|---|
| Auth User | `notification.commands.v1` | `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED`, `PHONE_OTP_REQUESTED` | Template do command chỉ định; category `SECURITY`. | 🟡 `PHONE_OTP_REQUESTED` với `channel=SMS` vào DLQ (không có SMS). |
| Message | `notification.commands.v1` | `MESSAGE_RECEIVED` | Template do command chỉ định; category `CONVERSATION`. | 🟡 Template `message-received-v1` chưa seed. |
| Rating-Comment | `notification.commands.v1` | `REVIEW_REQUESTED` | Template do command chỉ định; category `REVIEW`. | ✅ |
| Order-Commerce | `order.events.v1` | `order.confirmed`, `order.paid`, `order.cancelled` | Xem bảng §6.2. | ✅ |
| Order-Commerce | `invoice.events.v1` | `invoice.issued` | Xem bảng §6.2. | ✅ |
| Shipment | `shipment.events.v1` | `shipment.delivered`, `shipment.failed` | Xem bảng §6.2. | ✅ |
| Payment-Wallet | `payment.events.v1` | `payment.succeeded`, `payment.failed` | Xem bảng §6.2. | ✅ |
| Payment-Wallet | `wallet.events.v1` | `payout.succeeded`, `payout.failed` | Xem bảng §6.2. | 🟡 Recipient `owner_*` chưa đọc. |

Event khác trên các topic trên (vd. `payment.expired`, `payment.refunded`, `order.delivered`) **chưa được map** → ghi `processed_events` và bỏ qua.

| Event thiết kế | Template dự kiến | Kênh | Trạng thái |
|---|---|---|---|
| `payment.expired` | `payment-expired-v1` | EMAIL + IN_APP | ⏳ |
| `payment.refunded` | `payment-refunded-v1` | EMAIL + IN_APP | ⏳ |

### 6.2 Contract — command và domain event

Notification nhận **hai loại input, không được nhầm lẫn**:

1. **Command** trên `notification.commands.v1` — producer chỉ định template/kênh. `command_type` hợp lệ: `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED`, `PHONE_OTP_REQUESTED`, `MESSAGE_RECEIVED`, `REVIEW_REQUESTED`.
2. **Domain event** trên topic của service chủ — Notification tự map event → template; producer không cần biết template.

> **Không có command `ORDER_SUCCESS`.** Email xác nhận đơn đến từ domain event `order.confirmed`, map sang `order-success-v1`.

Envelope command (`NotificationCommandSchema`):

```json
{
  "event_id": "01912f55-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "command_type": "PASSWORD_RESET_REQUESTED",
  "occurred_at": "2026-08-30T09:00:00Z",
  "dedupe_key": "password-reset:user-01912f31:token-01912f56",
  "user_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
  "channel": "EMAIL",
  "recipient": "minhanh@example.com",
  "template": "auth-password-reset-v1",
  "data": {
    "display_name": "Nguyễn Minh Anh",
    "reset_url": "https://taca.vn/reset-password?t=opaque-token-reference",
    "expires_in_minutes": 30
  }
}
```

| Field | Rule |
|---|---|
| `event_id` | Bắt buộc, 1–64 ký tự; khoá dedupe ở `processed_events`. |
| `schema_version` | Bắt buộc, số nguyên dương. |
| `command_type` | Một trong 5 giá trị trên; giá trị khác → `SCHEMA_INVALID` → DLQ. |
| `occurred_at` | ISO-8601 datetime. |
| `dedupe_key` | Bắt buộc, 1–255 ký tự; unique ở `processed_events`. |
| `user_id` | Bắt buộc, 1–36 ký tự. |
| `channel` | `EMAIL` hoặc `IN_APP`. `SMS` → `SCHEMA_INVALID` → DLQ (không có phản hồi đồng bộ cho producer). |
| `recipient` | Email, ≤254 ký tự, mặc định `''`; bỏ qua khi `channel=IN_APP`. `EMAIL` mà rỗng → delivery `FAILED`. |
| `template` | Key template `PUBLISHED` (≤100); không tồn tại → `NOTIFICATION_TEMPLATE_NOT_FOUND` → DLQ. Code **không** kiểm template có khớp `command_type`. |
| `data` | Object, mặc định `{}`; chỉ giữ key có trong placeholder của template. Producer không gửi token raw (auth-user tự dựng `verification_url`/`reset_url`). |

Envelope domain event (`DomainEventSchema`):

```json
{
  "event_id": "01912f91-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "event_type": "order.confirmed",
  "occurred_at": "2026-08-30T09:03:15Z",
  "aggregate_type": "order",
  "aggregate_id": "order-01912f91",
  "payload": {
    "order_id": "order-01912f91",
    "order_code": "TC-20260830-0001",
    "display_name": "Nguyễn Minh Anh",
    "buyer": { "user_id": "01912f10-7a1b-7c12-9c55-8b1c34a6d921", "email": "minhanh@example.com" }
  }
}
```

`schema_version` mặc định 1; `aggregate_type`/`aggregate_id` optional (≤64). Field template (`order_code`, `display_name`, `amount`...) phải nằm **top-level** trong `payload` mới vào được template.

Ánh xạ domain event → template (`DOMAIN_EVENT_MAP`):

| Event nguồn | Template | Kênh | Category | `reference.type` / id lấy từ | Trạng thái |
|---|---|---|---|---|---|
| `order.confirmed` | `order-success-v1` | EMAIL + IN_APP | `ORDER` | `ORDER` / `order_id` | ✅ |
| `order.paid` | `payment-received-v1` | IN_APP | `ORDER` | `ORDER` / `order_id` | ✅ |
| `order.cancelled` | `order-cancelled-v1` | EMAIL + IN_APP | `ORDER` | `ORDER` / `order_id` | ✅ |
| `invoice.issued` | `invoice-issued-v1` | EMAIL | `ORDER` | `ORDER` / `invoice_id` | ✅ |
| `shipment.delivered` | `shipment-delivered-v1` | IN_APP | `SHIPMENT` | `SHIPMENT` / `shipment_id` | ✅ |
| `shipment.failed` | `shipment-failed-v1` | EMAIL + IN_APP | `SHIPMENT` | `SHIPMENT` / `shipment_id` | ✅ |
| `payment.succeeded` / `payment.failed` | `payment-result-v1` | EMAIL + IN_APP | `PAYMENT` | `PAYMENT` / `payment_id` | ✅ |
| `payout.succeeded` / `payout.failed` | `payout-result-v1` | EMAIL + IN_APP | `PAYMENT` | `PAYMENT` / `payout_id` | 🟡 recipient `owner_*` chưa đọc |

`reference_id` fallback về `aggregate_id` khi payload thiếu key.

Category cho command: `AUTH_VERIFICATION_REQUESTED`/`PASSWORD_RESET_REQUESTED`/`PHONE_OTP_REQUESTED` → `SECURITY`; `MESSAGE_RECEIVED` → `CONVERSATION`; `REVIEW_REQUESTED` → `REVIEW`. Command không có `reference`.

Recipient trong domain event (`extractRecipient`):

1. Field đã chốt: `payload.buyer.user_id` (bắt buộc) / `payload.buyer.email` (optional) — xem `order-commerce-docs/docs/lld/order.md` §6.1.
2. Fallback (log warning `consumer.recipient_fallback`): `payload.recipient.user_id/email`, rồi flat `user_id|buyer_id|recipient_user_id` và `email|buyer_email|recipient_email`.

> Payout: theo `payment-wallet-docs/docs/lld/payment-wallet.md` §6.1, code Payment-Wallet hiện không publish `payout.*` ra Kafka và payload không có recipient (không có `owner_user_id`/`owner_email`). Thiết kế cũ dùng `owner_user_id`/`owner_email`, nhưng `extractRecipient` cũng chưa đọc hai field này, nên email payout chưa gửi được (xem §8).

### 6.3 Reliability

- Commit offset thủ công sau khi persist notification + `processed_events`; dispatch chạy sau đó, độc lập với offset.
- Message lỗi vĩnh viễn (JSON hỏng, sai schema, `AppException` như template không tồn tại, thiếu recipient, idempotency conflict) → DLQ rồi commit. Lỗi hạ tầng → không commit, redeliver.
- DLQ `notification.events.dlq.v1`: key giữ nguyên, value `{topic, error, original}` với `original` đã `redact()`; `error` ∈ `INVALID_JSON`, `SCHEMA_INVALID` hoặc `ErrorCode`.
- Consumer không tự tạo topic; topic phải có trước khi app khởi động.
- Kết quả delivery publish qua transactional outbox (`delivery_outbox`), không publish trực tiếp trong luồng gửi.

### 6.4 Event phát ra

| Topic | Header `event_type` | Kafka key | Khi nào | Trạng thái |
|---|---|---|---|---|
| `notification.delivered.v1` | `notification.delivered` | `dedupeKey` | Attempt `SENT` (EMAIL hoặc IN_APP). | ✅ |
| `notification.failed.v1` | `notification.failed` | `dedupeKey` | Notification chốt `FAILED`. | ✅ |

Không phát event cho `SKIPPED`. Header còn có `event_id`. Payload (`DeliveryStatusEvent`, JSON **camelCase**):

```json
{
  "eventId": "01912fe6-7a1b-7c12-9c55-8b1c34a6d921",
  "outcome": "failed",
  "dedupeKey": "password-reset:user-01912f31:token-01912f56",
  "notificationId": "01912fe0-7a1b-7c12-9c55-8b1c34a6d921",
  "channel": "EMAIL",
  "templateKey": "auth-password-reset-v1",
  "errorCode": "SMTP_REJECTED",
  "occurredAt": "2026-08-30T09:00:07.120Z"
}
```

Relay chạy mỗi `OUTBOX_RELAY_INTERVAL_MS`, lấy tối đa 100 row `published_at IS NULL` order `created_at ASC`; at-least-once, consumer dedupe theo `eventId`. Auth-user khớp kết quả với command qua `dedupeKey` (xem `auth-user-docs/docs/lld/auth-user.md` §3.12 — doc đó ghi `dedupe_key`, xem §8).

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa | Nơi phát sinh |
|---|---:|---|---|
| `NOTIFICATION_INVALID_INPUT` | 400 | Query/body sai; message Kafka không phải command/event; event thiếu recipient. | ValidationPipe, consumer (→ DLQ). |
| `NOTIFICATION_UNAUTHENTICATED` | 401 | Thiếu/sai Bearer JWT. | `AuthGuard`. |
| `NOTIFICATION_FORBIDDEN` | 403 | Không có role admin. | `AdminGuard`. |
| `NOTIFICATION_NOT_FOUND` | 404 | Notification không tồn tại / không thuộc user / không phải in-app. | `PATCH /notifications/{id}/read`. |
| `NOTIFICATION_TEMPLATE_NOT_FOUND` | 409 | Template/version chưa có. | Consumer (→ DLQ), delivery (→ `FAILED`). |
| `NOTIFICATION_CHANNEL_DISABLED` | 409 | Reserved — chưa có đường code nào phát. | — |
| `NOTIFICATION_PREFERENCE_LOCKED` | 403 | Tắt category `SECURITY`/row `locked`. | `PUT /notifications/preferences`. |
| `NOTIFICATION_PROVIDER_UNAVAILABLE` | 503 | Reserved — chưa có đường code nào phát. | — |
| `NOTIFICATION_DELIVERY_FAILED` | 503 | Chỉ dùng làm `error_code` của attempt `RETRY_EXHAUSTED` khi vào dispatch đã hết budget. | `DeliveryService`. |
| `NOTIFICATION_IDEMPOTENCY_CONFLICT` | 409 | Cùng `(recipient_user_id, dedupe_key, channel)` nhưng khác `source_event_id`. | Consumer (→ DLQ). |
| `NOTIFICATION_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại; cũng là `error_code` attempt khi lỗi ngoài SMTP. | Filter, `DeliveryService`. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Notification dùng NestJS + MySQL, Email/In-app theo lựa chọn 1C. | Ảnh hưởng schema/consumer/deployment. | Tech lead |
| 2 | SMTP adapter v1 (Mailhog local, Gmail/SMTP thật); provider email production/DKIM chưa chốt. | Ảnh hưởng deliverability, SLA/retry. | DevOps/Product |
| 3 | **Gap code ↔ contract:** `payment.expired`, `payment.refunded` chưa có trong `DOMAIN_EVENT_MAP`; template `payment-expired-v1`, `payment-refunded-v1`, `message-received-v1` chưa seed → `MESSAGE_RECEIVED` của Message Service sẽ vào DLQ với `NOTIFICATION_TEMPLATE_NOT_FOUND`. | Buyer không nhận thông báo hết hạn/hoàn tiền/tin nhắn mới. | Notification owner |
| 4 | **Gap contract:** payload `notification.delivered/failed` là camelCase (`dedupeKey`...), auth-user doc mong `dedupe_key`. | Auth-user không khớp được kết quả delivery. | Notification + Auth owner |
| 5 | **Gap contract:** payout recipient `owner_user_id`/`owner_email` chưa được `extractRecipient` đọc. | Event payout vào DLQ do thiếu recipient. | Notification + Payment owner |
| 6 | Command `channel=SMS` bị reject ở schema → DLQ; auth-user doc ghi `409 NOTIFICATION_CHANNEL_DISABLED` nhưng không có phản hồi đồng bộ nào. | Phone OTP không hoàn tất (gap đã chấp nhận 2026-09-18). | Auth owner/Product |
| 7 | Chưa implement: retention in-app 90 ngày, ghi `notification_audits`, metrics/OpenTelemetry, readiness check Kafka/SMTP, trạng thái `EXPIRED`. | Storage/compliance/ops visibility. | Tech lead/Security |
| 8 | Data template chỉ lấy key top-level của payload; field lồng (vd. `buyer.display_name`) không render được. | Template hiển thị thiếu tên/mã nếu producer chỉ gửi lồng. | Notification owner + producer |
