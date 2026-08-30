# API Spec — Order-Commerce Service

> Nguồn: `docs/lld/order-commerce.md` · `docs/db/order-commerce.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Cart/checkout/order cần JWT; buyer/seller scope kiểm tra tại service. |
| Request ID | `X-Request-ID` tối đa 64; Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate`; propagate sang Product/Inventory/Payment/Shipment và Kafka. |
| Timestamp | ISO-8601 UTC; DB UTC `DATETIME(6)`. |
| Money | Integer VND, `currency=VND`; không float. |
| Pagination | page 1, size 20, max 100. |
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
| 5 | `GET /vouchers/validate` | Buyer | Preview voucher trên cart. |
| 6 | `GET /checkout/shipping-fee` | Buyer | Preview shipping fee. |
| 7 | `POST /checkout/preview` | Buyer | Tính preview tổng tiền. |
| 8 | `POST /checkout` | Buyer + Idempotency | Reserve Inventory và tạo order split. |
| 9 | `GET /orders` | Buyer | Own order list. |
| 10 | `GET /orders/{orderId}` | Buyer | Own order detail. |
| 11 | `GET /orders/{orderId}/status` | Buyer | Status timeline/projection. |
| 12 | `PATCH /orders/{orderId}/cancel` | Buyer | Cancel trước shipping. |
| 13 | `GET /orders/{orderId}/invoice` | Buyer | Invoice metadata/download URL. |
| 14 | `GET /seller/orders` | Seller/staff | Shop order list. |
| 15 | `GET /seller/orders/{orderId}` | Seller/staff | Shop order detail. |
| 16 | `PATCH /seller/orders/{orderId}/fulfill` | Seller/staff | Accept/ship flow. |
| 17 | `POST /seller/orders/{orderId}/cancel` | Seller/staff | Seller cancel theo policy. |
| 18 | `GET /seller/orders/{orderId}/invoice` | Seller/staff | Invoice. |
| 19 | `POST /seller/vouchers` | Seller | Tạo shop voucher. |
| 20 | `GET /seller/vouchers` | Seller | List voucher. |
| 21 | `PUT /seller/vouchers/{id}` | Seller | Update voucher. |
| 22 | `DELETE /seller/vouchers/{id}` | Seller | Soft archive/inactivate. |
| 23 | `POST /admin/vouchers` | Admin (`VOUCHER_MANAGE`) | Tạo platform voucher (`scope=PLATFORM`). |
| 24 | `GET /admin/vouchers` | Admin | List platform voucher + usage metric. |
| 25 | `PUT /admin/vouchers/{id}` | Admin | Update platform voucher. |
| 26 | `DELETE /admin/vouchers/{id}` | Admin | Soft archive/inactivate. |

## 3. Chi tiết endpoint

### 3.1 Cart endpoints

- `GET /cart`: trả cart Redis hiện tại; empty là `200 data.items=[]`.
- `POST /cart/items` request `{product_id,sku_id,quantity}`; validate Product active và quantity `1..999`; cart không reserve stock.
- `PUT /cart/items/{itemId}` request `{quantity}`; quantity `0` không dùng update, phải DELETE.
- `DELETE /cart/items/{itemId}` trả `204`; item khác user trả `404 ORDER_CART_ITEM_NOT_FOUND` theo anti-enumeration policy.

### 3.2 `GET /vouchers/validate?code=&cart_id=`

Preview áp mã trên cart hiện tại. **Không** giữ chỗ voucher; `usage_limit`/redemption được kiểm tra lại (và tăng) trong `POST /checkout`.

| Query | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `code` | string | Có | 1–32 ký tự, normalize uppercase/trim trước khi so |
| `cart_id` | string (UUIDv7) | Có | Phải thuộc buyer trong token |

```json
{
  "data": {
    "valid": true,
    "code": "SHOP10",
    "scope": "SHOP",
    "shop_id": "shop-01912f31",
    "discount_type": "PERCENT",
    "value": 10,
    "discount_amount": 25000,
    "applicable_item_ids": ["item-1", "item-2"],
    "min_order": 200000,
    "expires_at": "2026-09-30T16:59:59Z",
    "warnings": []
  },
  "meta": { "request_id": "01912fa1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Mã không hợp lệ vẫn trả `200` với `valid:false` + `reason` (`NOT_FOUND`/`EXPIRED`/`MIN_ORDER_NOT_MET`/`USAGE_LIMIT_REACHED`/`SCOPE_MISMATCH`) để UI hiển thị — **không** dùng `404` cho mã sai (tránh dò mã).

### 3.3 `GET /checkout/shipping-fee`

| Query | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `from_shop_id` | string | Có | — |
| `to_address_id` | string | Có | Phải thuộc buyer, nếu không → `403 ORDER_FORBIDDEN` |
| `items` | array | Có | `sku_id:quantity`, tối đa 100 dòng |

```json
{
  "data": {
    "shipping_fee": 32000,
    "currency": "VND",
    "estimate": { "min_days": 2, "max_days": 4 },
    "quote_expires_at": "2026-08-30T09:30:00Z"
  },
  "meta": { "request_id": "01912fa2-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Giá lấy từ Shipment `GET /internal/shipping/quote`. Dependency down → `503 ORDER_DEPENDENCY_UNAVAILABLE` (UI phải cho retry, không tự đặt fee = 0).

### 3.4 `POST /checkout/preview`

Tính lại toàn bộ tiền trước khi đặt. **Không reserve kho, không charge.**

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `cart_id` | string | Có | Thuộc buyer |
| `selected_item_ids` | string[] | Có | 1–100 phần tử, thuộc cart |
| `address_id` | string | Có | Thuộc buyer |
| `voucher_code` | string | Không | — |

```json
{
  "data": {
    "checkout_group_id": "group-01912f90",
    "shops": [
      {
        "shop_id": "shop-01912f31",
        "shop_name": "Anker Official",
        "items": [
          {
            "item_id": "item-1",
            "product_id": "product-01912f31",
            "sku_id": "sku-01912f33",
            "title": "Sạc Anker 65W",
            "variant_label": "Đen / 65W",
            "unit_price": 590000,
            "quantity": 2,
            "line_total": 1180000,
            "available_qty": 7
          }
        ],
        "subtotal": 1180000,
        "shipping_fee": 32000,
        "discount": 118000,
        "shop_total": 1094000
      }
    ],
    "summary": {
      "subtotal": 1180000,
      "shipping_fee": 32000,
      "discount": 118000,
      "tax": 0,
      "grand_total": 1094000,
      "currency": "VND"
    },
    "warnings": [
      { "code": "PRICE_CHANGED", "item_id": "item-1", "old_price": 620000, "new_price": 590000 }
    ],
    "expires_at": "2026-08-30T09:10:00Z"
  },
  "meta": { "request_id": "01912fa3-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`warnings` không chặn đặt hàng; UI phải hiển thị để buyer xác nhận lại. Mã warning: `PRICE_CHANGED`, `STOCK_LOW`, `ITEM_UNAVAILABLE`, `VOUCHER_NOT_APPLICABLE`. Item `ITEM_UNAVAILABLE` sẽ bị loại khỏi `POST /checkout`.

### 3.5 `POST /checkout`

Headers: `Idempotency-Key` bắt buộc. Request:

```json
{
  "cart_id":"cart-01912f80",
  "selected_item_ids":["item-1"],
  "address_id":"address-1",
  "payment_method":"VNPAY",
  "voucher_code":"SHOP10"
}
```

Response `201`:

```json
{"data":{"checkout_group_id":"group-1","orders":[{"order_id":"order-1","order_number":"TC-20260830-0001","shop_id":"shop-1","status":"PENDING_PAYMENT","reservation_id":"reservation-1","payment_status":"PENDING"}],"payment_next_action":{"type":"VNPAY_QR","payment_id":"payment-1"},"reservation_expires_at":"2026-08-30T09:15:00Z"},"meta":{"request_id":"01912f81"}}
```

Order revalidates Product/price, calls Inventory atomic reserve, then commits local orders/items/invoice/outbox. Reserve fail không tạo order; retry cùng idempotency key trả cùng response.

### 3.6 Buyer order endpoints

- `GET /orders`: query `status?`, `page`, `size` (max 100); chỉ buyer own orders.
- `GET /orders/{id}`: trả snapshot; **không** query lại giá Product.
- `GET /orders/{id}/status`: status + timeline từ Order/Shipment events.
- `PATCH /orders/{id}/cancel` headers `Idempotency-Key`, body `{reason, version}`; chỉ trước shipping, release Inventory/refund compensation theo payment state.
- `GET /orders/{id}/invoice`: `200` nếu issued; `409 ORDER_INVOICE_NOT_READY` nếu pending.

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
    "amounts": { "subtotal": 1180000, "shipping_fee": 32000, "discount": 118000, "tax": 0, "grand_total": 1094000, "currency": "VND" },
    "created_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "01912fa4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Mọi field `*_snapshot` là bản chụp tại thời điểm đặt hàng — **không** đồng bộ khi Product/Address đổi về sau. `phone_masked` không bao giờ trả số đầy đủ ở list; detail chỉ trả cho chính buyer và seller sở hữu đơn.

`GET /orders/{orderId}/status` response:

```json
{
  "data": {
    "order_id": "order-01912f91",
    "status": "SHIPPED",
    "timeline": [
      { "status": "PENDING_PAYMENT", "occurred_at": "2026-08-30T09:00:00Z", "source": "ORDER" },
      { "status": "PAID", "occurred_at": "2026-08-30T09:03:12Z", "source": "PAYMENT" },
      { "status": "PROCESSING", "occurred_at": "2026-08-30T09:05:00Z", "source": "SELLER" },
      { "status": "SHIPPED", "occurred_at": "2026-08-31T02:10:00Z", "source": "SHIPMENT" }
    ]
  },
  "meta": { "request_id": "01912fa5-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

### 3.7 Seller order/voucher endpoints

- Seller list/detail filter theo `shop_id` auth context, không tin shop_id query/body.
- `PATCH /seller/orders/{id}/fulfill` body `{action:"ACCEPT"|"SHIP",tracking_code?}`; Shipment tạo tracking; seller không set `DELIVERED` trực tiếp.
- `POST /seller/orders/{id}/cancel` body `{reason,version}`; phải có policy/refund/stock release.
- Voucher create/update body gồm `code`, `scope=SHOP`, `discount_type` (`PERCENT`/`FIXED`), `value`, `min_order`, `usage_limit`, `starts_at`, `expires_at`; seller không tạo PLATFORM voucher. **Không có** field audience/targeting/segment trong v1 — các màn Penpot Seller "Voucher audience / Selected users / Import user IDs / Tab followers/VIP" ngoài phạm vi v1.
- Voucher DELETE là soft `INACTIVE/ARCHIVED`, không xóa redemption history.

### 3.8 Admin platform voucher (`/admin/vouchers`)

- Quyền: `VOUCHER_MANAGE` (do Auth User RBAC cấp) + step-up 2FA cho mutation; Gateway coarse-gate role admin, `order-commerce` enforce permission. v1 không có microservice admin riêng — admin console gọi trực tiếp route này (`System_Overview.md` §6.3).
- Body create/update: `code` (normalized unique), `scope=PLATFORM` (cố định), `discount_type`, `value`, `min_order`, `usage_limit`, `starts_at`, `expires_at`, `status`. Không có audience/targeting.
- `GET /admin/vouchers`: query `status`, `code`, `page`, `size`; trả voucher + `used_count`/`usage_limit`.
- `DELETE` là soft `INACTIVE/ARCHIVED`; redemption history giữ nguyên. Validation, usage increment và scope-check khi checkout dùng chung code path với shop voucher.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `ORDER_INVALID_INPUT` | 400 | Payload/query sai. |
| `ORDER_UNAUTHENTICATED` | 401 | Thiếu login. |
| `ORDER_FORBIDDEN` | 403 | Sai buyer/shop scope. |
| `ORDER_CART_EMPTY` | 409 | Không có item được chọn. |
| `ORDER_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active. |
| `ORDER_PRICE_CHANGED` | 409 | Giá snapshot cũ. |
| `ORDER_STOCK_UNAVAILABLE` | 409 | Inventory reserve fail. |
| `ORDER_RESERVATION_EXPIRED` | 409 | Reservation hết TTL. |
| `ORDER_VOUCHER_INVALID` | 400 | Voucher invalid/scope/time. |
| `ORDER_VOUCHER_LIMIT_REACHED` | 409 | Hết usage. |
| `ORDER_ADDRESS_INVALID` | 400 | Address không thuộc user/sai. |
| `ORDER_STATE_INVALID` | 409 | State transition sai. |
| `ORDER_CANCEL_NOT_ALLOWED` | 409 | Không còn được cancel. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Key khác request hash. |
| `ORDER_DEPENDENCY_UNAVAILABLE` | 503 | Product/Inventory/Payment/Shipment down. |
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice pending. |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | `POST /checkout` tạo order sau Inventory reserve, Payment charge tách service. | Ảnh hưởng saga/compensation/payment UX. | Order/Payment/Inventory owner |
| 2 | Multi-shop checkout split order theo shop nhưng chung `checkout_group_id`. | Ảnh hưởng invoice/shipment/voucher. | Product owner |
| 3 | Shipping fee endpoint dùng Shipment mock/quote; exact carrier contract chưa có. | Ảnh hưởng total/timeout. | Shipment owner |
| 4 | Voucher stacking/priority chưa chốt. | Ảnh hưởng discount calculation. | Product/Finance |
| 5 | Seller cancel/refund policy chưa đủ trong HLD. | Ảnh hưởng inventory/payment compensation. | Product/Finance |
| 6 | Voucher v1 chỉ theo mã, không audience/targeting; các màn Penpot Seller voucher targeting ngoài v1. | Nếu bật targeting cần schema `voucher_audiences` + follow shop. | Product owner |
| 7 | Platform voucher CRUD đặt tại `order-commerce` (`/admin/vouchers`, `VOUCHER_MANAGE`). | Nếu phạm vi Admin tách service riêng, service đó gọi vào contract này chứ không sở hữu voucher aggregate. | Architecture |
