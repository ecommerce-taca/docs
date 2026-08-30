# API Spec — Product Catalog Service

> Nguồn: `docs/lld/product-catalog.md` và `docs/db/product-catalog.md` · Cập nhật: `2026-08-30`
> Base path nội bộ: `/api/v1` · Public traffic đi qua API Gateway · Content-Type: `application/json`

## 1. Quy ước API chung

### 1.1 Authentication và authorization

| Nhóm route | Authentication | Authorization |
|---|---|---|
| Buyer/public `GET` | Có thể anonymous | Chỉ trả `ACTIVE` và visibility hợp lệ. |
| Seller `/seller/*` | JWT từ Gateway | `SELLER`/`SELLER_STAFF`; product mutation phải đúng `shop_id` scope. |
| Admin `/admin/catalog/*` | JWT + step-up context khi destructive | `CATALOG_ADMIN` hoặc `SUPER_ADMIN`. |
| Internal integration | mTLS/service token qua Gateway hoặc internal policy | Chỉ endpoint contract được allowlist; không truy cập MongoDB trực tiếp. |

Product Catalog lấy `actor_user_id`, role và shop scope từ auth context do Gateway xác thực. Không nhận `shop_id` body để quyết định tenant. `401`/`403` format do Gateway chuẩn hóa; service-specific errors bên dưới dùng envelope chung.

### 1.2 Request/response và pagination

```json
{
  "data": {},
  "meta": {
    "request_id": "req-01912f50",
    "as_of": "2026-08-30T09:00:00Z"
  }
}
```

- List response: `{ data: [], meta: { request_id, page, size, total, has_next } }`.
- Default `size=20`, tối đa `100`; `page` bắt đầu từ `1`.
- Timestamp ISO-8601 UTC; tiền là integer VND (`currency: "VND"`).
- Public catalog list không phải full-text search engine; full-text/facet nâng cao thuộc Search Service.
- Mutation có `version` để optimistic concurrency; mismatch trả `409 PRODUCT_VERSION_CONFLICT`.
- API không trả KYC document, token, media bytes, `reserved`/`committed` như quyền trừ stock. Stock chỉ là projection display.

### 1.3 Error envelope

```json
{
  "error": {
    "code": "PRODUCT_VERSION_CONFLICT",
    "message": "Sản phẩm đã được thay đổi. Vui lòng tải lại.",
    "details": [{ "field": "version", "reason": "VERSION_MISMATCH" }],
    "trace_id": "01912f50-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

`details` optional; không leak stack trace, secret, internal MongoDB query hoặc KYC data.

## 2. Danh sách endpoint

| # | Method + path | Actor | Mục đích |
|---:|---|---|---|
| 1 | `GET /products` | Public | Catalog listing cơ bản. |
| 2 | `GET /products/{productId}` | Public | Product detail + display stock projection. |
| 3 | `GET /categories` | Public | Category tree. |
| 4 | `GET /categories/{categoryId}` | Public | Category detail/children. |
| 5 | `GET /shops/{shopId}/products` | Public | Product listing theo shop. |
| 6 | `POST /seller/products` | Seller | Tạo SPU draft. |
| 7 | `GET /seller/products` | Seller | Seller product list. |
| 7a | `GET /seller/products/export` | Seller | Xuất danh sách sản phẩm ra file (.xlsx/.csv). |
| 8 | `GET /seller/products/{productId}` | Seller | Seller editor detail. |
| 9 | `PATCH /seller/products/{productId}` | Seller | Cập nhật product fields. |
| 10 | `PUT /seller/products/{productId}/skus` | Seller | Replace/update SKU set. |
| 11 | `PUT /seller/products/{productId}/categories` | Seller | Gán 1 primary + tối đa 2 secondary. |
| 12 | `POST /seller/products/{productId}/publish` | Seller | Publish/resume product. |
| 13 | `POST /seller/products/{productId}/unpublish` | Seller | Đưa `ACTIVE` về `INACTIVE`. |
| 14 | `POST /seller/products/{productId}/archive` | Seller | Soft archive. |
| 15 | `POST /seller/products/{productId}/media/upload-url` | Seller | Nhận signed upload URL. |
| 16 | `POST /seller/products/{productId}/media/complete` | Seller | Verify upload và lưu metadata READY. |
| 17 | `GET /admin/catalog/products` | Admin | Catalog moderation/read. |
| 18 | `GET /admin/catalog/products/{productId}` | Admin | Xem product bất kể public visibility. |
| 19 | `POST /admin/catalog/products/{productId}/block` | Admin | Post-publication block. |
| 20 | `POST /admin/catalog/products/{productId}/unblock` | Admin | Restore an toàn về `INACTIVE`. |
| 21 | `GET /admin/catalog/categories` | Admin | Quản trị taxonomy. |
| 22 | `POST /admin/catalog/categories` | Admin | Tạo category. |
| 23 | `PATCH /admin/catalog/categories/{categoryId}` | Admin | Sửa category/parent/status. |
| 24 | `POST /admin/catalog/categories/{categoryId}/archive` | Admin | Soft archive category. |

Không có endpoint Product để reserve/deduct stock. Cart/Checkout phải gọi Inventory contract để xác minh availability và reserve atomically.

## 3. Chi tiết endpoint

### 3.1 Public catalog

#### `GET /products`

Query: `page`, `size`, `category_id`, `shop_id`, `status` (chỉ internal/admin; public mặc định `ACTIVE`), `min_price`, `max_price`, `sort` (`newest`, `price_asc`, `price_desc`), `product_ids` (danh sách phân tách bằng dấu phẩy, tối đa 100 — dùng để hydrate thẻ sản phẩm cho Favorites/Cart; khi có `product_ids` thì bỏ qua các filter khác và trả đúng các product `ACTIVE` visible, giữ thứ tự client gửi nếu có thể).

Response `200`:

```json
{
  "data": [{
    "product_id": "product-01912f31",
    "shop": { "shop_id": "shop-01912f30", "name": "Taca Shop", "slug": "taca-shop", "logo_url": null },
    "primary_category_id": "cat-01912f20",
    "tax_rate_bps": 1000,
    "title": "Áo khoác cotton",
    "slug": "ao-khoac-cotton",
    "price": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
    "cover_media": { "media_id": "media-01912f32", "url": "https://cdn.example/signed", "content_type": "image/webp" },
    "stock_display": { "status": "IN_STOCK", "as_of": "2026-08-30T08:59:59Z" }
  }],
  "meta": { "page": 1, "size": 20, "total": 1, "has_next": false, "request_id": "req-01912f50" }
}
```

Ràng buộc: `min_price/max_price` là integer VND; public không được query draft/blocked/archived; stock display có thể `UNKNOWN`/`STALE` và không phải purchase guarantee. Với `product_ids`, product không tồn tại/không visible bị bỏ khỏi kết quả (không lỗi) — caller tự phát hiện ID nào biến mất.

`tax_rate_bps` là **thuế suất VAT của sản phẩm**, đơn vị basis point (10% = `1000`). Product Catalog đã giải quyết xong việc thừa kế theo cây danh mục và trả ra giá trị cuối cùng — caller không phải leo cây. Order-Commerce snapshot giá trị này vào `order_items` tại lúc checkout để tách thuế ghi hoá đơn (`order-commerce-docs/docs/lld/order-commerce.md` §6.4). Giá trong `price` **đã bao gồm** VAT.

#### `GET /products/{productId}`

Response `200` trả `ProductDetail` gồm `product_id`, shop snapshot, title/description/brand, price summary, active categories, active SKUs, media READY và `stock_display` theo SKU:

```json
{
  "data": {
    "product_id": "product-01912f31",
    "status": "ACTIVE",
    "title": "Áo khoác cotton",
    "primary_category_id": "cat-01912f20",
    "tax_rate_bps": 1000,
    "price": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
    "attributes": [{ "key": "material", "label": "Chất liệu", "type": "ENUM", "values": ["cotton"] }],
    "skus": [{
      "sku_id": "sku-01912f33",
      "seller_sku": "AC-COTTON-01",
      "attributes": { "material": "cotton", "capacity_liter": 2.0 },
      "price": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
      "stock_display": { "status": "LOW_STOCK", "available_qty_snapshot": 3, "as_of": "2026-08-30T08:59:59Z" }
    }]
  },
  "meta": { "request_id": "req-01912f50", "as_of": "2026-08-30T09:00:00Z" }
}
```

`404 PRODUCT_NOT_FOUND` nếu product không public hoặc không tồn tại. Không synchronous-call Inventory ở baseline.

#### `GET /categories` và `GET /categories/{categoryId}`

`GET /categories`: trả category `ACTIVE` theo tree, `depth ≤ 5`, kèm `children`.

```json
{
  "data": [
    {
      "category_id": "cat-01912f20",
      "name": "Điện tử",
      "slug": "dien-tu",
      "path": "/cat-01912f20",
      "depth": 1,
      "tax_rate_bps": 1000,
      "children": [
        { "category_id": "cat-01912f21", "name": "Điện thoại", "slug": "dien-thoai", "path": "/cat-01912f20/cat-01912f21", "depth": 2, "tax_rate_bps": null, "children": [] }
      ]
    }
  ],
  "meta": { "request_id": "req-01912f51" }
}
```

`GET /categories/{categoryId}`: trả metadata category và children; category archived không public.

```json
{
  "data": { "category_id": "cat-01912f21", "name": "Điện thoại", "slug": "dien-thoai", "path": "/cat-01912f20/cat-01912f21", "depth": 2, "tax_rate_bps": null, "children": [] },
  "meta": { "request_id": "req-01912f52" }
}
```

`tax_rate_bps: null` ở category con nghĩa là **thừa kế từ cha** — client hiển thị số thực tế phải leo `path` để resolve, hoặc dùng giá trị đã resolve sẵn ở `tax_rate_bps` của product (§3.1). `404 PRODUCT_NOT_FOUND` cho category không tồn tại theo public visibility; category invalid trả `PRODUCT_CATEGORY_INVALID` ở assignment endpoint.

#### `GET /shops/{shopId}/products`

Query giống `GET /products`, mặc định filter `ACTIVE`, lọc theo `shop_id` trong shop snapshot. Response **cùng hình dạng** với `GET /products` ở §3.1 (mảng product card + `meta`) — không lặp lại ví dụ ở đây. Shop suspended có visibility policy do Auth User event; public không được dùng route này để bypass policy.

> **Khoá là `shop_id` (UUID), không phải slug.** Toàn bộ family `/api/v1/shops/{shopId}/**` dùng chung một loại khoá: `GET /shops/{shopId}` (hồ sơ, `auth-user`), `POST|DELETE /shops/{shopId}/follow`, `GET /shops/{shopId}/followers/count` và route này. Trước đây route này nhận slug trong khi các route anh em nhận UUID — màn Shop gọi hai route cùng prefix với hai kiểu khoá khác nhau sẽ `404`. FE có URL dạng slug thì resolve slug → `shop_id` bằng `GET /shops/{slug}` của `auth-user` (endpoint đó nhận cả hai dạng), rồi dùng `shop_id` cho mọi lời gọi sau.

### 3.2 Seller product

#### `POST /seller/products`

Request:

```json
{
  "title": "Áo khoác cotton",
  "slug": "ao-khoac-cotton",
  "description": "Mô tả sản phẩm",
  "brand": "Taca Brand",
  "price_summary": { "base_price": 299000, "sale_price": 249000, "currency": "VND" }
}
```

Response `201`: `{ data: { product_id, shop_id, status: "DRAFT", version: 1 }, meta }`.

Ràng buộc: shop chưa KYC `APPROVED` vẫn tạo/sửa draft được; `shop_id` lấy từ token scope; slug conflict `409 PRODUCT_SLUG_CONFLICT`.

#### `GET /seller/products`

Query: `page`, `size`, `status`, `q` (title/slug basic), `sort`. Trả tất cả product của shop actor, gồm draft/inactive; không trả product shop khác.

```json
{
  "data": [
    {
      "product_id": "product-01912f31",
      "title": "Áo khoác cotton",
      "slug": "ao-khoac-cotton",
      "status": "DRAFT",
      "primary_category_id": "cat-01912f20",
      "price_summary": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
      "sku_count": 3,
      "cover_media": { "media_id": "media-01912f32", "url": "https://cdn.example/signed" },
      "updated_at": "2026-08-30T09:00:00Z"
    }
  ],
  "meta": { "request_id": "req-01912f53", "page": 1, "size": 20, "total": 84 }
}
```

#### `GET /seller/products/export`

Query giống `GET /seller/products` (`status?`, `q?`, `updated_from?`, `updated_to?`) cộng `format` (`xlsx` mặc định | `csv`). Response `200`:

```json
{
  "data": {
    "export_url": "https://storage.example/signed-download/products-shop-01912f30-20260830.xlsx",
    "format": "xlsx",
    "row_count": 84,
    "generated_at": "2026-08-30T09:00:00Z",
    "expires_at": "2026-08-30T09:30:00Z"
  },
  "meta": { "request_id": "01912fbb-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Đúng chuẩn hệ thống "file lớn dùng signed URL, không qua Gateway body" (`api-gateway-docs/docs/api/api-gateway.md` §1.1). File **không** stream qua endpoint này — client GET `export_url` riêng để tải. `export_url` hết hạn sau 30 phút (`expires_at`), gọi lại endpoint để có URL mới. Cột export theo thứ tự: `product_id, title, slug, status, primary_category, base_price, sale_price, sku_count, updated_at`. Chỉ export sản phẩm của shop trong token — không tham số `shop_id`. Vượt `PRODUCT_EXPORT_MAX_ROWS` (10.000) → `400 PRODUCT_EXPORT_TOO_LARGE`, seller phải lọc hẹp hơn bằng `updated_from`/`updated_to`.

#### `GET /seller/products/{productId}`

Trả editor detail gồm product fields, definitions, SKU set, category assignments, media metadata, current shop/KYC projection và `version`. Seller được xem product blocked/archived để hiển thị lý do/lịch sử theo policy.

```json
{
  "data": {
    "product_id": "product-01912f31",
    "status": "DRAFT",
    "version": 3,
    "title": "Áo khoác cotton",
    "slug": "ao-khoac-cotton",
    "description": "Mô tả sản phẩm",
    "brand": "Taca Brand",
    "price_summary": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
    "attribute_definitions": [
      { "key": "material", "label": "Chất liệu", "type": "ENUM", "is_variant_dimension": true, "allowed_values": ["cotton", "linen"] }
    ],
    "skus": [
      { "sku_id": "sku-01912f33", "seller_sku": "AC-COTTON-01", "attributes": { "material": "cotton" }, "price_override": null, "status": "ACTIVE" }
    ],
    "categories": { "primary_category_id": "cat-01912f20", "secondary_category_ids": [] },
    "media": [{ "media_id": "media-01912f32", "url": "https://cdn.example/signed", "status": "READY", "is_cover": true }],
    "shop_projection": { "shop_id": "shop-01912f30", "status": "ACTIVE", "kyc_status": "APPROVED" },
    "block_reason": null
  },
  "meta": { "request_id": "req-01912f54" }
}
```

`block_reason` chỉ có giá trị khi `status="BLOCKED"`; các state khác trả `null`.

#### `PATCH /seller/products/{productId}`

Request fields optional:

```json
{
  "version": 1,
  "title": "Áo khoác cotton premium",
  "slug": "ao-khoac-cotton-premium",
  "description": "Mô tả mới",
  "brand": "Taca Brand",
  "price_summary": { "base_price": 319000, "sale_price": 269000, "currency": "VND" }
}
```

Response `200`: product summary + new version. Không được đổi `product_id`, `shop_id`, audit history; archived/blocked theo state policy trả `PRODUCT_ARCHIVED`/`PRODUCT_BLOCKED`.

#### `PUT /seller/products/{productId}/skus`

Request:

```json
{
  "version": 3,
  "attribute_definitions": [
    { "key": "material", "label": "Chất liệu", "type": "ENUM", "is_variant_dimension": true, "allowed_values": ["cotton", "linen"] },
    { "key": "color", "label": "Màu sắc", "type": "ENUM", "is_variant_dimension": true, "allowed_values": ["black", "white"], "display_as": "COLOR_SWATCH", "value_meta": { "black": { "swatch_hex": "#111111" }, "white": { "swatch_hex": "#FFFFFF" } } },
    { "key": "capacity_liter", "label": "Dung tích", "type": "NUMBER", "is_variant_dimension": false, "unit": "L" }
  ],
  "skus": [{
    "sku_id": "sku-01912f33",
    "seller_sku": "AC-COTTON-01",
    "attributes": { "material": "cotton", "capacity_liter": 2.0 },
    "price_override": null,
    "status": "ACTIVE"
  }]
}
```

Server tự canonicalize `variant_key`; duplicate hoặc sai type trả `PRODUCT_SKU_DUPLICATE`/`PRODUCT_ATTRIBUTE_INVALID`. Toàn request atomic, không partial write. `sku_id` đã được Inventory biết không hard-delete; chuyển lifecycle và phát event.

`display_as` (`PLAIN`/`COLOR_SWATCH`/`IMAGE_THUMB`) và `value_meta` (`swatch_hex`, `swatch_media_id`) là **tùy chọn, chỉ hint render** cho Frontend Seller SKU builder ("Attribute type / Image | Color | Text"); không tham gia canonicalize `variant_key`, không bắt buộc khi publish. Bỏ trống = `PLAIN`.

#### `PUT /seller/products/{productId}/categories`

Request:

```json
{
  "version": 4,
  "primary_category_id": "category-01912f20",
  "secondary_category_ids": ["category-01912f21"]
}
```

Response `200`: assignments + version. Bắt buộc đúng 1 primary, tối đa 2 secondary; category phải `ACTIVE`, không trùng; replace atomic.

#### `POST /seller/products/{productId}/publish`

Request: `{ "version": 4 }`.

```json
{
  "data": {
    "product_id": "product-01912f31",
    "status": "ACTIVE",
    "published_at": "2026-08-30T09:05:00Z",
    "version": 5,
    "stock_display": { "status": "IN_STOCK", "as_of": "2026-08-30T09:04:50Z" }
  },
  "meta": { "request_id": "req-01912f55" }
}
```

Preconditions: status `DRAFT`/`INACTIVE`; KYC `APPROVED`; shop không `SUSPENDED`; title/description, primary category ACTIVE, ≥1 SKU ACTIVE, prices hợp lệ, đúng 1 cover media READY. Stock `0` không chặn publish.

Errors: `PRODUCT_KYC_REQUIRED`, `PRODUCT_SHOP_SUSPENDED`, `PRODUCT_CATEGORY_REQUIRED`, `PRODUCT_SKU_REQUIRED`, `PRODUCT_PRICE_INVALID`, `PRODUCT_MEDIA_REQUIRED`, `PRODUCT_STATE_INVALID`, `PRODUCT_VERSION_CONFLICT`.

#### `POST /seller/products/{productId}/unpublish`

Request: `{ "version": 7, "reason": "Tạm dừng bán" }`.

```json
{ "data": { "product_id": "product-01912f31", "status": "INACTIVE", "version": 8 }, "meta": { "request_id": "req-01912f56" } }
```

Chỉ từ `ACTIVE`; không xóa SKU/price/media; phát `product.unpublished`.

#### `POST /seller/products/{productId}/archive`

Request: `{ "version": 7, "reason": "Ngừng kinh doanh" }`.

```json
{ "data": { "product_id": "product-01912f31", "status": "ARCHIVED", "version": 8 }, "meta": { "request_id": "req-01912f57" } }
```

Soft lifecycle; không archive nếu state/workflow policy không cho phép; phát `product.archived`.

#### `POST /seller/products/{productId}/media/upload-url`

Request:

```json
{
  "scope": "SPU",
  "sku_id": null,
  "content_type": "image/webp",
  "size_bytes": 524288,
  "sha256": "aabbcc...",
  "is_cover": true
}
```

Response `201`:

```json
{
  "data": {
    "media_id": "media-01912f32",
    "object_key": "products/shop-01912f30/product-01912f31/media-01912f32.webp",
    "upload_url": "https://storage.example/signed-upload",
    "expires_at": "2026-08-30T09:10:00Z",
    "status": "UPLOADING"
  },
  "meta": { "request_id": "req-01912f50" }
}
```

Server sinh object key và kiểm quota/type/size. Image ≤20 MiB, video ≤200 MiB, quota 12 image + 3 video/product; client không upload bytes qua Gateway.

#### `POST /seller/products/{productId}/media/complete`

Request: `{ "media_id": "media-01912f32", "object_key": "...", "sha256": "aabbcc..." }`.

```json
{ "data": { "media_id": "media-01912f32", "status": "READY", "url": "https://cdn.example/media-01912f32.webp" }, "meta": { "request_id": "req-01912f58" } }
```

`status` nếu scan async thì `SCANNING` và chưa thỏa publish. Sai metadata trả `PRODUCT_MEDIA_INVALID`.

### 3.3 Admin catalog

#### `GET /admin/catalog/products` và `GET /admin/catalog/products/{productId}`

Admin list hỗ trợ `page`, `size`, `status`, `shop_id`, `category_id`, `q`, `updated_from`, `updated_to`.

```json
{
  "data": [
    { "product_id": "product-01912f31", "title": "Áo khoác cotton", "shop_id": "shop-01912f30", "status": "ACTIVE", "category_id": "cat-01912f20", "updated_at": "2026-08-30T09:05:00Z" }
  ],
  "meta": { "request_id": "req-01912f59", "page": 1, "size": 20, "total": 4210 }
}
```

Detail trả seller/admin fields, audit summary, KYC/shop projection và inventory display; không trả secret:

```json
{
  "data": {
    "product_id": "product-01912f31",
    "status": "ACTIVE",
    "shop_projection": { "shop_id": "shop-01912f30", "status": "ACTIVE", "kyc_status": "APPROVED" },
    "audit_summary": [{ "action": "PUBLISHED", "actor": "seller-usr-01912f10", "occurred_at": "2026-08-30T09:05:00Z" }],
    "inventory_display": { "status": "IN_STOCK", "as_of": "2026-08-30T08:59:59Z" }
  },
  "meta": { "request_id": "req-01912f60" }
}
```

#### `POST /admin/catalog/products/{productId}/block`

Request: `{ "version": 8, "reason": "Vi phạm chính sách" }`.

```json
{ "data": { "product_id": "product-01912f31", "status": "BLOCKED", "blocked_at": "2026-08-30T09:10:00Z", "version": 9 }, "meta": { "request_id": "req-01912f61" } }
```

Bắt buộc reason + admin permission + step-up 2FA context. Block là post-publication action, phát `product.blocked`, không xóa dữ liệu.

#### `POST /admin/catalog/products/{productId}/unblock`

Request: `{ "version": 9, "reason": "Đã xử lý" }`.

```json
{ "data": { "product_id": "product-01912f31", "status": "INACTIVE", "next_status": "INACTIVE", "version": 10 }, "meta": { "request_id": "req-01912f62" } }
```

Status mặc định `INACTIVE`; seller phải publish lại. Admin quyền cao có thể restore `ACTIVE` chỉ khi publish policy đạt và phải có audit reason; baseline endpoint không nhận `ACTIVE` trực tiếp.

### 3.4 Admin category

#### `GET /admin/catalog/categories`

Query: `status`, `parent_id`, `page`, `size`; trả cả active/inactive/archived theo permission.

```json
{
  "data": [
    { "category_id": "cat-01912f20", "name": "Điện tử", "slug": "dien-tu", "parent_id": null, "path": "/cat-01912f20", "depth": 1, "status": "ACTIVE", "tax_rate_bps": 1000, "version": 1 }
  ],
  "meta": { "request_id": "req-01912f63", "page": 1, "size": 20, "total": 42 }
}
```

#### `POST /admin/catalog/categories`

Request: `{ "parent_id": null, "name": "Điện tử", "slug": "dien-tu", "sort_order": 10, "tax_rate_bps": 1000 }`.

```json
{ "data": { "category_id": "cat-01912f20", "name": "Điện tử", "slug": "dien-tu", "parent_id": null, "path": "/cat-01912f20", "depth": 1, "status": "ACTIVE", "tax_rate_bps": 1000, "version": 1 }, "meta": { "request_id": "req-01912f64" } }
```

Validate cycle/depth/slug/name. `tax_rate_bps` là thuế suất VAT của danh mục, đơn vị basis point (10% = `1000`), khoảng hợp lệ 0–10000 — xem `docs/db/product-catalog.md` §3.x. **Bắt buộc khi tạo category root** (không cha), vì thuế suất phải thừa kế được xuống category con; category con để trống thì thừa kế từ cha. Thiếu `tax_rate_bps` ở category root → `400 PRODUCT_CATEGORY_INVALID`.

#### `PATCH /admin/catalog/categories/{categoryId}`

Request: `{ "version": 1, "parent_id": "category-...", "name": "Điện tử gia dụng", "status": "ACTIVE", "tax_rate_bps": 1000 }`.

```json
{ "data": { "category_id": "cat-01912f20", "name": "Điện tử gia dụng", "slug": "dien-tu-gia-dung", "parent_id": "category-01912f01", "path": "/category-01912f01/cat-01912f20", "depth": 2, "status": "ACTIVE", "tax_rate_bps": 1000, "version": 2 }, "meta": { "request_id": "req-01912f65" } }
```

Move subtree phải cập nhật `path/depth` atomically; không vượt depth 5; collision trả `PRODUCT_SLUG_CONFLICT` hoặc `PRODUCT_CATEGORY_INVALID`.

`tax_rate_bps` bỏ trống trong PATCH giữ nguyên giá trị cũ (không set về `null`). Đổi `tax_rate_bps` **không hồi tố** đơn đã checkout — Order-Commerce đã snapshot thuế suất vào `order_items` tại thời điểm đặt hàng (`order-commerce-docs/docs/lld/order-commerce.md` §6.4). Đổi thuế suất của category có category con **không** tự đổi giá trị con đã set riêng — chỉ ảnh hưởng con đang thừa kế (con để `tax_rate_bps=null`).

#### `POST /admin/catalog/categories/{categoryId}/archive`

Request: `{ "version": 2, "reason": "Taxonomy mới" }`.

```json
{ "data": { "category_id": "cat-01912f20", "status": "ARCHIVED", "version": 3 }, "meta": { "request_id": "req-01912f66" } }
```

Không hard-delete; không nhận assignment mới; phải có migration policy nếu category đang được dùng.

## 4. Mã lỗi chung và mapping

| Code | HTTP | Áp dụng |
|---|---:|---|
| `PRODUCT_INVALID_INPUT` | 400 | JSON/field không hợp lệ. |
| `PRODUCT_TITLE_REQUIRED` | 400 | Thiếu/vượt title. |
| `PRODUCT_DESCRIPTION_INVALID` | 400 | Description publish không hợp lệ/vượt limit. |
| `PRODUCT_CATEGORY_REQUIRED` | 400 | Publish thiếu primary. |
| `PRODUCT_CATEGORY_INVALID` | 400 | Category không active/assignment sai. |
| `PRODUCT_ATTRIBUTE_INVALID` | 400 | Sai typed attribute/enum/key. |
| `PRODUCT_PRICE_INVALID` | 400 | Price không phải integer VND/range. |
| `PRODUCT_MEDIA_REQUIRED` | 400 | Thiếu cover READY. |
| `PRODUCT_MEDIA_INVALID` | 400 | Object metadata/checksum/type sai. |
| `PRODUCT_EXPORT_TOO_LARGE` | 400 | Vượt `PRODUCT_EXPORT_MAX_ROWS` (10.000), cần lọc hẹp hơn. |
| `PRODUCT_NOT_FOUND` | 404 | Không tồn tại/không visible trong scope. |
| `PRODUCT_FORBIDDEN` | 403 | Sai owner/shop/permission. |
| `PRODUCT_KYC_REQUIRED` | 403 | Shop chưa APPROVED. |
| `PRODUCT_SHOP_SUSPENDED` | 403 | Shop suspended. |
| `PRODUCT_BLOCKED` | 403 | Product đang blocked. |
| `PRODUCT_STATE_INVALID` | 409 | Transition không hợp lệ. |
| `PRODUCT_VERSION_CONFLICT` | 409 | Version mismatch. |
| `PRODUCT_SKU_DUPLICATE` | 409 | Duplicate variant/seller SKU. |
| `PRODUCT_SKU_LIMIT_EXCEEDED` | 409 | >1.000 SKU/product. |
| `PRODUCT_MEDIA_LIMIT_EXCEEDED` | 409 | >12 ảnh hoặc >3 video. |
| `PRODUCT_SLUG_CONFLICT` | 409 | Slug collision. |
| `CATEGORY_DEPTH_EXCEEDED` | 409 | Category >5 level. |
| `CATEGORY_CYCLE_DETECTED` | 409 | Parent cycle. |
| `PRODUCT_ARCHIVED` | 409 | Mutation archived product. |
| `INVENTORY_PROJECTION_STALE` | 200 metadata | Snapshot stale; không phải deduction error. |
| `CATALOG_EVENT_PUBLISH_FAILED` | 503 | Outbox chưa publish; request xử lý domain theo transaction nhưng integration đang retry. |
| `INTERNAL_ERROR` | 500 | Unexpected error. |

Rate limit, trace ID, JWT invalid/expired và generic 401/403 có thể được API Gateway chuẩn hóa; Product vẫn phải log `request_id` và actor scope để audit.

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | API version là `/api/v1`, Gateway expose route tương ứng. | Ảnh hưởng routing, backward compatibility và OpenAPI base URL. | API Gateway owner |
| 2 | Seller product mutation dùng `version` trong body thay vì HTTP `If-Match`. | Nếu platform chuẩn hóa ETag, cần đổi tất cả mutation contract. | Platform/API owner |
| 3 | `GET /products` chỉ catalog filter cơ bản; full-text/facet thuộc Search. | Nếu Product phải search full-text, cần thêm index/query contract và SLA. | Search owner |
| 4 | Product response hiển thị stock projection nhưng không reserve/deduct. | Nếu UI gọi Product để mua, phải đổi flow sang Inventory availability/reserve. | Inventory/Order owner |
| 5 | `price_override=null` fallback theo product price policy. | Cần chốt schema effective price nếu SKU có pricing độc lập. | Product + Order owner |
| 6 | Media complete có thể trả `SCANNING` nếu virus scan async. | Ảnh hưởng publish readiness và frontend polling. | Security/DevOps |
| 7 | Admin unblock baseline luôn về `INACTIVE`. | Nếu cần restore active một bước, phải thêm permission/endpoint explicit. | Product owner |
| 8 | Shop/KYC event projection có đủ `shop_status`, `kyc_status`, `source_version`. | Ảnh hưởng publish gate và shop visibility. | Auth User owner |
| 9 | `GET /products?product_ids=` (batch ≤100) dùng để hydrate thẻ sản phẩm cho Favorites (`auth-user`) và Cart (`order-commerce`); public shop **profile** `GET /shops/{id}` thuộc `auth-user`, Product Catalog chỉ giữ `GET /shops/{slug}/products`. | Nếu Shop hero cần một endpoint hợp nhất, cần BFF hoặc chuyển ownership. | Product + Auth User owner |
| 10 | `display_as`/`value_meta` chỉ là hint render, không đổi `variant_key`; khớp chip "Attribute type Image/Color/Text" trong Penpot mà không thêm `AttributeType` mới. | Nếu cần ràng buộc swatch bắt buộc, thêm validate. | Product + Frontend |
