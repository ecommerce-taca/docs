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
| `CartController` | Cart CRUD cho buyer | `buyer_user_id` chỉ lấy từ `X-User-ID` do Gateway tin cậy inject; chặn request giả mạo header/tới service trực tiếp; không tin `user_id` trong body. |
| `InternalCartController` | `GET /internal/carts/{buyerUserId}`, `POST /internal/carts/{buyerUserId}/checked-out` | Bắt buộc `X-Internal-Service-Token` so sánh constant-time và network isolation; chỉ `checkout-service` gọi. |
| `CartService` | Gộp quantity khi thêm SKU trùng, cap 999, tối đa 100 dòng | Mutation CAS theo cart `version`; tăng `item_revision` khi quantity thực sự thay đổi, tránh Checkout xoá nhầm item mới sửa. |
| `ProductCatalogClient` | `GET /products/{productId}` cho mỗi product khác nhau trong giỏ, song song tối đa 10, cache 30s theo `product_id` | Không dùng batch `GET /products?product_ids=` (thiếu `skus[]`); timeout 2s/call, tổng hydrate 3s. Cache 30s chỉ phục vụ GET hydrate; POST validate Product/SKU bằng lần đọc mới, không dựa vào cache GET. GET response-only, Product timeout → snapshot cũ. |
| `InventoryClient` | `GET /internal/inventory/availability` read-only phục vụ hiển thị | Không dùng để reserve; timeout/lỗi → `availability_status=UNKNOWN` nếu Product hợp lệ, không làm `GET /cart` lỗi. Không gọi Inventory cho SKU đã xác định không hợp lệ; giới hạn fan-out và tuân thủ ngân sách hydrate 3s. DTO, cách gọi batch/từng SKU và giới hạn đồng thời cụ thể phải đối chiếu contract Inventory. |

### 2.1 Chuẩn observability

- Log JSON stdout với field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST propagate `traceparent`/`tracestate` và `X-Request-ID`.
- Không log Authorization, token nội bộ hoặc toàn bộ nội dung cart; chỉ log metadata cần thiết như `cart_id`, `sku_id`, số dòng (cân nhắc hash/mask ID theo chính sách privacy).
- Gateway phải strip `X-User-ID`/`X-User-Roles` từ client rồi inject context đã xác thực; Cart chỉ chấp nhận request buyer qua đường nội bộ tin cậy (network policy hoặc mTLS). Không public `/internal/**`, không tin header tự khai khi truy cập service trực tiếp. Internal token lấy từ secret store/env, so sánh fixed-length digest bằng constant-time và không log. Bật deny-by-default cho route ngoài danh sách.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MongoDB.
- Error envelope `{error:{code,message,details,trace_id}}`.

## 3. Luồng xử lý

### 3.1 Buyer thao tác giỏ

1. Gateway xác thực JWT, inject `X-User-ID`.
2. `GET /cart` đọc document theo `buyer_user_id`; chưa có thì trả giỏ rỗng (`cart_id=null`, `items=[]`, `item_count=0`, `updated_at=null`), không tạo document. Đọc một snapshot từ DB, nhóm theo distinct `product_id`; dùng Product cache 30s, tối đa 10 request song song, timeout 2s/request và **timeout tổng hydrate 3s** (kể cả Inventory). Inventory chỉ dùng thời gian còn lại trong budget chung, không fan-out không giới hạn; không mặc định endpoint Inventory hỗ trợ batch trước khi kiểm tra contract. Hết budget thì dừng fan-out và trả snapshot còn lại.
3. Hydrate **chỉ thay đổi response, không ghi MongoDB**, không tăng cart `version`/item `item_revision`/`updated_at`. Thứ tự xử lý cho từng SKU: Product 404/inactive hoặc SKU vắng trong `skus[]` → `available=false`, `availability_status=PRODUCT_UNAVAILABLE` (ưu tiên hơn mọi kết quả Inventory); Product timeout/5xx → giữ snapshot, `availability_status=STALE` (không đổi thành `UNKNOWN` vì Inventory lỗi); chỉ khi Product hợp lệ mới xét Inventory: đủ quantity → `AVAILABLE`/`available=true`, thiếu → `OUT_OF_STOCK`/`available=false`, lỗi/timeout/chưa kịp kiểm tra → `UNKNOWN` và giữ `available` best-effort. Khi `UNKNOWN`/`STALE`, `available` **không** khẳng định có stock. Nếu Product hydrate thành công, `unit_price_snapshot` trong **response** lấy giá SKU mới; nếu không thì giữ giá snapshot DB. `line_total = unit_price_snapshot` **trong response** `* quantity` bằng `long`, kiểm tra overflow; tuyệt đối không trộn giá mới với line total từ giá cũ. Mọi giá/availability chỉ để hiển thị; Checkout revalidate nguồn chính thức.
4. `POST /cart/items`: validate actor, body và `Idempotency-Key` (nếu có), tra key còn hiệu lực để replay trước khi gọi Product (cùng SHA-256 canonical body → trả nguyên response 201, khác body → 409). Request mới validate Product và SKU active/thuộc Product **trước khi mở transaction** bằng lần đọc mới (không dùng cache GET 30s), Product down → 503, không thay đổi cart. Không dùng Inventory để quyết định add. Có key: transaction MongoDB (replica set) đọc hoặc claim `_id={buyer_user_id,key}`; key còn hạn thì replay hoặc 409; key đã hết hạn nhưng chưa được TTL xoá thì **conditional replacement** với `expires_at <= now` ngay trong transaction; key chưa tồn tại thì insert. Trong transaction, đọc cart theo buyer, tính quantity `min(999, old+incoming)`, enforce max 100 dòng, cập nhật CAS `version` nếu trạng thái cart thực sự đổi, lưu response 201 vào key kể cả khi add bị cap 999 (cart no-op), rồi commit **nguyên tử**. Không có key: chỉ cần atomic CAS trên document cart; create cart bằng unique index buyer. Duplicate key/write conflict đã xác định chưa commit → abort, re-read và retry hữu hạn (đề xuất 3), hết budget → `409 CART_VERSION_CONFLICT`; lỗi Mongo → `503 CART_STORAGE_UNAVAILABLE`. Nếu kết quả commit **chưa rõ**, không được coi như write conflict thông thường: với key, xác minh kết quả bằng idempotency record và hash trước khi quyết định replay/retry; nếu chưa thể xác minh, trả lỗi storage và không tự phát mutation mới. Không có key, kết quả ghi chưa rõ thì **không tự retry POST** để tránh cộng đôi; client phải đọc lại giỏ trước khi quyết định gửi thao tác mới. FE nên gửi `Idempotency-Key` cho mỗi ý định add.
5. PUT/DELETE: tìm item trong đúng buyer, CAS `version`, re-read và re-evaluate khi conflict; PUT quantity giống cũ là no-op. `item_revision=1` lúc tạo item; tăng 1 khi POST/PUT làm quantity **thực sự đổi**; không tăng khi GET hydrate/no-op. Khi buyer DELETE item rồi thêm lại SKU, cấp `item_id` **mới**. CAS tránh lost update trong service nhưng **không** bảo đảm thứ tự ý định giữa các PUT từ nhiều thiết bị vì API v1 chưa nhận `expected_version` từ client (ghi nhận limitation, không tự thêm field). Availability từ Inventory chỉ dùng hiển thị, không giữ stock.

### 3.2 Checkout đọc giỏ và xoá item đã đặt

```text
checkout-service ──GET /internal/carts/{buyerUserId}?item_ids=──► cart-service
      (place checkout thành công)
checkout-service ──POST /internal/carts/{buyerUserId}/checked-out {cart_id, item_ids, checkout_group_id, expected_item_revisions}──► cart-service
      → xoá các item đó khỏi giỏ nếu còn (gọi lại an toàn)
```

- `GET /internal/carts/{buyerUserId}` không truyền `item_ids` → trả mọi item; truyền `?item_ids=` rỗng → `400`; danh sách 1–100 ID, loại trùng và giữ thứ tự. Item không thuộc giỏ trả trong `missing_item_ids`, không lỗi — Checkout quyết định `CHECKOUT_CART_EMPTY`. Internal GET trả `item_revision` của mỗi item; Checkout phải lưu revision của **snapshot thực sự dùng lúc place**.
- `checked-out` bắt buộc `expected_item_revisions` (map chính xác item_id → revision nguyên dương), không fallback xoá mù theo `item_id`. Với mỗi item: item không còn → `missing_item_ids`; revision khác (buyer sửa quantity khi đang checkout) → **giữ nguyên**, đưa vào `skipped_modified_item_ids`; revision trùng → xoá và đưa vào `removed_item_ids`. Áp dụng CAS `version` một lần cho cart; xung đột phải re-read/re-classify rồi retry. Nếu không xoá gì, không tăng version. Gọi lại (kể cả G1 → G2 → G1) trả `removed_item_ids=[]` nếu item trước đó đã xoá, không được xoá item buyer vừa sửa hoặc item mới tạo; Checkout không dựa vào response cleanup để quyết định đơn.
- Trường hợp buyer đổi quantity từ 2 lên 5 sau khi Checkout đọc revision cũ, Cart v1 **giữ cả dòng 5**, không tự trừ 2 (không suy luận ý định buyer). Đây là trade-off để tránh mất thao tác mới; FE nên cho buyer xem lại giỏ. `checkout_group_id` chỉ để trace/correlation, không có exactly-once ledger ở v1. Nếu `checked-out` thất bại, đơn vẫn hợp lệ; Checkout retry **nguyên payload** qua outbox/job. **Phải đồng bộ thay đổi internal contract với `checkout-docs` và Checkout caller trước integration**.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| `CART_MAX_ITEMS` | 100 | — | Số dòng, không phải tổng quantity. |
| `ITEM_QTY_MIN` | 1 | — | Integer. |
| `ITEM_QTY_MAX` | 999 | — | Mỗi SKU. |
| Cart TTL | Không có | — | Giỏ lưu lâu dài (quyết định 2026-11-01). |
| `PRODUCT_TIMEOUT` | 2s | `PRODUCT_TIMEOUT` | Mỗi Product detail call; lỗi khi GET dùng snapshot, khi POST trả 503. |
| `CART_HYDRATE_TIMEOUT` | 3s | `CART_HYDRATE_TIMEOUT` | Ngân sách tổng hydrate cho GET; hết hạn trả snapshot. |
| `PRODUCT_CACHE_TTL` | 30s | `PRODUCT_CACHE_TTL` | Cache Product detail theo `product_id` chỉ cho GET hydrate, giới hạn capacity, không cache vô hạn; POST validate bằng lần đọc mới. |
| `PRODUCT_MAX_CONCURRENCY` | 10 | `PRODUCT_MAX_CONCURRENCY` | Số Product detail request tối đa đồng thời. |
| `CART_IDEMPOTENCY_TTL` | 24h | `CART_IDEMPOTENCY_TTL` | Replay window, TTL index chỉ GC. |
| `CART_WRITE_RETRY_MAX` | 3 | `CART_WRITE_RETRY_MAX` | Retry bounded cho CAS/transaction transient conflicts. |
| `CART_INTERNAL_TOKEN` | — | `CART_INTERNAL_TOKEN` | Secret từ environment/secret store; constant-time compare. |
| `MONGODB_URI` | — | `MONGODB_URI` | Database `cartdb`, kết nối replica set cho transaction. |

## 5. Enum & trạng thái

Cart không có trạng thái: document tồn tại suốt đời buyer, item được thêm/xoá. Trước đây có `CartStatus` (`ACTIVE`, `EXPIRED`, `CHECKED_OUT`); bỏ từ 2026-11-01 vì giỏ không hết hạn và item đã đặt bị xoá khỏi giỏ. `availability_status` chỉ là enum **trong response GET**: `AVAILABLE`, `OUT_OF_STOCK`, `PRODUCT_UNAVAILABLE`, `STALE`, `UNKNOWN`; không phải CartStatus hay trạng thái bắt buộc lưu ở DB.

## 6. Event phát ra / lắng nghe

Cart không phát và không lắng nghe event Kafka.

| Gọi ra (REST) | Contract |
|---|---|
| Product Catalog `GET /products/{productId}` | `product-catalog-docs/docs/api/product-catalog.md` §3.1. Giá = `skus[].price.sale_price`, nhãn từ `skus[].attributes`; product 404/SKU vắng hoặc inactive → `PRODUCT_UNAVAILABLE`. Trạng thái active field cụ thể **phải đối chiếu contract Product**, không tự giả định; timeout GET dùng snapshot, timeout POST trả 503. Không dùng batch `/products?product_ids=` (thiếu `skus[]`). |
| Inventory `GET /internal/inventory/availability` | `inventory-docs/docs/api/inventory.md` #1, read-only. Dùng quantity đang hiển thị để xác định `AVAILABLE`/`OUT_OF_STOCK`; lỗi/timeout → `UNKNOWN`, không reserve. Chưa đủ thông tin trong 4 Cart docs để khẳng định DTO/path prefix thực tế; phải đối chiếu Inventory contract trước implement client. |

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
| `CART_STORAGE_UNAVAILABLE` | 503 | MongoDB không sẵn sàng/transaction lỗi hạ tầng. |
| `CART_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Đã chốt (2026-11-01): Cart là service riêng, MongoDB, không hết hạn. | — | Đã đóng |
| 2 | Repo `cart-service` chưa tạo. | Header doc cần cập nhật repo/nhánh/commit khi có code. | Tech lead |
| 3 | Giỏ không hết hạn nên dữ liệu tăng theo số buyer; chưa có job dọn giỏ của tài khoản bị xoá. | Có thể cần nghe `user.status_changed` hoặc job dọn theo yêu cầu privacy. | Product/Security |
| 4 | Giỏ của khách chưa đăng nhập (guest cart) ngoài v1. | Nếu cần, phải thêm merge giỏ khi login. | Product owner |
| 5 | `expected_item_revisions` trong API checked-out là thay đổi contract từ bản gốc. | Checkout caller/LLD và contract test phải được sửa đồng bộ trước tích hợp. | Checkout owner |
| 6 | Field active của Product, schema Inventory (DTO, batch/từng SKU, concurrency), route Gateway chưa được cung cấp cùng 4 tài liệu này. | P0: phải đối chiếu contract và ingress/network policy trước các phase tích hợp; không tự suy đoán hoặc tuyên bố đã tương thích. | Product/Inventory/Gateway owner |
| 7 | API PUT v1 không nhận `expected_version` từ client. | CAS chống lost update nội bộ nhưng không ngăn PUT đến muộn từ một thiết bị thay thế quantity mới hơn; giữ nguyên contract v1, cần Product owner xác nhận nếu muốn semantics mạnh hơn. | Product owner |
