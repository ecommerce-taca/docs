# LLD — Cart Service

> Nguồn: `docs/api/cart.md` · `docs/db/cart.md` · `docs/test/cart.md` · HLD `EcommercePlatform-v4(6).excalidraw` · Cập nhật: `2026-11-17`
> Tech stack: Java 25 · Spring Boot 4 · MongoDB (replica set cho transaction; integration tests dùng `mongo:7.0`) · REST Product Catalog/Inventory · không Kafka.

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
                                          ├─ REST Product Catalog: GET /api/v1/products/{productId} (GET hydration và POST validation trên hai read path riêng)
                                          └─ REST Inventory: GET /internal/inventory/availability?sku_ids=... (read-only, service đích chưa có endpoint)
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

Các class nêu dưới đây là những thành phần liên quan trực tiếp đến Cart API; không phải toàn bộ cây source.

```text
com.taca.cart
├── api/                   # CartController, internal API, response DTO/error
├── application/           # CartQueryService, mutation/idempotency services
│                           # CartHydrationOrchestrator, CartProductResponseHydrator
│                           # CartInventoryResponseHydrator, CachedProductHydrationService
│                           # ProductHydrationResult, InventoryStock
├── domain/                # Cart, CartItem, quantity/version/item_revision rules
├── integrations/          # ProductCatalogClient (POST fresh validation)
│                           # ProductHydrationClient/HttpProductHydrationClient (GET)
│                           # InventoryAvailabilityClient/HttpInventoryAvailabilityClient (GET)
├── persistence/           # CartCasPersistence, Mongo documents/mappers/indexes
├── config/                # CartProperties, ProductProperties, InventoryProperties
├── security/              # Gateway buyer/internal actor context và access control
└── observability/         # Request ID, health, logs/metrics
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `CartController` | Cart CRUD cho buyer | `buyer_user_id` chỉ lấy từ `X-User-ID` do Gateway inject; request còn phải có `X-Gateway-Service-Token` khớp `CART_GATEWAY_TOKEN` (64 hex). V1 không đọc `X-User-Roles` hay `user_id` trong body. |
| `InternalCartController` | `GET /internal/carts/{buyerUserId}`, `POST /internal/carts/{buyerUserId}/checked-out` | Bắt buộc `X-Internal-Service-Token` so sánh constant-time và network isolation; chỉ `checkout-service` gọi. |
| `CartAddItemService`, `CartUpdateItemService`, `CartRemoveItemService`, `CartMutationService`, idempotency services | POST/PUT/DELETE, quantity cap, buyer isolation, CAS/transaction + replay | Mutation tách biệt GET hydrate; GET không persist giá hay availability mới. |
| `ProductCatalogClient` / `HttpProductCatalogClient` | Validate Product và SKU cho POST bằng HTTP request mới | Không dùng cache GET; lỗi dependency là `503`, Product/SKU không bán được là `409`. |
| `ProductHydrationClient` / `HttpProductHydrationClient` | GET `/api/v1/products/{productId}`, parse Product `ACTIVE`, các active SKU, giá VND, nhãn attributes | `FOUND`, `UNAVAILABLE` hoặc `STALE`; lỗi 404/inactive → unavailable, timeout/HTTP lỗi/schema sai → stale. |
| `CachedProductHydrationService` | In-memory cache 30s theo `product_id`, single-flight theo key, `Semaphore` giới hạn HTTP calls đang chạy | Cache `FOUND`/`UNAVAILABLE` nhưng không cache `STALE`; capacity giới hạn xấp xỉ 10.000 entries mỗi instance. |
| `CartHydrationOrchestrator` | Phân biệt product IDs, chạy song song có giới hạn, dùng chung budget GET, gọi Inventory cho SKU đã validate | Mặc định product stage ≤2s, hydrate tổng 3s; partial results và fallback khi hết budget. |
| `CartProductResponseHydrator` / `CartInventoryResponseHydrator` | Dựng `CartResponse` từ snapshot trong RAM | Chỉ cập nhật DTO; Product status có ưu tiên hơn Inventory. |
| `InventoryAvailabilityClient` / `HttpInventoryAvailabilityClient` | GET read-only batch `?sku_ids=` tối đa 100 SKU, header `X-Internal-Service-Token` | HTTP timeout mặc định 1s; semaphore 10 calls/instance; trả map rỗng khi lỗi/unconfigured; **chưa xác minh tương thích với Inventory endpoint thật**. |

### 2.1 Chuẩn observability

- Log JSON stdout với field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST propagate `traceparent`/`tracestate` và `X-Request-ID`.
- Không log Authorization, token nội bộ hoặc toàn bộ nội dung cart; chỉ log metadata cần thiết như `cart_id`, `sku_id`, số dòng (cân nhắc hash/mask ID theo chính sách privacy).
- Gateway phải strip `X-User-ID`/`X-User-Roles` từ client rồi inject context đã xác thực. Cart yêu cầu thêm `X-Gateway-Service-Token` khớp `CART_GATEWAY_TOKEN` (64 hex) trước khi tin `X-User-ID`; `X-User-Roles` hiện không được dùng bởi Cart v1. Không public `/internal/**`, không tin header tự khai khi truy cập service trực tiếp. Internal token lấy từ secret store/env, so sánh fixed-length digest bằng constant-time và không log. Bật deny-by-default cho route ngoài danh sách.
- `/health/live` không phụ thuộc DB; `/health/ready` kiểm MongoDB. Cả hai route hiện `permitAll` để phục vụ probe. `/actuator/prometheus` yêu cầu Bearer token `CART_MONITORING_TOKEN` (64 hex).
- Error envelope `{error:{code,message,details,trace_id}}`.

### 2.2 GET Cart Hydration

```text
GET /api/v1/cart
  -> CartController (buyer authentication, no-store, request_id)
  -> CartQueryService: CartCasPersistence.findByBuyerUserId(buyer)
  -> CartHydrationOrchestrator.hydrate(Optional<Cart>)
      -> distinct product_id -> CachedProductHydrationService
          -> cache 30s / single-flight / semaphore -> HttpProductHydrationClient
      -> CartProductResponseHydrator (response-only, ưu tiên Product)
      -> chỉ SKU có availability_status=UNKNOWN sau Product hydrate
          -> InventoryAvailabilityClient.fetchAvailability(Set<sku_id>) theo batch
      -> CartInventoryResponseHydrator -> CartResponse
  -> HTTP 200 (không ghi MongoDB)
```

- **Empty cart:** buyer chưa có cart trả `cart_id=null`, `items=[]`, `item_count=0`, `updated_at=null` và không tạo document; cart tồn tại nhưng rỗng giữ `cart_id`/`updated_at` và bỏ qua dependency calls.
- **Product:** cache in-memory theo product ID, TTL tính từ lúc fetch xong; cache `FOUND` và `UNAVAILABLE`, không cache `STALE`. Single-flight gộp cache miss đồng thời của cùng Product trên một JVM. `PRODUCT_MAX_CONCURRENCY=10` là giá trị mặc định và phải cấu hình không lớn hơn 10 nếu cần bảo toàn giới hạn 10 HTTP calls toàn instance; property hiện không tự chặn số >10. Việc xếp lịch tối đa 10 tác vụ Product mỗi GET nằm ở orchestrator, HTTP concurrency trên mỗi instance nằm ở semaphore của cached service.
- **Deadline:** `CART_HYDRATE_TIMEOUT=3s` tính từ lúc bắt đầu hydrate sau MongoDB read; product stage có trần từ `PRODUCT_TIMEOUT=2s` và dành phần ngân sách cho Inventory. Các `Future` chưa xong bị cancel/interrupt và item tương ứng fallback; hủy `Future` không đảm bảo HTTP call đang chạy dừng ngay. Đây là budget cho hydration dependency, không phải SLA cứng cho toàn bộ request kể cả MongoDB, scheduling và JSON serialization.
- **Inventory:** chỉ gọi sau Product validation với các SKU tương ứng `UNKNOWN`; batch `sku_ids` distinct, mặc định tối đa 100 ID/request (`INVENTORY_MAX_BATCH_SIZE`). Inventory client có timeout 1s/request, giới hạn concurrency mặc định 10 và không cache stock. Thiếu URL/token, HTTP lỗi, timeout, schema sai hoặc thiếu SKU → không có thông tin xác minh → `UNKNOWN` cho item Product hợp lệ. Status Inventory `ACTIVE` + đủ quantity → `AVAILABLE`, thiếu → `OUT_OF_STOCK`; `INACTIVE`/`DISABLED` → `OUT_OF_STOCK`, status không rõ → `UNKNOWN`.
- **Độ nhất quán:** read một Cart snapshot rồi hydrate ngoài transaction. GET và PUT/DELETE có thể chạy đồng thời: GET được trả snapshot đã đọc trước mutation, nhưng tuyệt đối không được ghi đè mutation. Product price/title/label chỉ cập nhật field mang hậu tố `*_snapshot` trong **response**; `line_total = unit_price_snapshot * quantity` từ chính response. Checkout phải revalidate giá và reserve stock từ source of truth.
- **Giới hạn tích hợp:** Inventory availability chưa có endpoint nghiệp vụ trên `inventory-service`; cần consumer/provider contract test và xác minh secret/network identity trước khi kết nối môi trường thực tế.

## 3. Luồng xử lý

### 3.1 Buyer thao tác giỏ

1. Gateway xác thực JWT, strip/inject `X-User-ID` và gửi `X-Gateway-Service-Token` khớp `CART_GATEWAY_TOKEN`; Cart không dùng `X-User-Roles` ở v1.
2. `GET /cart` (`GET /api/v1/cart` trên wire) đọc **một** Cart snapshot từ MongoDB theo buyer; không có document → trả giỏ rỗng không tạo document. Sau đó orchestrator lấy các `product_id` khác nhau, gọi Product read path có cache 30s, single-flight và giới hạn concurrency; Product stage mặc định tối đa 2s, hydrate tổng mặc định 3s. Sau Product mapping, chỉ các SKU có Product hợp lệ mới được gửi Inventory qua một hoặc nhiều batch `sku_ids` (mặc định tối đa 100 ID/batch); các batch dùng ngân sách hydration còn lại. Các Product chưa trả lời đúng deadline giữ snapshot và `STALE`.
3. Hydrate **chỉ thay đổi response, không ghi MongoDB**, không tăng cart `version`, item `item_revision`, hay `updated_at`. Product 404/inactive hoặc SKU không còn active/vắng trong `skus[]` → `PRODUCT_UNAVAILABLE` và `available=false` (ưu tiên cao hơn Inventory); Product timeout/5xx/schema invalid hoặc thiếu cấu hình → `STALE`, giữ giá và `available` từ snapshot. Product hợp lệ + Inventory đủ quantity → `AVAILABLE`/`true`; thiếu quantity hoặc SKU inventory bị disable → `OUT_OF_STOCK`/`false`; Inventory lỗi/timeout/chưa xác minh → `UNKNOWN` với hint `available` từ snapshot. Khi Product hợp lệ, response dùng title/label/giá mới và tính lại `line_total` từ giá mới nhân quantity; trường hợp Product stale/unavailable giữ snapshot. `UNKNOWN`/`STALE` không bảo đảm còn stock; Checkout luôn kiểm lại.
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
| `PRODUCT_MAX_CONCURRENCY` | 10 | `PRODUCT_MAX_CONCURRENCY` | Semaphore trên một JVM; cần đặt ≤10 để giữ đúng giới hạn thiết kế; orchestrator tối đa 10 tasks/GET. |
| `PRODUCT_BASE_URL` | rỗng | `PRODUCT_BASE_URL` | Chưa cấu hình → GET Product `STALE`; POST cần validation fresh và fail closed. |
| `INVENTORY_BASE_URL` | rỗng | `INVENTORY_BASE_URL` | URL Inventory private; chưa cấu hình → Product hợp lệ nhưng stock `UNKNOWN`. |
| `INVENTORY_INTERNAL_TOKEN` | rỗng | `INVENTORY_INTERNAL_TOKEN` | Secret outbound riêng của Cart→Inventory; header `X-Internal-Service-Token`, cần khớp secret của Inventory. |
| `INVENTORY_TIMEOUT` | 1s | `INVENTORY_TIMEOUT` | Timeout HTTP Inventory batch, nằm trong hydrate budget chung. |
| `INVENTORY_MAX_BATCH_SIZE` | 100 | `INVENTORY_MAX_BATCH_SIZE` | 1–100 SKU/batch theo Inventory API contract. |
| `INVENTORY_MAX_CONCURRENCY` | 10 | `INVENTORY_MAX_CONCURRENCY` | Semaphore outbound Inventory calls/instance; orchestrator hiện xử lý batches trong GET tuần tự. |
| `CART_IDEMPOTENCY_TTL` | 24h | `CART_IDEMPOTENCY_TTL` | Replay window, TTL index chỉ GC. |
| `CART_WRITE_RETRY_MAX` | 3 | `CART_WRITE_RETRY_MAX` | Retry bounded cho CAS/transaction transient conflicts. |
| `CART_INTERNAL_TOKEN` | — | `CART_INTERNAL_TOKEN` | Secret từ environment/secret store; constant-time compare. |
| `CART_GATEWAY_TOKEN` | — | `CART_GATEWAY_TOKEN` | Secret 64 hex giữa Gateway và Cart cho mọi buyer route. |
| `CART_MONITORING_TOKEN` | — | `CART_MONITORING_TOKEN` | Bearer secret 64 hex cho `GET /actuator/prometheus`. |
| `MONGODB_URI` | — | `MONGODB_URI` | Database `cartdb`, kết nối replica set cho transaction. |

## 5. Enum & trạng thái

Cart không có trạng thái: document tồn tại suốt đời buyer, item được thêm/xoá. Trước đây có `CartStatus` (`ACTIVE`, `EXPIRED`, `CHECKED_OUT`); bỏ từ 2026-11-01 vì giỏ không hết hạn và item đã đặt bị xoá khỏi giỏ. `availability_status` chỉ là enum **trong response GET**: `AVAILABLE`, `OUT_OF_STOCK`, `PRODUCT_UNAVAILABLE`, `STALE`, `UNKNOWN`; không phải CartStatus hay trạng thái bắt buộc lưu ở DB.

## 6. Event phát ra / lắng nghe

Cart không phát và không lắng nghe event Kafka.

| Gọi ra (REST) | Contract |
|---|---|
| Product Catalog `GET /api/v1/products/{productId}` | `product-catalog-docs/docs/api/product-catalog.md` §3.1. GET read dùng `ProductHydrationClient` và Product cache; POST fresh validation dùng `HttpProductCatalogClient`. Giá SKU từ `skus[].price.sale_price`; inactive/404/vắng SKU → `PRODUCT_UNAVAILABLE`; lỗi/timeout GET → `STALE`. |
| Inventory `GET /internal/inventory/availability?sku_ids=...` | Contract `inventory-docs/docs/api/inventory.md` §3.1, `data[]` với `sku_id`, `available_qty`, `status`, `as_of`; read-only batch tối đa 100, token nội bộ. Adapter Cart đã có và kiểm thử bằng HTTP mock; Inventory Service chưa có endpoint nghiệp vụ để xác minh contract end-to-end. |

## 7. Mã lỗi

Các mã lỗi dưới đây là contract của Cart API đã triển khai; dependency lỗi khi GET hydrate chuyển thành status trên item, không trả `CART_DEPENDENCY_UNAVAILABLE` cho GET.

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `CART_INVALID_INPUT` | 400 | Payload/query sai, quantity ngoài `1..999`. |
| `CART_UNAUTHENTICATED` | 401 | Thiếu/không hợp lệ `X-User-ID`, gateway token, internal token hoặc monitoring token. |
| `CART_FORBIDDEN` | 403 | Principal đã xác thực nhưng truy cập route bị deny. |
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
| 3 | Giỏ không hết hạn nên dữ liệu tăng theo số buyer; chưa có job dọn giỏ của tài khoản bị xoá. | Có thể cần nghe `user.status_changed` hoặc job dọn theo yêu cầu privacy. | Product/Security |
| 4 | Giỏ của khách chưa đăng nhập (guest cart) ngoài v1. | Nếu cần, phải thêm merge giỏ khi login. | Product owner |
| 5 | `expected_item_revisions` trong API checked-out là thay đổi contract từ bản gốc. | Checkout caller/LLD và contract test phải được sửa đồng bộ trước tích hợp. | Checkout owner |
| 6 | Product read adapter đã kiểm tra Product `ACTIVE`/SKU fields theo Product contract. Inventory batch request/response đã triển khai phía Cart theo tài liệu, nhưng endpoint Inventory thật chưa sẵn sàng để contract test. | Cần xác minh đường dẫn, encoding `sku_ids`, auth secret và behavior khi missing SKU cùng Inventory owner trước khi bật tích hợp production. | Product/Inventory/Gateway owner |
| 7 | API PUT v1 không nhận `expected_version` từ client. | CAS chống lost update nội bộ nhưng không ngăn PUT đến muộn từ một thiết bị thay thế quantity mới hơn; giữ nguyên contract v1, cần Product owner xác nhận nếu muốn semantics mạnh hơn. | Product owner |
