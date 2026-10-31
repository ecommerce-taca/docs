# Contract Matrix — giữa các service

> Nguồn · Code đối chiếu: `user-auth-service@develop` (`7c268a0`), `Product-Catalog-Service@develop` (`fe1e0d7`), `search-service@develop` (`9750a2f`), `inventory-service@develop` (`92c1454`), `shipment-service@develop` (`a375a41`), `payment-wallet-service@main` (`859b7b1`), `notification-service@main` (`6c6d9a2`). Order-Commerce, Message, Rating-Comment chỉ có README nên dùng doc làm spec. · Cập nhật: 2026-10-31
> Ký hiệu: ✅ khớp · ❌ lệch (ghi rõ phía nào sai) · ⏳ một phía chưa có code, chưa kiểm được end-to-end.

File này liệt kê mọi contract **giữa** các service: Kafka topic/event, envelope, payload, header, REST gọi chéo, route Gateway, health path, error envelope và tên biến môi trường. Doc từng service vẫn là nguồn chi tiết; khi code đổi, cập nhật dòng tương ứng ở đây.

## 1. Kết luận nhanh

Chưa đồng nhất. Phần doc đã được sửa cho khớp code trong cùng PR với file này. Các lệch còn lại nằm ở **code** và cần chủ service xử lý:

| # | Lệch (code) | Ảnh hưởng | Chủ |
|---|---|---|---|
| C1 | Connector Debezium của Catalog không đặt `event_id` (và `aggregate_id`) vào value; Search và consumer của Catalog đều bắt buộc `event_id` trong body. | Mọi event Catalog → Search vào DLQ `search.events.dlq.v1`. | Product Catalog |
| C2 | Payment-Wallet publish record Java camelCase phẳng (`eventId`, `paymentId`…), không có envelope `event_type`/`payload`. | Notification/Order không parse được (Notification → DLQ `INVALID_MESSAGE`). | Payment-Wallet |
| C3 | Payment-Wallet chỉ route `payment.created`, `wallet.allocated`; `payment.succeeded/failed/expired/refunded`, `refund.*`, `payout.*` chỉ nằm trong outbox. | Order không chuyển `PENDING_PAYMENT → CONFIRMED`; Notification không gửi kết quả thanh toán/payout. | Payment-Wallet |
| C4 | Event Payment-Wallet không có recipient; doc payout cũ dùng `owner_user_id`/`owner_email`, Notification chỉ đọc `buyer.user_id`/`buyer.email`. | Không gửi được thông báo payment/payout. | Payment-Wallet + Notification |
| C5 | `product.shop_snapshot_updated` có `aggregate_id = shop_id`, không có `product_id`; Search dùng `aggregate_id` làm ID document. | Đổi tên shop không cập nhật được index (patch vào doc không tồn tại). | Product Catalog + Search |
| C6 | Auth-user `shop.updated` để field đổi trong `payload.snapshot` (lồng) và không gửi `slug`/`logo_url`; Catalog đọc `payload.name`/`slug`/`logo_url` phẳng. `shop.created` chỉ có `shop_id`, `owner_user_id`, `status`. | Shop snapshot ở Catalog giữ tên mặc định `Shop xxxxxxxx`, không bao giờ cập nhật tên/logo. | Auth-user + Product Catalog |
| C7 | Search bỏ qua `product.media_deleted`; còn nhánh chờ `product.media_updated` mà Catalog không phát. | Xoá ảnh cover không cập nhật `cover_url`. | Search |
| C8 | Catalog chưa phát `sku.updated`, `sku.status_changed`, `category.*`. | Search không cập nhật SKU lẻ/category tree (đỡ một phần nhờ `product.skus_updated`). | Product Catalog |
| C9 | Notification phát `notification.delivered/failed` bằng camelCase (`dedupeKey`, `eventId`); command đầu vào và mọi topic khác dùng snake_case. Auth-user chưa consume. | Khi auth-user consume sẽ phải map tay. | Notification |
| C10 | Auth-user không có `/health/live`, chỉ `/actuator/health[/liveness|/readiness]`; Gateway probe `/health/live`. | Kong active healthcheck đánh dấu auth-user unhealthy. | Auth-user |
| C11 | Notification thiếu template `message-received-v1`; `PHONE_OTP_REQUESTED` channel `SMS` vào DLQ. | Thông báo tin nhắn và OTP SMS không gửi được. | Notification |
| C12 | Tên env Kafka không thống nhất (xem §7). | Cấu hình deploy dễ sai. | Tất cả |

## 2. Envelope Kafka theo producer

| Producer | Field trong value | Casing | Header Kafka | Khớp chuẩn? |
|---|---|---|---|---|
| Auth-user (event) | `event_id`, `event_type`, `schema_version`, `occurred_at`, `aggregate_type`, `aggregate_id`, `actor_user_id`, `payload` | snake | `event_id`, `event_type`, `schema_version`, `aggregate_type`, `aggregate_id` | ✅ (chuẩn tham chiếu) |
| Auth-user (command) | `event_id`, `schema_version`, `command_type`, `occurred_at`, `dedupe_key`, `user_id`, `channel`, `recipient`, `template`, `data` | snake | `event_id`, `command_type`, `schema_version` | ✅ khớp `NotificationCommandSchema` |
| Shipment (event) | `event_id`, `schema_version`, `event_type`, `occurred_at`, `aggregate_type`, `aggregate_id`, `correlation_id?`, `causation_id?`, `payload` | snake | không có | ✅ |
| Shipment (command vào) | `message_id`, `message_type=shipment.create`, `schema_version=1`, `correlation_id`, `causation_id?`, `occurred_at`, `payload` | snake | — | ⚠️ khác tên với command của Notification (`event_id`/`command_type`) |
| Product Catalog (qua Debezium) | `payload` + envelope `version`, `actor_user_id`, `event_type`, `schema_version`, `occurred_at`, `aggregate_type`; key = `aggregate_id`; `event_id` ở header `id` | snake | `id`, `traceparent` | ❌ C1 |
| Payment-Wallet | record domain, ví dụ `eventId`, `occurredAt`, `paymentId`, `checkoutGroupId`, `orderIds`, `amount`, `method`, `status` | camel | `event_id`, `request_id`, `traceparent`, `tracestate` | ❌ C2 |
| Notification (delivered/failed) | `eventId`, `outcome`, `dedupeKey`, `notificationId`, `channel`, `templateKey`, `errorCode`, `occurredAt` | camel | `event_id`, `event_type` | ❌ C9 |
| Order, Inventory, Rating, Message | theo doc: envelope snake như auth-user | snake | — | ⏳ chưa có code |

## 3. Kafka topic / event

| Topic | Event | Producer | Consumer | Field consumer đọc | Trạng thái |
|---|---|---|---|---|---|
| `user.events.v1` | `user.created/updated/email_verified/password_changed/role_changed/status_changed` | Auth-user ✅ | Order (doc, `user.status_changed`) ⏳ | — | ⏳ |
| `shop.events.v1` | `shop.created` | Auth-user ✅ | Catalog ✅ | `shop_id`, `name`, `slug`, `logo_url`, `status`, `version` | ❌ C6 (thiếu `name`/`slug`) |
| `shop.events.v1` | `shop.updated` | Auth-user ✅ | Catalog ✅ | `name`, `slug`, `logo_url` phẳng | ❌ C6 (`snapshot` lồng, `logo_object_key`) |
| `shop.events.v1` | `shop.status_changed` | Auth-user ✅ | Catalog ✅ | `new_status`/`status` | ✅ |
| `shop.events.v1` | `shop.kyc.submitted/approved/needs_info/rejected/expired` | Auth-user ✅ | Catalog ✅; Payment (doc) ⏳ | event_type → KYC status | ✅ |
| `shop.events.v1` | `kyc.document.delete_requested` | Auth-user ✅ | không ai | — | ✅ (Catalog bỏ qua) |
| `notification.commands.v1` | `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED` | Auth-user ✅ | Notification ✅ | schema §2 | ✅ |
| `notification.commands.v1` | `PHONE_OTP_REQUESTED` | Auth-user ✅ | Notification | `channel=SMS` | ❌ C11 (DLQ) |
| `notification.commands.v1` | `MESSAGE_RECEIVED` | Message ⏳ | Notification | template `message-received-v1` | ❌ C11 (template chưa seed) |
| `notification.commands.v1` | `REVIEW_REQUESTED` | Rating ⏳ | Notification ✅ | template `review-request-v1` (đã seed) | ⏳ |
| `product.events.v1` | `product.created/updated/published/unpublished/archived/blocked/unblocked` | Catalog ✅ | Search ✅ | envelope + `payload` | ❌ C1 |
| `product.events.v1` | `product.skus_updated` | Catalog ✅ | Search ✅ | `product_id`, `skus[]`, `version` | ❌ C1 |
| `product.events.v1` | `product.media_uploaded` | Catalog ✅ | Search ✅ | `is_cover`, `url` | ❌ C1 |
| `product.events.v1` | `product.media_deleted` | Catalog ✅ | Search ✗ | — | ❌ C7 |
| `sku.events.v1` | `sku.created` | Catalog (chỉ import) ✅ | Search ✅ | `sku_id`, `product_id` | ❌ C1 |
| `sku.events.v1` | `sku.updated`, `sku.status_changed` | Catalog ✗ | Search ✅ | — | ❌ C8 |
| `category.events.v1` | `category.created/updated/status_changed` | Catalog ✗ | Search ✅ | — | ❌ C8 |
| `catalog.events.v1` | `product.category_changed` | Catalog ✅ | Search ✅ | `primary_category_id`, `secondary_category_ids` | ❌ C1 |
| `catalog.events.v1` | `product.media_updated` | Catalog ✗ | Search ✅ | `media[0].url`/`cover_url` | ❌ C7 |
| `catalog.events.v1` | `product.shop_snapshot_updated` | Catalog ✅ | Search ✅ | `aggregate_id` làm product ID | ❌ C5 |
| `inventory.events.v1` | `inventory.stock_snapshot.updated` | Inventory ⏳ (scaffold) | Catalog ✅ | `sku_id`, `product_id`, `available_qty`, `reserved_qty`, `committed_qty`, `source_version`, `as_of` | ⏳ (doc inventory đã chốt đúng tên field) |
| `inventory.events.v1` | `inventory.reservation.*`, `inventory.adjusted` | Inventory ⏳ | Order ⏳ | — | ⏳ |
| `rating.events.v1` | `rating.aggregate.updated` | Rating ⏳ | Catalog ✅, Search ✅ | `product_id`, `avg`, `count` | ⏳ (doc khớp consumer) |
| `order.events.v1` | `order.confirmed/paid/cancelled` | Order ⏳ | Notification ✅ | `buyer.user_id`, `buyer.email`, `order_id` | ⏳ (doc khớp consumer) |
| `order.events.v1` | `order.cancelled` | Order ⏳ | Shipment ✅ | `event_id`, `payload.order_id`, `reason`, `correlation_id?` | ⏳ (doc khớp consumer) |
| `order.events.v1` | `order.delivered` | Order ⏳ | Rating ⏳ | `buyer`, `items[]` | ⏳ |
| `invoice.events.v1` | `invoice.issued` | Order ⏳ | Notification ✅ | `buyer.*`, `invoice_id` | ⏳ |
| `shipment.commands.v1` | `shipment.create` | Order ⏳ (doc Order chưa mô tả; Order gọi REST `POST /api/v1/internal/shipments`) | Shipment ✅ | envelope `message_*`, `payload.command_id`, `shop_order_id`, `buyer`, `from/to_address_snapshot`, `items[]` | ⏳ |
| `shipment.events.v1` | `shipment.delivered`, `shipment.failed` | Shipment ✅ | Notification ✅, Payment ✅, Order ⏳ | `shipment_id`, `order_id`, `delivered_at`, `buyer.user_id`, `buyer.email` | ✅ |
| `shipment.events.v1` | `shipment.created/status_changed/cancelled/creation_pending/creation_failed` | Shipment ✅ | Order ⏳ | — | ⏳ |
| `payment.events.v1` | `payment.created` | Payment ✅ | không ai | — | ✅ |
| `payment.events.v1` | `payment.succeeded/failed/expired/refunded` | Payment ✗ (chỉ outbox) | Order ⏳, Notification ✅ | `event_type`, `payload.buyer`, `payment_id` | ❌ C2, C3, C4 |
| `wallet.events.v1` | `wallet.allocated` | Payment ✅ | không ai | — | ✅ |
| `wallet.events.v1` | `payout.succeeded/failed` | Payment ✗ | Notification ✅ | `buyer.*`, `payout_id` | ❌ C3, C4 |
| `notification.delivered.v1` / `notification.failed.v1` | `notification.delivered/failed` | Notification ✅ | Auth-user ✗ (chưa consume) | — | ❌ C9 (casing) |
| DLQ | `auth-user.events.dlq.v1`, `product-catalog.events.dlq.v1`, `search.events.dlq.v1`, `notification.events.dlq.v1`, `shipment.events.dlq.v1`, `shipment.consumer.dlq.v1`, `payment-wallet.outbox.dlq.v1` | mỗi service | ops | — | ⚠️ tên không theo một mẫu (xem §8) |

## 4. REST gọi chéo và Gateway

| Caller → Callee | Path | Doc caller | Code callee | Trạng thái |
|---|---|---|---|---|
| Order → Shipment | `POST /api/v1/internal/shipping/quote` | Đã sửa (trước ghi `GET /internal/shipping/quote`) | `QuoteController` POST | ✅ |
| Order → Shipment | `POST /api/v1/internal/shipments` | Đã sửa prefix + link §3.4 | `ShipmentController` | ✅ |
| Order → Inventory | `POST /internal/inventory/reservations[/{id}/commit|release]` | khớp doc Inventory | chưa có code | ⏳ |
| Order → Payment | `POST /api/v1/payments`, `POST /api/v1/payments/{id}/refunds` | khớp | `PaymentController` | ✅ |
| Gateway → Auth-user | `/api/v1/locations/vn/**` | Đã thêm vào route table | `VietnamLocationController` (permitAll) | ✅ |
| Gateway → mọi upstream | `/api/v1/...` | route table | Catalog (`GLOBAL_PREFIX=api/v1`), Search, Notification, Auth, Payment, Shipment đều có prefix `/api/v1` | ✅ |
| Notification → Auth-user | JWKS `/.well-known/jwks.json` (`JWT_JWKS_URL`) | khớp | `SecurityConfig` permitAll | ✅ |

Header HTTP nội bộ: `X-Request-ID`, `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`, `X-MFA-Step-Up`, `X-Internal-Service-Token` dùng cùng tên ở mọi service có code (HTTP header không phân biệt hoa thường). ✅

## 5. Health path

| Service | Liveness | Readiness | Gateway probe `/health/live` |
|---|---|---|---|
| Auth-user | `/actuator/health/liveness` | `/actuator/health/readiness` | ❌ C10 |
| Product Catalog | `/health/live` | `/health/ready` | ✅ |
| Search | `/health/live` | `/api/v1/search/health` | ✅ |
| Inventory | `/health/live` | `/health/ready` | ✅ |
| Shipment | `/health/live` | `/health/ready` | ✅ |
| Payment-Wallet | `/health/live` (+ `/health/liveness`) | `/health/ready` (+ `/health/readiness`) | ✅ |
| Notification | `/health/live` | `/health/ready` | ✅ |

## 6. Error envelope và mã lỗi

| Service | Body lỗi | Prefix mã lỗi |
|---|---|---|
| Auth-user | `{error:{code,message,details,trace_id}}` | `AUTH_`, `KYC_` |
| Product Catalog | `{error:{code,message,details,trace_id}}` | `PRODUCT_`, `MEDIA_` |
| Search | `{error:{code,message,details,trace_id,request_id}}` | `SEARCH_`, `REINDEX_` |
| Inventory | `{error:{code,message,details,trace_id}}` | `INVENTORY_` |
| Shipment | `{error:{code,message,details,trace_id}, meta:{request_id}}` | `SHIPMENT_` |
| Notification | `{error:{code,message,details,trace_id}}` | `NOTIFICATION_` |
| Payment-Wallet | `{error:{code,message}, meta:{request_id}}` — thiếu `details`, `trace_id` | `PAYMENT_`, `REFUND_`, `SETTLEMENT_`… |

Prefix theo service là thống nhất. Hình dạng `error` gần khớp; Payment-Wallet thiếu `trace_id` mà Gateway (`taca-error-envelope`) sẽ phải bổ sung.

## 7. Biến môi trường dùng chung

| Ý nghĩa | Auth-user | Catalog | Search | Inventory | Shipment | Payment | Notification |
|---|---|---|---|---|---|---|---|
| Kafka broker | `SPRING_KAFKA_BOOTSTRAP_SERVERS` | `KAFKA_BROKERS` | `KAFKA_BROKERS` | `SPRING_KAFKA_BOOTSTRAP_SERVERS` | `KAFKA_BOOTSTRAP_SERVERS` | `KAFKA_BOOTSTRAP_SERVERS` | `KAFKA_BROKERS` |
| Consumer group | — | `KAFKA_GROUP_ID` | `KAFKA_GROUP_ID` | — | property `shipment.kafka.*-consumer-group-id` | `KAFKA_CONSUMER_GROUP_ID` | `KAFKA_CONSUMER_GROUP_ID` |
| Bật consumer | — | `KAFKA_CONSUMER_ENABLED` | `KAFKA_CONSUMER_ENABLED` | — | `SHIPMENT_KAFKA_CONSUMER_ENABLED` | — | — |
| Topic | `auth.kafka.topics.*` | hằng số | hằng số + `KAFKA_TOPIC_PREFIX` | — | `SHIPMENT_EVENTS_TOPIC`, `ORDER_EVENTS_TOPIC`… | `app.kafka.topics.*` | `KAFKA_TOPIC_*` |
| Port HTTP | `SERVER_PORT` | `PORT` | `PORT` | `SERVER_PORT` | `SERVER_PORT` | `SERVER_PORT` | `PORT` |
| Token nội bộ | — | — | — | `inventory.security.internal-token` | `SHIPMENT_INTERNAL_TOKEN` | — | — |

## 8. Convention đề xuất

1. **Envelope Kafka:** dùng đúng envelope của auth-user/shipment cho mọi event: `event_id`, `event_type`, `schema_version`, `occurred_at` (ISO-8601 UTC), `aggregate_type`, `aggregate_id`, `version?`, `actor_user_id?`, `correlation_id?`, `causation_id?`, `payload`. Mọi field trong value và payload dùng **snake_case**. Payment-Wallet và output của Notification phải đổi (Jackson `SNAKE_CASE` + wrapper envelope; Notification đổi `DeliveryStatusEvent`). Catalog thêm `event_id:envelope:event_id,aggregate_id:envelope:aggregate_id` vào connector.
2. **Command:** thống nhất một kiểu: `event_id` + `command_type` (như `notification.commands.v1`), Shipment đổi `message_id`/`message_type` hoặc cả hai cùng đổi sang `command_id`/`command_type`.
3. **Header Kafka:** mọi producer gửi `event_id`, `event_type`, `schema_version`, `traceparent` (và `tracestate` nếu có).
4. **Recipient:** mọi event cần thông báo mang `payload.buyer.{user_id,email}` cho buyer và `payload.recipient.{user_id,email}` cho seller/owner; Notification đọc cả hai, bỏ fallback tên rời.
5. **Event cho read model phải mang ID của document đích:** event fan-out theo shop (`product.shop_snapshot_updated`) phát mỗi product một event, hoặc consumer update-by-query theo `shop_id`.
6. **Health:** mọi service expose `/health/live` và `/health/ready` ở root, không prefix.
7. **Env:** `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_CONSUMER_GROUP_ID`, `KAFKA_CONSUMER_ENABLED`, `<SERVICE>_INTERNAL_TOKEN`, `SERVER_PORT` cho Java và `PORT` cho Node (giữ theo framework).
8. **DLQ:** `<service>.<source>.dlq.v1`, ví dụ `search.events.dlq.v1`, `shipment.commands.dlq.v1`.
