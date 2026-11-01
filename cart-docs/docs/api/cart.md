# API Spec — Cart Service

> Nguồn: `docs/lld/cart.md` · `docs/db/cart.md` · `docs/test/cart.md` · Code: chưa có repo (`ecommerce-taca/cart-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Endpoint buyer cần JWT (Gateway verify). Endpoint `/internal/**` bắt buộc `X-Internal-Service-Token` khớp `CART_INTERNAL_TOKEN`, Gateway không expose. |
| Request ID / trace | `X-Request-ID` tối đa 64; W3C `traceparent`/`tracestate`. |
| Actor context | `X-User-ID`, `X-User-Roles` do Gateway inject (client không gửi được). |
| Timestamp | ISO-8601 UTC. |
| Money | Integer VND; giá trong cart chỉ để hiển thị. |
| Idempotency | Không cần cho buyer CRUD (thao tác tự nhiên idempotent theo `item_id`/SKU). `checked-out` idempotent theo `checkout_group_id`. |
| Response | `{data,meta:{request_id}}`; error `{error:{code,message,details,trace_id}}`. |
| Log | JSON theo LLD §2.1; không log token/nội dung giỏ. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /cart` | Buyer | Giỏ hiện tại (hydrate giá/tình trạng). | ⏳ |
| 2 | `POST /cart/items` | Buyer | Thêm SKU. | ⏳ |
| 3 | `PUT /cart/items/{itemId}` | Buyer | Đổi quantity. | ⏳ |
| 4 | `DELETE /cart/items/{itemId}` | Buyer | Xoá item. | ⏳ |
| 5 | `GET /internal/carts/{buyerUserId}` | Internal (`checkout-service`) | Đọc item đã chọn để preview/place. | ⏳ |
| 6 | `POST /internal/carts/{buyerUserId}/checked-out` | Internal (`checkout-service`) | Xoá item đã đặt hàng thành công. | ⏳ |
| 7 | `GET /health/live` | Internal/ops | Liveness process-only. | ⏳ |
| 8 | `GET /health/ready` | Internal/ops | Readiness MongoDB. | ⏳ |

## 3. Chi tiết endpoint

### 3.1 `GET /cart`

Trả giỏ của buyer; chưa có giỏ thì `200` với `items=[]` và `cart_id=null`.

```json
{
  "data": {
    "cart_id": "cart-01912f80",
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
    "updated_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "01912fbe-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`available:false` khi Product/SKU đã inactive/hết hàng kể từ lúc thêm — item vẫn hiển thị để buyer tự xoá, nhưng bị loại ở checkout preview/place. Giỏ không hết hạn nên không còn field `status`/`expires_at` (bỏ từ 2026-11-01).

### 3.2 `POST /cart/items`

Request `{product_id, sku_id, quantity}`. Response `201`:

```json
{ "data": { "cart_id": "cart-01912f80", "item_id": "item-1", "quantity": 2, "line_total": 498000 }, "meta": { "request_id": "01912fbf-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Validate Product active và quantity `1..999`; cart không reserve stock. SKU đã có → cộng dồn `quantity` (giữ `item_id`), cap `999`. Dòng thứ 101 → `409 CART_TOO_MANY_ITEMS`. Product down → `503 CART_DEPENDENCY_UNAVAILABLE`.

### 3.3 `PUT /cart/items/{itemId}`

Request `{quantity}`. Response `200`:

```json
{ "data": { "item_id": "item-1", "quantity": 3, "line_total": 747000 }, "meta": { "request_id": "01912fc0-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Quantity `0` không dùng update, phải DELETE.

### 3.4 `DELETE /cart/items/{itemId}`

Trả `204`; item không thuộc giỏ của buyer → `404 CART_ITEM_NOT_FOUND` (anti-enumeration).

### 3.5 `GET /internal/carts/{buyerUserId}?item_ids=`

Gọi bởi `checkout-service` ở preview và place. `item_ids` phân tách bằng dấu phẩy, 1–100 phần tử.

```json
{
  "data": {
    "cart_id": "cart-01912f80",
    "buyer_user_id": "user-01912f10",
    "items": [
      { "item_id": "item-1", "product_id": "product-01912f31", "sku_id": "sku-01912f33", "shop_id": "shop-01912f30", "quantity": 2, "unit_price_snapshot": 249000 }
    ],
    "missing_item_ids": []
  },
  "meta": { "request_id": "01912fc1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Item không có trong giỏ trả ở `missing_item_ids`, không lỗi. `unit_price_snapshot` chỉ để Checkout phát hiện `PRICE_CHANGED`; giá chốt luôn đọc lại từ Product.

### 3.6 `POST /internal/carts/{buyerUserId}/checked-out`

```json
{ "cart_id": "cart-01912f80", "checkout_group_id": "group-01912f90", "item_ids": ["item-1", "item-2"] }
```

Response `200`:

```json
{ "data": { "cart_id": "cart-01912f80", "removed_item_ids": ["item-1", "item-2"], "remaining_item_count": 3 }, "meta": { "request_id": "01912fc2-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Idempotent theo `checkout_group_id`: gọi lại trả cùng `removed_item_ids`. `cart_id` không thuộc `buyerUserId` → `404 CART_ITEM_NOT_FOUND`.

## 4. Mã lỗi chung

⏳ Chưa có code: mọi mã là thiết kế.

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `CART_INVALID_INPUT` | 400 | Payload/query sai. |
| `CART_UNAUTHENTICATED` | 401 | Thiếu login hoặc token nội bộ. |
| `CART_ITEM_NOT_FOUND` | 404 | Item/cart không thuộc buyer. |
| `CART_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active. |
| `CART_TOO_MANY_ITEMS` | 409 | Vượt 100 dòng. |
| `CART_VERSION_CONFLICT` | 409 | Ghi đồng thời. |
| `CART_DEPENDENCY_UNAVAILABLE` | 503 | Product down khi thêm item. |
| `CART_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Path buyer giữ nguyên `/api/v1/cart/**` như thời Order-Commerce, chỉ đổi upstream ở Gateway. | FE không phải đổi URL. | Đã chốt |
| 2 | Bỏ `status`/`expires_at` khỏi response vì giỏ không hết hạn. | FE nào đang đọc hai field này phải bỏ. | FE owner |
