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

- `GET /notifications`: query `page,size,read_status,category`; chỉ recipient user hiện tại, newest-first; trả title/body safe, type, read status, created_at, reference IDs allowlist.
- `GET /notifications/unread-count`: `200 {data:{unread_count},meta}`; count không âm.
- Empty list là `200 data=[]`, không phải 404.

### 3.2 Mark read

- `PATCH /notifications/{id}/read` body `{version?}`; user scope, idempotent.
- `PATCH /notifications/read-all` body `{before?}`; chỉ in-app notifications của user; không thay đổi email delivery status.

### 3.3 Preferences

`GET` trả channel/category status. `PUT` body `{channel:"EMAIL",category:"ORDER",status:"DISABLED"}`; security-critical Auth command có thể không opt-out theo policy; client không tạo category/template mới.

### 3.4 Admin delivery

`GET /admin/notifications/deliveries` query status/channel/template/date/page/size; trả safe attempt/provider status/error code, recipient masked/hash, không raw body/token. Admin role/permission do Gateway/Auth User enforce.

### 3.5 Internal command contract

Kafka command `ORDER_SUCCESS`, `INVOICE_ISSUED`, `SHIPMENT_DELIVERED`, `PASSWORD_RESET_REQUESTED` có event ID, dedupe key, template key, allowlisted data; consumer persist rồi dispatch Email/In-app. Producer không gọi public API để bypass dedupe.

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
