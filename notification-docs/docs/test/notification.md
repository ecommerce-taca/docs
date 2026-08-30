# Test Plan — Notification Service

> Nguồn: `docs/lld/notification.md` · `docs/db/notification.md` · `docs/api/notification.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture |
|---|---|
| Runtime | NestJS + Jest/Supertest, MySQL/Kafka containers. |
| Providers | SMTP/Mock Email adapter, provider timeout/retry/error. |
| Events | Auth verification/reset, order paid/invoice issued, shipment delivered, payment/payout. |
| Data | Queued/processing/sent/failed/skipped, unread/read, duplicate dedupe key, template versions. |
| Actors | User own notifications, other user, admin delivery read. |
| Telemetry | JSON log, W3C trace, X-Request-ID, PII/secret redaction scan. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| In-app center | List, unread count, mark one/all read | Đúng user scope, count không âm, retry idempotent. |
| Email | Order success/invoice/shipment delivered | Template đúng, không lộ token/payment/address. |
| Auth message | Reset/OTP/verification command | Secret không lưu/log raw; delivery retry đúng. |
| Preferences | Disable Email/In-app category | Preference áp dụng; security-critical policy rõ. |
| Failure | SMTP down, Kafka duplicate/DLQ | Không báo sent giả; replay không gửi duplicate theo dedupe. |
| UI states | Loading/empty/error/offline | Empty không nhầm error; trace/request ID khi báo lỗi. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| N-API-01 | `GET /notifications` own/other user | Own `200`; other denied/no leak. |
| N-API-02 | List pagination/filter/empty | Bounded page/size, `200 data=[]`. |
| N-API-03 | `GET /notifications/unread-count` | Exact non-negative count. |
| N-API-04 | Mark one/all read/repeat | Idempotent, email status unchanged. |
| N-API-05 | Preferences valid/invalid/unauth | Correct update/`400/401`; cannot create template. |
| N-API-06 | Admin delivery authorized/denied | Safe masked attempts only. |
| N-API-07 | Health live/ready | Live process-only; ready checks dependencies. |
| N-INT-01 | Duplicate command same dedupe key | One notification/delivery. |
| N-INT-02 | Same dedupe key different data | `NOTIFICATION_IDEMPOTENCY_CONFLICT`. |
| N-INT-03 | Provider retry 1–3 then fail | Correct attempt status/DLQ, no false sent. |
| N-INT-04 | Template missing/invalid fields | `NOTIFICATION_TEMPLATE_NOT_FOUND`, no dispatch. |
| N-SEC-01 | Raw token/PII/template log scan | No leakage. |

## 4. Unit test

| Module | Cases |
|---|---|
| Consumer | Envelope/schema/dedupe/offset commit after persist. |
| Template | Version/locale/allowlist/sanitize. |
| Dispatcher | Channel preference, critical override, provider mapping. |
| Delivery | Retry/backoff/DLQ/status transitions. |
| In-app | Scope, unread count, monotonic read. |
| Observability | Required JSON fields, trace/request propagation, redaction, bounded metrics. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | NestJS/MySQL + Email/In-app theo lựa chọn 1C. | Ảnh hưởng test harness/schema. | Tech lead |
| 2 | SMTP/Mock provider v1; provider/DKIM chưa chốt. | Cần provider contract test thật. | DevOps |
| 3 | Retention 90 ngày và critical preference override baseline. | Ảnh hưởng cleanup/opt-out tests. | Security/Product |
| 4 | Template catalog v1 tiếng Việt. | Cần i18n/golden template khi mở rộng. | Product |
