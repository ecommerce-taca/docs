# LLD — Cart Service

> Nguồn: `docs/api/cart.md` · `docs/db/cart.md` · `docs/test/cart.md` · HLD `EcommercePlatform-v4(6).excalidraw` · Code: chưa có repo (`ecommerce-taca/cart-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Tech stack đã chốt: Java 25 · Spring Boot · MongoDB 8.x (Spring Data MongoDB) · REST với Product Catalog/Inventory · không Kafka

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Giỏ hàng của buyer: get/add/update/delete item; cung cấp item đã chọn cho Checkout; xoá item sau khi đặt hàng thành công. |
| Database | MongoDB `cartdb`, collection `carts`, một document mỗi buyer. Giỏ được **lưu lâu dài, không hết hạn** (quyết định 2026-11-01). |
| Không thuộc service | Giữ stock (Inventory reserve chỉ xảy ra ở `POST /checkout` của `checkout-service`), giá chốt và voucher (Checkout snapshot lại), order. |

### 1.2 Boundary

```text
mfe-buyer ──Gateway /api/v1/cart/**──► cart-service ──► MongoDB cartdb.carts
                                          ├─ REST Product Catalog: GET /products/{productId} (hydrate + validate SKU)
                                          └─ REST Inventory: GET /internal/inventory/availability (read-only)
checkout-service ──REST /internal/carts/**──► cart-service
```

- Cart không phải order source of truth và **không** là reservation kho.
- Giá/tiêu đề trong cart chỉ là cache hiển thị; `checkout-service` luôn đọc lại Product khi preview/place.
- Không service nào khác ghi vào `cartdb`.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Order (Cart) | Get/add/update/delete item | Tách thành `cart-service` riêng (2026-11-01) để scale độc lập lúc sale. |
| Buyer Cart | Giỏ còn nguyên khi quay lại, đổi thiết bị | MongoDB durable, không TTL. |

## 2. Cấu trúc bên trong

⏳ Repo chưa có code; cây package dưới đây là thiết kế.

```text
com.taca.cart
├── api/                  # CartController (buyer), InternalCartController (checkout)
├── application/          # CartService: add/update/delete/hydrate/checked-out
├── domain/               # Cart, CartItem, rules quantity/max items
├── integrations/         # ProductCatalogClient, InventoryClient
├── persistence/          # CartRepository (Spring Data MongoDB)
├── security/             # actor context, internal token filter
└── observability/        # JSON log/trace/metric contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `CartController` | Cart CRUD cho buyer | `buyer_user_id` lấy từ `X-User-ID`; không tin `user_id` trong body. |
| `InternalCartController` | `GET /internal/carts/{buyerUserId}`, `POST /internal/carts/{buyerUserId}/checked-out` | Bắt buộc `X-Internal-Service-Token`; chỉ `checkout-service` gọi. |
| `CartService` | Gộp quantity khi thêm SKU trùng, cap 999, tối đa 100 dòng | Update optimistic theo `version` của document. |
| `ProductCatalogClient` | `GET /products/{productId}` cho mỗi product khác nhau trong giỏ, song song tối đa 10, cache 30s theo `product_id` | Batch `GET /products?product_ids=` không có `skus[]` nên không dùng. Timeout → vẫn trả cart với snapshot cũ, `available` giữ giá trị cuối. |

### 2.1 Chuẩn observability

- Log JSON stdout với field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST propagate `traceparent`/`tracestate` và `X-Request-ID`.
- Không log Authorization, token nội bộ hoặc toàn bộ nội dung cart; chỉ log `cart_id`, `sku_id`, số dòng.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MongoDB.
- Error envelope `{error:{code,message,details,trace_id}}`.

## 3. Luồng xử lý

### 3.1 Buyer thao tác giỏ

1. Gateway xác thực JWT, inject `X-User-ID`.
2. `GET /cart` đọc document theo `buyer_user_id`; chưa có thì trả giỏ rỗng (không tạo document).
3. Hydrate: gọi `GET /products/{productId}` cho các `product_id` khác nhau trong giỏ (song song, cache 30s), cập nhật `title_snapshot`, `sku_label_snapshot`, `unit_price_snapshot`, `available`. Product lỗi/timeout → trả snapshot đang lưu, không lỗi.
4. `POST /cart/items` có `Idempotency-Key` đã thấy trong `add_idempotency` → trả response đã lưu, không cộng. Sau đó validate Product/SKU active và quantity `1..999`; SKU đã có → cộng dồn, cap `999`; vượt `CART_MAX_ITEMS` dòng → `409 CART_TOO_MANY_ITEMS`. Document tạo lần đầu ở thao tác thêm đầu tiên (upsert).
5. Availability từ Inventory chỉ để hiển thị; không coi là reservation.

### 3.2 Checkout đọc giỏ và xoá item đã đặt

```text
checkout-service ──GET /internal/carts/{buyerUserId}?item_ids=──► cart-service
      (place checkout thành công)
checkout-service ──POST /internal/carts/{buyerUserId}/checked-out {cart_id, item_ids, checkout_group_id}──► cart-service
      → xoá các item đó khỏi giỏ nếu còn (gọi lại an toàn)
```

- Item không thuộc giỏ của buyer bị bỏ qua (trả trong `missing_item_ids`), không lỗi — Checkout quyết định `CHECKOUT_CART_EMPTY`.
- `checked-out` gọi lại (kể cả sau một group khác) chỉ xoá item còn tồn tại, nên không xoá thừa; `removed_item_ids` có thể là `[]` và Checkout không dựa vào nó.
- Nếu `checked-out` thất bại, đơn vẫn hợp lệ; `checkout-service` retry (outbox job). Item còn lại trong giỏ chỉ là trải nghiệm, không ảnh hưởng tiền.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| `CART_MAX_ITEMS` | 100 | — | Số dòng, không phải tổng quantity. |
| `ITEM_QTY_MIN` | 1 | — | Integer. |
| `ITEM_QTY_MAX` | 999 | — | Mỗi SKU. |
| Cart TTL | Không có | — | Giỏ lưu lâu dài (quyết định 2026-11-01). |
| `PRODUCT_TIMEOUT` | 2s | ⏳ chưa có | Hydrate; lỗi thì dùng snapshot cũ. |
| `CART_INTERNAL_TOKEN` | — | ⏳ chưa có | So constant-time với `X-Internal-Service-Token`. |
| `MONGODB_URI` | — | ⏳ chưa có | Database `cartdb`. |

## 5. Enum & trạng thái

Cart không có trạng thái: document tồn tại suốt đời buyer, item được thêm/xoá. Trước đây có `CartStatus` (`ACTIVE`, `EXPIRED`, `CHECKED_OUT`); bỏ từ 2026-11-01 vì giỏ không hết hạn và item đã đặt bị xoá khỏi giỏ.

## 6. Event phát ra / lắng nghe

Cart không phát và không lắng nghe event Kafka.

| Gọi ra (REST) | Contract |
|---|---|
| Product Catalog `GET /products/{productId}` | `product-catalog-docs/docs/api/product-catalog.md` §3.1. Map field giống `checkout-docs/docs/lld/checkout.md` §6.4: giá = `skus[].price.sale_price`, `sku_label_snapshot` từ `skus[].attributes`, SKU không có trong `skus[]` → `available=false`. |
| Inventory `GET /internal/inventory/availability` | `inventory-docs/docs/api/inventory.md` #1, read-only. |

## 7. Mã lỗi

⏳ Chưa có code: mọi mã dưới đây là thiết kế.

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `CART_INVALID_INPUT` | 400 | Payload/query sai, quantity ngoài `1..999`. |
| `CART_UNAUTHENTICATED` | 401 | Thiếu login hoặc thiếu/sai token nội bộ. |
| `CART_ITEM_NOT_FOUND` | 404 | Item không có trong giỏ của buyer (anti-enumeration). |
| `CART_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active khi thêm. |
| `CART_TOO_MANY_ITEMS` | 409 | Vượt `CART_MAX_ITEMS` dòng. |
| `CART_VERSION_CONFLICT` | 409 | Ghi đồng thời; client tải lại giỏ. |
| `CART_IDEMPOTENCY_CONFLICT` | 409 | `POST /cart/items` dùng lại `Idempotency-Key` với body khác. |
| `CART_DEPENDENCY_UNAVAILABLE` | 503 | Product down khi **thêm** item (không dùng cho GET). |
| `CART_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Đã chốt (2026-11-01): Cart là service riêng, MongoDB, không hết hạn. | — | Đã đóng |
| 2 | Repo `cart-service` chưa tạo. | Header doc cần cập nhật repo/nhánh/commit khi có code. | Tech lead |
| 3 | Giỏ không hết hạn nên dữ liệu tăng theo số buyer; chưa có job dọn giỏ của tài khoản bị xoá. | Có thể cần nghe `user.status_changed` hoặc job dọn theo yêu cầu privacy. | Product/Security |
| 4 | Giỏ của khách chưa đăng nhập (guest cart) ngoài v1. | Nếu cần, phải thêm merge giỏ khi login. | Product owner |
