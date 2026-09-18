# 00 — Quy ước dùng chung toàn hệ thống

> Vai trò: trang quy ước chung cho cả 11 service + API Gateway. Mọi doc service phải tuân theo;
> nơi nào lệch là bug doc, trừ ngoại lệ được ghi rõ trong trang này.
> Nguồn: chốt từ quyết định của Cecilia (2026-09-18, theo report `reports/no-task-docs-consistency/2026-09-18-review-docs-consistency-2.md`).
> Các mục đánh dấu `[agent-chosen — needs review]` do agent đề xuất khi chưa có chủ quyết định — cần Cecilia/tech lead duyệt.

## 1. Base path & envelope

- Base path công khai: `/api/v1` cho toàn bộ route business. Prefix ngoại lệ: `/internal/**` (service-to-service, **không** khai trong Gateway), `/webhooks/**` (callback provider, không JWT), `/ws/messages` (WebSocket), `/.well-known/**`, `/health/*`, `/metrics` (ops, không qua public ingress).
- Response thành công: `{ "data": ..., "meta": { "request_id", ... } }` — **không** có key top-level thứ ba (facet/aggregation phải nằm trong `meta` hoặc `data`).
- Response lỗi: `{ "error": { "code", "message", "details", "trace_id" } }`; `details` là object allowlist field:value (hoặc array rỗng khi không có chi tiết).
- JSON field và DB column: `snake_case` ở mọi service, kể cả service dùng MongoDB.

## 2. Mã lỗi

- Mọi mã lỗi phải có prefix theo service: `GATEWAY_*`, `AUTH_*`, `PRODUCT_*`, `INVENTORY_*`, `ORDER_*`, `PAYMENT_*`, `SHIPMENT_*`, `REVIEW_*`, `SEARCH_*`, `MESSAGE_*`, `NOTIFICATION_*`. Không dùng mã trần (`INTERNAL_ERROR`, `RATE_LIMITED`).
- Mã nội bộ (không trả cho client) được phép nằm ngoài bảng lỗi public của api.md nhưng phải ghi rõ hậu tố `/internal` hoặc chú thích "nội bộ, không expose qua API" ở lld §7.
- Bảng mã lỗi chính thức ở lld §7 và bảng §4 của api.md phải khớp nhau 1:1; mọi mã dùng trong thân bài phải có mặt ở cả hai.
- Mã lỗi của service dependency được pass-through nguyên trạng khi cần (ghi rõ trong endpoint); không đổi tên/status.

## 3. Định danh (ID)

- Entity ID: `<resource>-<ULID>` — một prefix cho một resource, dùng nhất quán trong toàn bộ ví dụ của service và ở mọi service tham chiếu chéo.
  Ví dụ: `product-01912f31`, `category-01912f20`, `order-01912f91`, `reservation-01912f82`, `payment-01912f95`, `refund-01912fa8`, `wallet-01912fb5`, `payout-01912fc1`, `bank_account-01912fc0`, `entry-01912fb7`, `posting-01912fb8`, `shipment` → `shp-`, `cv-`/`msg-`/`usr-` (message), `ntf-` (notification).
  `[agent-chosen — needs review]` Quy tắc prefix: ưu tiên tên resource đầy đủ (`payment-`, `refund-`, `wallet-`, `reservation-`, `category-`); service nào đã có bộ prefix ngắn ổn định và được dùng xuyên suốt 4 file thì giữ nguyên bộ đó (`shp-`, `cv-`/`msg-`/`usr-`, `ntf-`, `sku-`, `media-`, `vch-`).
- `request_id`/`event_id`/`trace_id`: UUIDv7 trần (không prefix) — đây là ID kỹ thuật, không phải entity ID.
- Không trộn UUID trần với prefix-ULID cho cùng một entity trong cùng ví dụ.

## 4. Pagination

- List dùng offset pagination: `?page=1&size=20` (page từ 1, size mặc định 20, tối đa 100) → `meta: { "request_id", "page", "size", "total", "total_pages" }`. `[agent-chosen — needs review]` (chọn `total_pages` làm canonical vì auth-user/search đang dùng; bỏ `has_next`).
- Ngoại lệ: message history dùng cursor pagination (`has_more`, `next_after_sequence`, tối đa 100).
- Mọi api.md phải khai 1 dòng "Pagination" trong §1 Quy ước chung; endpoint list nào không phân trang phải ghi rõ.

## 5. Thời gian & tiền tệ

- DB MySQL: cột thời gian `DATETIME(6)` UTC; MongoDB/Elasticsearch: BSON Date / ISO-8601 UTC. API/event: ISO-8601 UTC (`...Z`), không epoch, không local time.
- Tiền: `BIGINT` integer VND (không FLOAT, không lưu đơn vị khác); trường `currency` luôn `"VND"` trong v1.

## 6. Path param

- Không dùng `{id}` chung chung — đặt tên theo resource: `{orderId}`, `{productId}`, `{shopId}`, `{categoryId}`, `{reviewId}`, `{paymentId}`, `{addressId}`, `{userId}`, `{skuId}`, `{reservationId}`, `{conversationId}`, `{messageId}`, `{notificationId}`, `{itemId}`, `{batchId}`, `{jobId}`, `{carrier}`.
- Tên param phải khớp giữa: route registry của Gateway ↔ endpoint của service ↔ test plan của service.

## 7. Kafka (outbox domain event)

- Đồng bộ dữ liệu bằng **outbox domain event** (không CDC/Debezium/change-stream).
- Topic: `<domain>.events.v1` (product/sku/category/catalog/order/invoice/payment/wallet/shipment/rating/user/shop/inventory/message) và `notification.commands.v1` (command).
- Envelope: `event_id`, `schema_version`, `event_type`, `aggregate_type`, `aggregate_id`, `occurred_at`, `[version]`, `payload`. `traceparent`/`request_id` nằm ở **Kafka header**, không nằm trong body.
- Tên event phải ghi tách rời từng event (không ghi gộp `a/b`); publisher và consumer phải dùng đúng một tên.
- Event có kênh EMAIL phải mang field recipient (`buyer`/shop owner) trong payload.
- Consumer xử lý idempotent theo `event_id`, bỏ event cũ theo `version`/`source_version`.

## 8. Actor context từ Gateway

- Gateway inject: `X-User-ID` (JWT `sub`), `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope` (+ `X-Auth-Method`); **strip** các header này nếu client gửi. Service phải đọc actor từ các header này, không tin body/query.
- Mọi api.md của service phải có 1 dòng ở §1 trích dẫn các header này.
- Header client gửi trực tiếp chỉ gồm: `Authorization`, `Content-Type`, `X-Request-ID` (≤64), `Idempotency-Key`, `X-MFA-Step-Up` (step-up 2FA) — CORS allowlist của Gateway đã khớp bộ này; nếu service cần header mới phải cập nhật allowlist gateway.
- `traceparent`/`tracestate` do Gateway/service propagate, client không gửi.

## Ngoại lệ hiện tại (freeze)

- `auth-user` và `notification`: dev đã code theo docs hiện tại — **không sửa docs 2 service này** tới khi hết freeze. Các lệch đã biết: `RATE_LIMITED`/`INTERNAL_ERROR` trần (auth-user), filter admin thiếu `PROCESSING` (notification), `DATETIME` không (6) (notification), command payload SMS/token (2 phía). Xem report `reports/no-task-docs-consistency/2026-09-18-review-docs-consistency-2.md`.
