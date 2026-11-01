# API Spec — Checkout Service

> Nguồn: `docs/lld/checkout.md` · `docs/db/checkout.md` · `docs/test/checkout.md` · Code: chưa có repo (`ecommerce-taca/checkout-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

Tài liệu này là **tổng quan** API của Checkout Service và chi tiết endpoint checkout. Chi tiết endpoint voucher ở `docs/api/voucher.md`.

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Endpoint buyer/seller/admin cần JWT (Gateway verify), riêng `GET /vouchers` auth tuỳ chọn. Endpoint `/internal/**` bắt buộc `X-Internal-Service-Token` khớp `CHECKOUT_INTERNAL_TOKEN`; Gateway không expose. |
| Request ID | `X-Request-ID` tối đa 64; Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate`; propagate sang Cart/Order/Product/Inventory/Shipment/Auth và Kafka. |
| Actor context | `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`, `X-MFA-Step-Up` do Gateway inject (client không gửi được). |
| Timestamp | ISO-8601 UTC; DB UTC `DATETIME(6)`. |
| Money | Integer VND, `currency=VND`; không float. |
| Pagination | `page` từ 1, `size` mặc định 20, max 100; meta trả `request_id,page,size,total,total_pages`. |
| Idempotency | `Idempotency-Key` bắt buộc với `POST /checkout`, voucher mutation và recalculate; giữ 24h. |
| Response | `{data,meta:{request_id}}`; error `{error:{code,message,details,trace_id}}`. |
| Log | JSON theo LLD §2.1; không log token/address/payload giỏ. |
| Retry | Gateway không retry mutation; service dùng saga/idempotency. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /checkout/shipping-fee` | Buyer | Preview shipping fee. | ⏳ |
| 2 | `POST /checkout/preview` | Buyer | Tính preview tổng tiền. | ⏳ |
| 3 | `POST /checkout` | Buyer + Idempotency | Giữ voucher, reserve Inventory, tạo đơn ở `order-service`. | ⏳ |
| 4 | `GET /vouchers/validate` | Buyer | Preview một mã trên giỏ. | ⏳ |
| 5 | `GET /vouchers` | Public (auth tuỳ chọn) | Danh sách voucher khả dụng. | ⏳ |
| 6 | `GET /vouchers/redemptions` | Buyer | Lịch sử voucher đã dùng + tổng tiết kiệm. | ⏳ |
| 7 | `POST /seller/vouchers` | Seller | Tạo shop voucher. | ⏳ |
| 8 | `GET /seller/vouchers` | Seller | List voucher của shop. | ⏳ |
| 9 | `PUT /seller/vouchers/{voucherId}` | Seller | Sửa voucher. | ⏳ |
| 10 | `DELETE /seller/vouchers/{voucherId}` | Seller | Soft inactivate. | ⏳ |
| 11 | `POST /admin/vouchers` | Admin (`VOUCHER_MANAGE`) | Tạo platform/freeship voucher. | ⏳ |
| 12 | `GET /admin/vouchers` | Admin (`VOUCHER_MANAGE`) | List + usage metric. | ⏳ |
| 13 | `PUT /admin/vouchers/{voucherId}` | Admin (`VOUCHER_MANAGE`) | Sửa voucher. | ⏳ |
| 14 | `DELETE /admin/vouchers/{voucherId}` | Admin (`VOUCHER_MANAGE`) | Soft inactivate/archive. | ⏳ |
| 15 | `POST /internal/checkout-groups/{checkoutGroupId}/recalculate` | Internal (`order-service`) | Tính lại voucher khi huỷ đơn con. | ⏳ |
| 16 | `GET /health/live` | Internal/ops | Liveness process-only (Gateway active healthcheck). | ⏳ |
| 17 | `GET /health/ready` | Internal/ops | Readiness MySQL/Kafka. | ⏳ |

Chi tiết #4–#14 ở `docs/api/voucher.md`. Path buyer/seller/admin giữ nguyên như thời Order-Commerce; Gateway chỉ đổi upstream.

## 3. Chi tiết endpoint

### 3.1 `GET /checkout/shipping-fee`

| Query | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `from_shop_id` | string | Có | — |
| `to_address_id` | string | Có | Phải thuộc buyer, nếu không → `403 CHECKOUT_FORBIDDEN` |
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

Giá lấy từ Shipment `POST /api/v1/internal/shipping/quote` (`shipment-docs/docs/api/shipment.md` §3.2; header `X-Internal-Service-Token`). Dependency down → `503 CHECKOUT_DEPENDENCY_UNAVAILABLE` (UI phải cho retry, không tự đặt fee = 0).

### 3.2 `POST /checkout/preview`

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

### 3.3 `POST /checkout`

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

`voucher_codes` là mảng theo mô hình 3 slot ở `docs/lld/voucher.md` §3.2: tối đa 1 mã `PLATFORM`, tối đa 1 mã `FREESHIP`, và tối đa 1 mã `SHOP` **cho mỗi shop** trong `selected_item_ids`. Vì vậy độ dài tối đa là `2 + số shop được chọn` (giỏ 1 shop → tối đa 3; giỏ 3 shop → tối đa 5). Vượt giới hạn này luôn có hai mã cùng slot → `400 CHECKOUT_VOUCHER_SLOT_CONFLICT`.

| Lỗi | HTTP | Khi nào |
|---|---:|---|
| `CHECKOUT_VOUCHER_SLOT_CONFLICT` | 400 | Hai mã cùng `scope` (2 mã `PLATFORM`, hoặc 2 mã `SHOP` cho cùng một shop) |
| `CHECKOUT_VOUCHER_INVALID` | 400 | Mã không tồn tại / hết hạn / sai trạng thái |

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

Checkout đọc lại Product/giá, giữ lượt voucher, reserve Inventory rồi gọi `order-service` `POST /internal/orders` (saga ở `docs/lld/checkout.md` §3.3). Reserve fail không tạo đơn và nhả voucher; retry cùng `Idempotency-Key` trả cùng response. `order_id`, `order_number`, `status`, `payment_status`, `payment_next_action` lấy nguyên từ response của `order-service`.

Với COD, `order-service` gọi Inventory `commit` ngay khi tạo đơn (không có bước chờ tiền), nên `reservation_expires_at` là `null` và đơn không bị job payment-timeout đụng tới. FE **không** được hiển thị đồng hồ đếm ngược thanh toán cho đơn COD.

Nếu Payment-Wallet lỗi khi `order-service` tạo payment cho đơn VNPAY, đơn vẫn được tạo và `payment_next_action.type = "RETRY_LATER"`; FE hiển thị đơn chờ thanh toán, đơn tự huỷ khi hết hạn (`docs/lld/checkout.md` §3.3).

### 3.4 `POST /internal/checkout-groups/{checkoutGroupId}/recalculate`

Gọi bởi `order-service` sau khi huỷ một hoặc nhiều đơn con. Headers: `X-Internal-Service-Token`, `Idempotency-Key` (`cancel-{order_id}`).

```json
{
  "cancelled_order_ids": ["order-01912f92"],
  "remaining_orders": [
    { "order_id": "order-01912f91", "payment_captured": false }
  ]
}
```

Response `200`:

```json
{
  "data": {
    "checkout_group_id": "group-01912f90",
    "orders": [
      {
        "order_id": "order-01912f91",
        "amounts": {
          "subtotal": 800000,
          "shop_discount": 300000,
          "platform_discount": 0,
          "shipping_fee": 30000,
          "freeship_discount": 30000,
          "discount": 300000,
          "tax": 45455,
          "grand_total": 500000
        },
        "applied_vouchers": [
          { "scope": "SHOP", "code": "APPLE300K", "discount_amount": 300000 },
          { "scope": "FREESHIP", "code": "FREESHIPMAX", "discount_amount": 30000 }
        ],
        "absorbed_amount": 0,
        "adjustment": "VOUCHER_REMOVED"
      }
    ],
    "released_vouchers": [
      { "code": "TACA200K", "scope": "PLATFORM", "reason": "VOUCHER_REMOVED" }
    ]
  },
  "meta": { "request_id": "01912fd0-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

- `payment_captured=true` và voucher bị gỡ → `grand_total` giữ nguyên số cũ, phần chênh trả ở `absorbed_amount` (sàn chịu).
- `remaining_orders` rỗng (huỷ cả nhóm) → `orders=[]`, mọi voucher trong `released_vouchers`.
- Group không tồn tại → `404 CHECKOUT_GROUP_NOT_FOUND`; gọi lặp cùng key trả cùng response; cùng key khác body → `409 CHECKOUT_IDEMPOTENCY_CONFLICT`.

Quy tắc tính ở `docs/lld/voucher.md` §3.3; ví dụ trên là nhóm golden `CO-CALC-01` sau khi huỷ đơn Shop B, giả định `TACA200K` có `min_order` 900.000: base còn 500.000 nên voucher bị gỡ; `FREESHIPMAX` vẫn đủ điều kiện, giảm hết 30.000 phí ship còn lại.

## 4. Mã lỗi chung

Bảng đầy đủ ở `docs/lld/checkout.md` §7 (prefix `CHECKOUT_`, đổi từ `ORDER_` ngày 2026-11-01). Endpoint checkout trả: `CHECKOUT_FORBIDDEN`, `CHECKOUT_CART_EMPTY`, `CHECKOUT_PRODUCT_UNAVAILABLE`, `CHECKOUT_PRICE_CHANGED`, `CHECKOUT_STOCK_UNAVAILABLE`, `CHECKOUT_RESERVATION_EXPIRED`, `CHECKOUT_VOUCHER_INVALID`, `CHECKOUT_VOUCHER_SLOT_CONFLICT`, `CHECKOUT_VOUCHER_LIMIT_REACHED`, `CHECKOUT_ADDRESS_INVALID`, `CHECKOUT_IDEMPOTENCY_CONFLICT`, `CHECKOUT_DEPENDENCY_UNAVAILABLE`.

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | `POST /checkout` tạo đơn qua `order-service` sau Inventory reserve; Payment do `order-service` gọi. | Ảnh hưởng saga/compensation/payment UX. | Order/Payment/Inventory owner |
| 2 | Multi-shop checkout split đơn theo shop, chung `checkout_group_id`. | Ảnh hưởng invoice/shipment/voucher. | Product owner |
| 3 | Shipping fee dùng Shipment quote; carrier thật chưa có adapter. | Ảnh hưởng total/timeout. | Shipment owner |
| 4 | Đã chốt: 3 slot voucher (`PLATFORM`/`SHOP`/`FREESHIP`), thứ tự shop→platform→freeship (`docs/lld/checkout.md` §3.2). | — | Đã đóng |
| 5 | Seller cancel/refund policy chưa đủ trong HLD. | Ảnh hưởng recalculate/compensation. | Product/Finance |
