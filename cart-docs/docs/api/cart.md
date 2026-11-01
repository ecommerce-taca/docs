# API Spec — Cart Service

> Nguồn: `docs/lld/cart.md` · `docs/db/cart.md` · `docs/test/cart.md` · Code: chưa có repo (`ecommerce-taca/cart-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Endpoint buyer cần JWT (Gateway verify) và chỉ nhận actor context do Gateway đã xác thực. Endpoint `/internal/**` bắt buộc `X-Internal-Service-Token` khớp `CART_INTERNAL_TOKEN`, chỉ truy cập mạng nội bộ, Gateway không expose. Chặn truy cập trực tiếp và header giả mạo. |
| Request ID / trace | `X-Request-ID` tối đa 64; W3C `traceparent`/`tracestate`. |
| Actor context | `X-User-ID`, `X-User-Roles` do Gateway **xoá header client gửi rồi inject lại**; Cart chỉ tin request đến từ Gateway được xác thực ở network/service identity, không tin `user_id` trong body. Thiếu/không hợp lệ `X-User-ID` → `401`. |
| Timestamp | ISO-8601 UTC. |
| Money | Integer VND; giá trong cart chỉ để hiển thị. |
| Idempotency | `PUT`/`DELETE` idempotent về **trạng thái dữ liệu** (DELETE lặp lại có thể trả `404`). `POST /cart/items` **cộng dồn** nên nhận `Idempotency-Key` tuỳ chọn (§3.2). `checked-out` an toàn khi gọi lại nếu gửi đúng `expected_item_revisions` (§3.6). |
| Response | `{data,meta:{request_id}}`; error `{error:{code,message,details,trace_id}}`; riêng `DELETE` thành công trả `204` không có body. |
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
    "cart_id": "019a0011-2222-7333-8444-555566667777",
    "items": [
      {
        "item_id": "019a0011-2222-7333-8444-555566667778",
        "product_id": "product-01912f31",
        "sku_id": "sku-01912f33",
        "shop_id": "shop-01912f30",
        "title_snapshot": "Áo khoác cotton",
        "sku_label_snapshot": "Đen / L",
        "unit_price_snapshot": 249000,
        "quantity": 2,
        "line_total": 498000,
        "available": true,
        "availability_status": "AVAILABLE"
      }
    ],
    "item_count": 1,
    "updated_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "01912fbe-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`GET /cart` hydrate theo từng `product_id` (tối đa 10 Product request đồng thời, timeout 2s/request, ngân sách tổng 3s kể cả Inventory; Product cache 30s), **chỉ cập nhật response, không ghi MongoDB**. Theo thứ tự ưu tiên cho mỗi SKU: Product 404/inactive hoặc SKU không còn trong `skus[]` → `available=false`, `availability_status=PRODUCT_UNAVAILABLE`, không cần hỏi Inventory; Product lỗi/timeout → giữ snapshot, `availability_status=STALE` (không bị ghi đè thành `UNKNOWN`); nếu Product hợp lệ mới xét Inventory: thiếu quantity → `OUT_OF_STOCK`/`available=false`, đủ → `AVAILABLE`/`available=true`, lỗi/timeout/chưa xác minh → `UNKNOWN` và giữ `available` best-effort. Inventory dùng budget còn lại, không fan-out không giới hạn; batch/từng SKU phải theo contract Inventory đã xác minh. Giá/nhãn Product hydrate thành công chỉ cập nhật các field `*_snapshot` **trong response**; giá không hydrate được giữ giá từ DB. `line_total` luôn tính bằng `unit_price_snapshot` **trong response** nhân `quantity` (int64, kiểm tra overflow), không trộn giá mới với tổng tiền cũ. Các trạng thái và giá chỉ để hiển thị, Checkout luôn tự kiểm tra lại Product/Inventory và không được dựa vào giá hoặc `available` trong Cart. Chưa có giỏ: `item_count=0`, `updated_at=null`. Giỏ không hết hạn nên không có `status`/`expires_at`.

### 3.2 `POST /cart/items`

Headers: `Idempotency-Key` tuỳ chọn (1–128 ký tự ASCII in được, không chứa khoảng trắng/control). Request `{product_id, sku_id, quantity}`; không nhận `user_id`/`buyer_user_id`. Response `201`:

```json
{ "data": { "cart_id": "019a0011-2222-7333-8444-555566667777", "item_id": "019a0011-2222-7333-8444-555566667778", "quantity": 2, "line_total": 498000 }, "meta": { "request_id": "01912fbf-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Validate quantity nguyên `1..999`, Product active và SKU thuộc Product/còn active **trước MongoDB transaction**, bằng lần đọc Product mới (không dựa vào cache hydrate GET 30s); Product 404/SKU sai, inactive → `409 CART_PRODUCT_UNAVAILABLE`; Product timeout/5xx → `503 CART_DEPENDENCY_UNAVAILABLE` và không sửa cart. Không gọi Inventory để quyết định add, không reserve stock. SKU đã có → cộng dồn `quantity` (giữ `item_id`), cap `999` bằng `min(999, hiện tại + quantity)`; khi đã ở 999, add là no-op về cart (nếu có key vẫn commit record idempotency/response 201). Dòng thứ 101 → `409 CART_TOO_MANY_ITEMS`.

Có `Idempotency-Key`: phạm vi `(buyer_user_id, key)`, request hash SHA-256 của JSON canonical `{product_id,sku_id,quantity}`; trong 24h cùng key/cùng body trả lại **HTTP 201 và toàn bộ body gốc kể cả `meta.request_id`**, không cộng thêm; khác body → `409 CART_IDEMPOTENCY_CONFLICT`. Key đã hết hạn được phép sử dụng lại kể cả khi TTL MongoDB chưa dọn (conditional replacement); hai request đồng thời cùng key chỉ tạo một mutation. Không gửi key → mỗi lần gọi đều cộng dồn; FE **nên gửi key cho mỗi ý định add** và giữ nguyên key khi retry cùng thao tác. Nếu POST không key bị mất response/không rõ đã ghi hay chưa, **không tự retry**, client đọc lại `GET /cart` trước khi quyết định thao tác mới. Dữ liệu giỏ và response idempotency được commit nguyên tử trong MongoDB transaction; ghi đồng thời dùng CAS `version` và retry có giới hạn khi đã biết write conflict chưa commit. Với keyed POST có kết quả commit chưa rõ, Cart kiểm tra record key/hash trước khi replay/retry; không xác minh được vì storage lỗi → `503 CART_STORAGE_UNAVAILABLE`, không phát mutation mới (§3.1 LLD).

### 3.3 `PUT /cart/items/{itemId}`

Request `{quantity}`. Response `200`:

```json
{ "data": { "item_id": "019a0011-2222-7333-8444-555566667778", "quantity": 3, "line_total": 747000 }, "meta": { "request_id": "01912fc0-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Quantity `0` không dùng update, phải DELETE. Quantity phải là số nguyên `1..999`; PUT thay thế quantity, không cộng dồn. Nếu quantity không đổi thì không tăng `version`/`item_revision`; nếu đổi thì tăng `item_revision` để bảo vệ thay đổi của buyer khi Checkout dọn giỏ. Nếu ghi đụng độ và đã retry có giới hạn vẫn không thành công → `409 CART_VERSION_CONFLICT`. V1 không nhận `expected_version` từ client: CAS chống lost update nội bộ nhưng không đảm bảo PUT từ một thiết bị đến muộn không thay thế quantity được đổi trên thiết bị khác; đây là limitation được ghi nhận, **không thêm field API ở lần chỉnh này**.

### 3.4 `DELETE /cart/items/{itemId}`

Trả `204`; item không thuộc giỏ của buyer hoặc đã bị xoá → `404 CART_ITEM_NOT_FOUND` (anti-enumeration). DELETE là idempotent về trạng thái, nhưng lần gọi lại không bắt buộc trả cùng HTTP status. Xoá bằng CAS `version`, không xoá nhầm item của buyer khác.

### 3.5 `GET /internal/carts/{buyerUserId}?item_ids=`

Gọi bởi `checkout-service` ở preview, place và khi tính voucher (`validate`, `GET /vouchers` có `cart_id`). `item_ids` **tuỳ chọn**, phân tách bằng dấu phẩy; **không truyền tham số** → trả mọi item trong giỏ (`missing_item_ids=[]`), **truyền `?item_ids=` nhưng rỗng** → `400 CART_INVALID_INPUT`. Khi truyền danh sách: 1–100 ID hợp lệ, từ chối phần tử rỗng, loại trùng ID nhưng giữ thứ tự xuất hiện. Giỏ chưa có document → `cart_id=null`, `items=[]`, không `404`. Checkout phải dùng bản snapshot mới nhất được lấy cho place và lưu `item_revision` tương ứng.

```json
{
  "data": {
    "cart_id": "019a0011-2222-7333-8444-555566667777",
    "buyer_user_id": "user-01912f10",
    "items": [
      { "item_id": "019a0011-2222-7333-8444-555566667778", "product_id": "product-01912f31", "sku_id": "sku-01912f33", "shop_id": "shop-01912f30", "quantity": 2, "item_revision": 1, "unit_price_snapshot": 249000 }
    ],
    "missing_item_ids": []
  },
  "meta": { "request_id": "01912fc1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Item không có trong giỏ trả ở `missing_item_ids`, không lỗi. `item_revision` là số nguyên dương, bắt buộc trong internal response để Checkout nhận biết item buyer đã sửa. `unit_price_snapshot` chỉ để Checkout phát hiện `PRICE_CHANGED`; giá chốt luôn đọc lại từ Product. Đọc từ MongoDB primary để giảm nguy cơ đọc bản sao chậm khi place.

### 3.6 `POST /internal/carts/{buyerUserId}/checked-out`

```json
{ "cart_id": "019a0011-2222-7333-8444-555566667777", "checkout_group_id": "group-01912f90", "item_ids": ["019a0011-2222-7333-8444-555566667778"], "expected_item_revisions": {"019a0011-2222-7333-8444-555566667778": 1} }
```

Response `200`:

```json
{ "data": { "cart_id": "019a0011-2222-7333-8444-555566667777", "removed_item_ids": ["019a0011-2222-7333-8444-555566667778"], "missing_item_ids": [], "skipped_modified_item_ids": [], "remaining_item_count": 3 }, "meta": { "request_id": "01912fc2-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Chỉ Checkout gọi **sau khi place/order thành công**. `expected_item_revisions` **bắt buộc** chứa đúng các `item_id` trong `item_ids` (1–100 phần tử duy nhất), revision nguyên dương; thiếu/mismatch → `400 CART_INVALID_INPUT`. Cart chỉ xoá item còn tồn tại với revision trùng snapshot Checkout đã dùng; item được buyer sửa sau đó phải được giữ lại và ghi vào `skipped_modified_item_ids`; item vắng trả trong `missing_item_ids`. `removed_item_ids` chỉ liệt kê item thực sự bị xoá. Mutation dùng CAS `version` và re-evaluate khi conflict. Gọi lại cùng payload có thể trả `removed_item_ids=[]`, không xoá item mới; `checkout_group_id` dùng để trace, **không** là ledger exactly-once. Nếu Cart tạm lỗi, Checkout retry giữ nguyên payload; kết quả order không phụ thuộc cleanup. `cart_id` không thuộc `buyerUserId` → `404 CART_ITEM_NOT_FOUND`. **Đây là thay đổi contract cần đồng bộ Checkout caller và `checkout-docs` trước integration**.

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
| `CART_IDEMPOTENCY_CONFLICT` | 409 | Cùng `Idempotency-Key` khác body ở `POST /cart/items`. |
| `CART_DEPENDENCY_UNAVAILABLE` | 503 | Product down khi thêm item. |
| `CART_STORAGE_UNAVAILABLE` | 503 | MongoDB không sẵn sàng hoặc transaction lỗi hạ tầng. |
| `CART_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Path buyer giữ nguyên `/api/v1/cart/**` như thời Order-Commerce, chỉ đổi upstream ở Gateway. | FE không phải đổi URL. | Đã chốt |
| 2 | Bỏ `status`/`expires_at` khỏi response vì giỏ không hết hạn. | FE nào đang đọc hai field này phải bỏ. | FE owner |
| 3 | `expected_item_revisions` bắt buộc trong checked-out để tránh xoá item buyer vừa sửa. | Thay đổi request/response internal; cần đồng bộ Checkout LLD và caller trước khi deploy. | Checkout owner |
| 4 | Product/Inventory contract thực tế chưa nằm trong 4 tài liệu Cart. | P0: phải kiểm tra field active, shape availability request/response (batch hay từng SKU), Gateway mapping và network policy trước implement adapter/tích hợp; không tự suy đoán schema. | Product/Inventory/Gateway owner |
