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
| 5 | `GET /vouchers/validate` | Buyer | Preview voucher trên cart (buyer đã biết mã). |
| 5a | `GET /vouchers` | Buyer | Danh sách voucher buyer đang dùng được (discovery). |
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
| 21 | `PUT /seller/vouchers/{id}` | Seller | Update voucher. |
| 22 | `DELETE /seller/vouchers/{id}` | Seller | Soft archive/inactivate. |
| 23 | `POST /admin/vouchers` | Admin (`VOUCHER_MANAGE`) | Tạo platform voucher (`scope=PLATFORM`). |
| 24 | `GET /admin/vouchers` | Admin | List platform voucher + usage metric. |
| 25 | `PUT /admin/vouchers/{id}` | Admin | Update platform voucher. |
| 26 | `DELETE /admin/vouchers/{id}` | Admin | Soft archive/inactivate. |

## 3. Chi tiết endpoint

### 3.1 Cart endpoints

`GET /cart` — trả cart Redis hiện tại; empty là `200 data.items=[]`.

```json
{
  "data": {
    "cart_id": "cart-01912f80",
    "status": "ACTIVE",
    "items": [
      {
        "item_id": "item-1",
        "product_id": "product-01912f31",
        "sku_id": "sku-01912f33",
        "shop_id": "shop-01912f30",
        "title_snapshot": "Áo khoác cotton",
        "sku_label_snapshot": "Đen / L",
        "unit_price_snapshot": 249000,
        "quantity": 2,
        "line_total": 498000,
        "available": true
      }
    ],
    "item_count": 1,
    "expires_at": "2026-09-29T09:00:00Z"
  },
  "meta": { "request_id": "01912fbe-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`available:false` khi Product/SKU đã inactive/hết hàng kể từ lúc thêm — item vẫn hiển thị trong giỏ để buyer tự xoá, nhưng bị loại khỏi `selected_item_ids` hợp lệ ở checkout preview/place.

`POST /cart/items` request `{product_id,sku_id,quantity}`. Response `201`:

```json
{ "data": { "cart_id": "cart-01912f80", "item_id": "item-1", "quantity": 2, "line_total": 498000 }, "meta": { "request_id": "01912fbf-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Validate Product active và quantity `1..999`; cart không reserve stock. Thêm SKU đã có trong giỏ → cộng dồn `quantity` (không tạo `item_id` mới), cap tại `999`.

`PUT /cart/items/{itemId}` request `{quantity}`. Response `200`:

```json
{ "data": { "item_id": "item-1", "quantity": 3, "line_total": 747000 }, "meta": { "request_id": "01912fc0-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Quantity `0` không dùng update, phải DELETE.

`DELETE /cart/items/{itemId}` trả `204`; item khác user trả `404 ORDER_CART_ITEM_NOT_FOUND` theo anti-enumeration policy.

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

### 3.2a `GET /vouchers` — danh sách voucher khả dụng

`validate` chỉ trả lời được khi buyer **đã biết mã**. HLD #23 yêu cầu buyer *"choose and apply voucher of system platform or shop"* — chọn từ danh sách. Endpoint này phục vụ mọi mặt hiển thị voucher: `Taca Buyer / My Vouchers`, `Overlay / Apply vouchers` ở checkout, `PDP / Voucher strip`, `Shop / Voucher strip`, `Mobile Buyer / Apply Voucher`.

| Query | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `scope` | enum | Không | `PLATFORM` \| `SHOP`; bỏ trống trả cả hai |
| `shop_id` | string (UUIDv7) | Không | Lọc voucher của một shop — dùng cho `Shop / Voucher strip` |
| `product_id` | string (UUIDv7) | Không | Voucher áp được cho sản phẩm — dùng cho `PDP / Voucher strip` |
| `cart_id` | string (UUIDv7) | Không | Có thì server tính `eligible` + `discount_amount` theo giỏ hiện tại |
| `status` | enum | Không | `AVAILABLE` (mặc định) \| `EXPIRING_SOON` \| `EXPIRED` |
| `page`, `size` | int | Không | `size` mặc định 20, max 100 |

```json
{
  "data": [
    {
      "code": "TACA200K",
      "scope": "PLATFORM",
      "shop_id": null,
      "title": "Giảm 200K cho đơn từ 2 triệu",
      "discount_type": "FIXED",
      "value": 200000,
      "min_order": 2000000,
      "starts_at": "2026-08-01T00:00:00Z",
      "expires_at": "2026-09-30T16:59:59Z",
      "usage_limit": 1000,
      "used_count": 812,
      "eligible": true,
      "ineligible_reason": null,
      "discount_amount": 200000
    },
    {
      "code": "TACA9SALE",
      "scope": "PLATFORM",
      "shop_id": null,
      "title": "Giảm 8% tối đa 1 triệu",
      "discount_type": "PERCENT",
      "value": 8,
      "min_order": 5000000,
      "starts_at": "2026-09-09T00:00:00Z",
      "expires_at": "2026-09-09T16:59:59Z",
      "usage_limit": 500,
      "used_count": 500,
      "eligible": false,
      "ineligible_reason": "USAGE_LIMIT_REACHED",
      "discount_amount": 0
    }
  ],
  "meta": { "request_id": "01912fa7-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 12 }
}
```

Quy tắc:

- `eligible`/`discount_amount` **chỉ có nghĩa khi truyền `cart_id`**; không truyền thì `eligible` trả `null` và UI chỉ hiển thị điều kiện.
- `ineligible_reason` dùng đúng bộ mã của `validate`: `EXPIRED`, `MIN_ORDER_NOT_MET`, `USAGE_LIMIT_REACHED`, `SCOPE_MISMATCH`, `ALREADY_USED`. Đây là nguồn cho trạng thái `Voucher option / TACA9SALE disabled` trên Penpot.
- Chỉ trả voucher `status=ACTIVE` và còn trong cửa sổ thời gian; voucher `DRAFT`/`ARCHIVED` không bao giờ lộ ra buyer.
- Endpoint này **không** giữ chỗ và **không** tăng `used_count`; redemption vẫn chốt ở `POST /checkout`.
- `EXPIRING_SOON` = hết hạn trong 7 ngày; phục vụ metric `Voucher metric / Sắp hết hạn`.

> **v1 không có "lưu/đổi voucher".** Không có bảng sở hữu voucher theo user; "Voucher của tôi" = danh sách voucher buyer **đang đủ điều kiện dùng**, tính động. Nút `CTA / Đổi voucher` trên Penpot (thu thập voucher về ví cá nhân) ngoài phạm vi v1 — xem `System_Overview.md` §6.2.

### 3.2b `GET /vouchers/redemptions` — voucher đã dùng

Phục vụ metric `Voucher metric / Đã tiết kiệm` và `Chip / Đã sử dụng 28` trên màn My Vouchers.

Query: `page`, `size` (max 100), `from?`, `to?` (ISO-8601).

```json
{
  "data": [
    {
      "code": "APPLE300K",
      "scope": "SHOP",
      "shop_id": "shop-01912f31",
      "order_id": "order-01912f91",
      "discount_amount": 300000,
      "redeemed_at": "2026-08-27T04:11:00Z"
    }
  ],
  "meta": {
    "request_id": "01912fa8-7a1b-7c12-9c55-8b1c34a6d921",
    "page": 1, "size": 20, "total": 28,
    "total_saved": 1840000, "currency": "VND"
  }
}
```

`total_saved` là tổng `discount_amount` của **toàn bộ** redemption khớp filter, không phải chỉ trang hiện tại. Redemption của order đã huỷ/hoàn không tính vào `total_saved`.

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
| `voucher_codes` | string[] | Không | Tối đa 1 mã mỗi `scope`; giỏ nhiều shop được nhiều mã `SHOP`. Mã không áp được → `warning`, không lỗi |

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
  "selected_item_ids":["item-1","item-2"],
  "address_id":"address-1",
  "payment_method":"VNPAY",
  "voucher_codes":["TACA200K","APPLE300K","FREESHIPMAX"]
}
```

`voucher_codes` là mảng, tối đa **3 phần tử**, mỗi phần tử thuộc một `scope` khác nhau (`PLATFORM` / `SHOP` / `FREESHIP`) — xem mô hình 3 slot ở `docs/lld/order-commerce.md` §3.5. Giỏ nhiều shop được gửi nhiều mã `SHOP` (mỗi shop một mã), khi đó mảng dài hơn 3.

| Lỗi | HTTP | Khi nào |
|---|---:|---|
| `ORDER_VOUCHER_SLOT_CONFLICT` | 400 | Hai mã cùng `scope` (2 mã `PLATFORM`, hoặc 2 mã `SHOP` cho cùng một shop) |
| `ORDER_VOUCHER_INVALID` | 400 | Mã không tồn tại / hết hạn / sai trạng thái |

Mã hợp lệ nhưng không áp được cho giỏ này (sai shop, chưa đủ `min_order`) **không** chặn đặt hàng: mã đó bị bỏ qua và trả `warning VOUCHER_NOT_APPLICABLE` kèm `code` để UI hiển thị.

`payment_method` nhận `VNPAY` hoặc `COD`. Hai giá trị này cho ra hai response khác nhau về `status` — FE phải xử lý cả hai.

Response `201` — **VNPAY** (chờ thanh toán):

```json
{"data":{"checkout_group_id":"group-1","orders":[{"order_id":"order-1","order_number":"TC-20260830-0001","shop_id":"shop-1","status":"PENDING_PAYMENT","reservation_id":"reservation-1","payment_status":"PENDING"}],"payment_next_action":{"type":"VNPAY_QR","payment_id":"payment-1"},"reservation_expires_at":"2026-08-30T09:15:00Z"},"meta":{"request_id":"01912f81"}}
```

Response `201` — **COD** (đơn chốt ngay):

```json
{"data":{"checkout_group_id":"group-2","orders":[{"order_id":"order-2","order_number":"TC-20260830-0002","shop_id":"shop-1","status":"CONFIRMED","reservation_id":"reservation-2","payment_status":"PENDING_COD"}],"payment_next_action":{"type":"NONE"},"reservation_expires_at":null},"meta":{"request_id":"01912f82"}}
```

| Field | VNPAY | COD |
|---|---|---|
| `status` | `PENDING_PAYMENT` | `CONFIRMED` |
| `payment_status` | `PENDING` | `PENDING_COD` |
| `payment_next_action.type` | `VNPAY_QR` | `NONE` |
| `reservation_expires_at` | timestamp — hết hạn thì đơn bị huỷ | `null` — kho đã commit ngay, không có hạn |

Order revalidates Product/price, calls Inventory atomic reserve, then commits local orders/items/invoice/outbox. Reserve fail không tạo order; retry cùng idempotency key trả cùng response.

Với COD, Inventory `commit` được gọi ngay trong luồng checkout (không có bước chờ tiền), nên `reservation_expires_at` là `null` và đơn không bị job payment-timeout đụng tới. FE **không** được hiển thị đồng hồ đếm ngược thanh toán cho đơn COD.

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

Cách đọc `amounts` (công thức đầy đủ ở `docs/lld/order-commerce.md` §3.2a):

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

### 3.7 Seller order endpoints

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
  "meta": { "request_id": "01912fc1-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 38 }
}
```

`GET /seller/orders/{orderId}` — cùng shape chi tiết với `GET /orders/{orderId}` phía buyer (§3.6) nhưng ẩn `buyer_name`/`phone` đầy đủ (chỉ `buyer_name_masked`), thêm `seller_note` nếu có.

`POST /seller/orders/{id}/cancel` body `{reason,version}`. Response `200`:

```json
{ "data": { "order_id": "order-01912f91", "status": "CANCELLED", "reason": "SELLER_REQUEST", "version": 5 }, "meta": { "request_id": "01912fc2-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Phải có policy/refund/stock release theo `CancellationReason` (§5 LLD).

### 3.7-export `GET /seller/orders/export`

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

### 3.7a `PATCH /seller/orders/{id}/fulfill` — accept 1 đơn

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

### 3.7b `POST /seller/orders/bulk-accept` — accept hàng loạt

Phục vụ Penpot `Overlay / Bulk fulfillment`, `CTA / Xác nhận 24 đơn`. **Chỉ thực hiện `ACCEPT`** (`CONFIRMED → PROCESSING`); tạo shipment/chọn carrier vẫn làm riêng từng đơn qua §3.7a — action `SHIP` không có trong bulk.

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

### 3.7c Seller voucher CRUD (`/seller/vouchers`)

`POST /seller/vouchers` request:

```json
{
  "code": "SHOP10",
  "discount_type": "PERCENT",
  "value": 10,
  "max_discount_amount": 100000,
  "min_order": 200000,
  "usage_limit": 500,
  "starts_at": "2026-09-01T00:00:00Z",
  "expires_at": "2026-09-30T16:59:59Z"
}
```

Response `201`:

```json
{
  "data": {
    "voucher_id": "vch-01912fc3",
    "code": "SHOP10",
    "scope": "SHOP",
    "shop_id": "shop-01912f31",
    "discount_type": "PERCENT",
    "value": 10,
    "max_discount_amount": 100000,
    "min_order": 200000,
    "usage_limit": 500,
    "used_count": 0,
    "starts_at": "2026-09-01T00:00:00Z",
    "expires_at": "2026-09-30T16:59:59Z",
    "status": "ACTIVE",
    "version": 1
  },
  "meta": { "request_id": "01912fc4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`scope` luôn `SHOP` (cố định, không nhận từ request) và `shop_id` lấy từ token — seller không tạo `PLATFORM`/`FREESHIP` voucher. `max_discount_amount` chỉ dùng với `PERCENT`, `null` = không trần. **Không có** field audience/targeting/segment trong v1 — các màn Penpot Seller "Voucher audience / Selected users / Import user IDs / Tab followers/VIP" ngoài phạm vi v1. Trùng `code` trong cùng shop → `409 ORDER_VOUCHER_CODE_EXISTS`.

`GET /seller/vouchers` — query `status?`, `code?`, `page`, `size`:

```json
{
  "data": [
    { "voucher_id": "vch-01912fc3", "code": "SHOP10", "discount_type": "PERCENT", "value": 10, "used_count": 128, "usage_limit": 500, "status": "ACTIVE", "expires_at": "2026-09-30T16:59:59Z" }
  ],
  "meta": { "request_id": "01912fc5-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 4 }
}
```

`PUT /seller/vouchers/{id}` request cùng field với create cộng `version`; response `200` cùng shape với create. Không đổi `code` sau khi đã có redemption (tránh sai lệch lịch sử) — đổi `code` khi `used_count > 0` → `409 ORDER_VOUCHER_CODE_LOCKED`.

`DELETE /seller/vouchers/{id}` body `{version}`. Response `200`:

```json
{ "data": { "voucher_id": "vch-01912fc3", "status": "INACTIVE" }, "meta": { "request_id": "01912fc6-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Soft `INACTIVE`, không xóa redemption history.

### 3.8 Admin platform voucher (`/admin/vouchers`)

Quyền: `VOUCHER_MANAGE` (do Auth User RBAC cấp) + step-up 2FA cho mutation; Gateway coarse-gate role admin, `order-commerce` enforce permission. v1 không có microservice admin riêng — admin console gọi trực tiếp route này (`System_Overview.md` §6.3).

`POST /admin/vouchers` request:

```json
{
  "code": "TACA200K",
  "scope": "PLATFORM",
  "discount_type": "FIXED",
  "value": 200000,
  "min_order": 2000000,
  "usage_limit": 1000,
  "starts_at": "2026-09-01T00:00:00Z",
  "expires_at": "2026-09-30T16:59:59Z"
}
```

Response `201` cùng shape với `POST /seller/vouchers` nhưng `shop_id: null`. `scope` nhận `PLATFORM` hoặc `FREESHIP` (không nhận `SHOP` — admin không tạo voucher hộ shop). Voucher freeship là `scope=FREESHIP`, `discount_type` chỉ `PERCENT`/`FREESHIP`; nó chỉ trừ vào phí ship, không trừ tiền hàng — đây là mã kiểu `FREESHIPMAX` trên Penpot.

`GET /admin/vouchers` — query `status?`, `code?`, `scope?`, `page`, `size`:

```json
{
  "data": [
    { "voucher_id": "vch-01912fc7", "code": "TACA200K", "scope": "PLATFORM", "discount_type": "FIXED", "value": 200000, "used_count": 812, "usage_limit": 1000, "status": "ACTIVE", "expires_at": "2026-09-30T16:59:59Z" }
  ],
  "meta": { "request_id": "01912fc8-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 12 }
}
```

`PUT /admin/vouchers/{id}` cùng field với create cộng `version`, response `200` cùng shape. `DELETE /admin/vouchers/{id}` body `{version}` → `200 { data: { voucher_id, status: "INACTIVE" }, meta }`, soft `INACTIVE/ARCHIVED`, redemption history giữ nguyên. Validation, usage increment và scope-check khi checkout dùng chung code path với shop voucher.

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
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice pending. |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | `POST /checkout` tạo order sau Inventory reserve, Payment charge tách service. | Ảnh hưởng saga/compensation/payment UX. | Order/Payment/Inventory owner |
| 2 | Multi-shop checkout split order theo shop nhưng chung `checkout_group_id`. | Ảnh hưởng invoice/shipment/voucher. | Product owner |
| 3 | Shipping fee endpoint dùng Shipment mock/quote; exact carrier contract chưa có. | Ảnh hưởng total/timeout. | Shipment owner |
| 4 | ~~Voucher stacking/priority chưa chốt~~ → **Đã chốt**: 3 slot (`PLATFORM`/`SHOP`/`FREESHIP`), tối đa 1 mã mỗi slot, thứ tự shop→platform→freeship. Công thức đầy đủ + phân bổ + làm tròn ở `docs/lld/order-commerce.md` §3.2a. | — | Đã đóng |
| 5 | Seller cancel/refund policy chưa đủ trong HLD. | Ảnh hưởng inventory/payment compensation. | Product/Finance |
| 6 | Voucher v1 chỉ theo mã, không audience/targeting; các màn Penpot Seller voucher targeting ngoài v1. | Nếu bật targeting cần schema `voucher_audiences` + follow shop. | Product owner |
| 7 | Platform voucher CRUD đặt tại `order-commerce` (`/admin/vouchers`, `VOUCHER_MANAGE`). | Nếu phạm vi Admin tách service riêng, service đó gọi vào contract này chứ không sở hữu voucher aggregate. | Architecture |
