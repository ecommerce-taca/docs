# API Spec — Notification Service

> Nguồn: `docs/lld/notification.md` · `docs/db/notification.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · Email + In-app · Consumer command không gọi trực tiếp từ buyer

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | In-app API cần JWT; internal command dùng service auth/mTLS. |
| Request ID/trace | `X-Request-ID`, W3C `traceparent`/`tracestate`; error có `trace_id`. |
| Time | ISO-8601 UTC; pagination page 1/size 20/max100. |
| Response/error | `{data,meta:{request_id}}` / `{error:{code,message,details,trace_id}}`. |
| Log | JSON field chuẩn `timestamp,level,service,env,version,event,trace_id,span_id,request_id,route,method,status_code,duration_ms`. |
| Redaction | Không log recipient raw, template body, OTP/reset token, payment/address/Authorization. |
| Delivery | API không hứa exactly-once external email; dedupe/retry theo command. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /notifications` | Authenticated | In-app notification list. |
| 2 | `GET /notifications/unread-count` | Authenticated | Unread count. |
| 3 | `PATCH /notifications/{id}/read` | Authenticated | Mark read. |
| 4 | `PATCH /notifications/read-all` | Authenticated | Mark all read. |
| 5 | `GET /notifications/preferences` | Authenticated | Channel/category preferences. |
| 6 | `PUT /notifications/preferences` | Authenticated | Update preferences. |
| 7 | `GET /admin/notifications/deliveries` | Admin/ops | Delivery/retry diagnostics. |
| 8 | `GET /health/live` | Ops | Liveness. |
| 9 | `GET /health/ready` | Ops | Readiness. |

Không có public endpoint để client tự gửi Email hoặc chọn template arbitrary.

## 3. Chi tiết endpoint

### 3.1 In-app list/count

- `GET /notifications`: query `page`, `size` (max 100), `read_status` (`ALL`|`READ`|`UNREAD`), `category`; chỉ recipient hiện tại, newest-first.
- `GET /notifications/unread-count`: `200 {data:{unread_count},meta}`; count không âm.
- Empty list là `200 data=[]`, **không phải** `404`.

```json
{
  "data": [
    {
      "notification_id": "ntf-01912fe0",
      "category": "ORDER",
      "template": "order-success-v1",
      "title": "Đặt hàng thành công",
      "body": "Đơn TC-20260830-0001 đã được thanh toán.",
      "reference": { "type": "ORDER", "id": "order-01912f91" },
      "read": false,
      "created_at": "2026-08-30T09:03:20Z"
    }
  ],
  "meta": { "request_id": "01912fe1-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 12, "unread_count": 3 }
}
```

`reference.type` nằm trong allowlist `ORDER | SHIPMENT | PAYMENT | REVIEW | CONVERSATION | SHOP` — client dùng để deep-link. **Không** trả email/phone người nhận, không trả nội dung email thô.

### 3.2 Mark read

- `PATCH /notifications/{id}/read` body `{version?}`; user scope, idempotent.
- `PATCH /notifications/read-all` body `{before?}`; chỉ in-app notifications của user; không thay đổi email delivery status.

### 3.3 Preferences

```json
{
  "data": {
    "preferences": [
      { "category": "ORDER",    "channel": "EMAIL",  "status": "ENABLED",  "locked": false },
      { "category": "ORDER",    "channel": "IN_APP", "status": "ENABLED",  "locked": false },
      { "category": "MARKETING","channel": "EMAIL",  "status": "DISABLED", "locked": false },
      { "category": "SECURITY", "channel": "EMAIL",  "status": "ENABLED",  "locked": true }
    ]
  },
  "meta": { "request_id": "01912fe2-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`PUT /notifications/preferences` body `{ "channel": "EMAIL", "category": "MARKETING", "status": "DISABLED" }`.

`locked: true` = **không được opt-out** (category `SECURITY`: verification, reset password, OTP, cảnh báo đăng nhập) — cố tình tắt trả `403 NOTIFICATION_PREFERENCE_LOCKED`. Client **không** tạo được `category`/`template` mới; giá trị ngoài enum → `400`.

### 3.4 Admin delivery

`GET /admin/notifications/deliveries` query status/channel/template/date/page/size; trả safe attempt/provider status/error code, recipient masked/hash, không raw body/token. Admin role/permission do Gateway/Auth User enforce.

### 3.5 Internal command contract

Notification nhận **hai loại input** (chi tiết ở LLD §6.2):

1. **Domain event** trên topic của service chủ — `order.paid`, `order.cancelled`, `invoice.issued`, `shipment.delivered`, `shipment.failed`, `payment.succeeded/failed`, `payout.succeeded/failed`. Notification **tự** map event → template; producer không cần biết template.
2. **Command** trên `notification.commands.v1` — chỉ gồm `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED`, `PHONE_OTP_REQUESTED`, `MESSAGE_RECEIVED`, `REVIEW_REQUESTED`.

> **Không có command `ORDER_SUCCESS`** — email đơn hàng đến từ domain event `order.paid`, map sang template `order-success-v1`.

Mọi input có `event_id`, `dedupe_key`, `schema_version` và data theo allowlist; consumer persist rồi mới dispatch. Producer **không** được gọi public API để bypass dedupe.

### 3.6 Health

`/health/live` process-only; `/health/ready` kiểm MySQL/Kafka/config/SMTP adapter, trả `503 NOTIFICATION_PROVIDER_UNAVAILABLE` khi policy yêu cầu fail-closed.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `NOTIFICATION_INVALID_INPUT` | 400 | Query/preference sai. |
| `NOTIFICATION_UNAUTHENTICATED` | 401 | Thiếu login. |
| `NOTIFICATION_FORBIDDEN` | 403 | Sai recipient/admin scope. |
| `NOTIFICATION_NOT_FOUND` | 404 | Không tìm thấy. |
| `NOTIFICATION_TEMPLATE_NOT_FOUND` | 409 | Template/version chưa có. |
| `NOTIFICATION_CHANNEL_DISABLED` | 409 | Channel bị preference tắt. |
| `NOTIFICATION_PREFERENCE_LOCKED` | 403 | Cố tắt category bắt buộc (`SECURITY`). |
| `NOTIFICATION_PROVIDER_UNAVAILABLE` | 503 | MySQL/Kafka/SMTP down. |
| `NOTIFICATION_DELIVERY_FAILED` | 503 | Gửi fail sau retry. |
| `NOTIFICATION_IDEMPOTENCY_CONFLICT` | 409 | Dedupe key khác payload. |
| `NOTIFICATION_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Email/In-app và NestJS/MySQL theo lựa chọn 1C. | Ảnh hưởng MFE notification center/ops. | Tech lead |
| 2 | SMTP/Mock provider v1; provider thật/DKIM chưa chốt. | Ảnh hưởng delivery SLA. | DevOps |
| 3 | Preference security-critical override chưa có policy chi tiết. | Ảnh hưởng opt-out/compliance. | Security/Product |
| 4 | Template catalog và locale chỉ baseline tiếng Việt. | Cần thêm i18n contract nếu mở rộng. | Product owner |
