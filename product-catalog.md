# LLD — Product Catalog Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Node.js + NestJS · MongoDB/Mongoose · Kafka outbox · S3/MinIO cho media · REST nội bộ với Inventory/Search/Auth User

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Quản lý product/SPU, SKU/variant, dynamic attributes, category tree, product media metadata, displayed price, seller product CRUD, publish/unpublish, admin catalog action và shop snapshot. |
| Nguồn dữ liệu chính | MongoDB database riêng của `product-catalog`; không tạo cross-service foreign key. MongoDB chỉ lưu product/catalog state và các projection phục vụ đọc. |
| Tồn kho | `inventory` là source of truth duy nhất cho available/reserved/committed stock và thao tác trừ/reserve. Product không được decrement, reserve hoặc kết luận stock cuối cùng cho checkout. |
| Giá | `product-catalog` là source of truth cho `base_price`/`sale_price` hiển thị. Order-Commerce đọc giá qua API/event và lưu price snapshot khi checkout/order; voucher/discount rule thuộc Order-Commerce. |
| Publish | Seller có thể tạo/cập nhật draft không cần product approval/censor. Publish yêu cầu shop đã `KYC_APPROVED`; không có trạng thái `PENDING_APPROVAL`. |
| Inventory display | Product giữ read-only inventory projection từ event của Inventory để hiển thị `Còn hàng/Hết hàng` hoặc số lượng snapshot. Projection có thể eventual consistent và không dùng để trừ stock. |
| Search | Product phát event sau commit; `search` consume event và sở hữu search index. Product không đọc trực tiếp Elasticsearch. |
| Không thuộc service | User/profile/KYC decision, inventory deduction/reservation, cart/checkout/order, voucher calculation, payment, shipment, review, message và search index. |
| Được gọi bởi | Buyer Web, Seller Center và Admin Console qua API Gateway; Order/Inventory/Search dùng REST/event contract, không đọc MongoDB trực tiếp. |

### 1.2 Boundary với các service khác

```text
Buyer / Seller / Admin frontend
  │ HTTPS
  ▼
API Gateway
  │ internal REST
  ▼
Product Catalog
  ├─ MongoDB: product, SKU, category, media, price, shop snapshot
  ├─ S3/MinIO: product media bytes qua signed URL
  ├─ Kafka outbox → Search, Order, Inventory và consumers khác
  ├─ Kafka consumer ← Inventory stock snapshot
  └─ Kafka consumer ← Auth User shop/KYC status event
```

- Product Catalog chỉ tin `shop_id` và local shop/KYC projection để route nhanh; ownership cuối cùng phải được kiểm tra từ auth context và policy service.
- `shop_id`, `sku_id`, `category_id` và `user_id` từ service khác là reference ID, không phải cross-service FK.
- Product có thể hiển thị sản phẩm `ACTIVE` với stock bằng 0; trạng thái stock chỉ ảnh hưởng khả năng mua ở Cart/Checkout, không tự làm Product trừ stock.
- Nếu Inventory không phát event hoặc projection bị stale, Product phải đánh dấu snapshot stale; không được suy diễn rằng còn hàng để reserve.

### 1.3 Mapping với HLD và Penpot

| Nguồn | Màn hình/requirement | Quyết định trong Product Catalog |
|---|---|---|
| HLD — Product Service | Product CRUD, SPU/SKU, category, publish/unpublish. | Product document sở hữu catalog content, variant, display price và lifecycle. |
| Penpot — Buyer | Home, search, category landing, shop, product detail. | Public GET trả product/card/detail; stock hiển thị từ Inventory projection và có `as_of`. |
| Penpot — Seller | Seller dashboard, Products, SPU/SKU editor, product first step trong onboarding. | Seller tạo draft, thêm dynamic attribute/SKU, media, price và publish khi KYC đạt. |
| Penpot — Admin | Categories, Products/SKU, shop/KYC, catalog settings. | Admin quản lý taxonomy, xem catalog, block/unblock sau publish; không duyệt trước từng sản phẩm. |
| Penpot — States | Loading, empty, error, validation, offline. | API trả error code ổn định; mutation dùng version để chống ghi đè; đọc snapshot phải thể hiện stale nếu cần. |
| Penpot — KYC | KYC lock publish/withdraw. | Product chỉ gate publish theo local shop status `APPROVED`; Product không review KYC document. |
| Penpot — SKU | Không giới hạn business field ở color/size. | Attribute definitions typed và dynamic; variant dimension do product/category khai báo, không hard-code tên thuộc tính. |

## 2. Cấu trúc bên trong

### 2.1 Module trong NestJS

```text
src/
├── product/                 # SPU aggregate, title, description, slug, status, price
├── sku/                     # variant dimensions, typed attributes, canonical key
├── category/                # category tree, product-category assignment
├── media/                   # metadata, signed upload/complete adapter
├── shop-projection/         # shop + KYC status snapshot từ Auth User events
├── inventory-projection/    # read-only stock snapshot từ Inventory events
├── publish-policy/          # required fields, KYC gate, lifecycle transition
├── catalog-query/           # public/seller/admin read models and pagination
├── outbox/                  # transactionally persisted domain events
├── integrations/            # Kafka, S3/MinIO, Auth User and Inventory contracts
└── security/                # actor scope, seller ownership, admin permission
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `ProductController` | Buyer/public product list, detail, shop/category listing. | Chỉ trả product có visibility phù hợp; không trả draft/blocked cho public. |
| `SellerProductController` | Seller tạo/sửa/xóa draft, SKU, media, publish/unpublish. | Lấy actor/shop scope từ Gateway; kiểm tra ownership trong service, không tin `shop_id` body. |
| `AdminCatalogController` | Category CRUD, catalog read, block/unblock product, audit metadata. | Destructive action yêu cầu `CATALOG_ADMIN`/`SUPER_ADMIN` và step-up 2FA từ auth-user. |
| `ProductApplicationService` | Orchestrate create/update/publish/archive và transaction boundary. | Mỗi mutation ghi product change + outbox event theo cùng MongoDB transaction khi deployment hỗ trợ replica set. |
| `VariantResolver` | Validate dynamic attributes, tạo canonical combination key, phát hiện duplicate SKU. | Không hard-code `color`, `size`; không cho hai SKU trong cùng SPU có cùng normalized combination. |
| `PublishPolicy` | Kiểm tra required fields, active SKU, media, price, category và shop KYC projection. | Không gọi Inventory để decrement; stock zero không chặn publish. |
| `CategoryPolicy` | Kiểm tra tree depth, parent state, cycle và product assignment. | Category inactive/archived không nhận product assignment mới. |
| `MediaService` | Tạo signed upload URL, validate complete metadata, lưu media metadata. | File bytes nằm ở S3/MinIO; Product chỉ lưu object key/checksum/content type. |
| `ShopProjectionService` | Consume shop created/updated/status event và cập nhật local snapshot. | Không gọi Auth User trên mỗi product read. |
| `InventoryProjectionService` | Consume stock snapshot event và cập nhật read-only display projection. | Projection không phải inventory ledger và không được dùng để confirm checkout. |
| `CatalogQueryService` | Query public/seller/admin, pagination, filter status/category/shop. | Search full-text nâng cao thuộc Search Service; Product chỉ hỗ trợ query catalog cơ bản. |
| `OutboxPublisher` | Publish product/category/SKU event sau commit. | Retry 3 lần, backoff 2 giây, sau đó dead-letter; consumer dedupe theo `event_id`. |

### 2.2 MongoDB collections ở mức LLD

Chi tiết field type, validator, index và migration nằm ở `docs/db/product-catalog.md`; LLD chỉ chốt ownership và field nghiệp vụ chính.

| Collection | Field chính | Index cần có |
|---|---|---|
| `products` | `_id`, `shop_id`, `title`, `slug`, `description`, `brand`, `status`, `price_summary`, `primary_category_id`, `shop_snapshot`, `version`, timestamps | `(shop_id,status,updated_at)`, unique `(shop_id,slug)`, category/status, public listing time |
| `skus` hoặc embedded `products.skus` | `sku_id`, `product_id`, `seller_sku`, `attributes`, `variant_key`, `price_override`, `status`, media refs | Unique `(product_id,variant_key)`, `(product_id,status)`, unique seller SKU theo shop |
| `attribute_definitions` | `product_id/category_id`, `key`, `label`, `type`, `is_variant_dimension`, `allowed_values`, `sort_order` | Unique scope + key, product/category scope |
| `categories` | `_id`, `parent_id`, `name`, `slug`, `path`, `depth`, `status`, `sort_order` | Unique slug, parent/status, materialized path |
| `product_categories` | `product_id`, `category_id`, `is_primary`, `assigned_at` | Unique product/category, category/product, primary partial unique |
| `product_media` | `media_id`, `product_id`, `sku_id?`, `scope`, `object_key`, `content_type`, `size_bytes`, `sha256`, `sort_order`, `is_cover`, `status` | Product/scope/status, unique checksum/object key |
| `shop_snapshots` | `shop_id`, `name`, `slug`, `logo_url`, `kyc_status`, `shop_status`, `source_version`, `updated_at` | Unique shop ID, status |
| `inventory_projections` | `sku_id`, `product_id`, `available_qty_snapshot`, `stock_status`, `as_of`, `source_event_id` | Unique SKU, product/status, `as_of` |
| `outbox_events` | `event_id`, `aggregate_type`, `aggregate_id`, `event_type`, `schema_version`, `payload`, `published_at`, `attempt_count` | Published flag/time, aggregate, event type |
| `catalog_audits` | `actor_user_id`, `shop_id?`, `action`, `target_type`, `target_id`, `reason`, `metadata`, `occurred_at` | Target/time, actor/time, action |

### 2.3 Aggregate và ownership

| Aggregate/read model | Source of truth | Product Catalog được làm gì |
|---|---|---|
| Product/SPU | Product Catalog | Tạo, cập nhật, lifecycle, displayed fields, base/sale price. |
| SKU/variant | Product Catalog | Tạo/update status/attributes/price override; phát SKU event. |
| Category | Product Catalog | Quản lý tree và product assignment. |
| Product media metadata | Product Catalog | Validate metadata/object; bytes do S3/MinIO lưu. |
| Shop identity/KYC | Auth User | Lưu snapshot để đọc/gate; không quyết định KYC. |
| Available/reserved stock | Inventory | Chỉ consume snapshot; không reserve/deduct. |
| Search index | Search | Phát event; không ghi Elasticsearch. |
| Voucher/discount | Order-Commerce | Product trả giá niêm yết; không tính voucher. |
| Order price | Order-Commerce | Product cung cấp giá hiện tại; Order snapshot giá tại checkout. |

### 2.4 Dynamic attributes và canonical SKU

Attribute không giới hạn ở tên field cố định. Mỗi attribute definition có:

| Field | Ý nghĩa |
|---|---|
| `key` | Machine key lowercase `a-z0-9_`, unique trong product/category scope. |
| `label` | Nhãn hiển thị tiếng Việt hoặc locale tương ứng. |
| `type` | `STRING`, `NUMBER`, `BOOLEAN`, `ENUM`. |
| `is_variant_dimension` | Nếu `true`, value tham gia tạo SKU combination. |
| `allowed_values` | Bắt buộc với `ENUM`; có thể rỗng với type khác. |
| `unit` | Đơn vị hiển thị tùy attribute, ví dụ `kg`, `cm`; không dùng để đổi precision ngầm. |
| `sort_order` | Thứ tự hiển thị trong Product Detail/editor. |

Ví dụ SKU:

```json
{
  "sku_id": "sku-01912f31",
  "attributes": {
    "material": "cotton",
    "capacity_liter": 2.0,
    "is_insulated": true,
    "pattern": "striped"
  },
  "variant_key": "material=cotton|capacity_liter=2|is_insulated=true|pattern=striped"
}
```

Canonicalization:

1. Chỉ lấy attribute có `is_variant_dimension=true`.
2. Sort key theo Unicode code point, normalize string trim/lowercase theo type.
3. Serialize `key=value` bằng escaping chuẩn; number dùng decimal canonical, không float binary.
4. Join bằng `|` tạo `variant_key`.
5. Unique trong cùng `product_id`; duplicate trả lỗi trước khi ghi.

## 3. Luồng xử lý

### 3.1 Tạo SPU draft — `POST /api/v1/seller/products` — trigger: Seller Center → Products → Thêm sản phẩm

```text
1. Gateway gửi actor_user_id + seller/shop scope
  ↓
2. Product kiểm tra actor có SELLER/SELLER_STAFF và shop scope hợp lệ
  └─ sai → 403 PRODUCT_FORBIDDEN
  ↓ hợp lệ
3. Validate title, description, category draft assignment, price draft và attribute definitions
  └─ sai → 400 PRODUCT_INVALID_INPUT
  ↓
4. Tạo product(status=DRAFT), shop snapshot, version=1
  ├→ tạo outbox product.created
  └→ ghi catalog audit
  ↓ commit
5. Trả 201 product_id + version + draft summary
```

Ràng buộc:

- Shop chưa KYC approved vẫn được tạo/cập nhật draft; KYC gate chỉ chặn publish.
- `shop_id` lấy từ actor scope hoặc route context, không tin giá trị khác trong payload.
- Không gọi Inventory ở bước tạo draft; SKU có thể chưa có stock projection.
- Product draft không xuất hiện ở public API/Search index.

### 3.2 Tạo/cập nhật attribute và SKU — `PUT /api/v1/seller/products/{productId}/skus` — trigger: Seller SPU/SKU editor

```text
1. Load product theo product_id + shop ownership + version
2. Validate attribute definitions và xác định variant dimensions
3. Với mỗi SKU:
   ├─ validate type/value/allowed_values
   ├─ tạo variant_key canonical
   ├─ kiểm tra duplicate trong request và database
   └─ validate seller_sku unique trong shop
4. Validate price override, status và media mapping
5. Update SKU set + product version trong transaction
6. Ghi outbox sku.created/updated và product.updated
7. Trả danh sách SKU cùng validation result
```

| Điều kiện | Xử lý |
|---|---|
| Attribute không tồn tại hoặc sai type | Không ghi bất kỳ SKU nào; trả lỗi theo field. |
| `ENUM` value ngoài allowlist | Từ chối `400 PRODUCT_ATTRIBUTE_INVALID`. |
| Hai SKU có cùng `variant_key` | Từ chối toàn bộ request, không ghi partial update. |
| `seller_sku` trùng trong cùng shop | Từ chối và trả SKU gây conflict. |
| SKU đã được Inventory biết tới | Không xóa cứng; chuyển `INACTIVE/ARCHIVED` và phát event để Inventory xử lý lifecycle riêng. |
| Version không khớp | `409 PRODUCT_VERSION_CONFLICT`; client phải reload, merge và retry có chủ đích. |

### 3.3 Cập nhật product draft/active — `PATCH /api/v1/seller/products/{productId}` — trigger: Seller Product editor

```text
1. Kiểm tra actor, shop ownership, product status và optimistic version
2. Validate field thay đổi: title, description, brand, price, category, attributes, media refs
3. Kiểm tra immutable/reference fields không bị đổi tùy ý: product_id, shop_id, audit history
4. Update document + increment version
5. Ghi outbox product.updated; nếu thay đổi price/SKU/category thì ghi event tương ứng
6. Trả product summary + version mới
```

- Seller được sửa `DRAFT`, `ACTIVE`, `INACTIVE` theo permission; `BLOCKED` chỉ admin unblock/đưa về trạng thái an toàn.
- Đổi giá không tự sửa order đã tạo; Order dùng snapshot giá tại thời điểm checkout.
- Đổi product title/slug phải phát event để Search cập nhật; slug conflict trả 409.
- Không cho sửa `shop_id` sau khi product được tạo.

### 3.4 Publish product — `POST /api/v1/seller/products/{productId}/publish` — trigger: Seller Product editor → Xuất bản

```text
1. Kiểm tra actor owner/staff + product status DRAFT/INACTIVE
2. Load shop snapshot; kiểm tra shop.kyc_status=APPROVED và shop_status không bị SUSPENDED
  └─ không đạt → 403 PRODUCT_KYC_REQUIRED
3. Validate required fields:
   ├─ title/description hợp lệ
   ├─ có primary category ACTIVE
   ├─ có ít nhất 1 SKU ACTIVE
   ├─ mọi SKU có price hợp lệ
   ├─ có đúng 1 cover media READY
   └─ không có duplicate variant_key
4. Set product.status=ACTIVE, published_at=now, increment version
5. Ghi product.published vào outbox cùng transaction
6. Search consume event; Inventory tiếp tục quản lý stock riêng
7. Trả 200 product public summary + stock snapshot metadata nếu có
```

Quy tắc publish:

- Không yêu cầu admin approval/censor.
- Không yêu cầu `available_qty > 0`; sản phẩm có thể publish nhưng hiển thị `OUT_OF_STOCK`.
- KYC hết hạn/chuyển trạng thái sau khi product đã `ACTIVE` không tự động unpublish trong v1; nó chặn lần publish/resume tiếp theo. Đây là đúng boundary “KYC gate publish/withdraw”, không phải product moderation.
- Nếu shop bị `SUSPENDED`, public visibility xử lý theo shop policy; Product phải nhận event và cập nhật `shop_snapshot`, không tự xóa product.

### 3.5 Unpublish/archive product — `POST /api/v1/seller/products/{productId}/unpublish`, `POST /api/v1/seller/products/{productId}/archive`

```text
Unpublish:
  1. Check ownership/permission + status ACTIVE
  2. Set INACTIVE, published_at giữ lại, unpublished_at=now
  3. Publish product.unpublished → Search bỏ public visibility

Archive:
  1. Check ownership/admin permission + không còn workflow active cần giữ
  2. Set ARCHIVED và archive reason
  3. Publish product.archived → Search xóa/ẩn index
```

- Unpublish không xóa SKU, price history hoặc audit.
- Archive là soft lifecycle, không xóa cứng product/SKU/media metadata trong v1.
- Product đang được tham chiếu bởi Order vẫn phải đọc được snapshot/reference; public listing không hiển thị archived.

### 3.6 Admin block/unblock sau publish — `POST /api/v1/admin/catalog/products/{productId}/block`

```text
1. Admin gửi reason + step-up context
2. Product kiểm tra CATALOG_ADMIN/SUPER_ADMIN và 2FA requirement
3. Set status=BLOCKED, lưu actor/reason/audit
4. Publish product.blocked để Search ẩn public listing
5. Không xóa dữ liệu; seller vẫn xem được lý do và trạng thái trong Seller Center
```

- Đây là emergency/post-publication action, không phải pre-publish approval.
- Unblock mặc định đưa product về `INACTIVE`; chỉ admin có permission mới có thể restore trực tiếp `ACTIVE` nếu product vẫn đạt publish policy.
- Product không tự censor nội dung bằng AI hoặc workflow review trong v1.

### 3.7 Đọc product detail — `GET /api/v1/products/{productId}` — trigger: Buyer Product Detail

```text
1. Query product ACTIVE + shop snapshot + active category/SKU/media
2. Query inventory_projections theo SKU/product
3. Map stock display:
   ├─ projection mới → available_qty_snapshot + stock_status + as_of
   ├─ projection cũ → giữ status=STALE và không khẳng định số lượng realtime
   └─ chưa có projection → stock_status=UNKNOWN
4. Trả product content, display price và stock display metadata
```

- Không gọi Inventory synchronous trên mỗi product detail ở baseline `2A`.
- `available_qty_snapshot` chỉ để hiển thị; Cart/Checkout phải gọi Inventory reserve/availability API hoặc command riêng.
- Public không trả draft, inactive, blocked hoặc archived; seller/admin read route có policy riêng.
- Nếu một SKU không có snapshot, UI hiển thị theo `UNKNOWN`/`Hết khả dụng` policy do frontend chốt, không tự suy diễn `IN_STOCK`.

### 3.8 Category tree và assignment — `POST/PATCH /api/v1/admin/catalog/categories`, `PUT /api/v1/seller/products/{productId}/categories`

```text
Admin category:
  1. Validate name/slug/parent/status
  2. Check no cycle và depth ≤ 5
  3. Update materialized path/depth
  4. Publish category.created/updated/status_changed

Product assignment:
  1. Check category ACTIVE và product actor scope
  2. Validate đúng 1 primary + tối đa 2 secondary
  3. Replace assignment atomically
  4. Publish product.category_changed
```

- Không cho xóa cứng category đã có product; chuyển `INACTIVE` hoặc `ARCHIVED`.
- Category inactive/archived không nhận assignment mới nhưng product cũ cần migration trước khi archive taxonomy.
- Admin category mutation yêu cầu `CATALOG_ADMIN` hoặc `SUPER_ADMIN`; seller chỉ chọn category, không sửa taxonomy.

### 3.9 Media upload/complete — `POST /api/v1/seller/products/{productId}/media/upload-url`, `POST /api/v1/seller/products/{productId}/media/complete`

```text
1. Product kiểm tra actor ownership và media quota/content type
2. Tạo object key private + signed upload URL
3. Client upload trực tiếp S3/MinIO
4. Client gọi complete với object key/checksum
5. Product verify object metadata, size, content type, checksum
6. Lưu product_media(status=READY), gắn cover/sort order
7. Publish product.media_updated nếu product đã tồn tại
```

- Client không được tự chọn object key của product khác.
- Media chưa `READY` không được dùng làm cover và không thỏa publish policy.
- Product không upload bytes lớn qua API Gateway; Gateway chỉ proxy metadata/complete request.
- File scan/virus scan provider chưa có trong HLD; nếu bắt buộc, media phải ở `SCANNING` trước `READY`.

### 3.10 Inventory projection — lắng nghe `inventory.stock_snapshot.updated`

```text
1. Nhận event theo key sku_id
2. Deduplicate theo event_id/source_version
3. Validate product_id/sku_id reference và quantity integer ≥ 0
4. Upsert inventory_projections với available_qty_snapshot, status, as_of
5. Nếu event out-of-order → bỏ qua version cũ, ghi metric
6. Product detail đọc projection; không phát lệnh trừ/reserve
```

Ràng buộc tuyệt đối:

- Chỉ Inventory được trừ/reserve stock bằng atomic transaction/locking/idempotency của Inventory.
- Product không trả response “đã giữ hàng” hoặc “đã trừ hàng”.
- Nếu event mất, consumer replay từ Kafka hoặc Inventory resync snapshot; Product không tự bù bằng số lượng cũ.
- Stock display có thể trễ; purchase availability phải xác minh lại tại Cart/Checkout.

### 3.11 Shop/KYC projection — lắng nghe `shop.created/updated/status_changed`

```text
1. Nhận event từ Auth User theo key shop_id
2. Validate event schema/version và dedupe event_id
3. Upsert shop_snapshots
4. Cập nhật product read model/cache key nếu cần
5. Với KYC status đổi: áp dụng ở lần publish/unpublish/resume tiếp theo
```

- Product không đọc KYC document hoặc quyết định `APPROVED/REJECTED`.
- `shop_snapshot` là display/projection; Auth User vẫn là source of truth.
- Nếu shop đổi tên/slug/logo, Product cập nhật snapshot và phát `product.shop_snapshot_updated` nếu Search cần reindex.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Đơn vị | Ghi chú |
|---|---:|---|---|
| `PAGE_SIZE_DEFAULT` | 20 | product/SKU | `PAGE_SIZE_MAX=100`. |
| `PRODUCT_TITLE_MAX_LENGTH` | 200 | Unicode characters | Trim hai đầu; không rỗng khi publish. |
| `PRODUCT_DESCRIPTION_MAX_LENGTH` | 100000 | Unicode characters | Có sanitize rich text/HTML allowlist. |
| `BRAND_MAX_LENGTH` | 120 | Unicode characters | Optional nhưng normalize khi tìm kiếm. |
| `SLUG_MAX_LENGTH` | 160 | ký tự | Lowercase, hyphen, unique trong shop. |
| `MAX_ATTRIBUTE_DEFINITIONS` | 50 | definition/product | Không giới hạn tên business field; đây là safety cap. |
| `MAX_VARIANT_DIMENSIONS` | 10 | dimension/product | Không hard-code color/size. |
| `MAX_SKUS_PER_PRODUCT` | 1000 | SKU/product | Vượt giới hạn cần bulk/import workflow riêng. |
| `MAX_ATTRIBUTES_PER_SKU` | 50 | attribute/SKU | Bao gồm variant và descriptive attributes. |
| `ATTRIBUTE_KEY_MAX_LENGTH` | 64 | ký tự | `[a-z0-9_]+` sau normalize. |
| `ATTRIBUTE_LABEL_MAX_LENGTH` | 120 | Unicode characters | UI label. |
| `ATTRIBUTE_STRING_MAX_LENGTH` | 500 | Unicode characters | Với specification dài dùng description field. |
| `ATTRIBUTE_ENUM_MAX_VALUES` | 200 | value/definition | ENUM value unique trong definition. |
| `PRIMARY_CATEGORY_LIMIT` | 1 | category/product | Bắt buộc khi publish. |
| `SECONDARY_CATEGORY_LIMIT` | 2 | category/product | Tối đa 2 category phụ. |
| `CATEGORY_MAX_DEPTH` | 5 | level | Root là depth 1. |
| `CATEGORY_NAME_MAX_LENGTH` | 120 | Unicode characters | Unique theo cùng parent. |
| `PRICE_CURRENCY` | `VND` | currency | Số nguyên, không decimal. |
| `PRICE_MIN` | 1 | VND | Giá bán phải dương khi SKU ACTIVE. |
| `PRICE_MAX` | 999999999999 | VND | Giới hạn chống overflow/nhập nhầm. |
| `PRODUCT_IMAGE_MAX_COUNT` | 12 | file/product | Ít nhất 1 cover khi publish. |
| `PRODUCT_VIDEO_MAX_COUNT` | 3 | file/product | Direct upload; provider/scan cần chốt. |
| `PRODUCT_IMAGE_MAX_SIZE` | 20 | MiB/file | JPG/PNG/WebP baseline. |
| `PRODUCT_VIDEO_MAX_SIZE` | 200 | MiB/file | MP4 baseline; chưa áp cho chat attachment. |
| `MEDIA_SIGNED_URL_TTL` | 10 | phút | URL upload/download private. |
| `INVENTORY_SNAPSHOT_STALE_AFTER` | 60 | giây | Sau thời gian này hiển thị `STALE`; không ảnh hưởng deduction. |
| `LOW_STOCK_THRESHOLD` | 5 | quantity | Chỉ là display label; Inventory vẫn quyết định availability. |
| `SHOP_PROJECTION_MAX_STALE` | 30 | phút | Quá hạn thì không cho publish nếu không có status chắc chắn. |
| `MONGO_TRANSACTION_REQUIRED` | `true` | boolean | Deployment MongoDB phải là replica set/sharded transaction-capable. |
| `OUTBOX_RETRY_COUNT` | 3 | lần | Sau đó dead-letter. |
| `OUTBOX_RETRY_BACKOFF` | 2 | giây | Backoff baseline. |
| `INTERNAL_REQUEST_TIMEOUT` | 5 | giây | Auth/media metadata adapter; không dùng cho stock deduction. |
| `TIMESTAMP_STORAGE` | `UTC` | timezone | API/log ISO-8601. |
| `OPTIMISTIC_VERSIONING` | `true` | boolean | Mutation seller/admin phải gửi version nếu update document. |

## 5. Enum & trạng thái

### 5.1 `ProductStatus`

| Giá trị | Ý nghĩa | Public visible | Ai đổi được |
|---|---|---:|---|
| `DRAFT` | Đang tạo/chưa publish đủ điều kiện. | Không | Seller owner/staff |
| `ACTIVE` | Đã publish, có thể xuất hiện public. | Có | Seller owner/staff theo policy; admin |
| `INACTIVE` | Seller chủ động unpublish hoặc admin restore an toàn. | Không | Seller owner/staff; admin |
| `BLOCKED` | Admin chặn sau publish vì policy/risk. | Không | Admin có permission |
| `ARCHIVED` | Soft archived, không dùng lại trong v1. | Không | Seller/admin theo policy |

```text
                 ┌───────────────┐
                 │               ▼
DRAFT ──publish──► ACTIVE ──unpublish──► INACTIVE
  ▲                 │  ▲                 │
  │                 │  └──publish────────┘
  │                 └──admin block──► BLOCKED ──restore──► INACTIVE
  │                                                          │
  └──────────────────────── seller chỉnh sửa ────────────────┘

DRAFT/INACTIVE/ACTIVE ──archive──► ARCHIVED
```

Chuyển trạng thái:

| Chuyển trạng thái | Ai được phép | Điều kiện |
|---|---|---|
| `DRAFT → ACTIVE` | Seller owner/staff | KYC `APPROVED`, required fields, category/media/SKU/price hợp lệ. |
| `INACTIVE → ACTIVE` | Seller owner/staff | Chạy lại publish policy; KYC `APPROVED`. |
| `ACTIVE → INACTIVE` | Seller owner/staff hoặc admin | Không xóa dữ liệu; phát event search visibility. |
| `ACTIVE/INACTIVE → BLOCKED` | `CATALOG_ADMIN`, `SUPER_ADMIN` | Có reason và audit; step-up 2FA nếu policy auth yêu cầu. |
| `BLOCKED → INACTIVE` | Admin | Mặc định restore an toàn, yêu cầu seller publish lại. |
| `BLOCKED → ACTIVE` | Admin có quyền cao | Chỉ khi product đạt publish policy và có audit reason. |
| `DRAFT/INACTIVE/ACTIVE → ARCHIVED` | Owner/admin theo permission | Có archive reason; không còn public visibility. |
| `ARCHIVED → *` | Không trong v1 | Tạo product mới nếu cần bán lại. |

### 5.2 `SkuStatus`

| Giá trị | Ý nghĩa | Quy tắc |
|---|---|---|
| `DRAFT` | SKU chưa được bán | Có thể thiếu inventory projection. |
| `ACTIVE` | SKU được chọn trong product active | Phải có variant_key unique và price hợp lệ. |
| `INACTIVE` | Tạm dừng bán SKU | Không public selectable; không xóa lịch sử. |
| `ARCHIVED` | SKU không dùng lại | Nếu Inventory đã biết thì phát event lifecycle, không delete cứng. |

### 5.3 `AttributeType`, `MediaStatus`, `CategoryStatus`

| Enum | Giá trị hợp lệ |
|---|---|
| `AttributeType` | `STRING`, `NUMBER`, `BOOLEAN`, `ENUM` |
| `MediaScope` | `SPU`, `SKU` |
| `MediaStatus` | `UPLOADING`, `SCANNING`, `READY`, `REJECTED`, `DELETED` |
| `CategoryStatus` | `ACTIVE`, `INACTIVE`, `ARCHIVED` |
| `InventoryStockStatus` | `UNKNOWN`, `IN_STOCK`, `LOW_STOCK`, `OUT_OF_STOCK`, `STALE` |

### 5.4 Publish readiness

| Check | `DRAFT/INACTIVE` cho phép | `ACTIVE` yêu cầu |
|---|---:|---:|
| Product title/description hợp lệ | Có thể lưu lỗi draft | Có |
| Primary category ACTIVE | Có thể lưu draft chưa đủ | Có |
| Ít nhất 1 SKU ACTIVE | Có thể chưa có | Có |
| SKU price hợp lệ | Có thể lưu draft | Có |
| Cover media `READY` | Có thể upload sau | Có |
| Shop KYC `APPROVED` | Không bắt buộc để save | Bắt buộc khi publish/resume |
| Stock > 0 | Không | Không; Inventory quyết định availability |
| Admin approval | Không | Không |

### 5.5 Consistency state của Inventory projection

| State | Ý nghĩa | UI/use case |
|---|---|---|
| `UNKNOWN` | Chưa có event snapshot | Không khẳng định còn hàng. |
| `IN_STOCK` | Snapshot available quantity > threshold | Hiển thị còn hàng theo `as_of`. |
| `LOW_STOCK` | Snapshot quantity từ 1 đến threshold | Hiển thị sắp hết; không dùng để reserve. |
| `OUT_OF_STOCK` | Snapshot quantity = 0 | Disable mua theo display, Checkout vẫn revalidate. |
| `STALE` | Snapshot quá `60s` hoặc source gap | Hiển thị stale; không kết luận quantity realtime. |

## 6. Event phát ra / lắng nghe

### 6.1 Event envelope chung

```json
{
  "event_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "event_type": "product.published",
  "occurred_at": "2026-08-30T09:00:00Z",
  "aggregate_type": "PRODUCT",
  "aggregate_id": "product-01912f31",
  "actor_user_id": "user-01912f30",
  "payload": {}
}
```

### 6.2 Event phát ra

| Topic | Event type | Payload chính | Khi nào | Key |
|---|---|---|---|---|
| `product.events.v1` | `product.created` | `product_id`, `shop_id`, status, title, categories, version | Tạo SPU commit | `product_id` |
| `product.events.v1` | `product.updated` | changed fields, version, price summary, shop snapshot | Product mutation commit | `product_id` |
| `product.events.v1` | `product.published` | public product summary, active SKU IDs, price, published_at | Publish thành công | `product_id` |
| `product.events.v1` | `product.unpublished` | `product_id`, reason, version | Seller/admin unpublish | `product_id` |
| `product.events.v1` | `product.blocked` | `product_id`, reason, actor, blocked_at | Admin block | `product_id` |
| `product.events.v1` | `product.unblocked` | `product_id`, next_status, actor | Admin unblock | `product_id` |
| `product.events.v1` | `product.archived` | `product_id`, reason, archived_at | Archive | `product_id` |
| `sku.events.v1` | `sku.created` | `sku_id`, `product_id`, `variant_key`, attributes, price | SKU create | `sku_id` |
| `sku.events.v1` | `sku.updated` | changed fields, version | SKU update | `sku_id` |
| `sku.events.v1` | `sku.status_changed` | old/new status, reason | Activate/inactivate/archive SKU | `sku_id` |
| `category.events.v1` | `category.created/updated` | category, parent, path, status | Admin taxonomy mutation | `category_id` |
| `category.events.v1` | `category.status_changed` | old/new status, path | Category state change | `category_id` |
| `catalog.events.v1` | `product.category_changed` | product_id, primary/secondary category IDs | Assignment changed | `product_id` |
| `catalog.events.v1` | `product.media_updated` | media IDs, cover, version | Media ready/order changed | `product_id` |
| `catalog.events.v1` | `product.shop_snapshot_updated` | product_id, shop_id, changed snapshot fields | Shop event projection update | `product_id` |

Consumers phải xử lý idempotent theo `event_id`, kiểm tra `schema_version` và không đọc Product MongoDB trực tiếp.

### 6.3 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Auth User | `shop.created` | Tạo shop snapshot để seller/product read. |
| Auth User | `shop.updated` | Cập nhật name/slug/logo/business display snapshot. |
| Auth User | `shop.status_changed` | Cập nhật shop status; không tự đổi product content. |
| Auth User | `shop.kyc.approved/needs_info/rejected/expired/suspended` | Cập nhật KYC gate projection; chặn publish/resume nếu không APPROVED. |
| Inventory | `inventory.stock_snapshot.updated` | Upsert read-only SKU/product stock projection. |
| Inventory | `inventory.sku.deleted/disabled` | Đánh dấu projection/source warning; không tự delete SKU catalog. |

### 6.4 Mock contract — Inventory stock snapshot

Event đề xuất cho phần HLD chưa định nghĩa:

```json
{
  "event_id": "inventory-event-01912f40",
  "schema_version": 1,
  "event_type": "inventory.stock_snapshot.updated",
  "occurred_at": "2026-08-30T09:00:00Z",
  "aggregate_type": "SKU",
  "aggregate_id": "sku-01912f31",
  "payload": {
    "product_id": "product-01912f31",
    "sku_id": "sku-01912f31",
    "available_qty": 7,
    "reserved_qty": 2,
    "committed_qty": 11,
    "source_version": 42,
    "as_of": "2026-08-30T08:59:59Z"
  }
}
```

Ràng buộc:

- `available_qty`, `reserved_qty`, `committed_qty` là integer `>=0`; Product không sửa và không cộng/trừ chúng.
- `source_version` tăng đơn điệu theo `sku_id`; event cũ hơn bị bỏ qua.
- Event này chỉ phục vụ display projection; Inventory API/command mới là nơi đảm bảo deduction tuyệt đối.

### 6.5 Mock contract — shop/KYC event

```json
{
  "event_id": "shop-event-01912f41",
  "schema_version": 1,
  "event_type": "shop.kyc.approved",
  "occurred_at": "2026-08-30T09:00:00Z",
  "aggregate_type": "SHOP",
  "aggregate_id": "shop-01912f30",
  "payload": {
    "shop_id": "shop-01912f30",
    "owner_user_id": "user-01912f30",
    "shop_status": "ACTIVE",
    "kyc_status": "APPROVED",
    "source_version": 8
  }
}
```

- Product chỉ cần metadata shop/KYC được allowlist; không nhận KYC document bytes.
- Nếu nhận `shop.kyc.expired` hoặc `shop.status_changed=SUSPENDED`, Product ghi projection và chặn publish/resume mới.

### 6.6 Reliability, ordering và replay

- Domain mutation và outbox event ghi trong cùng MongoDB transaction.
- Kafka key theo `product_id`, `sku_id` hoặc `category_id` để giữ thứ tự trong aggregate.
- Publisher retry tối đa 3 lần, backoff 2 giây; sau đó đưa `product-catalog.events.dlq.v1`.
- Consumer ghi `processed_event_id` hoặc dùng unique event ID để dedupe; không xử lý lại event hoàn tất.
- Inventory projection cho phép replay/resync từ Inventory snapshot; Product không dùng event replay để suy ra deduction.
- Search có thể lag sau publish; Product response `ACTIVE` là source of truth catalog, Search eventually indexes theo event.

## 7. Mã lỗi

| Mã | HTTP | Khi nào xảy ra | Thông điệp cho người dùng |
|---|---:|---|---|
| `PRODUCT_INVALID_INPUT` | 400 | Payload/product field không hợp lệ | `Thông tin sản phẩm chưa đúng.` |
| `PRODUCT_NOT_FOUND` | 404 | Không tìm thấy product trong scope | `Không tìm thấy sản phẩm.` |
| `PRODUCT_FORBIDDEN` | 403 | Actor không sở hữu product/shop hoặc thiếu permission | `Bạn không có quyền thao tác sản phẩm này.` |
| `PRODUCT_STATE_INVALID` | 409 | Action không hợp lệ với product status | `Trạng thái sản phẩm không cho phép thao tác này.` |
| `PRODUCT_VERSION_CONFLICT` | 409 | Optimistic version không khớp | `Sản phẩm đã được thay đổi. Vui lòng tải lại.` |
| `PRODUCT_TITLE_REQUIRED` | 400 | Thiếu title hoặc vượt giới hạn | `Vui lòng nhập tên sản phẩm hợp lệ.` |
| `PRODUCT_DESCRIPTION_INVALID` | 400 | Description rỗng khi publish hoặc vượt limit | `Mô tả sản phẩm chưa hợp lệ.` |
| `PRODUCT_CATEGORY_REQUIRED` | 400 | Chưa có primary category khi publish | `Vui lòng chọn danh mục chính.` |
| `PRODUCT_CATEGORY_INVALID` | 400 | Category không active/không thuộc taxonomy hợp lệ | `Danh mục sản phẩm không hợp lệ.` |
| `CATEGORY_DEPTH_EXCEEDED` | 409 | Category vượt depth 5 | `Danh mục vượt quá số cấp cho phép.` |
| `CATEGORY_CYCLE_DETECTED` | 409 | Parent tạo cycle | `Cấu trúc danh mục không hợp lệ.` |
| `PRODUCT_SKU_REQUIRED` | 400 | Publish không có SKU active | `Sản phẩm cần ít nhất một phiên bản.` |
| `PRODUCT_SKU_DUPLICATE` | 409 | Trùng variant key hoặc seller SKU | `Phiên bản sản phẩm bị trùng.` |
| `PRODUCT_SKU_LIMIT_EXCEEDED` | 409 | Vượt 1.000 SKU/product | `Sản phẩm đã vượt giới hạn số phiên bản.` |
| `PRODUCT_ATTRIBUTE_INVALID` | 400 | Sai type, key, enum value hoặc dimension | `Thuộc tính phiên bản chưa hợp lệ.` |
| `PRODUCT_PRICE_INVALID` | 400 | Price không phải integer VND hoặc ngoài range | `Giá sản phẩm chưa hợp lệ.` |
| `PRODUCT_MEDIA_REQUIRED` | 400 | Thiếu cover media READY khi publish | `Vui lòng thêm ảnh đại diện sản phẩm.` |
| `PRODUCT_MEDIA_INVALID` | 400 | Sai type/size/checksum/object metadata | `Tệp media không hợp lệ.` |
| `PRODUCT_MEDIA_LIMIT_EXCEEDED` | 409 | Vượt số ảnh/video cho phép | `Sản phẩm đã vượt giới hạn media.` |
| `PRODUCT_SLUG_CONFLICT` | 409 | Slug trùng trong shop | `Đường dẫn sản phẩm đã tồn tại.` |
| `PRODUCT_KYC_REQUIRED` | 403 | Shop chưa APPROVED để publish/resume | `Vui lòng hoàn tất xác minh gian hàng trước.` |
| `PRODUCT_SHOP_SUSPENDED` | 403 | Shop bị suspend | `Gian hàng hiện không thể thực hiện thao tác này.` |
| `PRODUCT_BLOCKED` | 403 | Product bị admin block | `Sản phẩm đang bị tạm ẩn.` |
| `PRODUCT_ARCHIVED` | 409 | Mutation trên archived product | `Sản phẩm đã được lưu trữ.` |
| `INVENTORY_PROJECTION_STALE` | 200/metadata | Stock snapshot quá cũ | `Thông tin tồn kho đang được cập nhật.` |
| `CATALOG_EVENT_PUBLISH_FAILED` | 503 | Outbox publisher chưa phát được event | `Hệ thống đang đồng bộ dữ liệu. Vui lòng thử lại sau.` |
| `INTERNAL_ERROR` | 500 | Lỗi chưa phân loại | `Hệ thống đang bận. Vui lòng thử lại.` |

`INVENTORY_PROJECTION_STALE` không phải lỗi deduction; nó là metadata để frontend không hiểu nhầm số lượng display là realtime.

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Stack là Node.js + NestJS + MongoDB/Mongoose; exact Node.js/NestJS/MongoDB versions và HTTP adapter chưa chốt. | Ảnh hưởng ODM validator, transaction behavior, Docker image và performance. | Tech lead |
| 2 | Product Catalog sở hữu `base_price/sale_price` hiển thị; Inventory chỉ sở hữu stock; Order lưu price snapshot. | Nếu service khác sở hữu giá, phải đổi event/API và pricing consistency. | Architecture + Order owner |
| 3 | Inventory deduction/reservation phải atomic/idempotent và tuyệt đối chính xác; Product chỉ nhận snapshot eventual. | Nếu Inventory không có reserve contract rõ, Checkout có thể oversell dù Product hiển thị đúng snapshot. | Inventory/Order owner |
| 4 | `inventory.stock_snapshot.updated` là mock event; tên topic, payload, source_version và replay/resync API chưa có trong HLD. | Ảnh hưởng stock display freshness và recovery khi mất event. | Inventory owner |
| 5 | Không yêu cầu stock > 0 để publish; product active với quantity 0 vẫn có thể public nhưng không mua được. | Nếu business muốn ẩn hết hàng khỏi listing, cần Search/visibility policy riêng. | Product owner |
| 6 | KYC event chỉ chặn publish/resume; product ACTIVE không tự unpublish khi KYC hết hạn trong v1. | Nếu compliance yêu cầu ẩn ngay toàn bộ product, cần consumer policy và bulk action. | Risk/Product owner |
| 7 | Dynamic attributes không hard-code field nhưng có technical safety cap: 50 definitions, 10 dimensions, 1.000 SKU/product. | Nếu cần hơn, phải có bulk import/async generation thay vì request synchronous. | Tech lead/Product owner |
| 8 | Category tree tối đa 5 cấp, product có 1 primary + 2 secondary; đã dùng làm baseline từ lựa chọn 5A. | Thay đổi taxonomy sẽ ảnh hưởng category assignment, search facets và migration. | Catalog owner |
| 9 | Media baseline là 12 ảnh + 3 video/product, S3/MinIO signed URL; virus scan provider và exact product media policy chưa chốt. | Ảnh hưởng upload UX, storage cost, publish readiness và security. | Product/Security/DevOps |
| 10 | Admin `BLOCKED` là post-publication emergency action, không phải pre-approval/censor workflow. | Nếu cần review trước publish, phải thêm state machine, queue và SLA. | Product owner |
| 11 | Shop snapshot nhận từ Auth User event; event schema/version và field allowlist cần align với auth-user API/event spec. | Product detail có thể hiển thị shop stale hoặc publish gate sai. | Auth-user owner |
| 12 | Search eventual consistency qua Kafka; Product response ACTIVE không đợi Search index thành công. | User có thể thấy product detail trước khi tìm thấy qua search. | Search owner |
| 13 | Product detail không gọi Inventory synchronous; snapshot stale sau 60 giây hiển thị metadata `STALE`. | Nếu UX bắt buộc số tồn realtime, phải bổ sung read API/timeout/fallback. | Product + frontend |
| 14 | Realtime price history, promotion campaign, brand approval và AI content moderation chưa thuộc v1. | Nếu Penpot/HLD bổ sung, cần thêm aggregate/permission/event riêng. | Product owner |
