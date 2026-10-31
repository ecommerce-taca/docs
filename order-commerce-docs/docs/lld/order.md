# LLD — Order-Commerce Service · Order

> Nguồn: `docs/api/order.md` · `docs/db/order.md` · `docs/test/order.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 · Redis Cart · Kafka outbox · REST với Product/Inventory/Payment/Shipment/Auth

Phần **Order** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/lld/order-commerce.md`.

## 1. Phạm vi

| Mục | Nội dung |
|---|---|
| Trách nhiệm | Order aggregate và state machine; buyer order read/cancel; seller order read/fulfill/bulk-accept/export/cancel; invoice metadata; event order/invoice; xử lý event từ Payment/Inventory/Shipment. |
| Module | `com.taca.order.order` — `OrderController`, `SellerOrderController`, `OrderStateMachine`; `com.taca.order.invoice`. |
| Ràng buộc | Buyer chỉ thấy own orders; seller filter theo shop scope; snapshot immutable; không client nào tự set payment/shipment/inventory status. |
| Tạo order | Order được tạo ở place checkout — `checkout.md` §3.3. |

## 2. Cấu trúc bên trong

Package `com.taca.order.order` và `com.taca.order.invoice` trong cây module ở `order-commerce.md` §2.1. ⏳ Repo chưa có code nên chưa có cây package thật.

## 3. Luồng xử lý

### 3.1 Order lifecycle

`OrderStatus` là trạng thái **fulfillment**, không phải trạng thái tiền. Trạng thái tiền nằm ở `PaymentStatusProjection` và do Payment-Wallet sở hữu. Hai phương thức thanh toán vào cùng một cửa fulfillment (`CONFIRMED`) nhưng thu tiền ở hai thời điểm khác nhau.

```text
VNPAY:  [checkout] → PENDING_PAYMENT --payment.succeeded--> CONFIRMED → PROCESSING → SHIPPED → DELIVERED
                            │                                                                      (tiền đã thu từ trước)
                            └──timeout──> CANCELLED

COD:    [checkout] ────────────────────────────────────────► CONFIRMED → PROCESSING → SHIPPED → DELIVERED
                                                                                                   │
                                                                          capture tiền mặt ────────┘
                                                                          → payment.succeeded → payment_status=SUCCESS
```

| Transition | Actor | Điều kiện |
|---|---|---|
| `→ PENDING_PAYMENT` | Checkout (VNPAY) | Order tạo với phương thức trả trước; reservation `RESERVED`, chưa commit. |
| `→ CONFIRMED` | Checkout (COD) | Order tạo với `method=COD`; commit reservation ngay; payment projection `PENDING_COD`. |
| `PENDING_PAYMENT → CONFIRMED` | Payment event | `payment.succeeded` từ Payment-Wallet; commit reservation; payment projection `SUCCESS`. |
| `PENDING_PAYMENT → CANCELLED` | System/buyer timeout hoặc `payment.expired` | **Chỉ áp dụng cho đơn trả trước.** Release reservation; `PAYMENT_TIMEOUT`. Đơn COD không bao giờ ở trạng thái này nên không bị job này chạm tới. |
| `CONFIRMED → PROCESSING` | Seller/system | Seller accept đơn (hàng đợi "Chờ xác nhận" gồm **cả** đơn VNPAY đã trả và đơn COD). |
| `PROCESSING → SHIPPED` | Seller/Shipment | Shipment created/tracking valid. |
| `SHIPPED → DELIVERED` | Shipment event (`shipment.delivered`) | Carrier confirms delivered. Với COD, đây là điểm kích hoạt capture tiền ở Payment-Wallet — **không** đổi `OrderStatus` thêm lần nữa. |
| `SHIPPED` (giữ nguyên) | Shipment event (`shipment.failed`) | Carrier báo giao thất bại: order giữ `SHIPPED`, ghi timeline; refund/return do Payment/Support workflow xử lý (không tự huỷ/hiển thị mới). |
| `CONFIRMED/PROCESSING → CANCELLED` | Buyer/seller/system theo policy | Buyer chỉ cancel trước shipping. VNPAY: release reservation đã commit + refund. COD: release/hoàn kho, không refund vì chưa thu tiền. |
| `CANCELLED/DELIVERED` | Không reopen | Tạo refund/return workflow ở Payment/Support nếu cần. |

Ràng buộc bổ sung:

- `PAID` **không còn là một `OrderStatus`**. Trước đây trạng thái này vừa mang nghĩa "đã thu tiền" vừa mang nghĩa "được phép giao", làm COD không có đường đi. Nghĩa "đã thu tiền" nay đọc ở `payment_status = SUCCESS`; nghĩa "được phép giao" là `CONFIRMED`.
- Event `order.paid` vẫn giữ nguyên ngữ nghĩa **tiền đã về** và vẫn phát khi `payment.succeeded` — với VNPAY là lúc vào `CONFIRMED`, với COD là sau `DELIVERED`. Consumer nào cần "đơn đã chốt, chuẩn bị giao" phải dùng `order.confirmed`, không dùng `order.paid`.
- Không service nào được suy ra quyền fulfillment từ `payment_status`; chỉ đọc `OrderStatus`.

### 3.2 Invoice

- Invoice có số unique, total/tax breakdown snapshot và `pdf_url` nullable; PDF generation/notification có thể async.
- Invoice tạo `PENDING` cùng transaction place checkout; `invoice.issued` phát khi chuyển `ISSUED`. Buyer/seller gọi invoice khi còn `PENDING` → `409 ORDER_INVOICE_NOT_READY`.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| `ORDER_NUMBER_MAX_LENGTH` | 32 | ⏳ chưa có | Unique. |
| `ORDER_EXPORT_MAX_ROWS` | 10000 | ⏳ chưa có | Vượt trả `400 ORDER_EXPORT_TOO_LARGE`, seller lọc hẹp hơn. |
| `ORDER_EXPORT_URL_TTL` | 30 phút | ⏳ chưa có | Signed download URL hết hạn, gọi lại endpoint để có URL mới. |
| `SELLER_BULK_ACCEPT_MAX_ORDERS` | 100 | ⏳ chưa có | `POST /seller/orders/bulk-accept` — tối đa order_id/lần gọi. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `OrderStatus` | `PENDING_PAYMENT`, `CONFIRMED`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. Trạng thái fulfillment; `PENDING_PAYMENT` chỉ dùng cho phương thức trả trước (VNPAY). |
| `PaymentStatusProjection` | `PENDING`, `PENDING_COD`, `SUCCESS`, `FAILED`, `EXPIRED`, `REFUNDED`, `PARTIALLY_REFUNDED`. `EXPIRED` → order ở `PENDING_PAYMENT` chuyển sang `CANCELLED` (xem `payment.expired` ở §6.3). `PARTIALLY_REFUNDED` → order giữ nguyên `OrderStatus` hiện tại, chỉ cập nhật số tiền đã hoàn, không tự đổi order status. |
| `InvoiceStatus` | `PENDING`, `ISSUED`, `VOIDED`. |
| `CancellationReason` | `BUYER_REQUEST`, `SELLER_REQUEST`, `PAYMENT_TIMEOUT`, `STOCK_FAILURE`, `SYSTEM`. |

Order state là source of truth của Order-Commerce; payment/shipment/inventory statuses được projection và không cho client tự set.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `order.events.v1` | `order.created` | order/group, buyer, shop, items snapshot, total, payment method |
| `order.events.v1` | `order.confirmed` | order/group, shop, payment method, confirmed_at |
| `order.events.v1` | `order.paid` | order, payment reference, paid_at |
| `order.events.v1` | `order.processing` | order, shop |
| `order.events.v1` | `order.shipped` | order, shipment reference |
| `order.events.v1` | `order.delivered` | order snapshot + `buyer` (`user_id`, `email`) + `items[]` (`product_id`, `sku_id`) + `delivered_at` — `buyer`/`items[]` là bắt buộc để Rating-Comment dựng eligibility |
| `order.events.v1` | `order.cancelled` | order, reason, compensation action |
| `invoice.events.v1` | `invoice.issued` | invoice/order/number/total/pdf reference |

Phân biệt bắt buộc giữa hai event dễ nhầm:

| Event | Nghĩa | VNPAY phát lúc | COD phát lúc | Dùng cho |
|---|---|---|---|---|
| `order.confirmed` | Đơn đã chốt, **được phép giao** | Khi nhận `payment.succeeded` | Ngay tại checkout | Fulfillment, email xác nhận đơn, hàng đợi seller |
| `order.paid` | **Tiền đã về** | Cùng lúc `order.confirmed` | Sau `DELIVERED`, khi capture tiền mặt | Đối soát, ledger, settlement |

Với VNPAY hai event trùng thời điểm; với COD chúng cách nhau cả vòng đời giao hàng. Consumer nào cần "đơn sẵn sàng giao" mà dùng `order.paid` sẽ bỏ sót toàn bộ đơn COD.

### 6.2 Payload chi tiết — `order.confirmed` / `order.paid` / `order.cancelled` / `invoice.issued` / `order.delivered`

Field `buyer` **đã chốt** cho các consumer cần recipient (đặc biệt Notification Service —
xem `notification-docs/docs/lld/notification.md` §6.2) — **gồm** `order.delivered`: `buyer.user_id` bắt buộc, `buyer.email`
optional (`null` cho phép ở event chỉ phát `IN_APP`, ví dụ `order.paid`). Field khác trong `payload`
lấy tên đúng theo response `GET /orders/{orderId}` (`docs/api/order.md` §3.1) và bảng `invoices` (`docs/db/order.md` §3.3).

`order.confirmed`:

```json
{
  "order_id": "order-01912f91",
  "order_number": "TC-20260830-0001",
  "checkout_group_id": "group-01912f90",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "shop": { "shop_id": "shop-01912f31", "shop_name": "Anker Official" },
  "payment_method": "VNPAY",
  "confirmed_at": "2026-08-30T09:05:00Z"
}
```

`order.paid` (chỉ phát `IN_APP` — xem bảng phân biệt ở trên — nên `buyer.email` có thể `null`):

```json
{
  "order_id": "order-01912f91",
  "buyer": { "user_id": "user-01912f10", "email": null },
  "payment_reference": "payment-01912f95",
  "paid_at": "2026-08-30T09:05:00Z"
}
```

`order.cancelled`:

```json
{
  "order_id": "order-01912f91",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "reason": "Hết hàng",
  "cancelled_at": "2026-08-30T10:00:00Z"
}
```

> `compensation action` (nhắc ở bảng §6.1) chưa có schema field cụ thể — phụ thuộc seller
> cancel/refund policy chưa chốt (xem `order-commerce.md` §8 mục 7), nên chưa đưa vào ví dụ JSON để tránh set field
> giả không đúng thực tế.

`invoice.issued`:

```json
{
  "invoice_id": "invoice-01912fb0",
  "order_id": "order-01912f91",
  "invoice_number": "INV-20260830-0001",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "total_amount": 1094000,
  "pdf_url": "https://cdn.taca.vn/invoices/invoice-01912fb0.pdf",
  "issued_at": "2026-08-30T09:10:00Z"
}
```

`order.delivered`:

```json
{
  "event_id": "01912f99-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "event_type": "order.delivered",
  "occurred_at": "2026-08-31T10:20:00Z",
  "aggregate_type": "ORDER",
  "aggregate_id": "order-01912f91",
  "payload": {
    "order_id": "order-01912f91",
    "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
    "items": [
      { "product_id": "product-01912f31", "sku_id": "sku-01912f33" },
      { "product_id": "product-01912f41", "sku_id": "sku-01912f43" }
    ],
    "delivered_at": "2026-08-31T10:20:00Z"
  }
}
```

### 6.3 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Payment-Wallet | `payment.succeeded`, `payment.failed`, `payment.refunded` | Update payment projection, transition order, commit/release Inventory. |
| Payment-Wallet | `payment.expired` | Order ở `PENDING_PAYMENT` chuyển sang `CANCELLED` (payment_status projection = `EXPIRED`); không hoàn kho nếu Inventory tự xử lý expire riêng qua `inventory.reservation.expired`. |
| Inventory | `inventory.reservation.created`, `inventory.reservation.committed`, `inventory.reservation.released`, `inventory.reservation.expired` | Reconcile order intent/reservation; không tự sửa quantity. Tên event lấy đúng catalog của Inventory — **không có** `reservation.confirmed`/`reservation.rejected`/`stock_committed`: reserve thành công/thất bại là kết quả **đồng bộ** của `POST /internal/inventory/reservations`, event chỉ dùng để reconcile khi mất response. |
| Shipment | `shipment.created`, `shipment.status_changed` | Update shipment projection/order status. |
| Shipment | `shipment.delivered` | Order chuyển `SHIPPED → DELIVERED`; cập nhật `delivered_at`. |
| Shipment | `shipment.failed` | Ghi nhận delivery failed vào timeline/shipment projection; **giữ** `OrderStatus = SHIPPED` (enum không có FAILED) — xử lý tiếp qua refund/return workflow ở Payment/Support, không tự đổi order status. |
| Auth User | `user.status_changed` (mock policy) | Chặn hành động account mới nếu suspended; không đổi order history. |

## 7. Mã lỗi

⏳ Chưa có code. `ORDER_FORBIDDEN`, `ORDER_STATE_INVALID`, `ORDER_CANCEL_NOT_ALLOWED`, `ORDER_IDEMPOTENCY_CONFLICT`, `ORDER_INVOICE_NOT_READY`, `ORDER_EXPORT_TOO_LARGE`, `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` (pass-through) — định nghĩa ở `order-commerce.md` §7.

## 8. Giả định & câu hỏi mở

Xem `order-commerce.md` §8 mục 4 (split theo shop), 7 (seller cancel policy, return/dispute ngoài v1).
