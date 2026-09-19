# Database — Notification Service

> Nguồn: `docs/lld/notification.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Baseline: MySQL 8.x + TypeORM `notificationdb`; Email + In-app

## 1. Quy ước chung

| Quy ước | Quyết định | Ràng buộc |
|---|---|---|
| ID | UUIDv7 string | Dedupe key/event ID unique. |
| Time | DATETIME(6) UTC | API/event ISO-8601 UTC. |
| Source | Kafka command/event | Notification không là order/payment source. |
| Payload | Template data allowlist | Không lưu secret/OTP/password/payment credential. |
| Retention | Notification/log theo policy | In-app baseline 90 ngày (`docs/lld/notification.md` §4 `IN_APP_RETENTION`); TTL/archive chưa chốt compliance. |
| Observability | JSON log/W3C trace/X-Request-ID | Redact recipient/body/PII. |

## 2. Quan hệ

```mermaid
erDiagram
    NOTIFICATIONS ||--o{ DELIVERY_ATTEMPTS : has
    NOTIFICATIONS ||--o{ NOTIFICATION_AUDITS : audited
    NOTIFICATIONS ||--o{ DELIVERY_OUTBOX : emits
```

`notification_preferences` không có FK tới `notifications` — keyed theo `user_id` (unique `(user_id,channel,category)`).

## 3. Chi tiết bảng

### 3.1 `notifications`

| Field | Type | Ràng buộc |
|---|---|---|
| `_id` | string | UUIDv7. |
| `recipient_user_id` | string | Required Auth reference. |
| `channel` | enum | `EMAIL`, `IN_APP`. |
| `template_key`/`template_version` | string/int | Allowlist/versioned. |
| `dedupe_key` | string | Unique theo recipient/template/campaign policy. |
| `source_event_id` | string | Kafka event/command reference. |
| `data` | object | Validated allowlist, redacted fields only. |
| `status` | enum | `QUEUED`, `PROCESSING`, `SENT`, `FAILED`, `SKIPPED`, `EXPIRED`. |
| `read_status` | enum | In-app `UNREAD/READ`; null với email. |
| `version` | int | Optimistic concurrency cho `PATCH /notifications/{id}/read` (API gửi optional `version`, xem `docs/api/notification.md` §3.2). |
| `scheduled_at`/`sent_at` | DATETIME(6) | UTC. |
| `created_at`/`updated_at` | DATETIME(6) | UTC. |
| `recipient_encrypted` | text | Recipient email/phone mã hoá AES-256-GCM (không lưu plaintext); null với `IN_APP`. |
| `recipient_hash` | char(64) | HMAC-SHA256 của recipient (filter admin, không lộ thô). |
| `category` | enum | `ORDER`/`SHIPMENT`/`PAYMENT`/`REVIEW`/`CONVERSATION`/`SHOP`/`SECURITY`/`MARKETING`. |
| `reference_type`/`reference_id` | enum/string | Deep-link reference (nullable). |
| `processing_started_at` | DATETIME(6) | Nullable. Lease cho atomic claim; sweeper chỉ reclaim khi lease đã quá `staleBefore`. |

### 3.2 `delivery_attempts`

`_id`, `notification_id`, `attempt_no`, `provider`, `status`, `provider_status` (string, masked — trạng thái provider của attempt, nguồn của field `provider_status` trong API §3.4), `provider_message_id` masked, `error_code`, `started_at`, `finished_at`, `trace_id`, `request_id`. Không lưu rendered body/raw provider response.

### 3.3 `notification_preferences`

`_id`, `user_id`, `channel`, `category`, `status`, `locked` (`TINYINT(1)`, `0/1` — `1` cấm opt-out, chỉ set cho category `SECURITY`), `updated_at`, `version`. Unique `(user_id,channel,category)`. Security-critical Auth notification có thể bypass disable theo policy.

### 3.4 `templates`, `notification_audits`, `processed_events`

- `templates`: key/version/locale/subject/body safe/status/created_at; template published immutable.
- `notification_audits`: actor/action/target/reason/metadata/occurred_at; no secret/body.
- `processed_events`: event_id/dedupe_key/processed_at/status/error; unique event ID.

### 3.5 `delivery_outbox`

`_id` (event ID, UUIDv7), `aggregate_id` (→ `notifications._id`), `event_type` (`notification.delivered`/`notification.failed`), `payload` (JSON — toàn bộ delivery status event: `dedupe_key`/`notification_id`/`channel`/`template_key`/`error_code`/`occurred_at`), `created_at`, `published_at` (nullable, null tới khi relay worker publish thành công), `retry_count` (default `0`, tăng khi relay publish lỗi). Transactional outbox: ghi cùng 1 transaction với đổi `status` của `notifications` + tạo `delivery_attempts`, để relay worker publish Kafka (`notification.delivered.v1`/`notification.failed.v1`) sau mà không mất event nếu crash giữa persist và publish.

> Toàn bộ bảng trên là **MySQL** (`notificationdb`), không phải MongoDB — chữ "collection" ở các bản trước là gõ nhầm thuật ngữ, giữ nguyên schema/field như trên nhưng đọc là **bảng**.

## 4. Index

| Bảng | Index |
|---|---|
| `notifications` | unique `(recipient_user_id,dedupe_key,channel)`; `(recipient_user_id,created_at)`; `(recipient_user_id,read_status,created_at)`; `(status,scheduled_at)`; `(recipient_hash)`; `(status,processing_started_at)` (phục vụ `tryClaimProcessing`/`findPendingDelivery`) |
| `delivery_attempts` | `(notification_id,attempt_no)`; `(status,started_at)` |
| `notification_preferences` | unique `(user_id,channel,category)` |
| `templates` | unique `(key,version,locale)`; `(key,status)` |
| `processed_events` | unique `event_id`; unique `dedupe_key`; `(processed_at)` |
| `notification_audits` | `(target_id,occurred_at)`; `(actor_user_id,occurred_at)` |
| `delivery_outbox` | `(published_at,created_at)` (relay worker lấy batch event chưa publish, order theo `created_at`) |

## 5. Enum và rules

Channel `EMAIL/IN_APP`; status `QUEUED/PROCESSING/SENT/FAILED/SKIPPED/EXPIRED`; `AttemptStatus` `STARTED/SENT/SKIPPED/RETRYABLE_FAILED/RETRY_EXHAUSTED/PERMANENT_FAILED` (khớp `docs/lld/notification.md` §5); retry max 3 (tổng số lần gửi gồm lần đầu, `attempt_no` 1–3); no raw secret/template body in logs/events; unread count không âm.

## 6. Migration và seed

### 6.1 Thứ tự migration

| Thứ tự | Nội dung | Phụ thuộc |
|---:|---|---|
| 001 | Tạo database `notificationdb`, charset/collation, migration metadata | — |
| 002 | Tạo bảng `templates` (unique `(key,version,locale)`, index `(key,status)`) | — |
| 003 | Tạo bảng `notifications` | `templates` |
| 004 | Tạo bảng `delivery_attempts`, FK → `notifications` | `notifications` |
| 005 | Tạo bảng `notification_preferences` (unique `(user_id,channel,category)`) | — |
| 006 | Tạo bảng `notification_audits` | — |
| 007 | Tạo bảng `processed_events` (unique `event_id`, unique `dedupe_key`) | — |
| 008 | Thêm index trên `notifications`: unique `(recipient_user_id,dedupe_key,channel)`, `(recipient_user_id,created_at)`, `(recipient_user_id,read_status,created_at)`, `(recipient_hash)`, `(status,scheduled_at)` | `notifications` |
| 008a | Thêm bảng `delivery_outbox` (§3.5) + cột `processing_started_at` + mở rộng enum `delivery_attempts.status` — tương ứng migration `1750000000009-delivery-hardening.ts` ở code | `notifications` |
| 009 | Seed template v1 + fixture cho local/test (§6.2) | Tất cả bảng trên |
| 010 | Thêm index `(status,processing_started_at)` trên `notifications`, phục vụ `tryClaimProcessing`/`findPendingDelivery` | 008a |

> Ghi chú: bảng trên phản ánh thứ tự logic của docs; ở code, cột `processing_started_at`, bảng `delivery_outbox` và enum `delivery_attempts.status` vào cùng migration `1750000000009-delivery-hardening.ts` — đã ghi là dòng `008a` ở trên để khớp phụ thuộc của `010`.

### 6.2 Seed tối thiểu

| Seed | Giá trị |
|---|---|
| `templates` | `order-success-v1`, `payment-received-v1`, `order-cancelled-v1`, `invoice-issued-v1`, `shipment-delivered-v1`, `shipment-failed-v1`, `payment-result-v1`, `payment-expired-v1`, `payment-refunded-v1`, `payout-result-v1`, `message-received-v1`, `auth-email-verification-v1`, `auth-password-reset-v1`, `review-request-v1` — mỗi template locale `vi-VN`, `status=PUBLISHED`. |
| `notification_preferences` | 1 user với `category=SECURITY,locked=true` (test không opt-out được); 1 user tắt `category=MARKETING`. |
| `notifications` | 1 `SENT` in-app `read_status=UNREAD`; 1 `SENT` đã `READ`; 1 `FAILED` sau 3 lần gửi (gồm lần đầu, `attempt_no` 1–3); 1 `SKIPPED` (do preference disabled); 1 `EXPIRED` (reserved — test filter `status=EXPIRED`, xem `test/notification.md` N-API-02b). |
| `delivery_attempts` | Đủ attempt khớp fixture `FAILED` ở trên (3 attempt, `attempt_no` 1-3). |
| `processed_events` | 1 event đã xử lý — test dedupe không gửi lặp. |

Không seed OTP/password/payment secret hoặc recipient email/phone thật — dùng placeholder masked.

### 6.3 Kiểm tra bắt buộc trước khi chạy migration production

1. Verify `dedupe_key` không trùng trong cùng `(recipient_user_id,channel)` trước khi tạo unique index.
2. Verify mọi `template_key` trong `notifications` đang dùng có tồn tại trong `templates` (không FK cứng nhưng phải reconcile bằng script trước go-live).

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | MySQL/NestJS, Email + In-app theo lựa chọn 1C. | Ảnh hưởng schema/ops. | Tech lead |
| 2 | SMTP/Mock provider v1, exact provider/DKIM chưa chốt. | Ảnh hưởng delivery/security. | DevOps |
| 3 | Notification retention 90 ngày baseline. | Ảnh hưởng storage/privacy. | Product/Security |
| 4 | Dedupe theo recipient/template/event policy. | Ảnh hưởng email duplicate behavior. | Platform owner |
