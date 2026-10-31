# API Spec — Notification Service

> Nguồn: `docs/lld/notification.md` · `docs/db/notification.md` · Code: `ecommerce-taca/notification-service` @ `main` (`6c6d9a2`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — consumer Kafka, delivery Email/In-app + retry/outbox, notification center, preference và admin deliveries đã có code; retention, audit, metrics và một số mapping event/template chưa có.
> Base path: `/api/v1` (health ngoài prefix) · Email + In-app · Không có endpoint để service khác gửi thông báo — input chỉ qua Kafka

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | `Authorization: Bearer <access JWT>`; service tự verify RS256 qua JWKS auth-user (issuer/audience cấu hình). User = claim `sub`, role = claim `roles`. Thiếu/sai token → `401 NOTIFICATION_UNAUTHENTICATED`. |
| Admin | Route `/api/v1/admin/**` cần một trong các role `SUPER_ADMIN`, `RISK_MANAGER`, `CATALOG_ADMIN`, `FINANCE_OPS`, `SUPPORT_VIEWER`; thiếu → `403 NOTIFICATION_FORBIDDEN`. |
| Request ID/trace | `X-Request-ID` (≤64 ký tự, `[A-Za-z0-9._:-]`, sai thì sinh UUIDv7) luôn echo lại ở response header; W3C `traceparent` → `trace_id`/`span_id`. |
| Validation | Query/body validate bằng class-validator, `whitelist` + `forbidNonWhitelisted`: field lạ → `400 NOTIFICATION_INVALID_INPUT` với `details[] = {field, message}`. |
| Timestamp | ISO-8601 UTC (`toISOString()`, có mili giây). |
| Actor context | Không đọc `X-User-ID`/`X-User-Roles` của Gateway; actor lấy từ JWT do service tự verify. |
| Idempotency | Không dùng `Idempotency-Key`; mark-read/read-all/PUT preference idempotent theo bản chất. |
| Pagination | `page` ≥1 (mặc định 1), `size` 1–100 (mặc định 20); vượt giới hạn → `400`, không clamp. `meta` có `page, size, total`, **không** có `total_pages`. |
| Response | `{data, meta:{request_id, ...}}`. List thêm `meta.page`, `meta.size`, `meta.total`. Health không bọc envelope. |
| Error | `{error:{code,message,details,trace_id}}`; `message` tiếng Việt; `trace_id` fallback `request_id`. |
| Log | Xem `docs/lld/notification.md` §2.2. Không log recipient raw, token/OTP, Authorization. |
| Security header | `helmet()` mặc định. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /api/v1/notifications` | Authenticated | List in-app notification của user. | ✅ |
| 2 | `GET /api/v1/notifications/unread-count` | Authenticated | Số in-app chưa đọc. | ✅ |
| 3 | `PATCH /api/v1/notifications/{id}/read` | Authenticated | Đánh dấu một notification đã đọc. | ✅ |
| 4 | `PATCH /api/v1/notifications/read-all` | Authenticated | Đánh dấu tất cả đã đọc. | ✅ |
| 5 | `GET /api/v1/notifications/preferences` | Authenticated | Xem preference đã lưu. | ✅ |
| 6 | `PUT /api/v1/notifications/preferences` | Authenticated | Cập nhật một preference. | ✅ |
| 7 | `GET /api/v1/admin/notifications/deliveries` | Admin role | Delivery diagnostics. | ✅ |
| 8 | `GET /health/live` | Ops | Liveness. | ✅ |
| 9 | `GET /health/ready` | Ops | Readiness. | ✅ |

Gateway route: `/api/v1/notifications/**` và `/api/v1/admin/notifications/**` (xem `api-gateway-docs/docs/api/api-gateway.md` #17/#17a).

## 3. Chi tiết endpoint

### 3.1 `GET /api/v1/notifications`

Query: `page`, `size`, `read_status` (`ALL`|`READ`|`UNREAD`, mặc định không lọc), `category` (enum `NotificationCategory`). Chỉ trả notification `channel=IN_APP` của user hiện tại, sort `created_at DESC`. Empty list là `200 data=[]`.

```json
{
  "data": [
    {
      "notification_id": "01912fe0-7a1b-7c12-9c55-8b1c34a6d921",
      "category": "ORDER",
      "template": "order-success-v1",
      "title": "Đặt hàng thành công",
      "body": "Cảm ơn Nguyễn Minh Anh, đơn hàng TC-20260830-0001 của bạn đã được xác nhận. Chúng tôi sẽ sớm xử lý và giao hàng.",
      "reference": { "type": "ORDER", "id": "order-01912f91" },
      "read": false,
      "created_at": "2026-08-30T09:03:20.000Z"
    }
  ],
  "meta": { "page": 1, "size": 20, "total": 12, "unread_count": 3, "request_id": "01912fe1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

- `title`/`body` render lúc đọc từ template `PUBLISHED` mới nhất cùng key; không có template → chuỗi rỗng.
- `reference` = `null` khi notification không có reference (vd. từ command). `reference.type` ∈ `ORDER | SHIPMENT | PAYMENT | REVIEW | CONVERSATION | SHOP`.
- Không trả email/phone người nhận.

### 3.2 `GET /api/v1/notifications/unread-count`

```json
{ "data": { "unread_count": 3 }, "meta": { "request_id": "01912fe2-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Đếm in-app `read_status=UNREAD` của user.

### 3.3 `PATCH /api/v1/notifications/{id}/read`

Body optional `{ "version": 3 }` (số nguyên ≥1) — được validate nhưng **không dùng** (không có optimistic lock ở v1). Response `200`:

```json
{ "data": { "notification_id": "01912fe0-7a1b-7c12-9c55-8b1c34a6d921", "read": true }, "meta": { "request_id": "01912fe4-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Idempotent. Notification không tồn tại, của user khác hoặc là `EMAIL` → `404 NOTIFICATION_NOT_FOUND` (không lộ sự tồn tại).

### 3.4 `PATCH /api/v1/notifications/read-all`

Body optional `{ "before": "2026-08-30T09:00:00Z" }` (ISO date) — chỉ đánh dấu notification có `created_at <= before`. Response `200`:

```json
{ "data": { "marked_count": 8 }, "meta": { "unread_count": 4, "request_id": "01912fe5-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Chỉ in-app của user; không đổi delivery status.

### 3.5 `GET /api/v1/notifications/preferences`

Trả các row đã lưu của user (không có row = mặc định `ENABLED`, không được liệt kê):

```json
{
  "data": {
    "preferences": [
      { "category": "SECURITY",  "channel": "EMAIL", "status": "ENABLED",  "locked": true },
      { "category": "MARKETING", "channel": "EMAIL", "status": "DISABLED", "locked": false }
    ]
  },
  "meta": { "request_id": "01912fe2-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

### 3.6 `PUT /api/v1/notifications/preferences`

Body (cả 3 field bắt buộc, giá trị trong enum):

```json
{ "channel": "EMAIL", "category": "MARKETING", "status": "DISABLED" }
```

Response `200` trả **một** preference vừa lưu (upsert theo `(user, channel, category)`):

```json
{ "data": { "category": "MARKETING", "channel": "EMAIL", "status": "DISABLED", "locked": false }, "meta": { "request_id": "01912fe3-7a1b-7c12-9c55-8b1c34a6d921" } }
```

- `status=DISABLED` với `category=SECURITY` hoặc row `locked=true` → `403 NOTIFICATION_PREFERENCE_LOCKED`.
- Preference chỉ ảnh hưởng `EMAIL`; `IN_APP` luôn được tạo.

### 3.7 `GET /api/v1/admin/notifications/deliveries`

Query: `status?` (`QUEUED`|`PROCESSING`|`SENT`|`FAILED`|`SKIPPED`|`EXPIRED`), `channel?`, `template?` (≤100), `recipient_hash?` (≤64), `from?`/`to?` (ISO date, so với `created_at`), `page`, `size`. Trả cả `EMAIL` và `IN_APP`, sort `created_at DESC`.

```json
{
  "data": [
    {
      "notification_id": "01912fe0-7a1b-7c12-9c55-8b1c34a6d921",
      "channel": "EMAIL",
      "template": "order-success-v1",
      "status": "SENT",
      "attempt_count": 1,
      "recipient_masked": "mi***@example.com",
      "provider_status": "delivered",
      "error_code": null,
      "queued_at": "2026-08-30T09:03:15.000Z",
      "sent_at": "2026-08-30T09:03:20.000Z"
    },
    {
      "notification_id": "01912fe1-7a1b-7c12-9c55-8b1c34a6d921",
      "channel": "EMAIL",
      "template": "payout-result-v1",
      "status": "FAILED",
      "attempt_count": 3,
      "recipient_masked": "se***@shop.vn",
      "provider_status": "bounced",
      "error_code": "SMTP_REJECTED",
      "queued_at": "2026-08-30T08:00:00.000Z",
      "sent_at": null
    }
  ],
  "meta": { "page": 1, "size": 20, "total": 4210, "request_id": "01912fe3-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Nguồn field (đối chiếu `docs/db/notification.md`):

| Field | Nguồn |
|---|---|
| `attempt_count` | Số row `delivery_attempts` của notification. |
| `recipient_masked` | Giải mã `recipient_encrypted` rồi mask (`mi***@example.com`); `null` với `IN_APP`. |
| `provider_status` | Suy ra từ `status` của attempt cuối: `SENT`→`delivered`, `RETRYABLE_FAILED`→`temporary_failure`, `PERMANENT_FAILED`→`bounced`, khác (`SKIPPED`/`RETRY_EXHAUSTED`/`STARTED`)→`sending`; không có attempt → `null`. |
| `error_code` | `error_code` của attempt cuối. |
| `queued_at` | `scheduled_at`, fallback `created_at`. |
| `sent_at` | `notifications.sent_at`. |

`recipient_hash` filter dùng HMAC-SHA256 (xem db §3.1); không bao giờ trả email đầy đủ.

### 3.8 `GET /health/live`

Không bọc envelope, không cần auth. Luôn `200`:

```json
{ "status": "UP", "service": "notification-service", "time": "2026-08-30T09:00:00.000Z" }
```

### 3.9 `GET /health/ready`

Không bọc envelope, không cần auth. `200` khi MySQL `SELECT 1` thành công, ngược lại `503` với `status=DEGRADED`:

```json
{ "status": "DEGRADED", "service": "notification-service", "checks": { "config": "UP", "mysql": "DOWN" }, "trace_id": "01912fe7-7a1b-7c12-9c55-8b1c34a6d921", "time": "2026-08-30T09:00:00.000Z" }
```

Kafka/SMTP **không** nằm trong readiness ở v1.

### 3.10 Kafka input (không phải REST)

Contract command `notification.commands.v1` và domain event: xem `docs/lld/notification.md` §6.2. Event kết quả `notification.delivered.v1`/`notification.failed.v1`: xem lld §6.4.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `NOTIFICATION_INVALID_INPUT` | 400 | Query/body sai, field lạ. |
| `NOTIFICATION_UNAUTHENTICATED` | 401 | Thiếu/sai JWT. |
| `NOTIFICATION_FORBIDDEN` | 403 | Không có role admin. |
| `NOTIFICATION_NOT_FOUND` | 404 | Notification không thuộc user/không phải in-app. |
| `NOTIFICATION_PREFERENCE_LOCKED` | 403 | Cố tắt `SECURITY`/row `locked`. |
| `NOTIFICATION_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

Mã `NOTIFICATION_TEMPLATE_NOT_FOUND`, `NOTIFICATION_IDEMPOTENCY_CONFLICT`, `NOTIFICATION_DELIVERY_FAILED` chỉ xuất hiện ở luồng Kafka/delivery (DLQ reason hoặc attempt `error_code`), không trả qua REST. `NOTIFICATION_CHANNEL_DISABLED`, `NOTIFICATION_PROVIDER_UNAVAILABLE` có trong catalog nhưng chưa được phát. Route không tồn tại trả HTTP `404` với code `NOTIFICATION_INVALID_INPUT` (mapping mặc định của filter).

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Email/In-app và NestJS/MySQL theo lựa chọn 1C. | Ảnh hưởng MFE notification center/ops. | Tech lead |
| 2 | SMTP provider production/DKIM chưa chốt. | Ảnh hưởng delivery SLA. | DevOps |
| 3 | `GET /preferences` chỉ trả row đã lưu; client phải tự coi thiếu row = `ENABLED`. | MFE hiển thị thiếu toggle mặc định. | Frontend lead |
| 4 | `version` ở mark-read bị bỏ qua; nếu cần optimistic lock phải thêm cột `notifications.version`. | Race giữa thiết bị (ảnh hưởng thấp vì idempotent). | Notification owner |
| 5 | `provider_status` trả `sending` cho notification đã `FAILED` với attempt cuối `RETRY_EXHAUSTED`. | Admin đọc nhầm trạng thái. | Notification owner |
| 6 | Template catalog và locale chỉ `vi-VN`. | Cần i18n contract nếu mở rộng. | Product owner |
