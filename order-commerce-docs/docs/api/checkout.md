# API Spec — Order-Commerce Service · Checkout

> Nguồn: `docs/lld/checkout.md` · `docs/test/checkout.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

Phần **Checkout** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/api/order-commerce.md`.

## 1. Quy ước API

Theo `docs/api/order-commerce.md` §1 (base path `/api/v1`, auth qua Gateway, envelope `{data,meta}`, error `{error:{code,message,details,trace_id}}`, `Idempotency-Key` cho mutation, tiền integer VND).

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 6 | `GET /checkout/shipping-fee` | Buyer | Preview shipping fee. | ⏳ |
| 7 | `POST /checkout/preview` | Buyer | Tính preview tổng tiền. | ⏳ |
| 8 | `POST /checkout` | Buyer + Idempotency | Reserve Inventory và tạo order split. | ⏳ |

## 3. Chi tiết endpoint

### 3.1 `GET /checkout/shipping-fee`

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

`voucher_codes` là mảng, tối đa **3 phần tử**, mỗi phần tử thuộc một `scope` khác nhau (`PLATFORM` / `SHOP` / `FREESHIP`) — xem mô hình 3 slot ở `docs/lld/voucher.md` §3.2. Giỏ nhiều shop được gửi nhiều mã `SHOP` (mỗi shop một mã), khi đó mảng dài hơn 3.

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

## 4. Mã lỗi chung

`ORDER_FORBIDDEN`, `ORDER_CART_EMPTY`, `ORDER_PRODUCT_UNAVAILABLE`, `ORDER_PRICE_CHANGED`, `ORDER_STOCK_UNAVAILABLE`, `ORDER_RESERVATION_EXPIRED`, `ORDER_VOUCHER_INVALID`, `ORDER_VOUCHER_SLOT_CONFLICT`, `ORDER_VOUCHER_LIMIT_REACHED`, `ORDER_ADDRESS_INVALID`, `ORDER_IDEMPOTENCY_CONFLICT`, `ORDER_DEPENDENCY_UNAVAILABLE` — định nghĩa ở `docs/api/order-commerce.md` §4.

## 5. Giả định & câu hỏi mở

Xem `docs/api/order-commerce.md` §5.
