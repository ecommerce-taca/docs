# API Spec — Order-Commerce Service · Cart

> Nguồn: `docs/lld/cart.md` · `docs/db/cart.md` · `docs/test/cart.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

Phần **Cart** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/api/order-commerce.md`.

## 1. Quy ước API chung

Theo `docs/api/order-commerce.md` §1 (base path `/api/v1`, auth qua Gateway, envelope `{data,meta}`, error `{error:{code,message,details,trace_id}}`, `Idempotency-Key` cho mutation, tiền integer VND).

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /cart` | Buyer | Cart hiện tại. | ⏳ |
| 2 | `POST /cart/items` | Buyer | Thêm SKU. | ⏳ |
| 3 | `PUT /cart/items/{itemId}` | Buyer | Đổi quantity. | ⏳ |
| 4 | `DELETE /cart/items/{itemId}` | Buyer | Xóa item. | ⏳ |

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

## 4. Mã lỗi chung

`ORDER_INVALID_INPUT`, `ORDER_UNAUTHENTICATED`, `ORDER_CART_ITEM_NOT_FOUND`, `ORDER_PRODUCT_UNAVAILABLE` — định nghĩa ở `docs/api/order-commerce.md` §4.

## 5. Giả định & câu hỏi mở

Xem `docs/api/order-commerce.md` §5.
