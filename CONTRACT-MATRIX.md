# Contract Matrix — giữa các service

> Nguồn · Code đối chiếu: `user-auth-service@develop` (`7c268a0`), `Product-Catalog-Service@develop` (`6572ece`), `search-service@develop` (`13d36e3`), `inventory-service@develop` (`92c1454`), `shipment-service@develop` (`a375a41`), `payment-wallet-service@main` (`859b7b1`), `notification-service@main` (`6c6d9a2`). Cart, Checkout, Order (tách từ Order-Commerce ngày 2026-11-01), Message, Rating-Comment chưa có code nên dùng doc làm spec. · Cập nhật: 2026-11-01
> Ký hiệu: ✅ khớp · ❌ đang hỏng luồng dữ liệu · ℹ️ khác hình thức hoặc phần chưa triển khai, giữ nguyên (nhóm B §1) · ⏳ một phía chưa có code, chưa kiểm được end-to-end. · Recheck ngoài 2026-11-01: xem §9.

File này liệt kê mọi contract **giữa** các service: Kafka topic/event, envelope, payload, header, REST gọi chéo, route Gateway, health path, error envelope và tên biến môi trường. Doc từng service vẫn là nguồn chi tiết; khi code đổi, cập nhật dòng tương ứng ở đây.

## 1. Kết luận nhanh

**Nguyên tắc (chốt với team 2026-10-31):** contract mà service **đã triển khai** là chuẩn — doc và consumer theo code, không sửa lại producer đã chạy chỉ để đồng bộ hình thức. Service **chưa có code** (Cart, Checkout, Order — tách từ Order-Commerce ngày 2026-11-01 —, Message, Rating-Comment, phần Kafka của Inventory) phải theo convention đã có ở §8.

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

**C. Lệch giữa hai phía đều đã có code — cần team chốt trước khi sửa code (recheck 2026-11-01, §9):**

| # | Lệch | Ảnh hưởng hiện tại | Hướng đề xuất (chưa làm) |
|---|---|---|---|
| C14 | Tên role/permission. Auth-user chỉ cấp role `BUYER`, `SELLER`, `SELLER_STAFF`, `SUPER_ADMIN`, `RISK_MANAGER`, `CATALOG_ADMIN`, `FINANCE_OPS`, `SUPPORT_VIEWER` và chưa seed permission `SEARCH_ADMIN`/`VOUCHER_MANAGE`. Shipment nhận `SHOP_STAFF` (không phải `SELLER_STAFF`) và `ADMIN`; Inventory `requireAdminStepUp` kiểm role `ADMIN` (scaffold, chưa endpoint nào gọi); Search kiểm `SEARCH_ADMIN` trong **role**. | Khi Gateway chạy thật: nhân viên shop (`SELLER_STAFF`) bị Shipment trả 403; không admin nào của auth-user qua được admin route Search/Inventory hay `/internal/ops` Shipment bằng role `ADMIN`. | Lấy role của auth-user làm chuẩn: Shipment nhận thêm `SELLER_STAFF`, Inventory/Shipment map admin sang role auth-user (`SUPER_ADMIN`…), auth-user seed `SEARCH_ADMIN` và Search đọc nó từ `X-User-Permissions` (hoặc seed thành role). **Đã merge 2026-11-01:** shipment-service#12, inventory-service#15, search-service#28 (giữ tên cũ để tương thích). auth-user seed `SEARCH_ADMIN`/`VOUCHER_MANAGE` + claim `permissions` cho admin: user-auth-service#115. |
| C15 | Template `shipment-delivered-v1`/`shipment-failed-v1` dùng `{{order_code}}`; payload `shipment.delivered/failed` của Shipment (`ShipmentCommandService.eventPayload`) không có `order_code`, và request tạo shipment cũng không mang mã đơn hiển thị. | Event qua được validate nhưng thông báo giao hàng hiển thị mã đơn rỗng. | Order (chưa code) gửi `order_code` khi tạo shipment, Shipment lưu và phát `order_code` trong event. **Đã merge 2026-11-01:** shipment-service#12 (`order_code` optional trong create request/command, cột `shipments.order_code`, phát `payload.order_code`). Order doc đã ghi phải gửi `order_code`. |
| C16 | `X-MFA-Step-Up`: auth-user nhận **token opaque** và tự verify; Gateway doc forward token đó; Search chỉ chấp nhận `true`/`1`; Payment chấp nhận mọi giá trị khác rỗng. | Qua Gateway, admin Search luôn bị `403 SEARCH_ADMIN_FORBIDDEN` ở reindex/cancel; Payment chỉ kiểm header khác rỗng nên an toàn phụ thuộc hoàn toàn vào việc Gateway đã verify token — mà auth-user chưa có endpoint cho Gateway verify. | Gateway (chưa code) verify token với auth-user rồi gửi upstream `X-MFA-Step-Up: true`; riêng route auth-user giữ token gốc. **Đã merge 2026-11-01:** endpoint `POST /api/v1/auth/2fa/step-up/check` (user-auth-service#115); luồng Gateway ghi ở api-gateway LLD (bảng header). |

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
| Order, Checkout, Inventory, Rating, Message | theo doc: envelope snake như auth-user | snake | — | ⏳ chưa có code |

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
| `order.events.v1` | `order.confirmed/paid/cancelled` | Order ⏳ | Notification ✅ | `buyer.user_id`, `buyer.email`, `order_id` + placeholder top-level `order_code`, `display_name` (confirmed), `amount` (paid) | ⏳ (doc Order đã thêm field template, 2026-11-01) |
| `order.events.v1` | `order.cancelled` | Order ⏳ | Shipment ✅ | `event_id`, `payload.order_id`, `reason`, `correlation_id?` | ⏳ (doc khớp consumer) |
| `order.events.v1` | `order.delivered` | Order ⏳ | Rating ⏳ | `buyer`, `items[]` | ⏳ |
| `order.events.v1` | `order.cancelled` | Order ⏳ | Inventory ⏳ (reconcile) | `payload.order_id` | ⏳ |
| `order.events.v1` | `order.created`, `order.cancelled` | Order ⏳ | Message ⏳, Payment (doc, chưa có listener) | order context | ⏳ |
| `voucher.events.v1` | `voucher.redeemed/released` | Checkout ⏳ | chưa có (analytics) | — | ⏳ (mới 2026-11-01, trước là Order-Commerce) |
| `invoice.events.v1` | `invoice.issued` | Order ⏳ | Notification ✅ | `buyer.*`, `invoice_id`, `order_code`, `invoice_url` | ⏳ (doc Order đổi `pdf_url` → `invoice_url`, 2026-11-01) |
| `shipment.commands.v1` | `shipment.create` | Order ⏳ (doc Order chưa mô tả; Order gọi REST `POST /api/v1/internal/shipments`) | Shipment ✅ | envelope `message_*`, `payload.command_id`, `shop_order_id`, `buyer`, `from/to_address_snapshot`, `items[]` | ⏳ |
| `shipment.events.v1` | `shipment.delivered`, `shipment.failed` | Shipment ✅ | Notification ✅, Payment ✅, Order ⏳ | `shipment_id`, `order_id`, `order_code`, `delivered_at`, `reason`, `buyer.user_id`, `buyer.email` | ✅ C15 đã sửa (shipment-service#12, `ShipmentCommandService` phát `payload.order_code`); `order-service` phải gửi `order_code` khi tạo shipment |
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
| Checkout → Shipment | `POST /api/v1/internal/shipping/quote` | khớp (`checkout-docs` api §3.1) | `QuoteController` POST | ✅ |
| Order → Shipment | `POST /api/v1/internal/shipping/carrier-quotes` (façade `GET /seller/orders/{id}/shipment/carriers`) | khớp shape `QuoteService.compare` (`order-docs` api §3.6) | `QuoteController` | ✅ |
| Order → Shipment | `POST /api/v1/internal/shipments` | body đủ field `CreateShipmentRequest` (`buyer_email` bắt buộc, `cod_amount`, `order_code`) — `order-docs` api §3.4 | `ShipmentController` | ✅ |
| Checkout → Inventory | `POST /internal/inventory/reservations` (mỗi đơn con một reservation, `order_intent_id = order_id`), `…/{id}/release` (chỉ khi chưa có đơn) | khớp doc Inventory | chưa có code | ⏳ |
| Order → Inventory | `POST /internal/inventory/reservations/{id}/commit|release` | khớp doc Inventory | chưa có code | ⏳ |
| Order → Payment | `POST /api/v1/payments`, `POST /api/v1/payments/{id}/refunds` | khớp `CreatePaymentRequest`; `order-service` là caller `InternalService.ORDER_COMMERCE` duy nhất | `PaymentController` | ✅ |
| Checkout → Cart | `GET /internal/carts/{buyerUserId}`, `POST /internal/carts/{buyerUserId}/checked-out` | `cart-docs` api §3.5–3.6 | chưa có code | ⏳ |
| Checkout → Order | `POST /internal/orders`, `GET /internal/orders?checkout_group_id=` | `order-docs` api §3.7–3.8 | chưa có code | ⏳ |
| Order → Checkout | `POST /internal/checkout-groups/{id}/recalculate` | `checkout-docs` api §3.4 | chưa có code | ⏳ |
| Cart → Product | `GET /products/{productId}` (batch không có `skus[]`, không dùng) | khớp | Catalog ✅ | ✅ |
| Checkout → Product | `GET /products/{productId}` (giá SKU, SKU active, `tax_rate_bps`) | `checkout-docs` lld §6.3–6.4 | Catalog ✅ | ✅ |
| Gateway → Auth-user | `/api/v1/locations/vn/**` | Đã thêm vào route table | `VietnamLocationController` (permitAll) | ✅ |
| Gateway → mọi upstream | `/api/v1/...` | route table | Catalog (`GLOBAL_PREFIX=api/v1`), Search, Notification, Auth, Payment, Shipment đều có prefix `/api/v1` | ✅ |
| Notification → Auth-user | JWKS `/.well-known/jwks.json` (`JWT_JWKS_URL`) | khớp | `SecurityConfig` permitAll | ✅ |

Header HTTP nội bộ: `X-Request-ID`, `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`, `X-MFA-Step-Up`, `X-Internal-Service-Token` dùng cùng tên ở mọi service có code (HTTP header không phân biệt hoa thường). ✅ về tên; ⚠️ giá trị `X-User-Roles` lệch giữa service (C14) và `X-MFA-Step-Up` lệch ngữ nghĩa (C16).

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

Áp dụng cho Cart, Checkout, Order, Message, Rating-Comment, Kafka của Inventory và event mới của service đang có. Không dùng để sửa lại code đã chạy.

1. **Event mới:** dùng envelope như auth-user/shipment: `event_id`, `event_type`, `schema_version`, `occurred_at` (ISO-8601 UTC), `aggregate_type`, `aggregate_id`, `payload`, field snake_case. Riêng Payment-Wallet giữ format record đang dùng cho event của nó.
2. **Command tới Notification:** đúng `NotificationCommandSchema` (auth-user đang dùng). **Command tới Shipment:** đúng envelope `message_id`/`message_type=shipment.create` của Shipment.
3. **Recipient:** event cần thông báo buyer mang `payload.buyer.{user_id,email}` (Notification đang đọc, Shipment đang gửi).
4. **Consumer đọc event của service đã triển khai:** đọc đúng format producer đó (bảng §2), kể cả header (`id` của Debezium, `event_id` của Payment-Wallet).
5. **Health:** service mới expose `/health/live` và `/health/ready` ở root.
6. **Env:** service mới (Node) theo Search/Notification: `KAFKA_BROKERS`, `KAFKA_CLIENT_ID`; service mới (Java) theo Shipment/Payment: `KAFKA_BOOTSTRAP_SERVERS`.
7. **DLQ:** `<service>.<source>.dlq.v1`.

## 9. Recheck ngoài 2026-11-01 (16 mục trên docs@873b0a7)

Đối chiếu lại với code: `user-auth-service@develop` (`c031391`), `search-service@develop` (`13d36e3`), `inventory-service@develop` (`92c1454`), `shipment-service@develop` (`a375a41`), `payment-wallet-service@main` (`859b7b1`), `notification-service@main` (`6c6d9a2`). Phía chưa có code (Order, Message, Gateway) sửa doc theo phía đã có code.

| ID | Kết luận | Xử lý |
|---|---|---|
| V01 | Đúng. Shipment `SHOP_STAFF` ≠ auth-user `SELLER_STAFF`. | C14 — shipment-service#12. |
| V02 | Đúng, nhưng chỉ ở scaffold: `AuthorizationPolicy.requireAdminStepUp` chưa được endpoint nào gọi. | C14 — inventory-service#15. |
| V03 | Đúng. Thêm: auth-user **chưa seed** `SEARCH_ADMIN` dưới dạng nào. | search-service#28 đọc cả permission + `SUPER_ADMIN`; user-auth-service#115 seed `SEARCH_ADMIN` và đưa claim `permissions` vào token admin. |
| V04 | Đúng. | C16 — user-auth-service#115 (endpoint check) + doc Gateway. |
| V05 | Đúng (Message chưa code). | Đã sửa: thêm `messages.version` (optimistic lock), trả `version` trong history/send, lỗi `409 MESSAGE_VERSION_CONFLICT` theo mẫu `PRODUCT_VERSION_CONFLICT`. |
| K01–K04 | Đúng (Order chưa code). Notification chỉ lấy key top-level có trong placeholder. | Đã sửa doc Order: thêm `order_code`, `display_name`, `amount`, `invoice_url` top-level theo template Notification; bảng placeholder ở Notification LLD §6.2. |
| K05 | Đúng. Shipment đã code, không có `order_code`. | C15 — shipment-service#12. |
| A01 | Đúng. Code Inventory dùng `ApiResponse {data, meta}`. | Đã sửa mock LLD và ví dụ API về envelope. |
| A02 | Đúng. | Đã sửa golden example: `FREESHIPMAX` dùng `discount_type=FREESHIP`; công thức Bước 3 đổi nhãn `FIXED` → `FREESHIP`. |
| A03 | Đúng. | Đã thêm `FREESHIP` vào filter `scope` của `GET /vouchers`. |
| A04 | Đúng. | Đã viết lại: tối đa 1 mã mỗi slot, độ dài tối đa `2 + số shop`. |
| A05 | Đúng. Payment (đã code) bắt `amount > 0` và `shipping_fee < amount` cho mỗi đơn con. | Đã sửa checkout LLD: giữ tối thiểu 1 VND tiền hàng mỗi đơn con, `orders[].shipping_fee` = phí ship sau freeship. |
| A06 | Đúng. Code `SigninService` cho login phone chưa verify. | Đã sửa doc API/test theo code; mã lỗi giữ ⏳. |

## 10. Tách Order-Commerce thành Cart / Checkout / Order (2026-11-01)

Order-Commerce (chưa có code) được tách thành `cart-service` (`cart-docs`), `checkout-service` gồm voucher (`checkout-docs`) và `order-service` gồm invoice (`order-docs`). Đối chiếu lại với code: `payment-wallet-service@main` (`859b7b1`), `shipment-service@develop` (`7103c0a`), `inventory-service@develop`, `notification-service@main` (`6c6d9a2`). Không service đã có code nào phải sửa.

| # | Điểm kiểm | Kết luận | Xử lý trong doc |
|---|---|---|---|
| S01 | Payment-Wallet chỉ cho một internal caller `InternalService.ORDER_COMMERCE` gọi `POST /payments` và refund (`InternalCallerPolicy`). | Nếu Checkout gọi Payment thì Payment phải sửa code. | Mọi lời gọi Payment đặt ở `order-service` (tạo payment ngay khi nhận `POST /internal/orders`); `order-service` mang identity `ORDER_COMMERCE`. |
| S02 | Topic `order.events.v1`, `invoice.events.v1`, tên event, field top-level cho Notification. | Notification/Shipment không ràng buộc tên producer. | Giữ nguyên, `order-service` là producer. |
| S03 | Shipment `CreateShipmentRequest` bắt buộc `buyer_email` (`@Email`), `cod_amount`, `shipping_fee`, `currency`… — doc Order cũ chỉ nhắc `carrier`, `order_code`. | Order thiếu field sẽ bị `400`. | `order-docs` api §3.4 thêm bảng map đủ field; `orders.buyer_email` NOT NULL. |
| S04 | Gateway route `GET /seller/orders/{id}/shipment/carriers` → `shipment`, nhưng Shipment ghi đây là façade của Order và không có route này; doc Order cũ cũng thiếu endpoint. | Seller gọi carrier list sẽ `404`. | Gateway route sang `order-service`; thêm endpoint #9 vào `order-docs` api §3.6 (shape theo `QuoteService.compare`). |
| S05 | Shipment ghi Order chỉ cho xem carrier khi đơn `CONFIRMED`, nhưng `SHIP` yêu cầu `PROCESSING`. | Seller đã accept không xem được carrier. | Façade nhận `CONFIRMED` hoặc `PROCESSING`; sửa câu ở Shipment doc. |
| S06 | C15 (`order_code`) đã có trong code Shipment (`ShipmentCommandService` phát `payload.order_code`). | Bảng §3 còn ghi ⚠️. | Cập nhật ✅. |
| S07 | Doc DB Order cũ: `orders.discount = Σ order_vouchers.discount_amount`, nhưng API định nghĩa `discount` không gồm freeship. | Bất biến sai với đơn có freeship. | Chốt hai bất biến riêng (`order-docs` db §3.3). |
| S08 | Voucher validate thiếu lý do `ALREADY_USED` mà list voucher dùng. | UI phải map hai bộ mã. | Thêm vào validate. |
| S09 | `voucher_redemptions` có hai unique mâu thuẫn. | — | Chốt unique `(voucher_id, checkout_group_id)` + giới hạn mỗi buyer ở service. |
| S10 | Gateway LLD ghi auth-user đã có `/health/live`; code chưa có (C10). | Mâu thuẫn trong cùng file. | Sửa câu. |
| S11 | Payment-Wallet chưa có API đổi `amount` của payment COD sau khi huỷ một phần làm đổi `grand_total`. | Shipper thu đúng số mới (qua `cod_amount`), nhưng đối soát Payment lệch. | **Mở** — `order-docs` lld §8 mục 6, cần Payment owner quyết định. |
| S12 | Địa chỉ lấy hàng của shop (`from_address_snapshot`) chưa có contract đọc từ auth-user. | Không tạo được shipment/carrier quote thật. | **Mở** — `order-docs` api §5 mục 3. |
| S13 | Error code: `ORDER_CART_*`/`ORDER_VOUCHER_*`… đổi sang `CART_*`/`CHECKOUT_*`. Không service có code nào đọc các mã này; Gateway doc cập nhật `CHECKOUT_DEPENDENCY_UNAVAILABLE`. | — | Đã đổi. |

### 10.1 Recheck chi tiết sau khi tách (2026-11-01)

Một lượt rà contract chi tiết sau PR tách service tìm thêm 12 điểm (ký hiệu R để không trùng S11/S12 ở trên), lượt rà lại sau đó tìm thêm 3 điểm (F01–F03). Tất cả đã sửa trong docs cart/checkout/order; không đổi contract của service đã có code.

| # | Điểm kiểm | Xử lý |
|---|---|---|
| R01 | Một reservation cho cả group: huỷ một đơn con nhả luôn kho đơn khác. | Checkout reserve **mỗi đơn con một reservation** (`order_intent_id = order_id`); `orders[].reservation_id` trong `POST /internal/orders`. |
| R02 | Recalculate chỉ có tổng đơn, không tính lại thuế theo dòng và không trả phân bổ theo dòng. | Bảng `checkout_group_items`; `order_item_id` do Checkout sinh; response recalculate có `orders[].items[]`. |
| R03 | Reconcile chỉ đọc `GET /internal/orders`, không dựng lại được response checkout. | Reconcile gọi lại `POST /internal/orders` cùng key; buyer retry khi `RECONCILING` → `409 CHECKOUT_IN_PROGRESS`. |
| R04 | `code` chỉ unique theo `(scope, shop_id)` nhưng checkout nhận mã. | Checkout nhận `voucher_ids`; validate trả `voucher_id`, khớp nhiều → `AMBIGUOUS_CODE` + `candidates[]`. |
| R05 | Voucher cần đọc cả giỏ nhưng cart internal bắt buộc `item_ids`. | `item_ids` tuỳ chọn ở cart internal, validate và list voucher. |
| R06 | Batch `GET /products?product_ids=` không có `skus[]` (đã kiểm code Catalog). | Cart và Checkout dùng `GET /products/{productId}` (`checkout-docs` lld §6.4). |
| R07 | Bất biến `grand_total` bỏ sót `absorbed_amount`. | Thêm `− absorbed_amount` ở order DB và checkout DB. |
| R08 | `POST /cart/items` cộng dồn nhưng ghi là idempotent. | `Idempotency-Key` tuỳ chọn, `CART_IDEMPOTENCY_CONFLICT`. |
| R09 | `checked-out` hứa trả cùng kết quả nhưng chỉ lưu group gần nhất. | Bỏ lời hứa; xoá theo `item_id` vốn an toàn khi gọi lại. |
| R10 | `CO-CALC-06` còn giảm 200.000. | Sửa thành 199.999 theo trần `subtotal_s − 1`. |
| R11 | `status=EXPIRED` mâu thuẫn quy tắc chỉ trả `ACTIVE`. | Định nghĩa nhánh `EXPIRED` (30 ngày gần nhất, `eligible=false`). |
| R12 | Enum `DiscountType` nói `PERCENT` chỉ áp tiền hàng, DB cho `FREESHIP` + `PERCENT`. | Một bảng tổ hợp hợp lệ trong `lld/voucher.md` §5. |
| F01 | Reconcile gửi lại "cùng body" nhưng DB Checkout không lưu đủ field để dựng lại (email, địa chỉ, snapshot item, hạn reservation). | Lưu nguyên body vào `checkout_groups.order_create_request` trước lần gọi Order đầu tiên; reconcile gửi lại đúng body đó. |
| F02 | Cart hứa giữ `Idempotency-Key` 24h nhưng chỉ lưu 20 key mỗi giỏ. | Collection riêng `cart_idempotency_keys` (TTL 24h, không giới hạn số key), ghi cùng transaction với giỏ. |
| F03 | Test plan Cart và ghi chú Catalog còn nói Cart dùng batch. | Đổi sang `GET /products/{productId}`, thêm case SKU vắng/timeout/cache. |
