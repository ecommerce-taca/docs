# LLD — Order-Commerce Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 · Redis Cart · Kafka outbox · REST với Product/Inventory/Payment/Shipment/Auth

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Cart, checkout orchestration, order split theo shop, price/address snapshot, voucher, invoice metadata, buyer/seller order read và cancel/fulfill command. |
| Database | MySQL 8.4 `orderdb`; Redis chỉ lưu cart ephemeral/TTL và không là order source of truth. |
| Payment | Payment-Wallet sở hữu payment transaction, VNPAY/COD reconciliation, wallet/commission; Order chỉ lưu payment method/status projection. |
| Inventory | Inventory sở hữu availability/reserve/commit/release/deduct; Order gọi contract và không tự trừ stock. |
| Shipment | Shipment sở hữu carrier/tracking/ship status; Order lưu shipment snapshot/status projection. |
| Product | Product Catalog cung cấp content/giá hiện tại; Order snapshot title/sku/shop/unit price tại checkout. |
| Không thuộc service | Auth/KYC decision, product CRUD, inventory ledger, payment gateway, shipping carrier, rating/message/notification delivery. |

### 1.2 Boundary

```text
mfe-buyer/mfe-seller ──Gateway──► Order-Commerce
                              ├─ MySQL orderdb: orders/items/vouchers/invoices
                              ├─ Redis: cart:{user_id}, TTL
                              ├─ REST Product: validate price/product/sku
                              ├─ REST Inventory: availability/reserve/commit/release
                              ├─ REST Payment: create/reconcile payment
                              └─ Kafka outbox → Notification/Shipment/Payment/Analytics
```

- Order snapshot là bất biến sau place trừ workflow correction/audit; không đọc giá Product để thay đổi đơn cũ.
- Một checkout nhiều shop tạo nhiều order con, liên kết bằng `checkout_group_id`.
- Idempotency bắt buộc cho checkout/cancel/fulfill/voucher redemption; retry không tạo duplicate order.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Order | Cart + Checkout + Invoice + Voucher | Gom trong Order-Commerce. |
| Buyer Cart | Get/add/update/delete item | Redis cart + product validation khi add/checkout. |
| Buyer Checkout | Address, shipping fee, payment method, voucher | Preview rồi place; address snapshot từ Auth User; shipping fee qua Shipment mock/contract. |
| Buyer Orders | List/detail/status/cancel/invoice | Buyer scope từ JWT; cancel trước shipping theo state machine. |
| Seller Orders | List/detail/fulfill/cancel/invoice | Shop scope; seller không đổi buyer/payment/inventory ledger trực tiếp. |
| Payment | VNPAY QR hoặc COD | Payment-Wallet owns provider; Order phát pending payment/paid events. |

## 2. Cấu trúc bên trong

### 2.1 Module Spring Boot

```text
com.taca.order
├── cart/                 # Redis cart command/query
├── checkout/             # preview, price/voucher/shipping validation
├── order/                # aggregate, state machine, buyer/seller query
├── voucher/              # platform/shop voucher and redemption
├── invoice/              # invoice number/metadata/PDF reference
├── integrations/         # Product, Inventory, Payment, Shipment, Auth clients
├── outbox/               # transactional event publisher
├── security/             # actor/shop ownership and idempotency
└── observability/        # shared JSON log/trace/metric contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `CartController` | Cart CRUD | User ID từ auth context; Redis TTL; không tin user_id body. |
| `CheckoutController` | Preview/shipping fee/place | Place idempotency; reserve Inventory trước khi commit order. |
| `OrderController` | Buyer list/detail/status/cancel/invoice | Chỉ own orders; snapshot immutable. |
| `SellerOrderController` | Shop order list/detail/fulfill/cancel/invoice | Filter theo shop; không expose other shop data. |
| `VoucherService` | Validate/apply/redeem platform/shop voucher | Atomic usage limit; unique redemption theo voucher+user/order policy. |
| `OrderStateMachine` | Pending/paid/processing/shipped/delivered/cancelled | Không cho seller/buyer bypass transition. |
| `InventoryClient` | Reserve/commit/release | Timeout/failure không đánh dấu order paid; idempotency required. |
| `OutboxPublisher` | Publish order/invoice/voucher events | Transactional outbox, retry 3/backoff 2s/DLQ. |

### 2.2 Chuẩn observability dùng chung

- Log JSON stdout với field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST propagate `traceparent`/`tracestate` và `X-Request-ID`; Kafka headers giữ `traceparent`, `request_id`, `event_id`.
- Không log Authorization, refresh token, password, full address/phone/email, voucher raw secret, payment credential, VNPAY signature hoặc full cart/order payload.
- Chỉ log `order_id`, `checkout_group_id`, `shop_id`, `sku_id` theo allowlist; metric label không dùng `user_id/order_id`.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MySQL, Redis, Kafka và dependency policy.
- API error envelope `{error:{code,message,details,trace_id}}`; mọi error/audit có request/trace correlation.

## 3. Luồng xử lý

### 3.1 Cart

1. Authenticated user gọi get/add/update/delete cart.
2. Cart lưu Redis key `cart:{user_id}`, item gồm `product_id`, `sku_id`, `shop_id`, quantity và optional price display cache.
3. Add/update validate quantity >0 và Product/Inventory availability snapshot; không coi cart là reservation.
4. Giá/tiêu đề trong cart chỉ là cache hiển thị; Checkout phải re-read Product và tạo price snapshot.
5. Cart expired/empty không tạo order; mfe-buyer nhận `ORDER_CART_EMPTY`.

### 3.2 Checkout preview

```text
Cart → load selected items → Product validate active SKU/price
     → Inventory availability check (read-only)
     → validate voucher/usage/time/scope
     → calculate subtotal/discount/tax/shipping/total
     → return preview with warnings; không reserve, không charge
```

Preview không đảm bảo giá/stock đến lúc place; response ghi `expires_at` ngắn và Checkout place phải tính lại.

### 3.3 Place checkout

1. Validate JWT, cart ownership, idempotency key và selected items.
2. Re-read Product fields và giá; snapshot `title`, `sku`, `shop`, `unit_price`, `currency=VND`.
3. Validate/redeem voucher trong transaction hoặc reservation protocol; không vượt usage limit.
4. Gọi Inventory `reserve` với `reservation_id`, `order_intent_id`, item SKU/qty và TTL.
5. Nếu reserve fail: rollback order intent/voucher hold, trả `ORDER_STOCK_UNAVAILABLE`.
6. Tạo orders split theo shop + order_items + address snapshot + invoice pending + outbox `order.created` trong MySQL transaction.
7. Với VNPAY: tạo payment intent qua Payment-Wallet sau order commit/idempotency; với COD: trạng thái payment `PENDING_COD` theo contract.
8. Trả `order_id(s)`, `checkout_group_id`, payment next action và reservation expiry.

Không dùng distributed transaction giữa MySQL/Inventory/Payment; dùng idempotency, saga compensation và event reconciliation.

### 3.4 Order lifecycle

| Transition | Actor | Điều kiện |
|---|---|---|
| `PENDING_PAYMENT → PAID` | Payment event | Payment-Wallet xác nhận thành công; commit reservation. |
| `PENDING_PAYMENT → CANCELLED` | System/buyer timeout | Release reservation; payment chưa success hoặc refund policy. |
| `PAID → PROCESSING` | Seller/system | Order payment confirmed, seller accepts. |
| `PROCESSING → SHIPPED` | Seller/Shipment | Shipment created/tracking valid. |
| `SHIPPED → DELIVERED` | Shipment event | Carrier confirms delivered. |
| `PENDING_PAYMENT/PAID/PROCESSING → CANCELLED` | Buyer/seller/system theo policy | Buyer chỉ cancel trước shipping; release/refund compensation. |
| `CANCELLED/DELIVERED` | Không reopen | Tạo refund/return workflow ở Payment/Support nếu cần. |

### 3.5 Voucher và invoice

- Voucher scope `PLATFORM` hoặc `SHOP`; một order item/shop phải kiểm scope trước split.
- Voucher v1 chỉ áp theo **mã** (`code`) + điều kiện `min_order`/thời gian/`usage_limit`; ràng buộc `PERCENT` 1–100 hoặc `FIXED` VND. Không có audience/targeting (selected users, segment, VIP, followers, import user IDs). Các màn Penpot Seller "Voucher audience / Selected users / Tab followers / Import user IDs / Create targeted voucher" là **ngoài phạm vi v1**; Penpot phần này cần đơn giản hoá về form tạo voucher theo mã. Nếu bật targeting sau này cần bảng `voucher_audiences` + quan hệ follow shop (chưa có trong v1).
- Platform voucher (`scope=PLATFORM`) do admin tạo/sửa qua `order-commerce` (endpoint admin); seller chỉ tạo `scope=SHOP`.
- Usage increment/redemption idempotent; không giảm `used_count` nếu order không tạo được hoặc đã release theo policy.
- Invoice có số unique, total/tax breakdown snapshot và `pdf_url` nullable; PDF generation/notification có thể async.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `PAGE_SIZE_DEFAULT` | 20 | Max 100. |
| `CART_TTL` | 30 ngày | Redis; cart không reserve stock. |
| `CART_MAX_ITEMS` | 100 | Item line, không phải quantity. |
| `ITEM_QTY_MIN` | 1 | Integer. |
| `ITEM_QTY_MAX` | 999 | Per SKU/order line. |
| `CHECKOUT_PREVIEW_TTL` | 5 phút | Chỉ warning/UX, không giữ stock. |
| `INVENTORY_RESERVATION_TTL` | 15 phút | Exact TTL phải align Inventory. |
| `ORDER_NUMBER_MAX_LENGTH` | 32 | Unique. |
| `VND_MIN/MAX` | 1/999999999999 | Integer; snapshot không float. |
| `OUTBOX_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `INTEGRATION_TIMEOUT` | 5s | Checkout dependency baseline; Payment/Shipment có route override. |
| `IDEMPOTENCY_RETENTION` | 24h | Place/cancel/fulfill response snapshot. |
| `AUDIT_RETENTION` | 365 ngày | Không lưu payment credential/raw address ngoài snapshot nghiệp vụ cần thiết. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | DB DATETIME(6), API/event UTC. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `CartStatus` | `ACTIVE`, `EXPIRED`, `CHECKED_OUT`. |
| `OrderStatus` | `PENDING_PAYMENT`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. |
| `PaymentStatusProjection` | `PENDING`, `PENDING_COD`, `SUCCESS`, `FAILED`, `REFUNDED`. |
| `VoucherStatus` | `DRAFT`, `ACTIVE`, `INACTIVE`, `EXPIRED`, `ARCHIVED`. |
| `DiscountType` | `PERCENT`, `FIXED`. |
| `InvoiceStatus` | `PENDING`, `ISSUED`, `VOIDED`. |
| `CancellationReason` | `BUYER_REQUEST`, `SELLER_REQUEST`, `PAYMENT_TIMEOUT`, `STOCK_FAILURE`, `SYSTEM`. |

Order state là source of truth của Order-Commerce; payment/shipment/inventory statuses được projection và không cho client tự set.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `order.events.v1` | `order.created` | order/group, buyer, shop, items snapshot, total, payment method |
| `order.events.v1` | `order.paid` | order, payment reference, paid_at |
| `order.events.v1` | `order.processing` | order, shop |
| `order.events.v1` | `order.shipped` | order, shipment reference |
| `order.events.v1` | `order.delivered` | order, delivered_at |
| `order.events.v1` | `order.cancelled` | order, reason, compensation action |
| `invoice.events.v1` | `invoice.issued` | invoice/order/number/total/pdf reference |
| `voucher.events.v1` | `voucher.redeemed/released` | voucher/order/user/scope |

### 6.2 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Payment-Wallet | `payment.succeeded`, `payment.failed`, `payment.refunded` | Update payment projection, transition order, commit/release Inventory. |
| Inventory | `inventory.reservation.confirmed/rejected/expired`, `inventory.stock_committed` | Reconcile order intent/reservation; không tự sửa quantity. |
| Shipment | `shipment.created`, `shipment.status_changed` | Update shipment projection/order status. |
| Auth User | `user.status_changed` (mock policy) | Chặn hành động account mới nếu suspended; không đổi order history. |

### 6.3 Mock contract — Inventory reserve request

```json
{
  "request_id": "01912f80-7a1b-7c12-9c55-8b1c34a6d921",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "idempotency_key": "checkout-intent-01912f81",
  "order_intent_id": "intent-01912f81",
  "expires_at": "2026-08-30T09:15:00Z",
  "items": [{"sku_id": "sku-01912f33", "product_id": "product-01912f31", "quantity": 2}]
}
```

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `ORDER_INVALID_INPUT` | 400 | Payload/query sai. |
| `ORDER_UNAUTHENTICATED` | 401 | Cart/checkout/order cần login. |
| `ORDER_FORBIDDEN` | 403 | Không thuộc buyer/shop scope. |
| `ORDER_CART_EMPTY` | 409 | Cart không có selected item. |
| `ORDER_CART_ITEM_NOT_FOUND` | 404 | Item không tồn tại trong cart. |
| `ORDER_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active. |
| `ORDER_PRICE_CHANGED` | 409 | Giá hiện tại khác preview/cart; cần xác nhận lại. |
| `ORDER_STOCK_UNAVAILABLE` | 409 | Inventory reserve thất bại/quantity không đủ. |
| `ORDER_RESERVATION_EXPIRED` | 409 | Reservation hết TTL. |
| `ORDER_VOUCHER_INVALID` | 400 | Code/scope/time/value không hợp lệ. |
| `ORDER_VOUCHER_LIMIT_REACHED` | 409 | Hết usage/user limit. |
| `ORDER_ADDRESS_INVALID` | 400 | Address snapshot không hợp lệ. |
| `ORDER_STATE_INVALID` | 409 | Transition không hợp lệ. |
| `ORDER_CANCEL_NOT_ALLOWED` | 409 | Đã shipping hoặc policy không cho cancel. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Key dùng lại với payload khác. |
| `ORDER_DEPENDENCY_UNAVAILABLE` | 503 | Product/Inventory/Payment/Shipment timeout/down. |
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice chưa issued. |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Database Order là MySQL 8.4 và backend Spring Boot/Java 25 theo lựa chọn A. | Ảnh hưởng transaction, migration và deployment BOM. | Tech lead |
| 2 | Cart nằm Redis; Order/Invoice/Voucher nằm MySQL. | Nếu muốn cart durable, cần thêm schema và recovery policy. | Architecture |
| 3 | Checkout tạo order sau khi Inventory reserve; không charge/payment trong cùng distributed transaction. | Ảnh hưởng compensation, timeout và payment reconciliation. | Order/Payment/Inventory owner |
| 4 | Một checkout multi-shop split thành order con theo shop. | Ảnh hưởng voucher scope, shipment, seller view và invoice grouping. | Product owner |
| 5 | Shipping fee là mock/integration với Shipment; exact carrier calculation chưa có HLD contract. | Cần thêm quote contract/SLA trước production. | Shipment owner |
| 6 | Payment service tên `payment-wallet`; HLD có chỗ gọi Payment Service. | Cần chuẩn hóa topic/API service name ở platform registry. | Architecture |
| 7 | Buyer cancel được trước shipping; seller cancel policy cần business rule chi tiết. Return/dispute workflow **ngoài v1** — service `dispute` (v1.1) điều phối; nếu cần sớm thì làm module trong `order-commerce` (`System_Overview.md` §6.3). | Ảnh hưởng refund/stock release và dispute. | Product/Finance |
| 8 | Tax calculation baseline là snapshot/tax breakdown từ Order; tax authority/provider chưa chốt. | Ảnh hưởng invoice compliance. | Finance owner |
| 9 | Voucher v1 chỉ theo mã + scope PLATFORM/SHOP + `min_order`/thời gian/`usage_limit`; **không có** audience/targeting/segment/import user IDs. Các màn Penpot Seller voucher targeting là ngoài v1. | Nếu bật targeting cần bảng `voucher_audiences` + quan hệ follow shop; đổi checkout redemption. | Product owner |
| 10 | Đã chốt (`System_Overview.md` §6.3): v1 **không** tách microservice admin. Platform voucher (`scope=PLATFORM`) do `order-commerce` sở hữu; admin console gọi trực tiếp `/api/v1/admin/vouchers/**` của service này (`VOUCHER_MANAGE` + 2FA), Gateway chỉ coarse-gate role admin. | Nếu voucher aggregate chuyển service khác phải đổi event/redemption. | Product owner + Architecture |
