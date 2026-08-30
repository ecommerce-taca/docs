# Test Plan — Message Service

> Nguồn: `docs/lld/message.md` · `docs/db/message.md` · `docs/api/message.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture |
|---|---|
| Runtime | NestJS + Jest/Supertest, MongoDB/Kafka/S3 mocks/containers, WebSocket test client. |
| Conversations | Buyer-seller, support, closed/archived, wrong participant. |
| Messages | Text/attachment, sequence 1..n, edited/deleted, duplicate idempotency. |
| Actors | Buyer/seller/support/admin, other user, expired JWT. |
| Realtime | Connect/join/send/ack/read/typing/reconnect with `after_sequence`. |
| Telemetry | JSON logs, traceparent, X-Request-ID, raw body/PII/token scan. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Conversation list | Buyer/seller/support list/open | Chỉ thread có quyền; unread/last message đúng. |
| Chat realtime | Send, receive, ACK, reconnect | Message không mất/duplicate; REST sync missed sequence. |
| Offline | Recipient offline rồi reconnect | History cursor trả đủ message, socket không là source truth. |
| Read/typing | Read cursor, typing/presence | Read monotonic; typing ephemeral/rate-limited. |
| Attachment | Upload/scan/reject/large file | Signed URL private; chỉ READY public. |
| Security/UI | Closed thread, wrong shop, loading/error/offline | Không gửi trái quyền; error/retry rõ. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| M-API-01 | `GET /conversations` own/other | Own list; other denied/no leak. |
| M-API-02 | `POST /conversations` valid/duplicate/forged participant | Deterministic idempotent; forged scope denied. |
| M-API-03 | `GET /conversations/{id}` participant/support/nonparticipant | Correct scope/context safe. |
| M-API-04 | `GET /conversations/{id}/messages` cursor/bounds | Ordered sequence, max 100, no unbounded query. |
| M-API-05 | `POST messages` text/attachment valid | `201`, sequence/ACK, one message. |
| M-API-06 | Send closed/empty/oversized/rate limit | Correct `409/400/429`, no write. |
| M-API-07 | Duplicate/different idempotency payload | Same result / `409 MESSAGE_IDEMPOTENCY_CONFLICT`. |
| M-API-08 | Edit/delete own/other/expired | Correct owner/version/window; soft delete. |
| M-API-09 | Read cursor lower/equal/higher | Monotonic and idempotent. |
| M-API-10 | Attachment presign/complete wrong key/checksum | READY only for valid own object. |
| M-API-11 | Seller/support permission | Correct role/context, no cross-tenant leak. |
| M-API-12 | Health live/ready | Correct Mongo/Kafka/S3 readiness. |
| M-WS-01 | JWT connect/join/send/read/typing | Auth/scope/ACK/error trace correct. |
| M-WS-02 | Disconnect/reconnect after sequence | Missed messages recovered via REST cursor. |
| M-WS-03 | Handshake qua Gateway `/ws/messages` với `Sec-WebSocket-Protocol: bearer, <token>` | Upgrade `101`; Message Service nhận `X-User-ID` từ Gateway và vẫn tự validate `Authorization`. |
| M-WS-04 | Handshake thiếu/sai token (fallback `?access_token=` hợp lệ) | Thiếu/sai → từ chối trước upgrade; query token hợp lệ → OK và không log raw token. |
| M-WS-05 | Token hết hạn giữa phiên | Server đóng socket; client reconnect sau refresh; không mất message (recovery qua `conversation.sync`). |
| M-WS-06 | `message.send` không phải participant | `MESSAGE_FORBIDDEN` qua WS error dùng cùng code/trace; không ghi message. |

### 3.1 Contract/security/resilience

| ID | Case | Expected |
|---|---|---|
| M-INT-01 | Message notification event | Safe preview only; no raw body/token. |
| M-INT-02 | User/shop suspended event | Mutation blocked/closed per policy, history retained. |
| M-SEC-01 | JWT/participant/attachment IDOR | Denied; no object/message leak. |
| M-SEC-02 | XSS/NoSQL/socket flood | Sanitize/reject/rate-limit. |
| M-RES-01 | Mongo/Kafka/S3 unavailable | No partial send; outbox/retry; REST recovery. |

## 4. Unit test

| Module | Cases |
|---|---|
| Conversation policy | Participant key, support role, closed state. |
| Message validator | body/attachment requirement, 5.000 chars, sanitize. |
| Sequence/idempotency | Monotonic allocation, duplicate replay, conflict payload. |
| Read state | Cursor monotonic/count. |
| WebSocket gateway | JWT, join/send/read/typing, reconnect/ACK. |
| Attachment | Key binding, size/type/checksum/scan status. |
| Observability | JSON fields, W3C/request propagation, redaction, bounded metrics. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | REST + WebSocket, buyer-seller/support theo lựa chọn 2A. | Ảnh hưởng socket/Gateway test. | Tech lead |
| 2 | Group chat chưa v1. | Cần participant/load matrix nếu bật. | Product owner |
| 3 | Không AI/pre-moderation; report/block chưa v1. | Cần moderation/security suite nếu đổi. | Product/Security |
| 4 | Attachment scan/retention chưa chốt. | Cần provider/retention tests. | DevOps/Security |
