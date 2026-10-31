# API Spec — Order-Commerce Service

> Nguồn: `docs/lld/order-commerce.md` · `docs/db/order-commerce.md` · HLD/Penpot · Cập nhật: `2026-10-31`
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

Tài liệu này là **tổng quan** API: quy ước chung, danh sách endpoint, mã lỗi. Chi tiết request/response tách theo nghiệp vụ:

| Tài liệu | Endpoint |
|---|---|
| `docs/api/cart.md` | #1–4 |
| `docs/api/checkout.md` | #6–8 |
| `docs/api/order.md` | #9–18 |
| `docs/api/voucher.md` | #5, 5a, 5b, 19–26 |

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Cart/checkout/order cần JWT; buyer/seller scope kiểm tra tại service. |
| Request ID | `X-Request-ID` tối đa 64; Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate`; propagate sang Product/Inventory/Payment/Shipment và Kafka (do Gateway/service propagate, client không gửi). |
| Actor context | Đọc `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope` do Gateway inject (client không gửi được — Gateway strip). |
| Timestamp | ISO-8601 UTC; DB UTC `DATETIME(6)`. |
| Money | Integer VND, `currency=VND`; không float. |
| Pagination | `page` từ 1, `size` mặc định 20, max 100; meta trả `request_id,page,size,total,total_pages`. |
| Idempotency | `Idempotency-Key` bắt buộc với checkout/cancel/fulfill/voucher mutation; giữ 24h. |
| Response | `{data,meta:{request_id}}`; error `{error:{code,message,details,trace_id}}`. |
| Log | JSON field chuẩn auth-user/Gateway; không log token/payment credential/full address/order payload. |
| Retry | Gateway không retry mutation; service dùng saga/idempotency. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /cart` | Buyer | Cart hiện tại. |
| 2 | `POST /cart/items` | Buyer | Thêm SKU. |
| 3 | `PUT /cart/items/{itemId}` | Buyer | Đổi quantity. |
| 4 | `DELETE /cart/items/{itemId}` | Buyer | Xóa item. |
| 5 | `GET /vouchers/validate` | Buyer | Preview voucher trên cart (buyer đã biết mã). |
| 5a | `GET /vouchers` | **Public (auth tuỳ chọn)** | Danh sách voucher khả dụng (PDP/Shop voucher strip cho khách chưa đăng nhập); có JWT thì service tính thêm `eligible`. |
| 5b | `GET /vouchers/redemptions` | Buyer | Lịch sử voucher đã dùng + tổng tiền đã tiết kiệm. |
| 6 | `GET /checkout/shipping-fee` | Buyer | Preview shipping fee. |
| 7 | `POST /checkout/preview` | Buyer | Tính preview tổng tiền. |
| 8 | `POST /checkout` | Buyer + Idempotency | Reserve Inventory và tạo order split. |
| 9 | `GET /orders` | Buyer | Own order list. |
| 10 | `GET /orders/{orderId}` | Buyer | Own order detail. |
| 11 | `GET /orders/{orderId}/status` | Buyer | Status timeline/projection. |
| 12 | `PATCH /orders/{orderId}/cancel` | Buyer | Cancel trước shipping. |
| 13 | `GET /orders/{orderId}/invoice` | Buyer | Invoice metadata/download URL. |
| 14 | `GET /seller/orders` | Seller/staff | Shop order list. |
| 14a | `GET /seller/orders/export` | Seller/staff | Xuất danh sách đơn ra file (.xlsx/.csv). |
| 15 | `GET /seller/orders/{orderId}` | Seller/staff | Shop order detail. |
| 16 | `PATCH /seller/orders/{orderId}/fulfill` | Seller/staff | Accept/ship flow (1 đơn). |
| 16a | `POST /seller/orders/bulk-accept` | Seller/staff | Accept hàng loạt (chỉ ACCEPT, không ship). |
| 17 | `POST /seller/orders/{orderId}/cancel` | Seller/staff | Seller cancel theo policy. |
| 18 | `GET /seller/orders/{orderId}/invoice` | Seller/staff | Invoice. |
| 19 | `POST /seller/vouchers` | Seller | Tạo shop voucher. |
| 20 | `GET /seller/vouchers` | Seller | List voucher. |
| 21 | `PUT /seller/vouchers/{voucherId}` | Seller | Update voucher. |
| 22 | `DELETE /seller/vouchers/{voucherId}` | Seller | Soft archive/inactivate. |
| 23 | `POST /admin/vouchers` | Admin (`VOUCHER_MANAGE`) | Tạo platform voucher (`scope=PLATFORM`). |
| 24 | `GET /admin/vouchers` | Admin | List platform voucher + usage metric. |
| 25 | `PUT /admin/vouchers/{voucherId}` | Admin | Update platform voucher. |
| 26 | `DELETE /admin/vouchers/{voucherId}` | Admin | Soft archive/inactivate. |
| 27 | `GET /health/live` | Internal/ops | Liveness process-only (Gateway active healthcheck gọi — bắt buộc). |
| 28 | `GET /health/ready` | Internal/ops | Readiness: MySQL/Redis/Kafka theo policy (khớp `docs/lld/order-commerce.md` §2.2). |

## 3. Chi tiết endpoint

Chi tiết từng endpoint nằm ở file nghiệp vụ (bảng đầu tài liệu). `GET /health/live` và `GET /health/ready` theo chuẩn observability chung ở `docs/lld/order-commerce.md` §2.2.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `ORDER_INVALID_INPUT` | 400 | Payload/query sai. |
| `ORDER_UNAUTHENTICATED` | 401 | Thiếu login. |
| `ORDER_FORBIDDEN` | 403 | Sai buyer/shop scope. |
| `ORDER_CART_EMPTY` | 409 | Không có item được chọn. |
| `ORDER_CART_ITEM_NOT_FOUND` | 404 | Item không tồn tại trong cart (anti-enumeration). |
| `ORDER_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active. |
| `ORDER_PRICE_CHANGED` | 409 | Giá snapshot cũ. |
| `ORDER_STOCK_UNAVAILABLE` | 409 | Inventory reserve fail. |
| `ORDER_RESERVATION_EXPIRED` | 409 | Reservation hết TTL. |
| `ORDER_VOUCHER_INVALID` | 400 | Voucher invalid/scope/time. |
| `ORDER_VOUCHER_SLOT_CONFLICT` | 400 | Gửi 2 mã cùng `scope` trong `voucher_codes`. |
| `ORDER_VOUCHER_CODE_EXISTS` | 409 | Trùng `code` trong cùng shop/platform. |
| `ORDER_VOUCHER_CODE_LOCKED` | 409 | Đổi `code` khi voucher đã có redemption (`used_count > 0`). |
| `ORDER_VOUCHER_LIMIT_REACHED` | 409 | Hết usage. |
| `ORDER_EXPORT_TOO_LARGE` | 400 | Vượt `ORDER_EXPORT_MAX_ROWS` (10.000), cần lọc hẹp hơn. |
| `ORDER_ADDRESS_INVALID` | 400 | Address không thuộc user/sai. |
| `ORDER_STATE_INVALID` | 409 | State transition sai. |
| `ORDER_CANCEL_NOT_ALLOWED` | 409 | Không còn được cancel. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Key khác request hash. |
| `ORDER_DEPENDENCY_UNAVAILABLE` | 503 | Product/Inventory/Payment/Shipment down. |
| `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` | 400 | Pass-through nguyên trạng từ Shipment (`docs/api/order.md` §3.4) — không đổi tên/status. |
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice pending. |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

> Mã lỗi của service dependency (Shipment, Inventory, Payment-Wallet) được pass-through nguyên trạng khi Order-Commerce đứng giữa — không đổi tên/status; ví dụ `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` (`docs/api/order.md` §3.4).

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | `POST /checkout` tạo order sau Inventory reserve, Payment charge tách service. | Ảnh hưởng saga/compensation/payment UX. | Order/Payment/Inventory owner |
| 2 | Multi-shop checkout split order theo shop nhưng chung `checkout_group_id`. | Ảnh hưởng invoice/shipment/voucher. | Product owner |
| 3 | Shipping fee endpoint dùng Shipment mock/quote; exact carrier contract chưa có. | Ảnh hưởng total/timeout. | Shipment owner |
| 4 | ~~Voucher stacking/priority chưa chốt~~ → **Đã chốt**: 3 slot (`PLATFORM`/`SHOP`/`FREESHIP`), tối đa 1 mã mỗi slot, thứ tự shop→platform→freeship. Công thức đầy đủ + phân bổ + làm tròn ở `docs/lld/checkout.md` §2.2. | — | Đã đóng |
| 5 | Seller cancel/refund policy chưa đủ trong HLD. | Ảnh hưởng inventory/payment compensation. | Product/Finance |
| 6 | Voucher v1 chỉ theo mã, không audience/targeting; các màn Penpot Seller voucher targeting ngoài v1. | Nếu bật targeting cần schema `voucher_audiences` + follow shop. | Product owner |
| 7 | Platform voucher CRUD đặt tại `order-commerce` (`/admin/vouchers`, `VOUCHER_MANAGE`). | Nếu phạm vi Admin tách service riêng, service đó gọi vào contract này chứ không sở hữu voucher aggregate. | Architecture |
