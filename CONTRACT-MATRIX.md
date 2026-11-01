# Contract Matrix — giữa các service

> Nguồn · Code đối chiếu: `user-auth-service@develop` (`7c268a0`), `Product-Catalog-Service@develop` (`6572ece`), `search-service@develop` (`13d36e3`), `inventory-service@develop` (`92c1454`), `shipment-service@develop` (`a375a41`), `payment-wallet-service@main` (`859b7b1`), `notification-service@main` (`6c6d9a2`). Order-Commerce, Message, Rating-Comment chỉ có README nên dùng doc làm spec. · Cập nhật: 2026-10-31
> Ký hiệu: ✅ khớp · ❌ đang hỏng luồng dữ liệu · ℹ️ khác hình thức hoặc phần chưa triển khai, giữ nguyên (nhóm B §1) · ⏳ một phía chưa có code, chưa kiểm được end-to-end.

File này liệt kê mọi contract **giữa** các service: Kafka topic/event, envelope, payload, header, REST gọi chéo, route Gateway, health path, error envelope và tên biến môi trường. Doc từng service vẫn là nguồn chi tiết; khi code đổi, cập nhật dòng tương ứng ở đây.

## 1. Kết luận nhanh

**Nguyên tắc (chốt với team 2026-10-31):** contract mà service **đã triển khai** là chuẩn — doc và consumer theo code, không sửa lại producer đã chạy chỉ để đồng bộ hình thức. Service **chưa có code** (Order-Commerce, Message, Rating-Comment, phần Kafka của Inventory) phải theo convention đã có ở §8.

Các điểm khác nhau giữa service đã triển khai chia làm hai loại:

**A. Từng hỏng luồng dữ liệu — đã sửa ở phía consumer (merge 2026-10-31):** C1 + C5 → [search-service#27](https://github.com/ecommerce-taca/search-service/pull/27); C6 → [Product-Catalog-Service#44](https://github.com/ecommerce-taca/Product-Catalog-Service/pull/44).

| # | Lệch | Ảnh hưởng | Cách sửa đã merge (phía consumer) |
|---|---|---|---|
| C1 | Connector Debezium của Catalog đặt `event_id` ở header `id` và `aggregate_id` ở Kafka key, không trong value. Search bắt buộc `event_id` trong body. | Mọi event Catalog → Search vào DLQ. | Search: lấy `event_id` từ header `id`, `aggregate_id` từ key khi body thiếu. |
| C5 | `product.shop_snapshot_updated` có `aggregate_id = shop_id`; Search dùng `aggregate_id` làm ID document. | Đổi tên shop không cập nhật index. | Search: với event này, update-by-query theo `shop.shop_id`. |
| C6 | Auth-user `shop.updated` để field đổi trong `payload.snapshot`; Catalog đọc `payload.name` phẳng. | Tên shop ở Catalog không cập nhật. | Catalog consumer: đọc thêm `payload.snapshot.name`. |

**B. Khác hình thức nhưng không hỏng luồng nào đang chạy — giữ nguyên, doc đã ghi đúng code:**

| # | Khác biệt | Vì sao chưa cần sửa |
|---|---|---|
| C2 | Payment-Wallet publish record camelCase phẳng (`eventId`, `paymentId`…). | Chưa có consumer nào đọc `payment.created`/`wallet.allocated`. Consumer tương lai đọc đúng format này. |
| C3, C4 | Payment-Wallet chưa route `payment.succeeded/failed/…`, `payout.*`; event không có recipient. | Là phần **chưa triển khai** của Payment-Wallet. Khi làm, theo format Payment-Wallet đang dùng; Notification/Order sẽ phải thêm parser cho format này. |
| C7 | Search bỏ qua `product.media_deleted`. | Payload không có `is_cover`, Search không biết ảnh bị xoá có phải cover; ghi nhận là giới hạn. |
| C8 | Catalog chưa phát `sku.updated`, `sku.status_changed`, `category.*`. | SKU đi qua `product.skus_updated` (đã khớp). Category event là phần chưa triển khai của Catalog. |
| C9 | Notification phát `notification.delivered/failed` camelCase. | Auth-user chưa consume; consumer tương lai đọc camelCase. |
| C10 | Auth-user chỉ có `/actuator/health`; `/actuator/health/{liveness,readiness}` có khi chạy trên Kubernetes (Spring Boot tự bật probe, manifest `deploy/k8s/dev` dùng hai path này). | Gateway (chưa có code) cấu hình `http_path` riêng cho `up-auth-user`. |
| C11 | Notification chưa seed `message-received-v1`; SMS chưa có provider. | Message chưa có code; SMS là gap đã chấp nhận (2026-09-18). |
| C12 | Tên env Kafka khác nhau giữa service. | Mỗi service tự đọc env của mình; deploy theo bảng §7. |
| C13 | Auth-user `shop.updated` gửi logo dạng `snapshot.logo_object_key`; Catalog chỉ đọc `logo_url` phẳng. | Không làm hỏng luồng tên shop (C6 đã sửa); chỉ logo shop trong projection Catalog không đổi. Auth-user chỉ trả logo dạng presigned URL hết hạn sau 10 phút (`ShopAssetUrlAdapter`), nên Catalog không lưu được URL bền. Giữ nguyên: `shop.logo_url` trong response Catalog là `null`, FE lấy logo qua `GET /api/v1/shops/{shopId}` của auth-user (Catalog LLD §8 #27). |

## 2. Envelope Kafka theo producer

| Producer | Field trong value | Casing | Header Kafka | Khớp chuẩn? |
|---|---|---|---|---|
| Auth-user (event) | `event_id`, `event_type`, `schema_version`, `occurred_at`, `aggregate_type`, `aggregate_id`, `actor_user_id`, `payload` | snake | `event_id`, `event_type`, `schema_version`, `aggregate_type`, `aggregate_id` | ✅ (chuẩn tham chiếu) |
| Auth-user (command) | `event_id`, `schema_version`, `command_type`, `occurred_at`, `dedupe_key`, `user_id`, `channel`, `recipient`, `template`, `data` | snake | `event_id`, `command_type`, `schema_version` | ✅ khớp `NotificationCommandSchema` |
| Shipment (event) | `event_id`, `schema_version`, `event_type`, `occurred_at`, `aggregate_type`, `aggregate_id`, `correlation_id?`, `causation_id?`, `payload` | snake | không có | ✅ |
| Shipment (command vào) | `message_id`, `message_type=shipment.create`, `schema_version=1`, `correlation_id`, `causation_id?`, `occurred_at`, `payload` | snake | — | ⚠️ khác tên với command của Notification (`event_id`/`command_type`) |
| Product Catalog (qua Debezium) | `payload` + envelope `version`, `actor_user_id`, `event_type`, `schema_version`, `occurred_at`, `aggregate_type`; key = `aggregate_id`; `event_id` ở header `id` | snake | `id`, `traceparent` | ✅ C1 (đã sửa) |
| Payment-Wallet | record domain, ví dụ `eventId`, `occurredAt`, `paymentId`, `checkoutGroupId`, `orderIds`, `amount`, `method`, `status` | camel | `event_id`, `request_id`, `traceparent`, `tracestate` | ℹ️ C2 |
| Notification (delivered/failed) | `eventId`, `outcome`, `dedupeKey`, `notificationId`, `channel`, `templateKey`, `errorCode`, `occurredAt` | camel | `event_id`, `event_type` | ℹ️ C9 |
| Order, Inventory, Rating, Message | theo doc: envelope snake như auth-user | snake | — | ⏳ chưa có code |

## 3. Kafka topic / event

| Topic | Event | Producer | Consumer | Field consumer đọc | Trạng thái |
|---|---|---|---|---|---|
| `user.events.v1` | `user.created/updated/email_verified/password_changed/role_changed/status_changed` | Auth-user ✅ | Order (doc, `user.status_changed`) ⏳ | — | ⏳ |
| `shop.events.v1` | `shop.created` | Auth-user ✅ | Catalog ✅ | `shop_id`, `status` (Catalog tự sinh tên/slug tạm khi thiếu) | ✅ |
| `shop.events.v1` | `shop.updated` | Auth-user ✅ | Catalog ✅ | `payload.snapshot.name` (fallback `name` phẳng); `slug`, `logo_url` chỉ khi có ở dạng phẳng | ✅ C6 (đã sửa). Logo: auth-user gửi `snapshot.logo_object_key`, Catalog chưa đọc → logo không đổi (C13) |
| `shop.events.v1` | `shop.status_changed` | Auth-user ✅ | Catalog ✅ | `new_status`/`status` | ✅ |
| `shop.events.v1` | `shop.kyc.submitted/approved/needs_info/rejected/expired` | Auth-user ✅ | Catalog ✅; Payment (doc) ⏳ | event_type → KYC status | ✅ |
| `shop.events.v1` | `kyc.document.delete_requested` | Auth-user ✅ | không ai | — | ✅ (Catalog bỏ qua) |
| `notification.commands.v1` | `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED` | Auth-user ✅ | Notification ✅ | schema §2 | ✅ |
| `notification.commands.v1` | `PHONE_OTP_REQUESTED` | Auth-user ✅ | Notification | `channel=SMS` | ℹ️ C11 (DLQ) |
| `notification.commands.v1` | `MESSAGE_RECEIVED` | Message ⏳ | Notification | template `message-received-v1` | ℹ️ C11 (template chưa seed) |
| `notification.commands.v1` | `REVIEW_REQUESTED` | Rating ⏳ | Notification ✅ | template `review-request-v1` (đã seed) | ⏳ |
| `product.events.v1` | `product.created/updated/published/unpublished/archived/blocked/unblocked` | Catalog ✅ | Search ✅ | envelope + `payload` | ✅ C1 (đã sửa) |
| `product.events.v1` | `product.skus_updated` | Catalog ✅ | Search ✅ | `product_id`, `skus[]`, `version` | ✅ C1 (đã sửa) |
| `product.events.v1` | `product.media_uploaded` | Catalog ✅ | Search ✅ | `is_cover`, `url` | ✅ C1 (đã sửa) |
| `product.events.v1` | `product.media_deleted` | Catalog ✅ | Search ✗ | — | ℹ️ C7 |
| `sku.events.v1` | `sku.created` | Catalog (chỉ import) ✅ | Search ✅ | `sku_id`, `product_id` | ✅ C1 (đã sửa) |
| `sku.events.v1` | `sku.updated`, `sku.status_changed` | Catalog ✗ | Search ✅ | — | ℹ️ C8 |
| `category.events.v1` | `category.created/updated/status_changed` | Catalog ✗ | Search ✅ | — | ℹ️ C8 |
| `catalog.events.v1` | `product.category_changed` | Catalog ✅ | Search ✅ | `primary_category_id`, `secondary_category_ids` | ✅ C1 (đã sửa) |
| `catalog.events.v1` | `product.media_updated` | Catalog ✗ | Search ✅ | `media[0].url`/`cover_url` | ℹ️ C7 |
| `catalog.events.v1` | `product.shop_snapshot_updated` | Catalog ✅ | Search ✅ | `payload.shop_id`, `payload.name` → update-by-query theo `shop.shop_id` | ✅ C5 (đã sửa) |
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
| `payment.events.v1` | `payment.succeeded/failed/expired/refunded` | Payment ✗ (chỉ outbox) | Order ⏳, Notification ✅ | `event_type`, `payload.buyer`, `payment_id` | ℹ️ C2, C3, C4 |
| `wallet.events.v1` | `wallet.allocated` | Payment ✅ | không ai | — | ✅ |
| `wallet.events.v1` | `payout.succeeded/failed` | Payment ✗ | Notification ✅ | `buyer.*`, `payout_id` | ℹ️ C3, C4 |
| `notification.delivered.v1` / `notification.failed.v1` | `notification.delivered/failed` | Notification ✅ | Auth-user ✗ (chưa consume) | — | ℹ️ C9 (casing) |
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
| Auth-user | `/actuator/health/liveness` (chỉ trên K8s; ngoài K8s dùng `/actuator/health`) | `/actuator/health/readiness` (chỉ trên K8s) | ℹ️ C10 |
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

## 8. Convention cho phần chưa triển khai

Áp dụng cho Order-Commerce, Message, Rating-Comment, Kafka của Inventory và event mới của service đang có. Không dùng để sửa lại code đã chạy.

1. **Event mới:** dùng envelope như auth-user/shipment: `event_id`, `event_type`, `schema_version`, `occurred_at` (ISO-8601 UTC), `aggregate_type`, `aggregate_id`, `payload`, field snake_case. Riêng Payment-Wallet giữ format record đang dùng cho event của nó.
2. **Command tới Notification:** đúng `NotificationCommandSchema` (auth-user đang dùng). **Command tới Shipment:** đúng envelope `message_id`/`message_type=shipment.create` của Shipment.
3. **Recipient:** event cần thông báo buyer mang `payload.buyer.{user_id,email}` (Notification đang đọc, Shipment đang gửi).
4. **Consumer đọc event của service đã triển khai:** đọc đúng format producer đó (bảng §2), kể cả header (`id` của Debezium, `event_id` của Payment-Wallet).
5. **Health:** service mới expose `/health/live` và `/health/ready` ở root.
6. **Env:** service mới (Node) theo Search/Notification: `KAFKA_BROKERS`, `KAFKA_CLIENT_ID`; service mới (Java) theo Shipment/Payment: `KAFKA_BOOTSTRAP_SERVERS`.
7. **DLQ:** `<service>.<source>.dlq.v1`.
