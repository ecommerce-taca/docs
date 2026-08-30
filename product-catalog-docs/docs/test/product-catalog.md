# Test Plan — Product Catalog Service

> Nguồn: `docs/lld/product-catalog.md` · `docs/db/product-catalog.md` · `docs/api/product-catalog.md` · Cập nhật: `2026-08-30`
> Phạm vi: Product/SPU, SKU/variant, dynamic attributes, category, media metadata, shop/KYC projection, inventory display projection, outbox và public/seller/admin API.

## 1. Chuẩn bị kiểm thử

### 1.1 Môi trường và test double

| Thành phần | Baseline |
|---|---|
| Runtime | Node.js + NestJS, test bằng Jest/Supertest. |
| Database | MongoDB replica set test để kiểm transaction; seed idempotent. |
| Messaging | Kafka test container hoặc mock broker hỗ trợ retry, replay, out-of-order. |
| Object storage | S3/MinIO test bucket private; mock signed URL + HEAD/checksum. |
| Auth context | JWT/Gateway test fixture cho buyer, seller owner, seller staff, admin, user khác shop, token expired. |
| Service fixtures | Shop `ACTIVE/APPROVED`, `ACTIVE/NEEDS_INFO`, `SUSPENDED`; categories depth 1 và depth 5; product mọi lifecycle. |

### 1.2 Dữ liệu bắt buộc

- Product draft không có SKU/media/category.
- Product active có 2 SKU typed attributes: `STRING`, `NUMBER`, `BOOLEAN`, `ENUM`.
- Variant dimensions không dùng tên cố định `color`/`size`, có canonical key và duplicate fixture.
- Inventory projection `UNKNOWN`, `IN_STOCK`, `LOW_STOCK`, `OUT_OF_STOCK`, `STALE`, event source version cũ/mới.
- Media `READY` cover, `UPLOADING`, `SCANNING`, `REJECTED`, checksum sai, vượt 12 ảnh/3 video.
- Category root, child depth 5, parent inactive, parent cycle và assignment 1 primary + 2 secondary.
- Outbox pending, retry lần 1–3, dead-letter, duplicate consumer event.

### 1.3 Tiêu chí chung

1. Mỗi mutation domain phải atomic: lỗi validation/version/permission không để lại partial product/SKU/category/media/outbox/audit.
2. Public API không leak `DRAFT`, `INACTIVE`, `BLOCKED`, `ARCHIVED`, KYC document, secret hoặc stock ledger.
3. Product không reserve/deduct stock; `inventory_projections` chỉ là display projection.
4. Mọi event consumer idempotent, bỏ qua `source_version` cũ và hỗ trợ replay.
5. Error code, HTTP status, message tiếng Việt và `request_id` khớp API spec.

## 2. Manual QA theo UI/Penpot

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Buyer Home/Listing | Load catalog, filter category/shop/price, pagination | Chỉ thấy product `ACTIVE`; card có title, cover, price VND, stock display và `as_of`. |
| Buyer Listing | Snapshot `STALE`/`UNKNOWN`/`OUT_OF_STOCK` | UI không khẳng định số lượng realtime; trạng thái hiển thị đúng, không gọi Product để reserve. |
| Product Detail | Product nhiều SKU/dynamic attributes | Chọn được typed variant không hard-code color/size; price/stock đổi theo SKU. |
| Product Detail | Product active quantity 0 | Vẫn đọc được detail; nút mua bị disable/revalidate theo Inventory policy. |
| Shop page | Shop đổi name/slug/logo từ Auth event | Snapshot cập nhật sau event; product list không hiển thị shop suspended theo policy. |
| Seller Products | Empty/loading/error/offline | Hiển thị đúng empty state, spinner, retry/error message; không mất draft khi retry. |
| Seller Create | Tạo draft khi KYC `NEEDS_INFO` | Cho save draft, chặn publish với thông báo hoàn tất KYC. |
| Seller Editor | Sai title/description/price/category | Inline validation theo field; không gửi request nếu lỗi client; server vẫn validate. |
| Seller SKU | Tạo typed attributes và duplicate combination | Hiển thị lỗi duplicate/type/ENUM rõ ràng; không partial-save. |
| Seller Media | Upload signed URL, progress, scan/reject | Không upload bytes qua Gateway; cover chỉ được chọn khi `READY`; retry không tạo duplicate metadata. |
| Seller Publish | Thiếu category/SKU/price/cover/KYC | Publish bị chặn, hiển thị lỗi actionable; stock 0 không bị hiểu là thiếu điều kiện publish. |
| Seller Lifecycle | Publish → unpublish → publish; archive | Button/action phụ thuộc state; archived không có nút sửa/publish trong v1. |
| Concurrent edit | Hai tab sửa cùng product | Tab cũ nhận version conflict, reload/merge; không silently overwrite. |
| Admin Products | Filter mọi status, view reason | Admin thấy blocked/archived và audit, public không thấy. |
| Admin Block | Block có/không có reason, thiếu 2FA | Chỉ block khi permission + reason + step-up đạt; product biến mất khỏi public sau event. |
| Admin Category | Tạo/move/archive tree | Không tạo cycle/>5 level; category archived không nhận assignment mới. |
| Accessibility | Keyboard, focus, labels, contrast | Form/editor/list/detail có keyboard flow và accessible validation. |

## 3. API integration test cases

### 3.1 Public read API

| ID | Endpoint | Kiểm thử | Kỳ vọng |
|---|---|---|---|
| PC-API-001 | `GET /products` | Public listing mặc định | `200`, chỉ `ACTIVE`, pagination default 20. |
| PC-API-002 | `GET /products` | `size=101`, price âm/decimal | `400 PRODUCT_INVALID_INPUT` hoặc validation tương ứng; không query unbounded. |
| PC-API-003 | `GET /products` | category/shop/status filter | Kết quả đúng index/scope; public không override status để lấy draft. |
| PC-API-004 | `GET /products/{id}` | Active detail có nhiều SKU | `200`, dynamic attrs, price VND, media READY, stock per SKU. |
| PC-API-005 | `GET /products/{id}` | Draft/blocked/archived/nonexistent | `404 PRODUCT_NOT_FOUND`; không leak existence theo public policy. |
| PC-API-006 | `GET /products/{id}` | Inventory snapshot stale/unknown | `200`, metadata `STALE`/`UNKNOWN`, không synchronous deduction. |
| PC-API-007 | `GET /categories` | Tree depth ≤5 | `200`, chỉ category public `ACTIVE`, children đúng thứ tự. |
| PC-API-008 | `GET /categories/{id}` | Inactive/archived public | Không expose theo visibility policy; error/status ổn định. |
| PC-API-009 | `GET /shops/{slug}/products` | Shop active/suspended | Listing đúng shop snapshot/visibility; không bypass suspended policy. |
| PC-API-009b | `GET /products?product_ids=` | 100 ID (vài ID archived/không tồn tại), >100 ID | Trả đúng product `ACTIVE` visible, bỏ ID biến mất không lỗi; >100 ID → `400`. Dùng hydrate Favorites/Cart. |

### 3.2 Seller product API

| ID | Endpoint | Kiểm thử | Kỳ vọng |
|---|---|---|---|
| PC-API-010 | `POST /seller/products` | Seller tạo draft hợp lệ | `201`, `DRAFT`, version 1, outbox + audit cùng transaction. |
| PC-API-011 | `POST /seller/products` | Anonymous/buyer/other shop + body shop_id giả | `401/403 PRODUCT_FORBIDDEN`; scope lấy từ auth context. |
| PC-API-012 | `POST /seller/products` | Title/slug/price invalid hoặc slug trùng | `400` validation hoặc `409 PRODUCT_SLUG_CONFLICT`, no partial document. |
| PC-API-013 | `GET /seller/products` | Seller list | Chỉ product của shop; pagination/status filter đúng. |
| PC-API-014 | `GET /seller/products/{id}` | Owner đọc draft/blocked/archived | `200` theo seller policy, có version/status/reason. |
| PC-API-015 | `GET /seller/products/{id}` | Seller khác shop | `403 PRODUCT_FORBIDDEN` hoặc not-found policy, không leak. |
| PC-API-016 | `PATCH /seller/products/{id}` | Update draft hợp lệ | Version tăng 1, fields đúng, `product.updated` outbox. |
| PC-API-017 | `PATCH /seller/products/{id}` | Wrong version/concurrent tabs | `409 PRODUCT_VERSION_CONFLICT`, document giữ nguyên. |
| PC-API-018 | `PATCH /seller/products/{id}` | Đổi `shop_id`/product_id/archived | Từ chối immutable/state với code phù hợp. |
| PC-API-019 | `PUT /seller/products/{id}/skus` | Typed attributes + canonicalization | Variant key server-generated, SKU saved atomically. |
| PC-API-019b | `PUT /seller/products/{id}/skus` | `display_as=COLOR_SWATCH`/`IMAGE_THUMB` + `value_meta` | Lưu như hint; `variant_key` không đổi so với khi bỏ `display_as`; publish không bắt buộc `value_meta`. |
| PC-API-020 | `PUT /seller/products/{id}/skus` | Enum ngoài allowlist/sai NUMBER/unknown key | `400 PRODUCT_ATTRIBUTE_INVALID`, no partial SKU write. |
| PC-API-021 | `PUT /seller/products/{id}/skus` | Duplicate variant_key/seller_sku | `409 PRODUCT_SKU_DUPLICATE`; duplicate detection intra-request + DB. |
| PC-API-022 | `PUT /seller/products/{id}/skus` | >1.000 SKU hoặc >50 attrs | `409 PRODUCT_SKU_LIMIT_EXCEEDED`/`400 PRODUCT_ATTRIBUTE_INVALID`. |
| PC-API-023 | `PUT /seller/products/{id}/categories` | 1 primary + 2 secondary active | `200`, replace atomic, event emitted. |
| PC-API-024 | `PUT /seller/products/{id}/categories` | 0/>1 primary, >2 secondary, duplicate | `400 PRODUCT_CATEGORY_INVALID`/`PRODUCT_CATEGORY_REQUIRED`, no partial assignment. |
| PC-API-025 | `PUT /seller/products/{id}/categories` | Inactive/archived category | `400 PRODUCT_CATEGORY_INVALID`, no assignment. |
| PC-API-026 | `POST /seller/products/{id}/publish` | All readiness + KYC approved | `200 ACTIVE`, version increment, `product.published` outbox. |
| PC-API-027 | `POST /seller/products/{id}/publish` | KYC needs info/rejected/expired | `403 PRODUCT_KYC_REQUIRED`. |
| PC-API-028 | `POST /seller/products/{id}/publish` | Shop suspended | `403 PRODUCT_SHOP_SUSPENDED`. |
| PC-API-029 | `POST /seller/products/{id}/publish` | Missing title/description/category/SKU/price/cover | Correct field error, no state change. |
| PC-API-030 | `POST /seller/products/{id}/publish` | Stock projection quantity 0 | Publish succeeds if other policy passes; display is `OUT_OF_STOCK`. |
| PC-API-031 | `POST /seller/products/{id}/unpublish` | Active with valid version | `200 INACTIVE`, event emitted, no SKU/media delete. |
| PC-API-032 | `POST /seller/products/{id}/unpublish` | Draft/blocked/archived | `409 PRODUCT_STATE_INVALID`/`PRODUCT_BLOCKED`/`PRODUCT_ARCHIVED`. |
| PC-API-033 | `POST /seller/products/{id}/archive` | Valid lifecycle + reason | `200 ARCHIVED`, soft lifecycle + audit/event. |
| PC-API-034 | `POST /seller/products/{id}/archive` | Missing reason/wrong version | Validation/version conflict; no destructive partial action. |
| PC-API-035 | `POST /seller/products/{id}/media/upload-url` | Valid image/video and quota remaining | `201`, private server-generated key, signed URL TTL 10 min. |
| PC-API-036 | `POST /seller/products/{id}/media/upload-url` | Wrong type/size/13th image/4th video | `400 PRODUCT_MEDIA_INVALID` hoặc `409 PRODUCT_MEDIA_LIMIT_EXCEEDED`. |
| PC-API-037 | `POST /seller/products/{id}/media/complete` | HEAD/checksum/type hợp lệ | `200 READY`, metadata persisted, no media bytes in MongoDB. |
| PC-API-038 | `POST /seller/products/{id}/media/complete` | Object key khác product/checksum sai | `400 PRODUCT_MEDIA_INVALID`, media không READY. |

### 3.3 Admin catalog API

| ID | Endpoint | Kiểm thử | Kỳ vọng |
|---|---|---|---|
| PC-API-039 | `GET /admin/catalog/products` | Admin filter status/shop/category | `200`, xem được mọi lifecycle theo permission. |
| PC-API-040 | `GET /admin/catalog/products/{id}` | Admin detail | Có audit/shop-KYC/inventory display; không secret. |
| PC-API-041 | `POST /admin/catalog/products/{id}/block` | Admin + reason + step-up | `200 BLOCKED`, audit + event, public ẩn. |
| PC-API-042 | `POST /admin/catalog/products/{id}/block` | Seller/no reason/no 2FA | `403 PRODUCT_FORBIDDEN` hoặc validation; không block. |
| PC-API-043 | `POST /admin/catalog/products/{id}/unblock` | Blocked product | `200 INACTIVE`, audit; không tự active baseline. |
| PC-API-044 | `POST /admin/catalog/products/{id}/unblock` | Non-blocked/wrong version | `409 PRODUCT_STATE_INVALID`/`PRODUCT_VERSION_CONFLICT`. |
| PC-API-045 | `GET /admin/catalog/categories` | Filter all category states | Kết quả đúng permission/pagination. |
| PC-API-046 | `POST /admin/catalog/categories` | Root/child valid | `201`, path/depth/version đúng, event emitted. |
| PC-API-047 | `POST /admin/catalog/categories` | >depth 5/cycle/slug collision | `409 CATEGORY_DEPTH_EXCEEDED`/`CATEGORY_CYCLE_DETECTED`/slug conflict. |
| PC-API-048 | `PATCH /admin/catalog/categories/{id}` | Move subtree within depth | `200`, path/depth descendants update atomic. |
| PC-API-049 | `POST /admin/catalog/categories/{id}/archive` | Category referenced by product | Soft archive only; assignment mới bị chặn; không delete reference. |

### 3.4 Integration, security và reliability

| ID | Khu vực | Kiểm thử | Kỳ vọng |
|---|---|---|---|
| PC-INT-001 | Inventory | Event snapshot version 42 rồi 41 | Version 41 bị bỏ qua; projection vẫn 42. |
| PC-INT-002 | Inventory | Duplicate `event_id` replay | Không double-write/không đổi quantity. |
| PC-INT-003 | Inventory | Product endpoint bị gọi để reserve/deduct | Không tồn tại route; Product không mutate inventory. |
| PC-INT-004 | Auth User | Shop/KYC approved/needs_info/suspended event | Snapshot upsert/dedupe; publish gate đúng. |
| PC-INT-005 | Search | Product publish khi Kafka unavailable | Domain transaction giữ đúng; outbox retry/DLQ; không báo Search đã index. |
| PC-INT-006 | Outbox | Retry 3 lần rồi fail | `attempt_count`, error redacted, dead-letter đúng; replay được. |
| PC-INT-007 | MongoDB | Fail giữa domain/outbox/audit | Transaction rollback toàn bộ hoặc recovery rõ ràng; không orphan event. |
| PC-SEC-001 | IDOR | Seller đổi productId shop khác, object key khác | `403 PRODUCT_FORBIDDEN`; không đọc/sửa/upload chéo tenant. |
| PC-SEC-002 | Input | HTML/script, oversized JSON, prototype-like key | Sanitize/reject, không XSS/NoSQL injection. |
| PC-SEC-003 | Admin | Role thấp gọi block/category mutation | Deny + audit security event. |
| PC-SEC-004 | Price | Decimal/negative/overflow/float rounding | Reject; chỉ integer VND trong range. |

## 4. Unit test plan

| Module | Unit cases bắt buộc |
|---|---|
| `VariantResolver` | Sort key deterministic; trim/lowercase theo type; decimal canonical; escaping; duplicate variant key; không hard-code color/size. |
| `AttributeValidator` | STRING/NUMBER/BOOLEAN/ENUM; enum allowlist; max key/label/value; max 50 definitions, 10 dimensions, 50 attrs/SKU. |
| `PricePolicy` | Integer VND, min/max, sale/base relationship theo policy, null override fallback, overflow. |
| `PublishPolicy` | Required fields, active category/SKU, READY cover, KYC approved, suspended shop, stock zero allowed, no admin pre-approval. |
| `ProductStateMachine` | Mọi transition hợp lệ/không hợp lệ: DRAFT/ACTIVE/INACTIVE/BLOCKED/ARCHIVED; blocked unblock default INACTIVE. |
| `CategoryPolicy` | Depth 1–5, cycle detection, primary/secondary limits, inactive/archived assignment. |
| `MediaPolicy` | Type/size/quota, one READY cover, object key tenant binding, checksum. |
| `InventoryProjectionService` | Quantity non-negative, stock status threshold 5, stale after 60s, source version monotonic, dedupe. |
| `ShopProjectionService` | Allowlist fields, source version, KYC transition projection, suspended status. |
| `OutboxPublisher` | Retry 3/backoff 2s, redaction, DLQ, aggregate key, idempotent publish marker. |
| `AuthorizationPolicy` | Seller owner/staff scope, admin roles, step-up, public visibility. |
| `PaginationMapper` | Default 20, max 100, page bounds, stable sort, no unbounded query. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Test stack là Jest/Supertest theo NestJS baseline. | Có thể đổi harness/container nếu team dùng Vitest hoặc Pact. | Tech lead |
| 2 | MongoDB test chạy replica set để kiểm transaction. | Nếu local standalone, phải dùng integration environment khác; không hạ invariant. | DevOps |
| 3 | Inventory exactness được kiểm ở Inventory service; Product chỉ test không reserve/deduct và projection ordering. | Cần contract test riêng cho atomic reserve/deduct của Inventory. | Inventory owner |
| 4 | Virus scan có thể async qua `SCANNING`; chưa có provider cụ thể. | Cần bổ sung callback/polling contract và test security khi provider chốt. | Security/DevOps |
| 5 | Search indexing eventual qua outbox/Kafka. | Cần thêm consumer contract test ở Search để xác nhận schema/version. | Search owner |
| 6 | UI acceptance dựa trên Penpot flows hiện có; breakpoint/browser matrix chưa chốt. | Cần bổ sung visual regression matrix khi frontend chọn framework/browser support. | Frontend lead |
| 7 | API error `401/403` có thể được Gateway normalize. | Cần contract test Gateway ↔ Product để tránh mất `PRODUCT_FORBIDDEN`/request ID. | API Gateway owner |
