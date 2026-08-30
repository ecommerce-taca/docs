# LLD — Message Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Node.js + NestJS · MongoDB/Mongoose · REST + WebSocket · S3/MinIO signed attachment

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Conversation/message giữa buyer–seller, support escalation, message history, unread/read state, realtime delivery và attachment metadata. |
| Database | MongoDB `messagedb`; message/conversation là source of truth của service. |
| Realtime | WebSocket gateway; REST là fallback/history. |
| Attachment | S3/MinIO giữ bytes; Message lưu object key/checksum/status. |
| Moderation | Không AI/pre-moderation v1; chỉ validate/sanitize và reserved report/blocked status. |
| Không thuộc service | Order/product/payment/KYC/notification delivery; service chỉ giữ optional context snapshot/reference. |

### 1.2 Boundary

```text
Buyer/Seller/Support ──Gateway──► Message REST + WebSocket
Message ──S3/MinIO──► attachment bytes via signed URL
Message ──Kafka──► Notification (optional message notification command)
Message ──REST/event──► Auth/Product/Order reference validation
```

- User chỉ đọc/ghi conversation khi là participant hoặc support/admin được cấp scope.
- Seller scope dựa `shop_id`; không tin participant list từ client để cấp quyền.
- WebSocket auth dùng access JWT/handshake token policy; refresh token không được dùng làm socket credential.
- Client kết nối WebSocket qua API Gateway tại `GET /ws/messages` với `Upgrade: websocket`; token gửi qua `Sec-WebSocket-Protocol: bearer, <access-token>` (fallback query `?access_token=`). Gateway validate JWT ở handshake rồi proxy TCP upgrade kèm `X-User-ID`/`X-User-Roles`/`X-Trace-ID`; Message Service vẫn tự validate lại `Authorization`/context và authorize `conversation.join`/`message.send`/`message.read` theo participant scope. Gateway không parse/không buffer frame; reconnect + đồng bộ message miss là `conversation.sync` qua REST cursor.
- Message không quyết định order dispute/refund; support conversation chỉ tham chiếu target.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Message | MessageDB MongoDB | MongoDB/Mongoose. |
| Penpot Buyer/Seller | Buyer ↔ seller chat | Conversation 1:1 hoặc theo order/product context. |
| Support escalation | Buyer/seller → support | Conversation type `SUPPORT`, support role scoped. |
| Attachment | Message attachment | Signed upload/complete, size/type/checksum validation. |
| UI states | unread, read, typing, offline | WebSocket event + REST sync/read cursor. |

## 2. Cấu trúc bên trong

### 2.1 Module NestJS

```text
src/
├── conversation/         # participant/thread lifecycle
├── message/              # send/edit/delete/history
├── realtime/             # WebSocket gateway/presence/ack
├── attachment/           # signed URL/metadata
├── read-state/           # unread/read cursor
├── support/              # escalation/admin scope
├── integration/          # Auth/Product/Order/Notification contracts
├── security/             # participant/tenant/rate limits
└── observability/        # shared telemetry contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `ConversationController` | List/create/detail/archive conversation | Participant/role scope; no arbitrary participant injection. |
| `MessageController` | REST history/send/edit/delete/read | Cursor pagination; message ownership/state. |
| `MessageGateway` | WebSocket connect/send/ack/read/typing | JWT auth, connection limits, trace context. |
| `AttachmentService` | Signed upload/complete | Private object, quota/checksum/scan state. |
| `ReadStateService` | Read cursor/unread count | Monotonic cursor; idempotent. |
| `SupportPolicy` | Support escalation/access | `SUPPORT_AGENT`/admin permission; audit. |
| `ObservabilityModule` | Shared logs/traces/metrics/health | No raw message/PII logs. |

### 2.2 Chuẩn observability dùng chung

- Structured JSON stdout field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST/WebSocket/Kafka propagate W3C `traceparent`/`tracestate`, `X-Request-ID`; socket event có `connection_id` hashed/allowlist.
- Không log message body, attachment URL/token, password, access/refresh token, raw email/phone/address; metrics không label bằng user/conversation/message ID.
- `/health/live` process-only; `/health/ready` MongoDB/Kafka/S3 config; socket disconnect/error có safe reason.
- Error envelope REST `{error:{code,message,details,trace_id}}`; WebSocket error dùng same code/trace metadata.

## 3. Luồng xử lý

### 3.1 Create/open conversation

1. Buyer/seller gửi target participant/shop/order/product context.
2. Resolve actor scope từ auth; validate other participant/shop/target reference theo contract.
3. Tạo deterministic participant key hoặc tìm conversation active hiện có.
4. Persist conversation + participant/read state; không duplicate khi retry.

### 3.2 Send message

```text
REST/WebSocket → authenticate → participant authorization
  → validate text/attachments/idempotency
  → persist message SENT/ACCEPTED + outbox event (transaction)
  → update conversation last_message
  → publish realtime event + optional notification command (via outbox relay)
  → ACK with message_id/sequence
```

- Message sequence tăng trong conversation; client retry cùng idempotency key không tạo duplicate.
- Offline recipient nhận message khi reconnect qua REST cursor; WebSocket delivery không là source of truth.
- Edit/delete chỉ owner trong edit window baseline; soft delete hiển thị placeholder, giữ audit.

### 3.3 Read/typing/presence

- Read cursor monotonic theo `message_sequence`; duplicate read ACK an toàn.
- Typing/presence ephemeral, không lưu domain message; rate limit socket event.
- Reconnect gửi `after_sequence` để sync missed messages; cursor không được vượt message cuối.

### 3.4 Attachment

1. Client request signed URL cho conversation/actor.
2. Upload trực tiếp S3/MinIO.
3. Complete với object key/checksum; server HEAD/verify type/size/scan state.
4. Chỉ attachment `READY` mới link trong message; `SCANNING` không public.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `MESSAGE_MAX_LENGTH` | 5000 | Unicode characters. |
| `CONVERSATION_PARTICIPANT_MAX` | 20 | Support thread; buyer-seller baseline 2. |
| `ATTACHMENT_MAX_COUNT` | 6 | Per message. |
| `ATTACHMENT_MAX_SIZE` | 20 MiB | Image/document baseline. |
| `MESSAGE_PAGE_SIZE_DEFAULT` | 50 | Max 100, cursor-based. |
| `EDIT_WINDOW` | 15 phút | Baseline, cần confirm. |
| `WEBSOCKET_IDLE_TIMEOUT` | 30 phút | Reconnect via REST cursor. |
| `SOCKET_RATE_LIMIT` | 60 events/phút/connection | Bounded. |
| `OUTBOX_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `MEDIA_SIGNED_URL_TTL` | 10 phút | Private object. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | MongoDB Date. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `ConversationType` | `BUYER_SELLER`, `SUPPORT`, `ORDER_CONTEXT`. |
| `ConversationStatus` | `ACTIVE`, `ARCHIVED`, `CLOSED`, `BLOCKED`. |
| `MessageStatus` | `ACCEPTED`, `EDITED`, `DELETED`, `BLOCKED`. Trạng thái **nội dung** (moderation); phơi ra API dưới tên `moderation_status`. |
| `MessageDeliveryStatus` | `SENT`, `DELIVERED`, `READ`. Trạng thái **giao nhận**, server tính per-viewer từ `read_states`, không lưu trên document message. Client-side `QUEUED`/`SENDING`/`FAILED` không thuộc enum này. |
| `AttachmentStatus` | `UPLOADING`, `SCANNING`, `READY`, `REJECTED`, `DELETED`. |
| `ParticipantRole` | `BUYER`, `SELLER`, `SUPPORT_AGENT`, `ADMIN`. |

V1 không có pre-moderation; `BLOCKED` chỉ reserved cho security/report/admin policy tương lai.

Hai enum trên là **hai trục khác nhau** và không được gộp vào một field: một message có thể vừa `EDITED` (nội dung) vừa `READ` (giao nhận). API trả hai field riêng `moderation_status` + `delivery_status`.

`delivery_status` không được persist theo message (sẽ là O(số message × số participant) lần ghi); nó được **suy ra lúc đọc** bằng cách so `message.sequence` với `delivered_sequence`/`last_read_sequence` trong `read_states` của phía đối diện — chi phí O(1) mỗi conversation.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `message.events.v1` | `conversation.created/archived` | conversation/participants-safe summary |
| `message.events.v1` | `message.created/edited/deleted` | message ID/conversation/sequence/safe metadata |
| `message.events.v1` | `message.read` | conversation/user/read_sequence |
| `notification.commands.v1` | `MESSAGE_RECEIVED` | recipient/conversation/message preview allowlist |

### 6.2 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Auth User | `user.status_changed`, `shop.status_changed` | Chặn/đóng participant khi suspended theo policy. |
| Order-Commerce | `order.created/cancelled` | Update order context; không expose order payload raw. |
| Product Catalog | `product.archived` | Update context label; conversation history giữ nguyên. |

### 6.3 Mock contract — realtime message

```json
{
  "event":"message.created",
  "event_id":"message-event-1",
  "schema_version":1,
  "traceparent":"00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "conversation_id":"conversation-1",
  "message_id":"message-1",
  "sequence":42,
  "sender_id":"user-1",
  "created_at":"2026-08-30T12:00:00Z",
  "content_preview":"safe preview or omitted",
  "attachments_count":0
}
```

Không publish raw message/attachment URL vào event Notification nếu template không allowlist.

## 7. Mã lỗi

| Mã | HTTP/WS | Ý nghĩa |
|---|---:|---|
| `MESSAGE_INVALID_INPUT` | 400 | Text/attachment/context sai. |
| `MESSAGE_UNAUTHENTICATED` | 401 | Thiếu JWT/socket auth. |
| `MESSAGE_FORBIDDEN` | 403 | Không phải participant/support scope. |
| `CONVERSATION_NOT_FOUND` | 404 | Conversation không tồn tại. |
| `CONVERSATION_CLOSED` | 409 | Không gửi vào conversation closed. |
| `MESSAGE_NOT_FOUND` | 404 | Message không tồn tại. |
| `MESSAGE_EDIT_EXPIRED` | 409 | Quá edit window. |
| `MESSAGE_IDEMPOTENCY_CONFLICT` | 409 | Key khác payload. |
| `MESSAGE_ATTACHMENT_INVALID` | 400 | Type/checksum/object sai. |
| `MESSAGE_ATTACHMENT_LIMIT_EXCEEDED` | 409 | Vượt 6 file/message. |
| `MESSAGE_RATE_LIMITED` | 429 | Vượt message/socket limit. |
| `MESSAGE_DEPENDENCY_UNAVAILABLE` | 503 | Mongo/Kafka/S3 unavailable. |
| `MESSAGE_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | REST + WebSocket, NestJS + MongoDB theo lựa chọn 2A. WebSocket đi qua API Gateway `GET /ws/messages` (handshake JWT ở subprotocol `bearer, <token>`); Gateway đã chốt hỗ trợ trong v1. | Nếu đổi subprotocol/handshake format phải đồng bộ `api-gateway` §3.12. | Tech lead + Gateway owner |
| 2 | Buyer-seller/support, participant baseline; group chat mở rộng chưa chốt. | Ảnh hưởng participant/schema/notification. | Product owner |
| 3 | Không AI/pre-moderation v1; report/block workflow chưa có. | Nếu compliance yêu cầu, cần moderation service/state. | Product/Security |
| 4 | Attachment tối đa 20 MiB/6 file, scan provider chưa chốt. | Ảnh hưởng storage/security/UX. | Security/DevOps |
| 5 | Message edit window 15 phút và retention chưa chốt. | Ảnh hưởng audit/privacy/storage. | Product/Security |
