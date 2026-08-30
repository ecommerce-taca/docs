# LLD — Search Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Node.js + NestJS · Elasticsearch · Kafka consumer (CDC via Debezium) từ Product Catalog · REST qua API Gateway

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Full-text product search, autocomplete, category/shop/brand/price filter, facet, sort và index lifecycle. |
| Nguồn dữ liệu | Elasticsearch là read index; Product Catalog là source of truth cho product content/lifecycle/price. |
| Đồng bộ | Consume CDC events (Debezium/Kafka) từ Product Catalog; hỗ trợ replay/reindex. |
| Không thuộc service | Product CRUD, stock deduction/reservation, order/cart/voucher, review aggregate source, KYC, payment. |
| Public visibility | Chỉ index product đã `ACTIVE`/published; status index dùng `PUBLISHED` là projection từ Product `ACTIVE`. |
| Được gọi bởi | `mfe-buyer`, `mfe-seller`, `mfe-admin` qua API Gateway; internal reindex/health qua ops policy. |

### 1.2 Boundary

```text
Product Catalog ──CDC (Debezium/Kafka)──► Search Consumer ──► Elasticsearch products index
mfe-buyer/mfe-seller/mfe-admin ──API Gateway──► Search REST API
Inventory ──(không gọi trực tiếp Search)──► Product stock projection ──event──► Search
```

- Search không query MongoDB Product trong request path và không tự sửa product state.
- Event có thể đến trễ/duplicate/out-of-order; consumer dedupe theo `event_id` và kiểm aggregate version.
- Search result không phải purchase availability; Cart/Checkout phải revalidate Product/Inventory.
- Nếu index lag sau publish, Product detail vẫn là source of truth; Search trả metadata index lag cho observability, không bịa product.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Search | Elasticsearch product index, CDC/event từ Product | Dùng CDC qua Debezium/Kafka; không đọc database Product. |
| mfe-buyer (Home/Search/Category) | Search keyword, category, price | REST query có full-text, filter, facet, sort, pagination. |
| Product detail/card | `title`, `shopId`, `categoryPath`, `brand`, `price`, `ratingAvg`, `attributes` | Index document denormalized; URL/detail lấy Product API. |
| mfe-seller/mfe-admin | Reindex/index diagnostics | Route admin/internal riêng, không expose index management cho buyer. |
| UI states | loading, empty, error, offline | Stable error envelope, `request_id`, timeout và retry policy. |

## 2. Cấu trúc bên trong

### 2.1 Module NestJS

```text
src/
├── search-query/       # query parser, filter/facet/sort, result mapper
├── autocomplete/       # completion suggester
├── product-index/      # index mapping, document builder, bulk write
├── event-consumer/     # Kafka consumer, dedupe, version ordering
├── reindex/            # alias swap, backfill, resume checkpoint
├── security/           # route role, query validation, redaction
├── observability/      # shared JSON log/trace/metric contract
└── health/             # liveness/readiness/index lag
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `SearchController` | Public search/suggest/facet endpoint | Query bounded; không nhận arbitrary Elasticsearch DSL. |
| `AdminSearchController` | Reindex/status/lag diagnostics | `SEARCH_ADMIN`/ops only; destructive alias action phải step-up. |
| `QueryParser` | Validate keyword/filter/sort/facet | Allowlist fields; reject script/query injection. |
| `ProductDocumentBuilder` | Map Product event → index document | Map Product `ACTIVE` → index `PUBLISHED`; không ghi stock ledger. |
| `KafkaProductConsumer` | Consume event theo aggregate key | Dedupe `event_id`, bỏ event có version cũ; retry/DLQ. |
| `IndexWriter` | Bulk upsert/delete và refresh policy | Không refresh sau từng document; bulk bounded. |
| `ReindexService` | Backfill index mới, alias swap | Chạy checkpoint, không làm mất index đang phục vụ. |
| `ObservabilityModule` | Chuẩn hóa log/trace/metrics | Không log raw query có PII hoặc Authorization. |

### 2.2 Chuẩn observability dùng chung

- Log stdout dạng JSON, một dòng/event, field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- Propagate W3C `traceparent`/`tracestate` qua REST và Kafka headers; `X-Request-ID` tối đa 64 ký tự do Gateway tạo/propagate.
- Không log access token, Authorization, raw email/phone/address, query body, secret, Elasticsearch DSL hoặc full Kafka payload; chỉ log IDs đã allowlist và hash khi cần correlation.
- Metrics dùng OpenTelemetry/Prometheus-compatible exporter; không dùng `product_id`, `user_id`, `query` làm metric label.
- Liveness/readiness: `/health/live`, `/health/ready`; readiness kiểm Kafka consumer, Elasticsearch cluster/index alias và config bắt buộc.
- Error response dùng `{error:{code,message,details,trace_id}}`; mọi error log phải có `trace_id` + `request_id`.

## 3. Luồng xử lý

### 3.1 Search request

```text
Gateway → validate request context → SearchController
  → QueryParser (allowlist + bounds)
  → Elasticsearch bool query + filter/facet/sort
  → map SearchDocument → result card
  → return result + request_id + index_as_of
```

Ràng buộc: `size` mặc định 20, tối đa 100; keyword tối đa 200 ký tự; price integer VND; result chỉ `visibility_status=PUBLISHED`; empty result là `200`, không phải 404.

### 3.2 Product event indexing

1. Consumer nhận event với `event_id`, `aggregate_id`, `version`, `traceparent`.
2. Kiểm schema/version, dedupe event và đọc current indexed version.
3. `product.created/updated/published` → upsert document; `unpublished/blocked/archived` → đổi visibility hoặc delete logical.
4. Bulk write index, cập nhật indexed version/event ID và metric lag.
5. Lỗi retry tối đa 3 lần, backoff 2 giây; tiếp tục lỗi vào `search.events.dlq.v1`.

### 3.3 Reindex và alias swap

1. Tạo index version mới `products-v{timestamp}` với mapping/settings đã versioned.
2. Backfill từ event replay hoặc approved Product export; checkpoint theo aggregate ID/version.
3. Chạy count/checksum/sample comparison, kiểm document `PUBLISHED` visibility.
4. Atomic alias swap `products-read` sang index mới; giữ index cũ theo retention.
5. Không expose endpoint cho client tự chọn index name.

### 3.4 Autocomplete

- Dùng `completion`/edge-ngram field từ title, brand, category và shop display fields.
- Prefix tối đa 100 ký tự; kết quả chỉ gợi ý document published và giới hạn 10 item.
- Không ghi raw user search history vào v1; không dùng keyword để log.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `PAGE_SIZE_DEFAULT` | 20 | `PAGE_SIZE_MAX=100`. |
| `SUGGEST_LIMIT` | 10 | Max suggestion/request. |
| `QUERY_MAX_LENGTH` | 200 | Unicode characters. |
| `PREFIX_MAX_LENGTH` | 100 | Autocomplete. |
| `ES_REQUEST_TIMEOUT` | 2s | Search request; trả 503 khi quá hạn. |
| `ES_BULK_BATCH_SIZE` | 500 | Max documents/bulk request. |
| `ES_REFRESH_INTERVAL` | 1s | Không refresh từng document. |
| `EVENT_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `EVENT_MAX_LAG_ALERT` | 60s | Alert consumer lag. |
| `INDEX_ALIAS` | `products-read` | Client không truyền index name. |
| `MAPPING_VERSION` | 1 | Đổi mapping phải reindex. |
| `INTERNAL_TIMEOUT` | 5s | Admin/reindex adapter; không dùng trong public query. |
| `TIMESTAMP_FORMAT` | ISO-8601 UTC | Log/response/event metadata. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `visibility_status` | `PUBLISHED`, `HIDDEN`, `DELETED`. |
| `index_document_state` | `CURRENT`, `STALE`, `FAILED`. |
| `consumer_state` | `RUNNING`, `PAUSED`, `DEGRADED`, `DLQ`. |
| `reindex_state` | `REQUESTED`, `RUNNING`, `VALIDATING`, `SWAPPED`, `FAILED`, `CANCELLED`. |

`PUBLISHED` chỉ là Search projection của Product Catalog `ACTIVE`; Search không tự publish sản phẩm. `STALE`/`FAILED` không làm Search suy diễn stock.

## 6. Event phát ra / lắng nghe

### 6.1 Event lắng nghe

| Topic | Event | Xử lý |
|---|---|---|
| `product.events.v1` | `product.created`, `product.updated`, `product.published`, `product.unpublished`, `product.blocked`, `product.archived` | Build/upsert/hide/delete logical document. |
| `sku.events.v1` | `sku.created`, `sku.updated`, `sku.status_changed` | Cập nhật SKU/variant/price searchable. |
| `category.events.v1` | `category.created`, `category.updated`, `category.status_changed` | Cập nhật category path/facet hoặc reindex affected docs. |
| `catalog.events.v1` | `product.category_changed`, `product.media_updated`, `product.shop_snapshot_updated` | Reindex document affected. |
| `rating.events.v1` | `rating.aggregate.updated` | Cập nhật `rating_avg`/`rating_count` của product; contract do `rating-comment` sở hữu (payload: `product_id`, `avg`, `count`, `distribution`). |

### 6.2 Mock contract — Product event

```json
{
  "event_id": "01912f70-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "event_type": "product.published",
  "occurred_at": "2026-08-30T09:00:00Z",
  "aggregate_type": "PRODUCT",
  "aggregate_id": "product-01912f31",
  "version": 8,
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "payload": {
    "product_id": "product-01912f31",
    "shop_id": "shop-01912f30",
    "title": "Áo khoác cotton",
    "slug": "ao-khoac-cotton",
    "category_path": ["fashion", "outerwear"],
    "brand": "Taca Brand",
    "price": 249000,
    "currency": "VND",
    "attributes": {"material": "cotton"},
    "media": [{"url": "https://cdn.example/signed", "alt": "Áo khoác"}],
    "visibility_status": "PUBLISHED"
  }
}
```

### 6.3 Reliability

- Partition key theo `aggregate_id`; consumer commit offset sau bulk write thành công.
- Duplicate/out-of-order event không được làm index lùi version.
- DLQ giữ event đã redacted; replay phải ghi `replay_of_event_id` và giữ trace correlation.
- Elasticsearch unavailable: public query trả `503 SEARCH_UNAVAILABLE`; không fallback sang MongoDB Product trong request path.

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `SEARCH_INVALID_INPUT` | 400 | Query/filter/sort ngoài allowlist. |
| `SEARCH_QUERY_TOO_LONG` | 400 | Keyword/prefix vượt giới hạn. |
| `SEARCH_UNAVAILABLE` | 503 | Elasticsearch/index alias unavailable. |
| `SEARCH_TIMEOUT` | 504 | Elasticsearch vượt timeout. |
| `SEARCH_INDEX_NOT_READY` | 503 | Alias/readiness chưa sẵn sàng. |
| `SEARCH_ADMIN_FORBIDDEN` | 403 | Thiếu quyền reindex/diagnostics. |
| `SEARCH_REINDEX_CONFLICT` | 409 | Reindex đang chạy hoặc alias conflict. |
| `SEARCH_EVENT_INVALID` | 400 | Event schema/version không hợp lệ. |
| `SEARCH_EVENT_VERSION_OLD` | 200/internal | Event cũ bị bỏ qua, ghi metric. |
| `SEARCH_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Backend Search là NestJS, engine là Elasticsearch theo lựa chọn A. | Ảnh hưởng client SDK, deployment và query DSL adapter. | Tech lead |
| 2 | Product event được đồng bộ qua CDC (Debezium/Kafka). | Ảnh hưởng replay, ordering và source export khi rebuild index. | Platform/Search owner |
| 3 | Product `ACTIVE` map thành Search `PUBLISHED`. | Nếu business dùng tên state khác, phải đổi mapping/API filter. | Product owner |
| 4 | `rating.aggregate.updated` do `rating-comment` phát (đã có trong bộ 11 service); Search consume để cập nhật `rating_avg`/`rating_count` phục vụ sort/facet `rating_desc`. Chỉ cần chốt tên topic + schema registry giữa hai service. | Nếu topic/schema lệch, `rating_desc` sort trả kết quả cũ/thiếu. | Search + Rating owner |
| 5 | Elasticsearch version, analyzer tiếng Việt, synonym và shard/replica chưa chốt. | Ảnh hưởng relevance, storage và query latency. | Search/DevOps |
| 6 | Search không dùng stock projection realtime làm source availability. | Nếu cần sort theo stock, phải có event/contract và freshness SLA riêng. | Inventory/Product owner |
