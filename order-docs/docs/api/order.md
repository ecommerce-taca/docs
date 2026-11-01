# API Spec — Order Service

> Nguồn: `docs/lld/order.md` · `docs/db/order.md` · `docs/test/order.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`, chỉ có `README.md`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Endpoint buyer/seller cần JWT (Gateway verify). `/internal/**` bắt buộc `X-Internal-Service-Token` khớp `ORDER_INTERNAL_TOKEN`; Gateway không expose. |
| Request ID / trace | `X-Request-ID` tối đa 64; W3C `traceparent`/`tracestate`; propagate sang Payment/Inventory/Shipment/Checkout và Kafka. |
| Actor context | `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope` do Gateway inject (client không gửi được). |
| Timestamp | ISO-8601 UTC; DB UTC `DATETIME(6)`. |
| Money | Integer VND, `currency=VND`. |
| Pagination | `page` từ 1, `size` mặc định 20, max 100; meta trả `request_id,page,size,total,total_pages`. |
| Idempotency | `Idempotency-Key` bắt buộc với cancel/fulfill/bulk-accept và `POST /internal/orders`; giữ 24h. |
| Optimistic lock | Mutation gửi `version`; lệch → `409 ORDER_VERSION_CONFLICT`. |
| Response | `{data,meta:{request_id}}`; error `{error:{code,message,details,trace_id}}`. |
| Log | JSON theo LLD §2.1. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /orders` | Buyer | Own order list. | ⏳ |
| 2 | `GET /orders/{orderId}` | Buyer | Own order detail. | ⏳ |
| 3 | `GET /orders/{orderId}/status` | Buyer | Status timeline. | ⏳ |
| 4 | `PATCH /orders/{orderId}/cancel` | Buyer | Cancel trước shipping. | ⏳ |
| 5 | `GET /orders/{orderId}/invoice` | Buyer | Invoice metadata/download URL. | ⏳ |
| 6 | `GET /seller/orders` | Seller/staff | Shop order list. | ⏳ |
| 7 | `GET /seller/orders/export` | Seller/staff | Xuất danh sách đơn (.xlsx/.csv). | ⏳ |
| 8 | `GET /seller/orders/{orderId}` | Seller/staff | Shop order detail. | ⏳ |
| 9 | `GET /seller/orders/{orderId}/shipment/carriers` | Seller/staff | Façade so sánh carrier (gọi Shipment). | ⏳ |
| 10 | `PATCH /seller/orders/{orderId}/fulfill` | Seller/staff | Accept/ship (1 đơn). | ⏳ |
| 11 | `POST /seller/orders/bulk-accept` | Seller/staff | Accept hàng loạt. | ⏳ |
| 12 | `POST /seller/orders/{orderId}/cancel` | Seller/staff | Seller cancel theo policy. | ⏳ |
| 13 | `GET /seller/orders/{orderId}/invoice` | Seller/staff | Invoice. | ⏳ |
| 14 | `POST /internal/orders` | Internal (`checkout-service`) | Tạo đơn từ checkout. | ⏳ |
| 15 | `GET /internal/orders?checkout_group_id=` | Internal (`checkout-service`) | Reconcile group không rõ kết quả. | ⏳ |
| 16 | `GET /health/live` | Internal/ops | Liveness process-only. | ⏳ |
| 17 | `GET /health/ready` | Internal/ops | Readiness MySQL/Kafka. | ⏳ |

Path buyer/seller giữ nguyên như thời Order-Commerce; Gateway chỉ đổi upstream. `GET /orders/{orderId}/shipment` và `GET /seller/orders/{orderId}/shipment` (tracking) **thuộc Shipment**, không thuộc `order-service`.

## 3. Chi tiết endpoint

### 3.1 Buyer order endpoints

- `GET /orders`: query `status?`, `page`, `size` (max 100); chỉ buyer own orders.
- `GET /orders/{orderId}`: trả snapshot; **không** query lại giá Product.
- `GET /orders/{orderId}/status`: status + timeline từ `order_status_history` (ghi khi transition, kể cả transition do Payment/Shipment event).
- `PATCH /orders/{orderId}/cancel` headers `Idempotency-Key`, body `{reason, version}`; chỉ trước shipping; compensation (release Inventory, refund, tính lại voucher) theo `docs/lld/order.md` §3.3.
- `GET /orders/{orderId}/invoice`: `200` nếu issued; `409 ORDER_INVOICE_NOT_READY` nếu pending.

`GET /orders/{orderId}` response:

```json
{
  "data": {
    "order_id": "order-01912f91",
    "order_number": "TC-20260830-0001",
    "checkout_group_id": "group-01912f90",
    "status": "PENDING_PAYMENT",
    "version": 1,
    "shop": { "shop_id": "shop-01912f31", "shop_name": "Anker Official" },
    "items": [
      {
        "order_item_id": "oi-1",
        "product_id": "product-01912f31",
        "sku_id": "sku-01912f33",
        "title_snapshot": "Sạc Anker 65W",
        "variant_label_snapshot": "Đen / 65W",
        "unit_price_snapshot": 590000,
        "quantity": 2,
        "line_total": 1180000
      }
    ],
    "address_snapshot": {
      "recipient_name": "Nguyễn Văn A",
      "phone_masked": "09****678",
      "line1": "12 Nguyễn Huệ",
      "ward": "Bến Nghé", "district": "Quận 1", "province": "TP.HCM"
    },
    "payment": { "method": "VNPAY", "status": "PENDING", "payment_id": "payment-01912f95" },
    "shipment": { "status": "NOT_CREATED", "carrier": null, "tracking_code": null },
    "amounts": {
      "subtotal": 1180000,
      "shop_discount": 100000,
      "platform_discount": 18000,
      "shipping_fee": 32000,
      "freeship_discount": 0,
      "discount": 118000,
      "tax": 96545,
      "grand_total": 1094000,
      "currency": "VND"
    },
    "applied_vouchers": [
      { "scope": "SHOP", "code": "SHOP10", "discount_amount": 100000 },
      { "scope": "PLATFORM", "code": "TACA200K", "discount_amount": 18000 }
    ],
    "created_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "01912fa4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Cách đọc `amounts` (công thức đầy đủ ở `checkout-docs/docs/lld/checkout.md` §3.2; `order-service` lưu đúng số Checkout gửi, không tự tính):

```
grand_total = subtotal − shop_discount − platform_discount + shipping_fee − freeship_discount
            = 1.180.000 − 100.000 − 18.000 + 32.000 − 0 = 1.094.000
```

| Field | Nghĩa |
|---|---|
| `shop_discount` / `platform_discount` / `freeship_discount` | Phần giảm của từng `scope` **đã phân bổ về đúng đơn con này**, không phải giá trị gốc của voucher |
| `discount` | Tổng tiền hàng được giảm = `shop_discount + platform_discount`. **Không** gồm `freeship_discount` (đó là giảm phí ship) |
| `tax` | VAT **đã nằm trong** `subtotal` (giá bán đã gồm thuế), tách ra để ghi hoá đơn. **Không cộng** vào `grand_total`. Tính trên tiền hàng sau giảm giá: `1.062.000 × 1000/11000 = 96.545` với thuế suất 10% |
| `applied_vouchers` | Tối đa 1 dòng mỗi `scope`; dùng để hiển thị chi tiết và đối soát ngân sách voucher |

FE hiển thị dòng "Đã bao gồm VAT {tax}" — **không** được cộng `tax` vào tổng phải trả.

Mọi field `*_snapshot` là bản chụp tại thời điểm đặt hàng — **không** đồng bộ khi Product/Address đổi về sau. `phone_masked` không bao giờ trả số đầy đủ ở list; detail chỉ trả cho chính buyer và seller sở hữu đơn.

`GET /orders/{orderId}/status` response:

```json
{
  "data": {
    "order_id": "order-01912f91",
    "status": "SHIPPED",
    "timeline": [
      { "status": "PENDING_PAYMENT", "occurred_at": "2026-08-30T09:00:00Z", "source": "ORDER" },
      { "status": "CONFIRMED", "occurred_at": "2026-08-30T09:03:12Z", "source": "PAYMENT" },
      { "status": "PROCESSING", "occurred_at": "2026-08-30T09:05:00Z", "source": "SELLER" },
      { "status": "SHIPPED", "occurred_at": "2026-08-31T02:10:00Z", "source": "SHIPMENT" }
    ]
  },
  "meta": { "request_id": "01912fa5-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`status` nhận một trong `PENDING_PAYMENT`, `CONFIRMED`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. Timeline chỉ ghi chuyển trạng thái **fulfillment**; biến động tiền đọc ở `payment.status`.

Đơn COD có timeline khác — **không** có `PENDING_PAYMENT`:

```json
{
  "data": {
    "order_id": "order-01912f92",
    "status": "DELIVERED",
    "timeline": [
      { "status": "CONFIRMED", "occurred_at": "2026-08-30T09:00:00Z", "source": "ORDER" },
      { "status": "PROCESSING", "occurred_at": "2026-08-30T09:05:00Z", "source": "SELLER" },
      { "status": "SHIPPED", "occurred_at": "2026-08-31T02:10:00Z", "source": "SHIPMENT" },
      { "status": "DELIVERED", "occurred_at": "2026-09-01T07:40:00Z", "source": "SHIPMENT" }
    ]
  },
  "meta": { "request_id": "01912fa6-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Với đơn COD, `payment.status` đi `PENDING_COD → SUCCESS` tại thời điểm giao thành công, tức là **sau** khi `status` đã là `DELIVERED`. FE không được coi `payment.status != SUCCESS` là "đơn chưa hợp lệ".

### 3.2 Seller order endpoints

`GET /seller/orders` — query `status?`, `page`, `size` (max 100); filter theo `shop_id` auth context, không tin shop_id query/body.

```json
{
  "data": [
    {
      "order_id": "order-01912f91",
      "order_number": "TC-20260830-0001",
      "status": "CONFIRMED",
      "payment": { "method": "COD", "status": "PENDING_COD" },
      "buyer_name_masked": "Nguyễn V. A",
      "grand_total": 1094000,
      "currency": "VND",
      "placed_at": "2026-08-30T09:00:00Z"
    }
  ],
  "meta": { "request_id": "01912fc1-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 38, "total_pages": 2 }
}
```

`GET /seller/orders/{orderId}` — cùng shape chi tiết với `GET /orders/{orderId}` phía buyer (§3.1) nhưng ẩn `buyer_name`/`phone` đầy đủ (chỉ `buyer_name_masked`), thêm `seller_note` nếu có.

`POST /seller/orders/{orderId}/cancel` body `{reason,version}`. Response `200`:

```json
{ "data": { "order_id": "order-01912f91", "status": "CANCELLED", "reason": "SELLER_REQUEST", "version": 5 }, "meta": { "request_id": "01912fc2-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Phải có policy/refund/stock release theo `CancellationReason` (`docs/lld/order.md` §5).

### 3.3 `GET /seller/orders/export`

Query giống `GET /seller/orders` (`status?`, `from?`, `to?`) cộng `format` (`xlsx` mặc định | `csv`). Response `200`:

```json
{
  "data": {
    "export_url": "https://storage.example/signed-download/orders-shop-01912f31-20260830.xlsx",
    "format": "xlsx",
    "row_count": 312,
    "generated_at": "2026-08-30T09:00:00Z",
    "expires_at": "2026-08-30T09:30:00Z"
  },
  "meta": { "request_id": "01912fbc-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Cùng convention với `product-catalog-docs/docs/api/product-catalog.md` §`GET /seller/products/export`: signed URL, không stream qua Gateway body. Cột export: `order_number, status, payment_method, payment_status, buyer_name_masked, subtotal, discount, shipping_fee, tax, grand_total, placed_at`. Vượt `ORDER_EXPORT_MAX_ROWS` (10.000) → `400 ORDER_EXPORT_TOO_LARGE`, seller lọc hẹp hơn bằng `from`/`to`.

### 3.4 `PATCH /seller/orders/{orderId}/fulfill` — accept 1 đơn

Body action `ACCEPT`:

```json
{ "action": "ACCEPT", "version": 3 }
```

Response `200`:

```json
{ "data": { "order_id": "order-01912f91", "status": "PROCESSING", "version": 4 }, "meta": { "request_id": "01912fb8-7a1b-7c12-9c55-8b1c34a6d921" } }
```

`CONFIRMED → PROCESSING`. Chỉ hợp lệ khi order đang `CONFIRMED`; sai trạng thái → `409 ORDER_STATE_INVALID`.

Body action `SHIP` — **bắt buộc `carrier`**, seller đã chọn ở `GET /seller/orders/{orderId}/shipment/carriers` (`shipment-docs/docs/api/shipment.md` §3.1a):

```json
{ "action": "SHIP", "carrier": "GHN", "version": 4 }
```

Response `200`:

```json
{
  "data": {
    "order_id": "order-01912f91",
    "status": "SHIPPED",
    "version": 5,
    "shipment": { "shipment_id": "shp-01912fb0", "carrier": "GHN", "tracking_code": "GHN123456789" }
  },
  "meta": { "request_id": "01912fb9-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`PROCESSING → SHIPPED`. Body gửi Shipment theo đúng `CreateShipmentRequest` đã có code (`shipment-service@develop`):

| Field Shipment | Lấy từ |
|---|---|
| `order_id`, `checkout_group_id`, `shop_id`, `buyer_user_id` | `orders` |
| `buyer_email` | `orders.buyer_email` (bắt buộc, `@Email`) |
| `carrier` | Seller chọn |
| `from_address_snapshot` | Địa chỉ lấy hàng của shop: đọc `GET /api/v1/internal/shops/{shopId}/pickup-address` của auth-user (`auth-user-docs/docs/api/auth-user.md` §3.52) lúc carrier quote và lúc SHIP, map `{contact_name, phone, line1, line2, ward, district, province, postal_code}`, snapshot vào shipment |
| `to_address_snapshot` | `orders.address_snapshot` |
| `items[]` (≤100) | `order_items` |
| `cod_amount` | `grand_total` hiện tại nếu `payment_method=COD` (đã gồm điều chỉnh recalculate), ngược lại `0` |
| `shipping_fee` | `shipping_fee − freeship_discount` |
| `payment_method`, `currency` | `orders.payment_method`, `VND` |
| `order_code` | `order_number` |

`order-service` gọi `POST /api/v1/internal/shipments` với `carrier` seller vừa chọn và `order_code` = `order_number` (để event giao hàng của Shipment mang mã đơn cho Notification), nhận `tracking_code` thật từ Shipment (không phải seller tự gõ) — xem `shipment-docs/docs/api/shipment.md` §3.4. `carrier` không nằm trong danh sách khả dụng của đơn (đã đổi từ lúc seller xem carrier list) → `400 SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` (`order-service` pass-through nguyên trạng lỗi từ Shipment). Seller **không** set `DELIVERED` trực tiếp — trạng thái đó chỉ do Shipment event xác nhận (carrier webhook).

### 3.5 `POST /seller/orders/bulk-accept` — accept hàng loạt

Phục vụ Penpot `Overlay / Bulk fulfillment`, `CTA / Xác nhận 24 đơn`. **Chỉ thực hiện `ACCEPT`** (`CONFIRMED → PROCESSING`); tạo shipment/chọn carrier vẫn làm riêng từng đơn qua §3.4 — action `SHIP` không có trong bulk.

Headers: `Idempotency-Key` bắt buộc. Request:

```json
{ "order_ids": ["order-01912f91", "order-01912f92", "order-01912f93"] }
```

Ràng buộc: tối đa 100 `order_id`/lần gọi, tất cả phải thuộc `shop_id` trong token.

Response `200` — **không atomic**, mỗi đơn có kết quả riêng:

```json
{
  "data": {
    "succeeded": [
      { "order_id": "order-01912f91", "status": "PROCESSING", "version": 4 },
      { "order_id": "order-01912f92", "status": "PROCESSING", "version": 2 }
    ],
    "failed": [
      { "order_id": "order-01912f93", "error_code": "ORDER_STATE_INVALID", "message": "Order không ở trạng thái CONFIRMED." }
    ]
  },
  "meta": { "request_id": "01912fba-7a1b-7c12-9c55-8b1c34a6d921", "requested": 3, "succeeded_count": 2, "failed_count": 1 }
}
```

Một đơn lỗi (sai state, sai shop scope, version conflict) **không** chặn các đơn còn lại — mỗi order transition trong transaction riêng. FE hiển thị tổng kết theo `succeeded_count`/`failed_count`, cho phép retry riêng các đơn `failed`. Gửi lại cùng `Idempotency-Key` với **cùng** `order_ids` trả cùng kết quả; `order_ids` khác → `409 ORDER_IDEMPOTENCY_CONFLICT`.

### 3.6 `GET /seller/orders/{orderId}/shipment/carriers` — façade chọn carrier

Kiểm đơn thuộc `X-User-Shop-Scope`, `status` là `CONFIRMED` hoặc `PROCESSING`, chưa có shipment; rồi gọi Shipment `POST /api/v1/internal/shipping/carrier-quotes` (`shipment-docs/docs/api/shipment.md` §3.3) với `order_id`, `shop_id`, `from_address_snapshot`, `to_address_snapshot`, `items[]`. Response trả nguyên `data` của Shipment:

```json
{
  "data": {
    "order_id": "order-01912f91",
    "carriers": [
      { "carrier": "GHN", "available": true, "fee": 32000, "currency": "VND", "estimate": { "min_days": 2, "max_days": 4 } },
      { "carrier": "SPX", "available": false, "unavailable_reason": "ADAPTER_NOT_CONFIGURED" },
      { "carrier": "J&T", "available": false, "unavailable_reason": "ADAPTER_NOT_CONFIGURED" }
    ],
    "quote_expires_at": "2026-08-30T09:30:00Z"
  },
  "meta": { "request_id": "01912fb4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Shape lấy theo code Shipment (`QuoteService.compare`, `shipment-service@develop`): carrier khả dụng có `fee`, `currency`, `estimate{min_days,max_days}`; không khả dụng có `unavailable_reason` (`ADAPTER_NOT_CONFIGURED`, `CARRIER_UNAVAILABLE`, `QUOTE_FAILED`). Sai trạng thái → `409 ORDER_STATE_INVALID`; Shipment down → `503 ORDER_DEPENDENCY_UNAVAILABLE`.

### 3.7 `POST /internal/orders` — tạo đơn từ checkout

Headers: `X-Internal-Service-Token`, `Idempotency-Key` = `checkout_group_id`. Gọi bởi `checkout-service` (`checkout-docs/docs/lld/checkout.md` §3.3). Toàn bộ số tiền do Checkout tính; `order-service` chỉ kiểm bất biến (`grand_total = subtotal − shop_discount − platform_discount + shipping_fee − freeship_discount`, `Σ order_vouchers = discount + freeship_discount`, `grand_total ≥ 1`) và lưu.

```json
{
  "checkout_group_id": "group-01912f90",
  "buyer": { "user_id": "user-01912f10", "email": "buyer@example.com" },
  "payment_method": "VNPAY",
  "reservation_expires_at": "2026-08-30T09:15:00Z",
  "address_snapshot": {
    "recipient_name": "Nguyễn Văn A",
    "phone": "0912345678",
    "line1": "12 Nguyễn Huệ",
    "ward": "Bến Nghé", "district": "Quận 1", "province": "TP.HCM"
  },
  "orders": [
    {
      "order_id": "order-01912f91",
      "reservation_id": "reservation-01912fa1",
      "shop": { "shop_id": "shop-01912f31", "shop_name": "Apple Store" },
      "items": [
        {
          "order_item_id": "oi-01912f9a",
          "product_id": "product-01912f31", "sku_id": "sku-01912f33",
          "title_snapshot": "iPhone ốp lưng", "variant_label_snapshot": "Đen", "seller_sku_snapshot": "OP-01",
          "unit_price": 500000, "quantity": 1, "line_total": 500000,
          "tax_rate_bps": 1000, "tax_amount": 22096, "discount_allocated": 256944
        }
      ],
      "amounts": {
        "subtotal": 800000, "shop_discount": 300000, "platform_discount": 111111,
        "shipping_fee": 30000, "freeship_discount": 30000,
        "discount": 411111, "tax": 35354, "grand_total": 388889
      },
      "applied_vouchers": [
        { "voucher_id": "vch-01912fc3", "scope": "SHOP", "code": "APPLE300K", "discount_amount": 300000 },
        { "voucher_id": "vch-01912fc7", "scope": "PLATFORM", "code": "TACA200K", "discount_amount": 111111 },
        { "voucher_id": "vch-01912fc9", "scope": "FREESHIP", "code": "FREESHIPMAX", "discount_amount": 30000 }
      ]
    }
  ]
}
```

(Ví dụ rút gọn một đơn con và một item; số khớp golden `CO-CALC-01`, đơn Shop A.)

- `orders[].reservation_id`: **mỗi đơn con một reservation** (Checkout reserve riêng từng đơn), để commit/release một đơn không đụng kho của đơn khác trong nhóm.
- `items[].order_item_id`: Checkout sinh trước, `order-service` dùng làm `order_items.id`. Recalculate trả `items[]` theo id này (v1 giá trị không đổi) (`checkout-docs/docs/api/checkout.md` §3.4).

Response `201` — VNPAY:

```json
{
  "data": {
    "checkout_group_id": "group-01912f90",
    "orders": [
      { "order_id": "order-01912f91", "order_number": "TC-20260830-0001", "shop_id": "shop-01912f31", "status": "PENDING_PAYMENT", "reservation_id": "reservation-01912fa1", "payment_status": "PENDING" }
    ],
    "payment_next_action": { "type": "VNPAY_QR", "payment_id": "payment-01912f95" },
    "reservation_expires_at": "2026-08-30T09:15:00Z"
  },
  "meta": { "request_id": "01912fd1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

COD: `status=CONFIRMED`, `payment_status=PENDING_COD`, `payment_next_action.type=NONE`, `reservation_expires_at=null` (đã commit kho). Payment-Wallet lỗi với VNPAY: `payment_next_action.type=RETRY_LATER`, `payment_id=null`.

| Lỗi | HTTP | Khi nào |
|---|---:|---|
| `ORDER_INVALID_INPUT` | 400 | Thiếu field, bất biến tiền sai. Checkout coi là lỗi cuối, release kho/voucher. |
| `ORDER_UNAUTHENTICATED` | 401 | Thiếu/sai token. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Cùng `checkout_group_id`, body khác. |

### 3.8 `GET /internal/orders?checkout_group_id=`

Trả `{data:{checkout_group_id, orders:[{order_id, order_number, status, payment_status}]}}`; group chưa có đơn → `orders=[]` (không `404`).

## 4. Mã lỗi chung

Bảng đầy đủ ở `docs/lld/order.md` §7. Endpoint buyer/seller trả: `ORDER_FORBIDDEN`, `ORDER_NOT_FOUND`, `ORDER_STATE_INVALID`, `ORDER_CANCEL_NOT_ALLOWED`, `ORDER_VERSION_CONFLICT`, `ORDER_IDEMPOTENCY_CONFLICT`, `ORDER_INVOICE_NOT_READY`, `ORDER_EXPORT_TOO_LARGE`, `ORDER_DEPENDENCY_UNAVAILABLE`, `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` (pass-through).

> Mã lỗi của Shipment được pass-through nguyên trạng khi `order-service` đứng giữa (không đổi tên/status).

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Seller cancel/refund policy chưa đủ trong HLD. | Ảnh hưởng compensation. | Product/Finance |
| 2 | `address_snapshot` lưu số điện thoại đầy đủ (cần cho Shipment); API chỉ trả `phone_masked`. | Ảnh hưởng privacy. | Security |
| 3 | Đã chốt contract (2026-11-01, đóng S12): `from_address_snapshot` đọc từ auth-user `GET /internal/shops/{shopId}/pickup-address` (auth-user đã có code, PR #116). `409 SHOP_WAREHOUSE_MISSING` → façade/SHIP trả `409 ORDER_STATE_INVALID` kèm `details.reason=SHOP_WAREHOUSE_MISSING`. | Đã có code ở auth-user; còn chờ order-service gọi. | Auth-user + Order owner |
