# API Spec — Order-Commerce · Order

> Thuộc service Order-Commerce — quy ước chung, danh sách endpoint đầy đủ và mã lỗi ở `docs/api/order-commerce.md` · Cập nhật: `2026-10-31`
> LLD: `docs/lld/order.md` · DB: `docs/db/order.md` · Test: `docs/test/order.md`

## 1. Quy ước API

Theo `docs/api/order-commerce.md` §1 (base path `/api/v1`, auth qua Gateway, envelope `{data,meta}`, error `{error:{code,message,details,trace_id}}`, `Idempotency-Key` cho mutation, tiền integer VND).

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
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

## 3. Chi tiết endpoint

### 3.1 Buyer order endpoints

- `GET /orders`: query `status?`, `page`, `size` (max 100); chỉ buyer own orders.
- `GET /orders/{orderId}`: trả snapshot; **không** query lại giá Product.
- `GET /orders/{orderId}/status`: status + timeline từ Order/Shipment events.
- `PATCH /orders/{orderId}/cancel` headers `Idempotency-Key`, body `{reason, version}`; chỉ trước shipping, release Inventory/refund compensation theo payment state.
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

Cách đọc `amounts` (công thức đầy đủ ở `docs/lld/checkout.md` §2.2):

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

Phải có policy/refund/stock release theo `CancellationReason` (`docs/lld/order.md` §4).

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

`PROCESSING → SHIPPED`. Order-Commerce gọi `POST /internal/shipments` với `carrier` seller vừa chọn, nhận `tracking_code` thật từ Shipment (không phải seller tự gõ) — xem `shipment-docs/docs/api/shipment.md` §3.3. `carrier` không nằm trong danh sách khả dụng của đơn (đã đổi từ lúc seller xem carrier list) → `400 SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` (Order-Commerce pass-through nguyên trạng lỗi từ Shipment). Seller **không** set `DELIVERED` trực tiếp — trạng thái đó chỉ do Shipment event xác nhận (carrier webhook).

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

## 4. Mã lỗi

`ORDER_FORBIDDEN`, `ORDER_STATE_INVALID`, `ORDER_CANCEL_NOT_ALLOWED`, `ORDER_IDEMPOTENCY_CONFLICT`, `ORDER_INVOICE_NOT_READY`, `ORDER_EXPORT_TOO_LARGE`, `SHIPMENT_CARRIER_UNAVAILABLE_FOR_ORDER` (pass-through) — định nghĩa ở `docs/api/order-commerce.md` §4.

## 5. Giả định & câu hỏi mở

Xem `docs/api/order-commerce.md` §5.
