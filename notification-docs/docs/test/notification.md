# Test Plan — Notification Service

> Nguồn: `docs/lld/notification.md` · `docs/db/notification.md` · `docs/api/notification.md` · Code: `ecommerce-taca/notification-service` @ `main` (`6c6d9a2`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — consumer Kafka, delivery Email/In-app + retry/outbox, notification center, preference và admin deliveries đã có code; retention, audit, metrics và một số mapping event/template chưa có.
> Môi trường: Jest unit test trong repo (`npm test`) · integration thủ công trên `docker compose` (MySQL, Kafka, Mailhog, mock JWKS)

## 1. Chuẩn bị

| Thành phần | Fixture |
|---|---|
| Runtime | NestJS + Jest (`npm test`, `npm run typecheck`); integration chạy với `docker compose up -d --build`. |
| Hạ tầng local | MySQL 8, Kafka (`kafka-init` tạo sẵn topic), Mailhog (`http://localhost:8025`), mock JWKS. |
| Auth | `node scripts/mock-auth/generate.mjs` sinh JWKS + test JWT (user thường và admin role). |
| Input | `./scripts/send-test-notification.sh <email>` gửi command `AUTH_VERIFICATION_REQUESTED`; fixture domain event cho từng `event_type` ở lld §6.2. |
| Data | Seed `npm run seed` (db §6.2): SENT/UNREAD, SENT/READ, FAILED + 3 attempt, SKIPPED, preference SECURITY locked, MARKETING disabled, processed event `evt-001`. |
| Actors | `user-1`, `user-2`, user không có role, admin (`SUPER_ADMIN`/`SUPPORT_VIEWER`). |
| Telemetry | Log JSON stdout, `X-Request-ID`, `traceparent`, redaction scan. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Email | Gửi command verification bằng script | Email hiện ở Mailhog, subject `Xác thực email`, không có token raw trong log. |
| Order flow | Publish `order.confirmed` (VNPAY và COD) | 1 EMAIL + 1 IN_APP `order-success-v1`; COD nhận ngay, không chờ `order.paid`. |
| In-app center | List, unread count, mark one/all read | Chỉ in-app của user; count giảm đúng; lặp lại không lỗi. |
| Preferences | Tắt `EMAIL`/`MARKETING`; thử tắt `SECURITY` | Email marketing `SKIPPED`; `SECURITY` → `403 NOTIFICATION_PREFERENCE_LOCKED`. |
| Failure | Tắt Mailhog/SMTP khi gửi | 3 attempt (2 `RETRYABLE_FAILED` + `RETRY_EXHAUSTED`), status `FAILED`, event `notification.failed`. |
| DLQ | Gửi JSON hỏng / `channel=SMS` / template không tồn tại | Message vào `notification.events.dlq.v1` với `INVALID_JSON` / `SCHEMA_INVALID` / `NOTIFICATION_TEMPLATE_NOT_FOUND`; consumer tiếp tục chạy. |
| Admin | Xem deliveries | Recipient mask, không có email đầy đủ. |

## 3. API integration

| ID | Endpoint/case | Expected | Trạng thái |
|---|---|---|---|
| N-API-01 | `GET /api/v1/notifications` không token / token sai | `401 NOTIFICATION_UNAUTHENTICATED`. | ✅ |
| N-API-02 | List của `user-1` | `200`, chỉ in-app của `user-1`, sort mới nhất trước, `meta.unread_count` đúng. | ✅ |
| N-API-03 | List `size=101`, `page=0`, `read_status=FOO`, field lạ | `400 NOTIFICATION_INVALID_INPUT` + `details`. | ✅ |
| N-API-04 | List user không có notification | `200 data=[]`. | ✅ |
| N-API-05 | `GET /unread-count` | Bằng số in-app `UNREAD`. | ✅ |
| N-API-06 | `PATCH /{id}/read` own / lặp lại | `200 {read:true}` cả hai lần. | ✅ |
| N-API-07 | `PATCH /{id}/read` của user khác / notification EMAIL / ID không tồn tại | `404 NOTIFICATION_NOT_FOUND`. | ✅ |
| N-API-08 | `PATCH /read-all` có/không `before` | `marked_count` đúng, `meta.unread_count` đúng, email không đổi. | ✅ |
| N-API-09 | `GET /preferences` | Chỉ row đã lưu, có `locked`. | ✅ |
| N-API-10 | `PUT /preferences` hợp lệ / thiếu field / enum sai | `200` trả 1 preference; `400`. | ✅ |
| N-API-11 | `PUT /preferences` tắt `SECURITY` | `403 NOTIFICATION_PREFERENCE_LOCKED`. | ✅ |
| N-API-12 | `GET /admin/notifications/deliveries` user thường / admin | `403 NOTIFICATION_FORBIDDEN` / `200` với `recipient_masked`, `attempt_count`, `provider_status`. | ✅ |
| N-API-13 | Admin filter `status=EXPIRED`, `recipient_hash`, `from/to` | `200`; `EXPIRED` trả rỗng ở seed hiện tại (giá trị reserved). | 🟡 |
| N-API-14 | `GET /health/live`, `/health/ready` (MySQL up/down) | Không envelope; ready `200 UP` / `503 DEGRADED`. | ✅ |
| N-API-15 | `X-Request-ID` hợp lệ / quá 64 ký tự | Echo nguyên / sinh UUIDv7 mới. | ✅ |

### 3.1 Kafka/delivery integration

| ID | Case | Expected | Trạng thái |
|---|---|---|---|
| N-INT-01 | Command cùng `event_id` gửi 2 lần | 1 notification, 1 lần gửi. | ✅ |
| N-INT-02 | 2 command khác `event_id`, cùng `dedupe_key` + user + channel | Lần 2 → DLQ `NOTIFICATION_IDEMPOTENCY_CONFLICT`. | ✅ |
| N-INT-03 | SMTP 4xx/timeout liên tục | Attempt 1–2 `RETRYABLE_FAILED`, attempt 3 `RETRY_EXHAUSTED`, `FAILED`, outbox `notification.failed`. | ✅ |
| N-INT-04 | SMTP 5xx | 1 attempt `PERMANENT_FAILED` `SMTP_REJECTED`, `FAILED`. | ✅ |
| N-INT-05 | Template không tồn tại | DLQ `NOTIFICATION_TEMPLATE_NOT_FOUND`, không tạo notification. | ✅ |
| N-INT-06 | Event không map (`payment.expired`) | Ghi `processed_events`, không tạo notification. | ✅ |
| N-INT-07 | `order.cancelled` | `order-cancelled-v1` EMAIL + IN_APP. | ✅ |
| N-INT-08 | `invoice.issued` | Chỉ EMAIL, `reference.type=ORDER`, id = `invoice_id`. | ✅ |
| N-INT-09 | Event dùng recipient fallback (`payload.user_id`) | Ingest được, log `consumer.recipient_fallback`. | ✅ |
| N-INT-10 | Event thiếu recipient | DLQ `NOTIFICATION_INVALID_INPUT`. | ✅ |
| N-INT-11 | Command `channel=IN_APP` có `recipient` | `recipient_encrypted`/`recipient_hash` null. | ✅ |
| N-INT-12 | Kill app khi notification `PROCESSING` | Sau `DELIVERY_STALE_AFTER_MS` sweeper reclaim, `attempt_no` tiếp tục tăng. | ✅ |
| N-INT-13 | Kafka publish lỗi khi relay | `retry_count` tăng, publish lại ở chu kỳ sau; payload camelCase. | ✅ |
| N-INT-14 | MySQL down khi consume | Không commit offset; message được xử lý lại khi DB lên. | ✅ |
| N-INT-15 | `MESSAGE_RECEIVED` với `message-received-v1` (chưa seed) | Hiện tại vào DLQ `NOTIFICATION_TEMPLATE_NOT_FOUND` — case theo dõi gap lld §8 #3. | ⏳ |

## 4. Unit test

Đã có trong repo (`*.spec.ts`):

| Module | Cases |
|---|---|
| `NotificationConsumer` | Command tạo notification + processed + dispatch; skip event đã xử lý; event không map vẫn mark processed; recipient từ `buyer` không log fallback; fallback có warning; IN_APP null hoá recipient. |
| `extractRecipient` (`contracts.spec`) | `buyer.user_id/email`; email optional; fallback `recipient.*`; fallback flat; không có field → `userId=null`. |
| `DispatcherService` | IN_APP không gọi provider; skip email khi preference disabled; bypass preference cho `SECURITY`; throw `DeliverySendError` khi gửi lỗi. |
| `DeliveryService` | SENT qua outbox; finalize lỗi không flip FAILED; SKIPPED; RETRY_EXHAUSTED; PERMANENT_FAILED; vào dispatch đã hết budget; lỗi lạ → `NOTIFICATION_INTERNAL_ERROR`; claim thất bại không dispatch; `attempt_no` đơn điệu. |
| `DeliveryOutboxRelayService` | Publish OK → mark published; lỗi → tăng `retry_count` + warn; một event lỗi không chặn event khác. |
| `DeliveryOutboxTypeOrmRepository` | `incrementRetryCount` atomic +1. |
| `InAppService` | List + unread count; mark read user khác/email → `NOT_FOUND`. |
| `PreferenceService` | Chặn tắt `SECURITY`; chặn tắt row locked; row `SECURITY` mới `locked=true`; thiếu row = enabled. |
| `TemplateService` | Render subject/body; HTML-escape; loại field ngoài allowlist; không có template → throw; resolve bản publish mới nhất. |
| `RequestIdMiddleware` | Tách trace/span từ `traceparent`; thiếu/hỏng → undefined. |
| `AppLogger` | Log `span_id` khi có; null khi không. |

Cần bổ sung: `SmtpEmailGateway.classifyError`, `AdminService` (`provider_status`/mask), `AllExceptionsFilter`, `KafkaConsumerAdapter` (DLQ/commit), `redact`.

## 5. Contract/security/resilience

- Contract Auth→Notification: envelope command (lld §6.2), `data` không chứa token; `channel=SMS` hiện vào DLQ.
- Contract Notification→Auth: payload `notification.delivered/failed` camelCase, key = `dedupeKey` — cần chốt với auth-user (lld §8 #4).
- Contract domain event: recipient `buyer.user_id/email`; payout `owner_*` chưa đọc (lld §8 #5).
- Security: JWT issuer/audience/JWKS; admin role; không trả email đầy đủ; log/DLQ scan không có email/token/OTP raw.
- Resilience: SMTP timeout, Kafka redeliver, crash giữa gửi và finalize (at-least-once), outbox relay retry.

## 6. Tiêu chí pass/phát hành

`npm run typecheck && npm test` pass trên CI; toàn bộ case §3 pass trên docker compose; không có email/token/OTP raw trong log hoặc DLQ; không có notification kẹt `PROCESSING` quá `DELIVERY_STALE_AFTER_MS` sau khi sweeper chạy.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | NestJS/MySQL + Email/In-app theo lựa chọn 1C. | Ảnh hưởng test harness/schema. | Tech lead |
| 2 | Chưa có integration/e2e test tự động trong repo; §3 chạy tay trên docker compose. | Regression khó phát hiện. | Notification owner |
| 3 | Provider SMTP production/DKIM chưa chốt. | Cần provider contract test thật. | DevOps |
| 4 | Template catalog v1 chỉ `vi-VN`. | Cần golden template khi mở rộng i18n. | Product |
