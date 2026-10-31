# LLD — Order-Commerce Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-10-31`
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 · Redis Cart · Kafka outbox · REST với Product/Inventory/Payment/Shipment/Auth

Tài liệu này là **tổng quan** của Order-Commerce. Chi tiết từng nghiệp vụ tách ra các file cùng thư mục; mỗi file cùng tên ở `docs/api/`, `docs/db/`, `docs/test/` là phần API/DB/test tương ứng.

| Tài liệu | Phạm vi |
|---|---|
| `order-commerce.md` | Tổng quan: ranh giới service, cấu trúc module, quy ước chung, mã lỗi, giả định. |
| `cart.md` | Giỏ hàng Redis. |
| `checkout.md` | Shipping fee, preview, công thức tính tiền, place checkout, contract Inventory/thuế. |
| `order.md` | Vòng đời order, buyer/seller order, fulfill, invoice, event order/invoice. |
| `voucher.md` | Voucher buyer/seller/admin, mô hình 3 slot, huỷ một phần. |

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

| Luồng | Tài liệu |
|---|---|
| Cart | `docs/lld/cart.md` §2 |
| Checkout preview | `docs/lld/checkout.md` §2.1 |
| Công thức tính tiền *(nguồn sự thật)* | `docs/lld/checkout.md` §2.2 |
| Place checkout (VNPAY/COD) | `docs/lld/checkout.md` §2.3 |
| Order lifecycle | `docs/lld/order.md` §2.1 |
| Invoice | `docs/lld/order.md` §2.2 |
| Voucher discovery/validate | `docs/lld/voucher.md` §2.1 |
| Cộng dồn voucher — mô hình 3 slot | `docs/lld/voucher.md` §2.2 |
| Huỷ một phần nhóm đơn — tính lại voucher | `docs/lld/voucher.md` §2.3 |

Không dùng distributed transaction giữa MySQL/Inventory/Payment; dùng idempotency, saga compensation và event reconciliation.

## 4. Hằng số & cấu hình dùng chung

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `PAGE_SIZE_DEFAULT` | 20 | Max 100. |
| `VND_MIN/MAX` | 1/999999999999 | Integer; snapshot không float. |
| `OUTBOX_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `INTEGRATION_TIMEOUT` | 5s | Checkout dependency baseline; Payment/Shipment có route override. |
| `IDEMPOTENCY_RETENTION` | 24h | Place/cancel/fulfill response snapshot. |
| `AUDIT_RETENTION` | 365 ngày | Không lưu payment credential/raw address ngoài snapshot nghiệp vụ cần thiết. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | DB DATETIME(6), API/event UTC. |

Hằng số riêng của từng nghiệp vụ nằm trong file tương ứng: cart (`CART_*`, `ITEM_QTY_*`) ở `cart.md` §3, checkout (`CHECKOUT_PREVIEW_TTL`, `INVENTORY_RESERVATION_TTL`) ở `checkout.md` §3, order (`ORDER_NUMBER_MAX_LENGTH`, `ORDER_EXPORT_*`, `SELLER_BULK_ACCEPT_MAX_ORDERS`) ở `order.md` §3.

## 5. Enum & trạng thái

| Enum | Định nghĩa tại |
|---|---|
| `CartStatus` | `docs/lld/cart.md` §4 |
| `OrderStatus`, `PaymentStatusProjection`, `InvoiceStatus`, `CancellationReason` | `docs/lld/order.md` §4 |
| `VoucherStatus`, `DiscountType`, `VoucherScope` | `docs/lld/voucher.md` §4 |

Order state là source of truth của Order-Commerce; payment/shipment/inventory statuses được projection và không cho client tự set.

## 6. Event phát ra / lắng nghe

| Topic / nguồn | Hướng | Chi tiết |
|---|---|---|
| `order.events.v1` | Phát ra | `docs/lld/order.md` §5.1, payload §5.2 |
| `invoice.events.v1` | Phát ra | `docs/lld/order.md` §5.1, payload §5.2 |
| `voucher.events.v1` | Phát ra | `docs/lld/voucher.md` §5 |
| Payment-Wallet / Inventory / Shipment / Auth User | Lắng nghe | `docs/lld/order.md` §5.3 |
| Inventory reserve (REST, đồng bộ) | Gọi ra | `docs/lld/checkout.md` §5.1 |
| Thuế suất từ Product Catalog (REST) | Gọi ra | `docs/lld/checkout.md` §5.2 |

Mọi event đi qua transactional outbox (`OUTBOX_RETRY_COUNT`, backoff 2s rồi DLQ — §4).

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
| `ORDER_VOUCHER_SLOT_CONFLICT` | 400 | Gửi 2 mã cùng `scope` trong `voucher_codes`. |
| `ORDER_VOUCHER_CODE_EXISTS` | 409 | Trùng `code` trong cùng shop/platform. |
| `ORDER_VOUCHER_CODE_LOCKED` | 409 | Đổi `code` khi voucher đã có redemption (`used_count > 0`). |
| `ORDER_EXPORT_TOO_LARGE` | 400 | Vượt `ORDER_EXPORT_MAX_ROWS`, seller cần lọc hẹp hơn. |
| `ORDER_VOUCHER_LIMIT_REACHED` | 409 | Hết usage/user limit. |
| `ORDER_ADDRESS_INVALID` | 400 | Address snapshot không hợp lệ. |
| `ORDER_STATE_INVALID` | 409 | Transition không hợp lệ. |
| `ORDER_CANCEL_NOT_ALLOWED` | 409 | Đã shipping hoặc policy không cho cancel. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Key dùng lại với payload khác. |
| `ORDER_DEPENDENCY_UNAVAILABLE` | 503 | Product/Inventory/Payment/Shipment timeout/down. |
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice chưa issued. |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |
| `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` | 400 | Pass-through nguyên trạng từ Shipment (`docs/api/order.md` §3.4). |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Database Order là MySQL 8.4 và backend Spring Boot/Java 25 theo lựa chọn A. | Ảnh hưởng transaction, migration và deployment BOM. | Tech lead |
| 2 | Cart nằm Redis; Order/Invoice/Voucher nằm MySQL. | Nếu muốn cart durable, cần thêm schema và recovery policy. | Architecture |
| 3 | Checkout tạo order sau khi Inventory reserve; không charge/payment trong cùng distributed transaction. | Ảnh hưởng compensation, timeout và payment reconciliation. | Order/Payment/Inventory owner |
| 4 | Một checkout multi-shop split thành order con theo shop. | Ảnh hưởng voucher scope, shipment, seller view và invoice grouping. | Product owner |
| 5 | Shipping fee là mock/integration với Shipment; exact carrier calculation chưa có HLD contract. | Cần thêm quote contract/SLA trước production. | Shipment owner |
| 6 | Payment service tên `payment-wallet`; HLD có chỗ gọi Payment Service. | Cần chuẩn hóa topic/API service name ở platform registry. | Architecture |
| 7 | Buyer cancel được trước shipping; seller cancel policy cần business rule chi tiết. Return/dispute workflow **ngoài v1** — service `dispute` (v1.1) điều phối; nếu cần sớm thì làm module trong `order-commerce` (quyết định đã chốt 2026-09-18). | Ảnh hưởng refund/stock release và dispute. | Product/Finance |
| 8 | Tax calculation baseline là snapshot/tax breakdown từ Order; tax authority/provider chưa chốt. | Ảnh hưởng invoice compliance. | Finance owner |
| 9 | Voucher v1 chỉ theo mã + scope PLATFORM/SHOP + `min_order`/thời gian/`usage_limit`; **không có** audience/targeting/segment/import user IDs. Các màn Penpot Seller voucher targeting là ngoài v1. | Nếu bật targeting cần bảng `voucher_audiences` + quan hệ follow shop; đổi checkout redemption. | Product owner |
| 10 | Đã chốt (2026-09-18): v1 **không** tách microservice admin. Platform voucher (`scope=PLATFORM`) do `order-commerce` sở hữu; admin console gọi trực tiếp `/api/v1/admin/vouchers/**` của service này (`VOUCHER_MANAGE` + 2FA), Gateway chỉ coarse-gate role admin. | Nếu voucher aggregate chuyển service khác phải đổi event/redemption. | Product owner + Architecture |
