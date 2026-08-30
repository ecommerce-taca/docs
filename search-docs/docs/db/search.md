# Database — Search Service

> Nguồn: `docs/lld/search.md` · HLD `EcommercePlatform-v4(6).excalidraw` · Cập nhật: `2026-08-30`
> Baseline: Elasticsearch 8.x · index riêng, không dùng database Product làm query store

## 1. Quy ước chung

| Quy ước | Quyết định | Ràng buộc |
|---|---|---|
| Storage | Elasticsearch index + alias | Không tạo relational FK/cross-service join. |
| Document ID | `product_id` string | Không dùng document ID ngẫu nhiên cho product index. |
| Source | Product/CDC event (Debezium/Kafka) | Search document là projection, có `source_version`, `source_event_id`. |
| Time | ISO-8601 UTC trong document | Không lưu local time. |
| Price | `long` integer VND | Không float/decimal trong query range. |
| Visibility | `PUBLISHED`/`HIDDEN`/`DELETED` | Product `ACTIVE` map thành `PUBLISHED`. |
| Alias | `products-read` | Client không truyền index name. |
| Observability | OpenTelemetry-compatible JSON log/trace/metric | Redact query/token/PII; field chuẩn giống auth-user/Gateway. |

## 2. Quan hệ dữ liệu

Search chỉ có quan hệ logic: một product document chứa denormalized shop/category/SKU/attribute/media display data; Product Catalog và Rating là source event. Không có FK hoặc transaction join.

## 3. Chi tiết index/document

### 3.1 `products-v{mapping_version}`

```json
{
  "product_id": "product-01912f31",
  "shop_id": "shop-01912f30",
  "title": "Áo khoác cotton",
  "title_exact": "Áo khoác cotton",
  "slug": "ao-khoac-cotton",
  "description": "Mô tả đã sanitize",
  "category_ids": ["category-1"],
  "category_path": ["fashion", "outerwear"],
  "brand": "Taca Brand",
  "price": 249000,
  "currency": "VND",
  "rating_avg": 4.5,
  "rating_count": 12,
  "sales_count": 48,
  "attributes": {"material": ["cotton"]},
  "suggest": ["Áo khoác cotton", "Taca Brand"],
  "visibility_status": "PUBLISHED",
  "source_version": 8,
  "source_event_id": "event-1",
  "indexed_at": "2026-08-30T09:00:00Z"
}
```

| Field | Mapping | Ràng buộc |
|---|---|---|
| `product_id`, `shop_id`, `slug` | `keyword` | Filter/identity. |
| `title` | `text` analyzer tiếng Việt baseline | Full-text; exact dùng `title_exact`. |
| `description` | `text` | Search relevance, không highlight raw HTML. |
| `category_ids`, `category_path`, `brand` | `keyword[]`/`keyword` | Facet/filter. |
| `price` | `long` | Range/sort VND. |
| `rating_avg` | `double`; `rating_count` `integer`; `sales_count` `long` | Sort/analytics; event rating/sales contract cần chốt. |
| `attributes` | `flattened` | Dynamic facet, không hard-code color/size. |
| `suggest` | `completion` | Prefix autocomplete. |
| `visibility_status` | `keyword` | Public query bắt buộc `PUBLISHED`. |
| `source_version` | `long` | Bỏ event cũ/out-of-order. |
| `source_event_id` | `keyword` | Dedupe/audit. |

### 3.2 `search_consumer_state`

Persist bằng index hoặc operational store tùy deployment; tối thiểu phải giữ `event_id`, `aggregate_id`, `source_version`, `processed_at`, `status`, `error_code`. Nếu dùng Elasticsearch, index này không phục vụ public query.

### 3.3 `reindex_jobs`

| Field | Type | Ràng buộc |
|---|---|---|
| `job_id` | keyword | Unique. |
| `target_index` | keyword | Server-generated. |
| `mapping_version` | integer | Immutable. |
| `state` | keyword | `REQUESTED/RUNNING/VALIDATING/SWAPPED/FAILED/CANCELLED`. |
| `checkpoint` | keyword | Resume token, không là public ID. |
| `processed_count`/`failed_count` | long | Non-negative. |
| `started_at`/`finished_at` | date | UTC. |
| `trace_id`/`request_id` | keyword | Correlation allowlist. |

## 4. Index và uniqueness

| Index/alias | Mapping/setting | Mục đích |
|---|---|---|
| `products-read` alias | Read alias tới một versioned index | Public/admin query. |
| `{product_id}` | `_id` chính | Upsert/delete deterministic. |
| `title` | Vietnamese analyzer + lowercase/ascii folding policy | Full-text. |
| `category_path` | keyword array | Category filter/facet. |
| `attributes` | flattened | Dynamic facets. |
| `suggest` | completion | Autocomplete. |
| `visibility_status` | filter-first keyword | Không query hidden public. |

Không dùng ES dynamic mapping không kiểm soát cho user-provided field names; attribute key phải sanitize/allowlist trước khi build document.

## 5. Enum và rules

| Rule | Giá trị |
|---|---|
| Mapping version | Bắt đầu 1; đổi analyzer/field type phải reindex. |
| Visibility | `PUBLISHED`, `HIDDEN`, `DELETED`. |
| Event version | Không apply event có `source_version` ≤ current indexed version. |
| Public filter | `visibility_status=PUBLISHED`. |
| Query limit | page 20/max 100; suggestion max 10. |

## 6. Migration và seed

1. Tạo index version mới với explicit mapping/settings.
2. Validate analyzer, dynamic attributes, completion và numeric range.
3. Backfill/replay event theo checkpoint.
4. Chạy count/sample/version comparison và query smoke test.
5. Atomic alias swap; giữ index cũ theo retention.
6. Seed test product bằng event fixture, không viết trực tiếp public index bằng tay trong integration test.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Elasticsearch 8.x là engine đã chốt; exact minor/version chưa chốt. | Ảnh hưởng mapping/API client/compatibility. | Search/DevOps |
| 2 | Vietnamese analyzer/synonym chưa được chọn. | Ảnh hưởng relevance/autocomplete. | Search owner |
| 3 | Consumer state/reindex job có thể dùng operational index, chưa chốt retention store. | Ảnh hưởng recovery/debug. | Platform owner |
| 4 | Rating aggregate event là mock. | Không bật rating sort/facet production trước khi contract có. | Rating owner |
| 5 | Product export/replay API chưa có; hiện dùng Kafka replay/mock fixture. | Ảnh hưởng full rebuild khi Kafka retention hết. | Product/Platform owner |
