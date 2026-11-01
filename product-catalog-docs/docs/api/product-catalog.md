# API Spec — Product Catalog Service

> Nguồn: `docs/lld/product-catalog.md` · `docs/db/product-catalog.md` · Code: `ecommerce-taca/Product-Catalog-Service` @ `develop` (`6572ece`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ✅ Đã triển khai — public/seller/admin REST, import/export Excel, media S3, moderation, outbox (CDC Debezium), consumer `shop`/`inventory`/`rating`. Lệch thiết kế ghi rõ ở từng mục (event SKU/category chưa phát, step-up admin chưa kiểm).
> Base path: `/api/v1` (env `GLOBAL_PREFIX`; `/health/*` không có prefix) · Port mặc định `3000` · Public traffic đi qua API Gateway · Content-Type: `application/json`

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Trace | W3C `traceparent`/`tracestate` qua REST và Kafka (do Gateway/service propagate, client không gửi). |
| Actor context | Đọc `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope` do Gateway inject (client không gửi được — Gateway strip). |

### 1.1 Authentication và authorization

| Nhóm route | Authentication | Authorization |
|---|---|---|
| Buyer/public `GET` | Có thể anonymous | Chỉ trả `ACTIVE` và visibility hợp lệ. |
| Seller `/seller/*` | JWT từ Gateway | `SELLER`/`SELLER_STAFF`; product mutation phải đúng `shop_id` scope. |
| Admin `/admin/catalog/*` | JWT từ Gateway | `CATALOG_ADMIN` hoặc `SUPER_ADMIN`. Thiết kế yêu cầu step-up 2FA cho block/unblock/archive nhưng **code chưa kiểm** (không đọc header step-up). |
| Internal integration | mTLS/service token qua Gateway hoặc internal policy | Chỉ endpoint contract được allowlist; không truy cập MongoDB trực tiếp. |

`ActorContextGuard` (global, deny-by-default): route không gắn `@Public()` mà thiếu `X-User-ID` → `401 UNAUTHORIZED`; sai role → `403 PRODUCT_FORBIDDEN`. Role đọc từ `X-User-Roles` (CSV, chuẩn hoá in hoa); `SUPER_ADMIN` vượt mọi kiểm tra role/permission. Shop scope lấy từ `X-User-Shop-Scope` (một giá trị).

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

- List response: `{ data: [], meta: { request_id, page, size, total, total_pages } }`.
- Envelope do `ResponseEnvelopeInterceptor` bọc; route gắn `@SkipEnvelope()` (tải file) trả raw. Số `BigInt`/BSON `Long` được chuyển sang `number`.
- `ValidationPipe` `whitelist + forbidNonWhitelisted + transform`: field lạ hoặc sai → `400 PRODUCT_INVALID_INPUT` với `details` là danh sách lỗi class-validator.
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

| # | Method + path | Actor | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /products` | Public | Catalog listing cơ bản. | ✅ |
| 2 | `GET /products/{productId}` | Public | Product detail + display stock projection. | ✅ |
| 3 | `GET /categories` | Public | Category tree. | ✅ |
| 4 | `GET /categories/{categoryId}` | Public | Category detail/children. | ✅ |
| 5 | `GET /shops/{shopId}/products` | Public | Product listing theo shop. | ✅ |
| 6 | `POST /seller/products` | Seller | Tạo SPU draft. | ✅ |
| 7 | `GET /seller/products` | Seller | Seller product list. | ✅ |
| 7a | `GET /seller/products/export` | Seller | Xuất danh sách sản phẩm ra file (.xlsx/.csv). | ✅ |
| 7b | `POST /seller/products/import/template` (và `GET`) | Seller | Tải file Excel template mẫu tùy biến theo bộ lọc (danh mục, sản phẩm, số dòng, layout). | ✅ |
| 7c | `POST /seller/products/import` | Seller | Upload file Excel (.xlsx, <= 100 SPUs, <= 2MB), phản hồi `202 Accepted` kèm `job_id`. | ✅ |
| 7d | `GET /seller/products/import/jobs/{jobId}` | Seller | Tra cứu tiến độ xử lý import theo thời gian thực. | ✅ |
| 7e | `GET /seller/products/import/jobs/{jobId}/result` | Seller | Tải file Excel báo cáo kết quả chi tiết các dòng lỗi. | ✅ |
| 8 | `GET /seller/products/{productId}` | Seller | Seller editor detail. | ✅ |
| 9 | `PATCH /seller/products/{productId}` | Seller | Cập nhật product fields. | ✅ |
| 10 | `PUT /seller/products/{productId}/skus` | Seller | Replace/update SKU set. | ✅ |
| 10a | `GET /seller/products/{productId}/skus` | Seller | Đọc danh sách SKU của product trong shop. | ✅ |
| 11 | `PUT /seller/products/{productId}/categories` | Seller | Gán 1 primary + tối đa 2 secondary. | ✅ |
| 12 | `POST /seller/products/{productId}/publish` | Seller | Publish/resume product. | ✅ |
| 13 | `POST /seller/products/{productId}/unpublish` | Seller | Đưa `ACTIVE` về `INACTIVE`. | ✅ |
| 14 | `POST /seller/products/{productId}/archive` | Seller | Soft archive. | ✅ |
| 15 | `POST /seller/products/{productId}/media/upload-url` | Seller | Nhận signed upload URL. | ✅ |
| 16 | `POST /seller/products/{productId}/media/complete` | Seller | Verify upload và lưu metadata READY. | ✅ |
| 16a | `GET /seller/products/{productId}/media` | Seller | Liệt kê media của product (`media_id, product_id, sku_id, scope, object_key, content_type, size_bytes, sort_order, is_cover, status, url, uploaded_by, created_at, updated_at`). | ✅ |
| 16b | `DELETE /seller/products/{productId}/media/{mediaId}` | Seller | Xoá media, trả `{ "success": true }`, phát `product.media_deleted`. | ✅ |
| 17 | `GET /admin/catalog/products` | Admin | Catalog moderation/read. | ✅ |
| 18 | `GET /admin/catalog/products/{productId}` | Admin | Xem product bất kể public visibility. | ✅ |
| 19 | `POST /admin/catalog/products/{productId}/block` | Admin | Post-publication block. | ✅ |
| 20 | `POST /admin/catalog/products/{productId}/unblock` | Admin | Restore an toàn về `INACTIVE`. | ✅ |
| 20a | `GET /admin/catalog/audits` | Admin | Tra cứu `catalog_audits`; query `target_id`, `target_type`, `actor_user_id`, `action`, `page` (≥1), `size` (1–100). | ✅ |
| 21 | `GET /admin/catalog/categories` | Admin | Quản trị taxonomy. | ✅ |
| 22 | `POST /admin/catalog/categories` | Admin | Tạo category. | ✅ |
| 23 | `PATCH /admin/catalog/categories/{categoryId}` | Admin | Sửa category/parent/status. | ✅ |
| 24 | `POST /admin/catalog/categories/{categoryId}/archive` | Admin | Soft archive category. | ✅ |
| 25 | `GET /health/live` | Internal/ops | Liveness process-only (Gateway active healthcheck gọi — bắt buộc). | ✅ |
| 26 | `GET /health/ready` | Internal/ops | Readiness: MongoDB sẵn sàng (app không có outbox consumer/publisher riêng dưới CDC — readiness của Debezium connector nằm ở Kafka Connect, ngoài phạm vi health endpoint của service này). | ✅ |

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
    "primary_category_id": "category-01912f20",
    "tax_rate_bps": 1000,
    "title": "Áo khoác cotton",
    "slug": "ao-khoac-cotton",
    "price": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
    "cover_media": { "media_id": "media-01912f32", "url": "https://cdn.example/signed", "content_type": "image/webp" },
    "rating_summary": { "avg": 4.5, "count": 128 },
    "stock_display": { "status": "IN_STOCK", "as_of": "2026-08-30T08:59:59Z" }
  }],
  "meta": { "page": 1, "size": 20, "total": 1, "total_pages": 1, "request_id": "req-01912f50" }
}
```

Ràng buộc: `min_price/max_price` là integer VND; public không được query draft/blocked/archived; stock display có thể `UNKNOWN`/`STALE` và không phải purchase guarantee. Với `product_ids`, product không tồn tại/không visible bị bỏ khỏi kết quả (không lỗi) — caller tự phát hiện ID nào biến mất.

`tax_rate_bps` là **thuế suất VAT của sản phẩm**, đơn vị basis point (10% = `1000`). Product Catalog đã giải quyết xong việc thừa kế theo cây danh mục và trả ra giá trị cuối cùng — caller không phải leo cây. `checkout-service` snapshot giá trị này tại lúc checkout (lưu ở `order_items` của `order-service`) để tách thuế ghi hoá đơn (`checkout-docs/docs/lld/checkout.md` §6.3). Giá trong `price` **đã bao gồm** VAT.

`rating_summary` là cache **read-only** từ event `rating.aggregate.updated` (sở hữu bởi `rating-comment`); `avg: null, count: 0` nếu sản phẩm chưa có review nào.

#### `GET /products/{productId}`

Response `200` trả `ProductDetail` gồm `product_id`, shop snapshot, title/description/brand, price summary, active categories, active SKUs, media READY và `stock_display` theo SKU:

```json
{
  "data": {
    "product_id": "product-01912f31",
    "status": "ACTIVE",
    "title": "Áo khoác cotton",
    "primary_category_id": "category-01912f20",
    "tax_rate_bps": 1000,
    "price": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
    "rating_summary": { "avg": 4.5, "count": 128 },
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
      "category_id": "category-01912f20",
      "name": "Điện tử",
      "slug": "dien-tu",
      "path": "/category-01912f20",
      "depth": 1,
      "tax_rate_bps": 1000,
      "children": [
        { "category_id": "category-01912f21", "name": "Điện thoại", "slug": "dien-thoai", "path": "/category-01912f20/category-01912f21", "depth": 2, "tax_rate_bps": null, "children": [] }
      ]
    }
  ],
  "meta": { "request_id": "req-01912f51" }
}
```

`GET /categories/{categoryId}`: trả metadata category và children; category archived không public.

```json
{
  "data": { "category_id": "category-01912f21", "name": "Điện thoại", "slug": "dien-thoai", "path": "/category-01912f20/category-01912f21", "depth": 2, "tax_rate_bps": null, "children": [] },
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
      "primary_category_id": "category-01912f20",
      "price_summary": { "base_price": 299000, "sale_price": 249000, "currency": "VND" },
      "sku_count": 3,
      "cover_media": { "media_id": "media-01912f32", "url": "https://cdn.example/signed" },
      "updated_at": "2026-08-30T09:00:00Z"
    }
  ],
  "meta": { "request_id": "req-01912f53", "page": 1, "size": 20, "total": 84, "total_pages": 5 }
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

#### `POST /seller/products/import/template` (Đồng thời hỗ trợ `GET`)

Sinh và tải về tệp bảng tính mẫu Excel (`.xlsx`) được tùy biến động theo bộ lọc và nhu cầu nhập liệu của người bán.

Request Body (`application/json`):
- `row_count` (integer, tuỳ chọn, 5..200, mặc định: 100): Số dòng dữ liệu trống khởi tạo sẵn định dạng (viền bảng, dropdown, prefill category). Người bán hoàn toàn không bắt buộc truyền trường này và không cần tự tính số dòng/biến thể; các dòng trống người bán không điền sẽ được worker tự động bỏ qua an toàn.
- `category_ids` (string[], tuỳ chọn): Danh sách mã danh mục muốn tạo mẫu. Tự động hiển thị tên thân thiện dạng `Tên Danh Mục [UUID]`.
- `product_ids` (string[], tuỳ chọn): Danh sách mã sản phẩm SPU hiện hữu muốn nhập bổ sung biến thể SKU.
- `layout_mode` (string, tuỳ chọn, `'MULTI_SHEET' | 'SINGLE_SHEET'`):
  - `MULTI_SHEET`: Tự động phân chia mỗi danh mục được chọn thành một tab sheet riêng mang tên danh mục đó (kèm cell protection `spinCount: 1`).
  - `SINGLE_SHEET`: Tất cả sản phẩm nằm trong 1 sheet dữ liệu ("Sản phẩm & Biến thể") kèm Data Validation dropdown chọn danh mục.

Cấu trúc Workbook:
1. **Sheet Dữ liệu (Multi-sheet hoặc "Sản phẩm & Biến thể")**: Chứa các cột SPU/SKU cố định và các cột thuộc tính động của từng danh mục. Với thuộc tính `ENUM`, nhúng Data Validation dropdown danh sách giá trị (`allowed_values`).
2. **Sheet "Ví dụ điền mẫu"**: Minh họa 3 sản phẩm thực tế với số lượng biến thể đa dạng theo Nguyên tắc Vàng (SPU 1 Polo 4 biến thể, SPU 2 Balo 1 biến thể đơn, SPU 3 Sneaker 3 biến thể; dòng đầu điền tên sản phẩm, các dòng sau để trống để kế thừa).
3. **Sheet "Hướng dẫn & Danh mục"**: Nhấn mạnh bảng "Nguyên Tắc Vàng Phân Cụm Biến Thể" (Cùng mã tham chiếu = Các biến thể của cùng 1 sản phẩm; Đổi mã tham chiếu = Sản phẩm mới), chú giải chi tiết kiểu dữ liệu, các trường bắt buộc, quy tắc giá VND (>= 1.000).

Content Negotiation:
- **Mặc định** (hoặc `Accept: application/json`): Trả về JSON envelope chứa Presigned Download URL (lưu cache trên MinIO S3 với TTL 1 giờ):
  ```json
  {
    "data": {
      "download_url": "https://storage.example/signed-download/templates/product_import_template_0191ec4d.xlsx",
      "filename": "product_import_template_0191ec4d.xlsx",
      "expires_at": "2026-08-30T10:00:00Z"
    },
    "meta": { "request_id": "req-01912f69", "as_of": "2026-08-30T09:00:00Z" }
  }
  ```
- **Binary Stream** (`Accept: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`): Stream trực tiếp file Excel nhị phân về client.

Tương thích ngược: Hệ thống đồng thời tiếp nhận `GET /seller/products/import/template?category_id={categoryId}` cho các client cũ.

Lỗi:
- `400 PRODUCT_CATEGORY_INVALID`: `category_id` không tồn tại hoặc không ở trạng thái `ACTIVE`.
- `403 PRODUCT_SHOP_SUSPENDED`: Shop đang bị đình chỉ hoạt động.

#### `POST /seller/products/import`

Tiếp nhận tệp Excel (`.xlsx`) do người bán tải lên để khởi tạo tiến trình nạp sản phẩm hàng loạt (Bulk Ingest).
- Content-Type: `multipart/form-data` (Field: `file`, định dạng `.xlsx`, dung lượng $\le 2\text{ MB}$, số lượng $\le 100\text{ SPUs}$ / tối đa 200 dòng SKU).
- Quá trình nạp là **Bất đồng bộ (Async Job)**: API lưu tệp tạm vào Object Storage (S3), tạo bản ghi `import_jobs` ở trạng thái `PENDING`, đưa vào hàng đợi xử lý và phản hồi ngay lập tức (`< 500ms`) kèm `job_id`.

Response `202 Accepted`:

```json
{
  "data": {
    "job_id": "01912f70-7a1b-7c12-9c55-8b1c34a6d921",
    "status": "PENDING",
    "total_rows": null,
    "created_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "req-01912f70" }
}
```

Ràng buộc & Giới hạn:
- **Tenant Throttle**: Mỗi shop tại một thời điểm chỉ được phép có tối đa **1 job ở trạng thái `PENDING` hoặc `PROCESSING`**. Nếu shop tải lên tệp mới khi job cũ chưa hoàn tất $\to$ trả lỗi `409 PRODUCT_IMPORT_JOB_RUNNING`.
- **Global Concurrency Throttling**: Hệ thống khống chế tối đa **3 worker chạy đồng thời** trên toàn cụm máy chủ (`GLOBAL_MAX_CONCURRENT_WORKERS = 3`) để bảo vệ tài nguyên CPU 4 nhân 8 luồng và giữ trọn vẹn Connection Pool MongoDB cho luồng API chính.
- Shop chưa hoàn tất KYC (`PENDING`) vẫn được phép import sản phẩm (toàn bộ sản phẩm tạo ra đều ở trạng thái `DRAFT`). Shop `SUSPENDED` bị chặn `403 PRODUCT_SHOP_SUSPENDED`.
- Vượt dung lượng 2MB $\to$ `400 PRODUCT_IMPORT_FILE_TOO_LARGE`. Tệp sai định dạng (không phải `.xlsx`) $\to$ `400 PRODUCT_IMPORT_FILE_TYPE_INVALID`.

#### `GET /seller/products/import/jobs/{jobId}`

Tra cứu trạng thái và tiến độ xử lý của tiến trình import theo thời gian thực (Zero-Trust IDOR check: Chỉ xem được job thuộc `shop_id` của actor).

Response `200`:

```json
{
  "data": {
    "job_id": "01912f70-7a1b-7c12-9c55-8b1c34a6d921",
    "status": "PROCESSING",
    "total_rows": 98,
    "processed_rows": 45,
    "success_count": 40,
    "error_count": 5,
    "started_at": "2026-08-30T09:00:02Z",
    "completed_at": null,
    "created_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "req-01912f71" }
}
```

Trạng thái `status`: `'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED'`.
Khi `status` đạt `'COMPLETED'` hoặc `'FAILED'`, trường `completed_at` được cập nhật.

Lỗi:
- `404 PRODUCT_NOT_FOUND`: `jobId` không tồn tại hoặc không thuộc quyền sở hữu của gian hàng hiện tại.

#### `GET /seller/products/import/jobs/{jobId}/result`

Tải về tệp Excel báo cáo kết quả đối soát chi tiết sau khi job hoàn tất (Áp dụng chiến lược **Partial Success**).
- Nếu `error_count == 0`: Trả về thông báo 100% sản phẩm đã được tạo thành công dạng `DRAFT`.
- Nếu `error_count > 0`: Sinh tệp Excel chứa nguyên trạng các dòng bị lỗi kèm thêm cột **"Lý do lỗi"** giải thích cụ thể bằng tiếng Việt (Ví dụ: *"Dòng 14: Mã SKU 'AO-01' đã tồn tại trong gian hàng"*, *"Dòng 25: Lỗi tải ảnh từ URL (404 Not Found)"*).
- **Phòng chống tấn công Formula Injection (CWE-1236)**: Mọi ô dữ liệu văn bản bắt đầu bằng ký tự `=`, `+`, `-`, `@` đều được tự động tiền tố thêm dấu nháy đơn `'` trước khi ghi vào tệp Excel kết quả.
- Tệp kết quả được lưu trữ tạm thời trên Object Storage (S3/MinIO) và cấp phát đường dẫn tải về an toàn thông qua Presigned Download URL có hiệu lực trong 30 phút.

Response `200`:

```json
{
  "data": {
    "job_id": "01912f70-7a1b-7c12-9c55-8b1c34a6d921",
    "result_file_url": "https://storage.example/signed-download/import-result-01912f70.xlsx",
    "total_rows": 98,
    "success_count": 92,
    "error_count": 6,
    "expires_at": "2026-08-30T09:35:00Z"
  },
  "meta": { "request_id": "req-01912f72" }
}
```

Lỗi:
- `400 PRODUCT_IMPORT_JOB_NOT_FINISHED`: Job đang xử lý (`PENDING` hoặc `PROCESSING`), chưa có file kết quả.
- `400 PRODUCT_IMPORT_NO_ERRORS`: Job hoàn thành 100% không có lỗi, không có tệp kết quả lỗi.
- `404 PRODUCT_NOT_FOUND`: Job không tồn tại hoặc không thuộc sở hữu của shop.

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
    "categories": { "primary_category_id": "category-01912f20", "secondary_category_ids": [] },
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

`price_summary` gửi ở endpoint này chỉ có tác dụng khi product **chưa có SKU nào**. Khi product đã có SKU, giá trị `price_summary` này bị ghi đè ở lần `PUT .../skus` kế tiếp (application service tự tính lại từ SKU `ACTIVE`) — đây là hành vi đúng theo thiết kế, không phải lỗi.

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

`price_override` **chỉ override `sale_price`** của SKU; `base_price` luôn kế thừa từ `products.price_summary.base_price`, không có field override riêng cho base. `skus[].price` trả về ở endpoint đọc (`GET /products/{productId}`, `GET /seller/products/{productId}`) là giá trị **đã resolve** (`sale_price = price_override ?? products.price_summary.sale_price`), không phải giá trị thô lưu trong `price_override`.

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
    { "product_id": "product-01912f31", "title": "Áo khoác cotton", "shop_id": "shop-01912f30", "status": "ACTIVE", "category_id": "category-01912f20", "updated_at": "2026-08-30T09:05:00Z" }
  ],
  "meta": { "request_id": "req-01912f59", "page": 1, "size": 20, "total": 4210, "total_pages": 211 }
}
```

Detail trả seller/admin fields, audit summary, KYC/shop projection và inventory display; không trả secret:

```json
{
  "data": {
    "product_id": "product-01912f31",
    "status": "ACTIVE",
    "shop_projection": { "shop_id": "shop-01912f30", "status": "ACTIVE", "kyc_status": "APPROVED" },
    "audit_summary": [{ "action": "PUBLISH", "actor": "seller-usr-01912f10", "occurred_at": "2026-08-30T09:05:00Z" }],
    "stock_display": { "status": "IN_STOCK", "as_of": "2026-08-30T08:59:59Z" }
  },
  "meta": { "request_id": "req-01912f60" }
}
```

#### `POST /admin/catalog/products/{productId}/block`

Request: `{ "version": 8, "reason": "Vi phạm chính sách" }`.

```json
{ "data": { "product_id": "product-01912f31", "status": "BLOCKED", "blocked_at": "2026-08-30T09:10:00Z", "version": 9 }, "meta": { "request_id": "req-01912f61" } }
```

Bắt buộc reason + role admin. Step-up 2FA là yêu cầu thiết kế, code hiện chưa kiểm (câu hỏi mở). Block là post-publication action, phát `product.blocked`, không xóa dữ liệu.

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
    { "category_id": "category-01912f20", "name": "Điện tử", "slug": "dien-tu", "parent_id": null, "path": "/category-01912f20", "depth": 1, "status": "ACTIVE", "tax_rate_bps": 1000, "version": 1 }
  ],
  "meta": { "request_id": "req-01912f63", "page": 1, "size": 20, "total": 42, "total_pages": 3 }
}
```

#### `POST /admin/catalog/categories`

Request: `{ "parent_id": null, "name": "Điện tử", "slug": "dien-tu", "sort_order": 10, "tax_rate_bps": 1000 }`.

```json
{ "data": { "category_id": "category-01912f20", "name": "Điện tử", "slug": "dien-tu", "parent_id": null, "path": "/category-01912f20", "depth": 1, "status": "ACTIVE", "tax_rate_bps": 1000, "version": 1 }, "meta": { "request_id": "req-01912f64" } }
```

Validate cycle/depth/slug/name. `tax_rate_bps` là thuế suất VAT của danh mục, đơn vị basis point (10% = `1000`), khoảng hợp lệ 0–10000 — xem `docs/db/product-catalog.md` §3.x. **Bắt buộc khi tạo category root** (không cha), vì thuế suất phải thừa kế được xuống category con; category con để trống thì thừa kế từ cha. Thiếu `tax_rate_bps` ở category root → `400 PRODUCT_CATEGORY_INVALID`.

#### `PATCH /admin/catalog/categories/{categoryId}`

Request: `{ "version": 1, "parent_id": "category-...", "name": "Điện tử gia dụng", "status": "ACTIVE", "tax_rate_bps": 1000 }`.

```json
{ "data": { "category_id": "category-01912f20", "name": "Điện tử gia dụng", "slug": "dien-tu-gia-dung", "parent_id": "category-01912f01", "path": "/category-01912f01/category-01912f20", "depth": 2, "status": "ACTIVE", "tax_rate_bps": 1000, "version": 2 }, "meta": { "request_id": "req-01912f65" } }
```

Move subtree phải cập nhật `path/depth` atomically; không vượt depth 5; collision trả `PRODUCT_SLUG_CONFLICT` hoặc `PRODUCT_CATEGORY_INVALID`.

`tax_rate_bps` bỏ trống trong PATCH giữ nguyên giá trị cũ (không set về `null`). Đổi `tax_rate_bps` **không hồi tố** đơn đã checkout — `checkout-service` đã snapshot thuế suất (lưu ở `order_items` của `order-service`) tại thời điểm đặt hàng (`checkout-docs/docs/lld/checkout.md` §6.3). Đổi thuế suất của category có category con **không** tự đổi giá trị con đã set riêng — chỉ ảnh hưởng con đang thừa kế (con để `tax_rate_bps=null`).

#### `POST /admin/catalog/categories/{categoryId}/archive`

Request: `{ "version": 2, "reason": "Taxonomy mới" }`.

```json
{ "data": { "category_id": "category-01912f20", "status": "ARCHIVED", "version": 3 }, "meta": { "request_id": "req-01912f66" } }
```

Không hard-delete; không nhận assignment mới; phải có migration policy nếu category đang được dùng.

## 4. Mã lỗi chung

| Code | HTTP | Áp dụng |
|---|---:|---|
| `PRODUCT_INVALID_INPUT` | 400 | JSON/field không hợp lệ. |
| `PRODUCT_TITLE_REQUIRED` | 400 | Thiếu/vượt title. |
| `PRODUCT_DESCRIPTION_INVALID` | 400 | Description publish không hợp lệ/vượt limit. |
| `PRODUCT_CATEGORY_REQUIRED` | 400 | Publish thiếu primary. |
| `PRODUCT_CATEGORY_INVALID` | 400 | Category không active/assignment sai. |
| `PRODUCT_ATTRIBUTE_INVALID` | 400 | Sai typed attribute/enum/key. |
| `PRODUCT_PRICE_INVALID` | 400 | Price không phải integer VND/range. |
| `PRODUCT_SKU_REQUIRED` | 400 | Publish không có SKU active. |
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
| `PRODUCT_IMPORT_FILE_TOO_LARGE` | 400 | Tệp Excel nạp vượt quá 2MB. |
| `PRODUCT_IMPORT_FILE_TYPE_INVALID` | 400 | Tệp nạp không đúng định dạng Excel .xlsx hoặc sai magic bytes. |
| `PRODUCT_IMPORT_JOB_RUNNING` | 409 | Shop đang có một import job khác đang chạy (Tenant Throttle). |
| `PRODUCT_IMPORT_JOB_NOT_FINISHED` | 400 | Tải kết quả khi job chưa hoàn tất (PENDING/PROCESSING). |
| `PRODUCT_IMPORT_NO_ERRORS` | 400 | Tải kết quả lỗi khi job thành công 100% không có lỗi. |
| `PRODUCT_IMPORT_TOO_MANY_ROWS` | 400 | Tệp nạp vượt quá giới hạn 200 dòng SKU. |
| `PRODUCT_IMPORT_TOO_MANY_SPUS` | 400 | Tệp nạp vượt quá giới hạn 100 SPUs. |
| `PRODUCT_SKU_DUPLICATE_VARIANT` | 400 | Trùng lặp tổ hợp thuộc tính biến thể trong cùng một SPU. |
| `PRODUCT_SPU_STATUS_INVALID` | 400 | Không thể nạp biến thể vào SPU đang ở trạng thái BLOCKED hoặc ARCHIVED. |
| `PRODUCT_SPU_REJECTED` | 400 | Biến thể bị từ chối do SPU cha có lỗi không thể khởi tạo. |
| `PRODUCT_IMPORT_EMPTY_FILE` | 400 | Tệp Excel không chứa dòng dữ liệu nào hợp lệ. |
| `PRODUCT_IMPORT_FILE_CORRUPT` | 400 | Tệp trên bộ lưu trữ S3 bị hỏng hoặc không thể đọc. |
| `PRODUCT_IMPORT_WORKER_TIMEOUT` | 500 / job error | Tiến trình xử lý import bị quá hạn do worker mất tín hiệu heartbeat hoặc sự cố hệ thống. |
| `PRODUCT_IMPORT_SERVICE_UNAVAILABLE` | 400 / 503 | Dịch vụ xử lý/xuất kết quả Excel chưa sẵn sàng. |
| `MEDIA_INVALID_URL_BLOCKED` | 400 | URL ảnh vi phạm an toàn SSRF (trỏ về IP nội bộ/metadata). |
| `DATABASE_TRANSACTION_FAILED` | 500 | Giao dịch cơ sở dữ liệu hàng loạt thất bại và đã rollback an toàn. |
| `PRODUCT_SKU_INVALID` | 400 | Media gắn `sku_id` không thuộc product. |
| `MEDIA_DOWNLOAD_FAILED` | job error | Import: tải ảnh từ URL thất bại (ghi vào `error_summary` của job, không trả HTTP). |
| `UNAUTHORIZED` | 401 | Thiếu `X-User-ID` ở route không public (`ActorContextGuard`). |
| `SERVICE_UNAVAILABLE` | 503 | Map mặc định cho `503` không có code riêng. |
| `INVENTORY_PROJECTION_STALE` | 200 metadata | ⏳ Chưa có trong code. |
| `INTERNAL_ERROR` | 500 | Unexpected error. |

Mapping trong `GlobalExceptionFilter`: lỗi class-validator → `PRODUCT_INVALID_INPUT`; `413`/`LIMIT_FILE_SIZE` (multer) → `400 PRODUCT_IMPORT_FILE_TOO_LARGE`; Mongo duplicate key `11000` → `409` với `PRODUCT_SLUG_CONFLICT` (slug), `PRODUCT_SKU_DUPLICATE` (`seller_sku`/`variant_key`) hoặc `PRODUCT_STATE_INVALID` (key khác); `HttpException` không có code → `401 UNAUTHORIZED`, `403 PRODUCT_FORBIDDEN`, `404 PRODUCT_NOT_FOUND`, `409 PRODUCT_STATE_INVALID`, `503 SERVICE_UNAVAILABLE`, còn lại `INTERNAL_ERROR`. `trace_id` lấy từ trace context (W3C), mặc định 32 số 0.

`CATALOG_EVENT_PUBLISH_FAILED` đã bị bỏ khỏi error catalog (Cecilia, 2026-09-19, DOCS-CONSISTENCY-01): dưới CDC, publish lên Kafka tách rời khỏi request/response của mutation API nên mã lỗi này không còn tình huống nào để trả về đồng bộ — xem `docs/lld/product-catalog.md` §7/giả định #23.

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
| 8 | Payload event `shop.kyc.*` thật của Auth User **không có** field `kyc_status`/`source_version`/`logo_url` sẵn (khớp `auth-user-docs/docs/lld/auth-user.md` §6.2) — Product tự suy `kyc_status` từ `event_type` (`submitted`→`PENDING`, `approved`→`APPROVED`, `needs_info`→`NEEDS_INFO`, `rejected`→`REJECTED`, `expired`→`EXPIRED`) và dedupe bằng `event_id`+`occurred_at`. | Nếu Auth User đổi payload, `ShopProjectionService` suy sai `kyc_status`. | Auth User owner |
| 9 | `GET /products?product_ids=` (batch ≤100) dùng để hydrate thẻ sản phẩm cho Favorites (`auth-user`) và Cart (`cart-service`); public shop **profile** `GET /shops/{shopId}` thuộc `auth-user`, Product Catalog chỉ giữ `GET /shops/{shopId}/products` — cả hai khoá bằng `shop_id` UUID, không phải slug. | Nếu Shop hero cần một endpoint hợp nhất, cần BFF hoặc chuyển ownership. | Product + Auth User owner |
| 10 | `display_as`/`value_meta` chỉ là hint render, không đổi `variant_key`; khớp chip "Attribute type Image/Color/Text" trong Penpot mà không thêm `AttributeType` mới. | Nếu cần ràng buộc swatch bắt buộc, thêm validate. | Product + Frontend |
