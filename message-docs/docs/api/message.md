# API Spec — Message Service

> Nguồn: `docs/lld/message.md` · `docs/db/message.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · REST + WebSocket · Buyer–seller/support

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | JWT cho REST/WebSocket handshake; participant/support scope bắt buộc. |
| Request ID/trace | `X-Request-ID`, W3C `traceparent`/`tracestate`; socket event có trace metadata. |
| Time | ISO-8601 UTC; history cursor pagination max 100. |
| Idempotency | Send/create conversation/attachment complete dùng `Idempotency-Key`. |
| Response/error | `{data,meta}` / `{error:{code,message,details,trace_id}}`. |
| Log | JSON chuẩn auth-user/Gateway; không log body/attachment URL/token/PII. |
| Realtime | WebSocket chỉ transport; REST cursor là recovery/source of truth. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /conversations` | Authenticated | List conversation của actor. |
| 2 | `POST /conversations` | Authenticated | Mở buyer-seller/support conversation. |
| 3 | `GET /conversations/{id}` | Participant/support | Detail/participants/context. |
| 4 | `GET /conversations/{id}/messages` | Participant/support | Message history cursor. |
| 5 | `POST /conversations/{id}/messages` | Participant/support | Gửi message REST. |
| 6 | `PATCH /messages/{id}` | Sender | Sửa message trong window. |
| 7 | `DELETE /messages/{id}` | Sender | Soft delete. |
| 8 | `PATCH /conversations/{id}/read` | Participant | Mark read cursor. |
| 9 | `POST /conversations/{id}/attachments/upload-url` | Participant | Signed upload. |
| 10 | `POST /attachments/{id}/complete` | Uploader | Complete metadata. |
| 11 | `GET /health/live` | Ops | Liveness. |
| 12 | `GET /health/ready` | Ops | Readiness. |

WebSocket path `/ws/messages` (qua API Gateway, `Upgrade: websocket`); event names: `conversation.join`, `message.send`, `message.created`, `message.ack`, `message.read`, `message.typing`, `conversation.sync`. Handshake: token qua `Sec-WebSocket-Protocol: bearer, <access-token>` (fallback query `?access_token=`); Gateway validate JWT rồi proxy upgrade, Message Service tự authorize từng event theo participant scope.

## 3. Chi tiết endpoint

### 3.1 Conversation

- `GET /conversations`: query `status,type,page,size`; chỉ participant/support scope.
- `POST /conversations`: body `{type,participant_user_ids[],shop_id?,context_type?,context_id?}`; server kiểm tra participant/scope và deterministic key; không cho arbitrary admin injection.
- `GET /conversations/{id}`: trả participants safe, context label/reference, last sequence/time, unread count; không trả private account data.

### 3.2 Message history/send

- `GET /conversations/{id}/messages?after_sequence=&before_sequence=&limit=`: cursor bounded, ordered sequence; reconnect dùng `after_sequence`.
- `POST /conversations/{id}/messages` header Idempotency-Key, body `{client_message_id?,body?,attachment_ids[]}`; body max 5.000, attachment max 6, ít nhất một body/attachment; response `201` sequence/message ID.
- `PATCH /messages/{id}` body `{version,body,attachment_ids[]}`; sender + edit window 15 phút baseline.
- `DELETE /messages/{id}` body `{version}`; soft delete placeholder.

### 3.3 Read/realtime

`PATCH /conversations/{id}/read` body `{last_read_sequence}`; cursor chỉ tăng, idempotent. WebSocket handshake tại `GET /ws/messages` qua Gateway: token ở `Sec-WebSocket-Protocol: bearer, <access-token>` (fallback `?access_token=`); Gateway validate JWT + kiểm connection cap + rate limit rồi proxy upgrade. Message Service authorize `conversation.join`/`message.send`/`message.read` theo participant scope, không tin danh sách participant từ client. Reconnect: client gọi `conversation.sync` (hoặc `GET /conversations/{id}/messages?after_sequence=`) qua REST — socket không giữ history. Token hết hạn giữa phiên: server đóng socket, client refresh rồi reconnect.

### 3.4 Attachment

`POST /conversations/{id}/attachments/upload-url` body `{content_type,size_bytes,sha256}`; server sinh private key, max 20 MiB/file. `POST /attachments/{id}/complete` HEAD/checksum/type/scan; `READY` mới message public, `SCANNING` chưa public.

### 3.5 Health

`/health/live` process-only; `/health/ready` MongoDB/Kafka/S3 config. WebSocket không được dùng để bypass readiness/auth.

## 4. Mã lỗi chung

| Mã | HTTP/WS | Ý nghĩa |
|---|---:|---|
| `MESSAGE_INVALID_INPUT` | 400 | Text/attachment/context sai. |
| `MESSAGE_UNAUTHENTICATED` | 401 | JWT/socket auth thiếu/sai. |
| `MESSAGE_FORBIDDEN` | 403 | Không participant/support. |
| `CONVERSATION_NOT_FOUND` | 404 | Không tồn tại. |
| `CONVERSATION_CLOSED` | 409 | Không gửi conversation closed. |
| `MESSAGE_NOT_FOUND` | 404 | Không tồn tại. |
| `MESSAGE_EDIT_EXPIRED` | 409 | Hết edit window. |
| `MESSAGE_IDEMPOTENCY_CONFLICT` | 409 | Key khác payload. |
| `MESSAGE_ATTACHMENT_INVALID` | 400 | Media/checksum/type sai. |
| `MESSAGE_ATTACHMENT_LIMIT_EXCEEDED` | 409 | Vượt 6 file. |
| `MESSAGE_RATE_LIMITED` | 429 | Vượt socket/message rate. |
| `MESSAGE_DEPENDENCY_UNAVAILABLE` | 503 | Mongo/Kafka/S3 down. |
| `MESSAGE_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | REST + WebSocket, buyer-seller/support theo lựa chọn 2A. | Ảnh hưởng Gateway/socket infrastructure. | Tech lead |
| 2 | Group chat chưa v1; participant baseline 2 hoặc support thread. | Ảnh hưởng schema/UX. | Product owner |
| 3 | Không AI/pre-moderation; report/block chưa triển khai. | Nếu cần compliance, thêm moderation workflow. | Product/Security |
| 4 | Attachment scan provider và retention chưa chốt. | Ảnh hưởng storage/security. | DevOps/Security |
