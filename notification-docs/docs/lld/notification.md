# LLD — Notification Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Node.js + NestJS · MySQL/TypeORM · Kafka consumer · Email + In-app · SMTP/Mock adapter

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Nhận notification command/event, render template, gửi Email/In-app, lưu delivery log, retry và đọc notification center. |
| Nguồn dữ liệu | MySQL `notificationdb` sở hữu notification log, preference và delivery attempt metadata. |
| Channel v1 | `EMAIL`, `IN_APP`; HLD yêu cầu email order success/invoice, schema có cả in-app. |
| Không thuộc service | Order/payment/shipment state, user profile/KYC, template business ownership, message chat, SMS/push provider. |
| Delivery | At-least-once với dedupe key; không đảm bảo exactly-once ở external SMTP. |
| Privacy | Không log raw token/OTP/password/payment/bank data; template data allowlist và redaction. |

### 1.2 Boundary

```text
Order/Payment/Shipment/Auth ──Kafka command/event──► Notification
Notification ──SMTP/Mock──► Email provider
Buyer/Seller/Admin ──Gateway──► In-app notification API
Notification ──Kafka/REST──► không mutate domain state
```

- Notification là delivery service, không quyết định order/payment/shipment state.
- Consumer dedupe theo `event_id`/`dedupe_key`; retry không gửi lặp nếu provider hỗ trợ idempotency, nếu không phải kiểm soát local attempt.
- User recipient identity lấy từ event/allowlist; không query UserDB trực tiếp trong request path.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Notification | Notification DB, Email delivery | MySQL vì lựa chọn `1C`; email provider adapter. |
| Order flow | Order success + invoice email | Consume `order.confirmed` và `invoice.issued` contract. `order.confirmed` phát cho **cả** VNPAY lẫn COD tại lúc đơn chốt; `order.paid` (tiền về) là event riêng, với COD chỉ tới sau khi giao hàng. |
| Shipment flow | Delivered → leave review prompt | Consume `shipment.delivered`, render review prompt. |
| UI | Notification center/read state | In-app list, unread count, mark read. |
| KYC/security | Verification/reset/security messages | Consume notification commands từ Auth User; không lưu secret raw. |

## 2. Cấu trúc bên trong

### 2.1 Module NestJS

```text
src/
├── notification/         # notification aggregate/query
├── template/             # versioned templates + locale
├── dispatcher/           # channel routing and provider adapter
├── consumer/             # Kafka event/command consumer
├── preference/           # user channel preference
├── delivery/             # retry/backoff/attempt state
├── in-app/               # unread/read API
├── security/             # payload allowlist/redaction
├── outbox/               # optional internal event/audit
└── observability/        # shared telemetry contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `NotificationConsumer` | Consume domain event/command | Dedupe event, validate schema, commit offset sau persist. |
| `TemplateService` | Resolve template/version/locale | Không render arbitrary HTML; template version immutable sau publish. |
| `Dispatcher` | Route Email/In-app | Không gửi channel ngoài allowlist; preference policy áp dụng trừ security-critical. |
| `EmailAdapter` | SMTP/Mock send | Timeout/retry bounded; không log raw content. |
| `InAppService` | Persist/read/unread | User scope; unread count consistent eventual. |
| `DeliveryService` | Attempt/retry/dead-letter | Retry 3/backoff 2s; status rõ ràng. |
| `ObservabilityModule` | Logs/traces/metrics/health | Chuẩn exact với auth-user/Gateway. |

### 2.2 Chuẩn observability dùng chung

- Structured JSON stdout, field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- Propagate W3C `traceparent`/`tracestate`, `X-Request-ID`; Kafka headers giữ `traceparent`, `request_id`, `event_id`.
- Không log email/phone raw, OTP/reset token, password, payment/bank/address data, full rendered body hoặc provider credential; chỉ recipient hash/notification ID allowlist.
- Metrics OpenTelemetry/Prometheus-compatible, label bounded (`channel`, `template`, `result`, `provider`), không label user/email/order ID.
- `/health/live` process-only; `/health/ready` kiểm MySQL/Kafka/SMTP config theo policy.
- Error envelope `{error:{code,message,details,trace_id}}`; delivery failure log phải có correlation nhưng không raw payload.

## 3. Luồng xử lý

### 3.1 Consume và dispatch

1. Nhận event/command, validate envelope/schema/version và tạo trace context.
2. Tính `dedupe_key`, kiểm notification đã completed/processing.
3. Resolve recipient/channel/template/version/locale từ allowlist.
4. Persist notification `QUEUED` và delivery attempt trong MySQL.
5. Dispatcher gửi Email/In-app; success → `SENT`, fail → retry hoặc `FAILED/DLQ`.
6. Commit Kafka offset sau persist đủ để replay an toàn.

### 3.2 In-app read/unread

- Notification center chỉ trả notification của actor user; newest-first pagination.
- `mark-read` idempotent; unread count eventual nhưng không âm.
- Notification expired/archived không xóa audit delivery; retention job chỉ xóa sau policy.

### 3.3 Template/security

- Template dùng versioned key (`order-success-v1`, `payment-received-v1`, `invoice-issued-v1`, `review-request-v1`).
- Data fields được allowlist theo template; HTML sanitize; link token phải opaque, TTL và không log.
- Security-critical Auth command không bị tắt bởi preference nếu policy yêu cầu; promotional notification chưa thuộc v1.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `PAGE_SIZE_DEFAULT` | 20 | Max 100. |
| `DELIVERY_RETRY_COUNT` | 3 | Tổng số lần gửi tối đa (gồm lần đầu). Backoff exponential + jitter, rồi DLQ. |
| `EMAIL_SEND_TIMEOUT` | 5s | SMTP/Mock adapter (`SMTP_TIMEOUT_MS`). |
| `OUTBOX_RELAY_INTERVAL_MS` | 5s | Relay publish delivery status từ outbox. |
| `IN_APP_RETENTION` | 90 ngày | Cần confirm compliance. |
| `TEMPLATE_KEY_MAX_LENGTH` | 100 | Allowlist only. |
| `RENDERED_BODY_MAX_LENGTH` | 100000 | Không log body. |
| `KAFKA_BATCH_SIZE` | 100 | Consumer bounded. |
| `CONSUMER_MAX_LAG_ALERT` | 60s | Alert. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | MySQL DATETIME. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `Channel` | `EMAIL`, `IN_APP`. |
| `NotificationStatus` | `QUEUED`, `PROCESSING`, `SENT`, `FAILED`, `SKIPPED`, `EXPIRED`. |
| `AttemptStatus` | `STARTED`, `SENT`, `SKIPPED`, `RETRYABLE_FAILED`, `RETRY_EXHAUSTED`, `PERMANENT_FAILED`. |
| `ReadStatus` | `UNREAD`, `READ`. |
| `PreferenceStatus` | `ENABLED`, `DISABLED`. |

Retryable provider failure không mark `SENT`; duplicate completed event trả idempotent result.

## 6. Event phát ra / lắng nghe

### 6.1 Event/command lắng nghe

| Nguồn | Event/command | Template/channel |
|---|---|---|
| Auth User | `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED`, `PHONE_OTP_REQUESTED` | Email/critical channel; không lưu raw secret. |
| Order-Commerce | `order.confirmed`, `order.paid`, `invoice.issued`, `order.cancelled` | Order/invoice/cancel Email + In-app. |
| Shipment | `shipment.delivered`, `shipment.failed` | Review prompt/tracking update. |
| Payment-Wallet | `payment.succeeded/failed`, `payout.succeeded/failed` | Payment/wallet notification. |

### 6.2 Contract — notification command

Notification nhận **hai loại input, không được nhầm lẫn**:

1. **Domain event** trên topic của service chủ (`order.events.v1`, `invoice.events.v1`, `shipment.events.v1`, `payment.events.v1`, `wallet.events.v1`) — Notification **tự** ánh xạ event → template. Producer không cần biết template.
2. **Command** trên `notification.commands.v1` — dùng khi producer cần chỉ định rõ template/kênh (verification, reset password, OTP, message, review request). Command type hợp lệ chỉ gồm: `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED`, `PHONE_OTP_REQUESTED`, `MESSAGE_RECEIVED`, `REVIEW_REQUESTED`.

> **Không có command `ORDER_SUCCESS`.** Email xác nhận đơn hàng đến từ domain event `order.confirmed` (không phải command), Notification map sang template `order-success-v1`.
>
> **Phải dùng `order.confirmed`, không dùng `order.paid`.** Với VNPAY hai event trùng thời điểm nên chọn cái nào cũng ra kết quả giống nhau; với COD `order.paid` chỉ tới **sau khi hàng đã giao xong**, nên nếu gắn email xác nhận đơn vào `order.paid` thì buyer COD không bao giờ nhận được email + hoá đơn tại lúc đặt hàng — trái yêu cầu HLD "Notify to user email when order success with invoice".

Envelope command:

```json
{
  "event_id": "event-01912fb0",
  "schema_version": 1,
  "command_type": "AUTH_VERIFICATION_REQUESTED",
  "occurred_at": "2026-08-30T12:00:00Z",
  "dedupe_key": "auth-verification:user-1:token-01912fb1",
  "user_id": "user-1",
  "channel": "EMAIL",
  "recipient": "masked-at-runtime",
  "template": "auth-email-verification-v1",
  "data": { "verification_url": "https://taca.vn/verify?t=…", "expires_in_minutes": 30 }
}
```

Ánh xạ domain event → template (Notification sở hữu bảng này):

| Event nguồn | Template | Kênh |
|---|---|---|
| `order.confirmed` | `order-success-v1` | EMAIL + IN_APP |
| `order.paid` | `payment-received-v1` | IN_APP |
| `order.cancelled` | `order-cancelled-v1` | EMAIL + IN_APP |
| `invoice.issued` | `invoice-issued-v1` | EMAIL |
| `shipment.delivered` | `shipment-delivered-v1` | IN_APP |
| `shipment.failed` | `shipment-failed-v1` | EMAIL + IN_APP |
| `payment.succeeded` / `payment.failed` | `payment-result-v1` | EMAIL + IN_APP |
| `payout.succeeded` / `payout.failed` | `payout-result-v1` | EMAIL + IN_APP |

Payload producer không gửi password/token/card; consumer phải reject field ngoài allowlist.

> Field recipient trong payload domain event: xem `order-commerce.md` §6.1 — `buyer.user_id`
> (bắt buộc) / `buyer.email` (optional với event chỉ phát `IN_APP`, ví dụ `order.paid`).

### 6.3 Reliability

- Kafka commit sau MySQL persist; provider send retry độc lập với consumer offset.
- Dedupe `dedupe_key` unique theo recipient/template/event; DLQ lưu redacted payload.
- MySQL/Kafka/SMTP down: readiness/degraded metric; không giả báo sent.

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `NOTIFICATION_INVALID_INPUT` | 400 | Command/query/template field sai. |
| `NOTIFICATION_UNAUTHENTICATED` | 401 | In-app API thiếu login. |
| `NOTIFICATION_FORBIDDEN` | 403 | Sai recipient/user scope. |
| `NOTIFICATION_NOT_FOUND` | 404 | Notification không tồn tại. |
| `NOTIFICATION_TEMPLATE_NOT_FOUND` | 409 | Template/version chưa có. |
| `NOTIFICATION_CHANNEL_DISABLED` | 409 | Channel bị tắt theo preference/policy. |
| `NOTIFICATION_PROVIDER_UNAVAILABLE` | 503 | SMTP/Kafka/MySQL unavailable. |
| `NOTIFICATION_DELIVERY_FAILED` | 503 | Gửi thất bại sau policy. |
| `NOTIFICATION_IDEMPOTENCY_CONFLICT` | 409 | Dedupe key khác payload. |
| `NOTIFICATION_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Notification dùng NestJS + MySQL, Email/In-app theo lựa chọn 1C. | Ảnh hưởng schema/consumer/deployment. | Tech lead |
| 2 | SMTP/Mock adapter v1; provider email thật chưa chốt. | Ảnh hưởng deliverability, DKIM/SLA/retry. | DevOps/Product |
| 3 | HLD nêu email order success/invoice và shipment delivered; template catalog chi tiết cần chốt. | Ảnh hưởng event/template coverage. | Product owner |
| 4 | In-app retention 90 ngày và preference policy là baseline. | Ảnh hưởng storage/privacy. | Product/Security |
| 5 | SMS/push/promotional notification chưa thuộc v1. | Cần channel/provider/template mới nếu bật. | Product owner |
