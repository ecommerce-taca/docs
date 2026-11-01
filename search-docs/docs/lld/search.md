# LLD — Search Service

> Nguồn: `HighLevelDesign/HLD.md` · `FrontendDesign/Taca.penpot.zip` · Code: `ecommerce-taca/search-service` @ `develop` (`13d36e3`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ✅ Đã triển khai — search, suggest, readiness, admin status/reindex/cancel, Kafka consumer 5 topic, DLQ; nhánh `main` mới chỉ có README, code nằm ở `develop`.
> Tech stack: Node.js + NestJS 11 · `@elastic/elasticsearch` 8.15 · `kafkajs` · `nestjs-pino` · `prom-client` · Elasticsearch · Kafka consumer (domain event do CDC — Debezium MongoDB Outbox Event Router — đọc outbox collection của Product Catalog và publish) · REST qua API Gateway

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Full-text product search, autocomplete, category/shop/brand/price filter, facet, sort và index lifecycle. |
| Nguồn dữ liệu | Elasticsearch là read index; Product Catalog là source of truth cho product content/lifecycle/price. |
| Đồng bộ | Consume domain event trên `product.events.v1`/`sku.events.v1`/`category.events.v1`/`catalog.events.v1`, được publish bởi CDC (Debezium MongoDB Outbox Event Router) đọc **outbox collection** của Product Catalog — không phải app tự poll, và không đọc trực tiếp `products`/`skus`/`categories`; topic/payload giữ nguyên như §6. Hỗ trợ replay/reindex. |
| Không thuộc service | Product CRUD, stock deduction/reservation, order/cart/voucher, review aggregate source, KYC, payment. |
| Public visibility | Chỉ index product đã `ACTIVE`/published; status index dùng `PUBLISHED` là projection từ Product `ACTIVE`. |
| Được gọi bởi | `mfe-buyer`, `mfe-seller`, `mfe-admin` qua API Gateway; internal reindex/health qua ops policy. |

### 1.2 Boundary

```text
Product Catalog outbox collection ──CDC (Debezium MongoDB Outbox Event Router)──► Kafka
  (product.events.v1 / sku.events.v1 / category.events.v1 / catalog.events.v1)
                                                                              │
                                                                              ▼
                                                                    Search Consumer ──► Elasticsearch products index
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
| HLD Search | Elasticsearch product index, đồng bộ qua CDC từ Product | Dùng CDC (Debezium MongoDB Outbox Event Router) đọc outbox collection của Product Catalog, publish domain event lên Kafka; không đọc trực tiếp collection nghiệp vụ (`products`/`skus`/`categories`) của Product. |
| mfe-buyer (Home/Search/Category) | Search keyword, category, price | REST query có full-text, filter, facet, sort, pagination. |
| Product detail/card | `title`, `shop_id`, `category_path`, `brand`, `price`, `rating_avg`, `attributes` | Index document denormalized; URL/detail lấy Product API. (tên field trên là quote HLD; index thực dùng snake_case theo db §2) |
| mfe-seller/mfe-admin | Reindex/index diagnostics | Route admin/internal riêng, không expose index management cho buyer. |
| UI states | loading, empty, error, offline | Stable error envelope, `request_id`, timeout và retry policy. |

## 2. Cấu trúc bên trong

### 2.1 Module NestJS

```text
src/
├── main.ts             # global prefix api/v1 (trừ health/live, metrics), ValidationPipe, SearchExceptionFilter
├── common/             # constants, enums, errors (SearchException, error-codes), context (RequestContextMiddleware), es error mapper
├── config/             # AppConfigService + env.validation (fail-fast)
├── search-query/       # SearchController, QueryParser, SearchRequestBuilder, price-buckets, FacetMapper, ResultMapper
├── autocomplete/       # SuggestController, SuggestService
├── product-index/      # mappings (products/consumer-state/reindex-jobs/search-categories v1), ProductDocumentBuilder, VariantAggregator, EsIndexWriter, CategoryProjection, IndexBootstrapService
├── event-consumer/     # KafkaEventConsumer, EventProcessor (dedupe/retry/park), DlqPublisher, handlers (product/sku/category/catalog/rating), ConsumerStatusService
├── reindex/            # AdminSearchController, ReindexService, ReindexRunner, state-machine, validation-gate, IndexCopySource, KafkaReplaySource, JobRepository
├── security/           # RolesGuard, StepUpGuard, @AdminOnly
├── observability/      # MetricsController (/metrics), MetricsService, MetricsMiddleware
└── health/             # HealthController (/health/live), ReadinessController (/api/v1/search/health)
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `SearchController` | Public search/suggest/facet endpoint | Query bounded; không nhận arbitrary Elasticsearch DSL. |
| `AdminSearchController` | Reindex/cancel/status/lag diagnostics | `SEARCH_ADMIN`/ops only; destructive alias action hoặc cancel phải step-up (`X-MFA-Step-Up: true\|1`). |
| `QueryParser` | Validate keyword/filter/sort/facet | Allowlist fields; reject script/query injection. |
| `ProductDocumentBuilder` | Map Product event → index document | Map Product `ACTIVE` → index `PUBLISHED`; không ghi stock ledger. |
| `KafkaEventConsumer` + `EventProcessor` | Consume 5 topic, route theo topic tới handler | Dedupe `event_id` qua index `search-consumer-state`; retry 3 lần (backoff 2s) rồi DLQ `search.events.dlq.v1`; event của product chưa được index thì được "park" (`pending:<event_id>`) và replay khi product được index (tối đa 100/lần). |
| `EsIndexWriter` | Bulk upsert/delete và refresh policy | Không refresh sau từng document; bulk bounded. |
| `ReindexService` | Backfill index mới, alias swap | Chạy checkpoint, không làm mất index đang phục vụ. |
| `ObservabilityModule` | Chuẩn hóa log/trace/metrics | Không log raw query có PII hoặc Authorization. |

### 2.2 Chuẩn observability dùng chung

- Log stdout dạng JSON, một dòng/event, field bắt buộc: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- Propagate W3C `traceparent`/`tracestate` qua REST và Kafka headers; `X-Request-ID` tối đa 64 ký tự do Gateway tạo/propagate.
- Không log access token, Authorization, raw email/phone/address, query body, secret, Elasticsearch DSL hoặc full Kafka payload; chỉ log IDs đã allowlist và hash khi cần correlation.
- Metrics dùng OpenTelemetry/Prometheus-compatible exporter; không dùng `product_id`, `user_id`, `query` làm metric label.
- Liveness: `GET /health/live` (process-only — Gateway active healthcheck gọi, bắt buộc); Readiness: `GET /search/health` (api §3.3) kiểm Kafka consumer, Elasticsearch cluster/index alias và config bắt buộc.
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

1. Consumer nhận event với `event_id`, `aggregate_id`, `version`, `traceparent`. Nếu value không có `event_id`/`aggregate_id` (format Debezium của Catalog), lấy `event_id` từ header `id` và `aggregate_id` từ Kafka key (`consumer/debezium.ts`); thiếu ở cả hai → DLQ.
2. Kiểm schema/version, dedupe event và đọc current indexed version.
3. `product.created/updated/published` → upsert document; `unpublished/blocked/archived` → đổi visibility hoặc delete logical; `unblocked` → theo `next_status` (`ACTIVE` hiện lại, `INACTIVE` giữ ẩn).
4. Bulk write index, cập nhật indexed version/event ID và metric lag.
5. Lỗi retry tối đa 3 lần, backoff 2 giây; tiếp tục lỗi vào `search.events.dlq.v1`.

### 3.3 Reindex và alias swap

1. **Kiểm soát đồng thời (Concurrency Lock)**: Chỉ duy nhất 1 job được active (`REQUESTED`, `RUNNING`, hoặc `VALIDATING`) tại một thời điểm trên toàn hệ thống (bất kể mapping version). Cơ chế enforced bằng document khoá create-only trong Elasticsearch index `search-reindex-jobs` (`_id = active-lock`, `op_type = create`). Khi job hoàn tất (`SWAPPED`, `FAILED`, hoặc `CANCELLED`), lock được giải phóng.
2. **Tạo index mới**: Index version đích được đặt tên theo format `products-v{mapping_version}-{yyyyMMddHHmmss}` (độ chính xác theo giây) với mapping/settings versioned. Không cho client tự chọn tên index.
3. **Thực thi backfill theo `source`**:
   - `INDEX_COPY` (mặc định): Gọi Elasticsearch native `_reindex` API để sao chép toàn bộ document từ index đang đứng sau alias `products-read` sang index mới. Thường dùng khi thay đổi mapping/settings/analyzer.
   - `KAFKA_REPLAY`: Replay các topic từ earliest offsets với dedicated consumer group `search-reindex-<jobId>`, ghi nhận checkpoint theo aggregate ID/version.
4. **Validation gates**: Trước khi swap alias, thực hiện kiểm tra nghiêm ngặt:
   - So sánh số lượng document giữa index mới và `total_estimate` của index active cũ (không lệch quá ngưỡng cho phép).
   - Kiểm tra version regression (`source_version` không được lùi).
   - Smoke test query (search cơ bản, facet, suggest).
5. **Atomic alias swap**: Chuyển alias `products-read` trỏ sang index mới; index cũ giữ lại theo retention policy để rollback nếu cần.
6. **Huỷ job (Cancel flow)**: Khi nhận yêu cầu cancel từ `POST /admin/search/reindex/{jobId}/cancel`, nếu job đang `REQUESTED`/`RUNNING`/`VALIDATING` thì tiến hành huỷ tác vụ đang chạy, xoá target index đang tạo dở, giữ nguyên alias `products-read`, giải phóng lock và cập nhật trạng thái job sang `CANCELLED`.

### 3.4 Autocomplete

- Dùng `completion`/edge-ngram field từ title, brand, category và shop display fields.
- Prefix tối đa 100 ký tự; kết quả chỉ gợi ý document published và giới hạn 10 item.
- Không ghi raw user search history vào v1; không dùng keyword để log.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---|---|---|
| `PAGE_SIZE_DEFAULT` | 20 | — | `PAGE_SIZE_MAX=100`. |
| `SUGGEST_LIMIT` | 10 | — | Max suggestion/request. |
| `QUERY_MAX_LENGTH` | 200 | — | Unicode characters. |
| `PREFIX_MAX_LENGTH` | 100 | — | Autocomplete. |
| `ES_REQUEST_TIMEOUT` | 2s | — | Search request; trả 504 SEARCH_TIMEOUT khi quá hạn. |
| `ES_BULK_BATCH_SIZE` | 500 | — | Max documents/bulk request. |
| `ES_REFRESH_INTERVAL` | 1s | — | Không refresh từng document. |
| `EVENT_RETRY_COUNT` | 3 | — | Backoff 2s rồi DLQ. |
| `EVENT_MAX_LAG_ALERT` | 60s | — | Alert consumer lag. |
| `INDEX_ALIAS` | `products-read` | — | Client không truyền index name. |
| `MAPPING_VERSION` | 1 | — | Đổi mapping phải reindex. |
| `INTERNAL_TIMEOUT` | 5s | — | Admin/reindex adapter; không dùng trong public query. |
| `TIMESTAMP_FORMAT` | ISO-8601 UTC | — | Log/response/event metadata. |
| `MAX_RESULT_WINDOW` | 10000 | — | `page × size` tối đa. |
| `MAX_QUERY_ATTRIBUTE_KEYS` / `_VALUES_PER_KEY` | 10 / 20 | — | Filter `attribute[key]`; key ≤64, value ≤100 ký tự. |
| `MAX_ATTRIBUTE_FACET_KEYS` / `_VALUES` | 10 / 30 | — | Facet attribute trả về. |
| `MAX_PRICE_VALUE` | 1 000 000 000 000 | — | `min_price`/`max_price`. |
| `PRICE_BUCKETS` | `lt_5m`, `5m_20m`, `gte_20m` | — | Facet giá cố định. |

Các hằng số trên ở `src/common/constants.ts` và `src/search-query/constants.ts`. Biến môi trường (`src/config/env.validation.ts`, thiếu biến bắt buộc → app không khởi động):

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---|---|---|
| Port HTTP | `3000` | `PORT` | |
| Tên service | `search-service` | `SERVICE_NAME` | |
| Môi trường / phiên bản | `development` / `1.0.0` | `APP_ENV` / `APP_VERSION` | Đưa vào log. |
| Elasticsearch | — (bắt buộc) | `ES_NODE` | URL Elasticsearch. |
| Tiền tố index | `""` | `ES_INDEX_PREFIX` | Tiền tố cho mọi index/alias. |
| Kafka broker | — (bắt buộc) | `KAFKA_BROKERS` | |
| Kafka client / group | `search-service` / `search-service-group` | `KAFKA_CLIENT_ID` / `KAFKA_GROUP_ID` | Reindex `KAFKA_REPLAY` dùng group `search-reindex-<jobId>`. |
| Tiền tố topic | `""` | `KAFKA_TOPIC_PREFIX` | Tiền tố cho mọi topic. |
| Bật consumer | `true` | `KAFKA_CONSUMER_ENABLED` | |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `visibility_status` | `PUBLISHED`, `HIDDEN`, `DELETED`. |
| `index_document_state` | `CURRENT`, `STALE`, `FAILED`. |
| `consumer_state` | `RUNNING`, `PAUSED`, `DEGRADED`, `DLQ`. |
| `reindex_state` | `REQUESTED`, `RUNNING`, `VALIDATING`, `SWAPPED`, `FAILED`, `CANCELLED`. |

`PUBLISHED` chỉ là Search projection của Product Catalog `ACTIVE`; Search không tự publish sản phẩm. `STALE`/`FAILED` không làm Search suy diễn stock.

> `index_document_state`/`consumer_state` là state nội bộ consumer (không lưu vào ES document, không phơi qua API); tên field API là `consumer` (xem api §3.4).

## 6. Event phát ra / lắng nghe

### 6.1 Event lắng nghe

Routing theo **topic** (không theo `event_type`), trừ `product.skus_updated`. `ProductHandler` xử lý mọi event trên `product.events.v1` qua `ProductDocumentBuilder` (visibility lấy từ `payload.visibility_status`/`next_status`); `product.media_uploaded` (khi `is_cover=true`) và `product.media_updated` chỉ cập nhật `cover_url`. Search không phát event; chỉ publish DLQ `search.events.dlq.v1`.

| Topic | Event | Xử lý |
|---|---|---|
| `product.events.v1` | `product.created`, `product.updated`, `product.published`, `product.unpublished`, `product.blocked`, `product.unblocked`, `product.archived` | Build/upsert/hide/delete logical document. `product.unblocked` mang `next_status`: nếu `ACTIVE` thì hiển thị lại document, nếu `INACTIVE` thì giữ ẩn — không bỏ qua event này, nếu không sản phẩm được gỡ chặn sẽ không bao giờ quay lại index. |
| `product.events.v1` | `product.media_uploaded` | `ProductDocumentBuilder`: chỉ khi `payload.is_cover=true` và có `payload.url` thì patch `cover_url`; ngược lại bỏ qua. `product.media_deleted` hiện **không** được xử lý (cover đã xoá vẫn còn trong index). |
| `product.events.v1` | `product.skus_updated` | Catalog gửi toàn bộ danh sách SKU của product trên topic product; route sang `SkuHandler.handleSkusUpdated`. |
| `sku.events.v1` | `sku.created`, `sku.updated`, `sku.status_changed` | Cập nhật SKU/variant/price searchable; gộp giá trị variant dimension của các SKU `ACTIVE` vào `attributes` (db §3.1). |
| `category.events.v1` | `category.created`, `category.updated`, `category.status_changed` | (Code ghi chú: Product Catalog hiện chưa publish category event, nên khi projection chưa biết category thì index leaf id.) Upsert projection `search-categories` (db §3.4); đổi path thì reindex affected docs, đổi tên thì không. |
| `catalog.events.v1` | `product.category_changed`, `product.media_updated`, `product.shop_snapshot_updated` | Reindex document affected. `product.media_updated` không được Catalog phát (Catalog dùng `product.media_uploaded`/`product.media_deleted` trên `product.events.v1`). `product.shop_snapshot_updated` của Catalog có `aggregate_id=shop_id` và không có `product_id`: `CatalogHandler` cập nhật mọi document của shop bằng update-by-query theo `shop.shop_id` (`IIndexWriter.updateShopSnapshot`). |
| `rating.events.v1` | `rating.aggregate.updated` | Cập nhật `rating_avg`/`rating_count` của product; contract do `rating-comment` sở hữu (payload: `product_id`, `avg`, `count`, `distribution`). |

### 6.2 Mock contract — Product event

> Contract đã chốt với product-catalog (2026-09-18): `price` = `payload.price.sale` (scalar long VND cho range/sort); `category_path` và `visibility_status` do Product gửi sẵn trong payload (`product-catalog-docs/docs/lld/product-catalog.md` §6.2). Ví dụ dưới là hình dạng payload đã thống nhất.

```json
{
  "event_id": "01912f70-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "event_type": "product.published",
  "occurred_at": "2026-08-30T09:00:00Z",
  "aggregate_type": "PRODUCT",
  "aggregate_id": "product-01912f31",
  "version": 8,
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

(traceparent/request_id nằm ở Kafka header, không nằm trong payload)

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
| `SEARCH_EVENT_INVALID` | 400/internal | Event schema/version không hợp lệ. |
| `SEARCH_EVENT_VERSION_OLD` | 200/internal | Event cũ bị bỏ qua, ghi metric (mã nội bộ, không expose qua public API — chỉ ghi metric). |
| `SEARCH_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Backend Search là NestJS, engine là Elasticsearch theo lựa chọn A. | Ảnh hưởng client SDK, deployment và query DSL adapter. | Tech lead |
| 2 | Product event được đồng bộ qua **CDC (Debezium MongoDB Outbox Event Router)** đọc outbox collection của Product Catalog, publish nguyên trạng lên topic `product.events.v1`/`sku.events.v1`/`category.events.v1`/`catalog.events.v1` — không đọc trực tiếp collection nghiệp vụ (`products`/`skus`/`categories`); topic/payload không đổi so với trước — đã đồng bộ với `product-catalog-docs/docs/lld/product-catalog.md` §1.1/§6.1–6.2 (2026-09-19, DOCS-CONSISTENCY-01). | Ảnh hưởng replay, ordering và source export khi rebuild index; cần connector Debezium theo dõi outbox collection còn sống. | Platform/Search owner |
| 3 | Product `ACTIVE` map thành Search `PUBLISHED`. | Nếu business dùng tên state khác, phải đổi mapping/API filter. | Product owner |
| 4 | `rating.aggregate.updated` do `rating-comment` phát (đã có trong bộ 11 service); Search consume để cập nhật `rating_avg`/`rating_count` phục vụ sort/facet `rating_desc`. Đã chốt tên topic + schema (2026-09-18): `rating.events.v1`, payload `product_id`/`avg`/`count`/`distribution` — khớp product-catalog §6.3. | Nếu topic/schema lệch, `rating_desc` sort trả kết quả cũ/thiếu. | Search + Rating owner |
| 5 | Elasticsearch version, analyzer tiếng Việt, synonym và shard/replica chưa chốt. | Ảnh hưởng relevance, storage và query latency. | Search/DevOps |
| 6 | Search không dùng stock projection realtime làm source availability. | Nếu cần sort theo stock, phải có event/contract và freshness SLA riêng. | Inventory/Product owner |
| 7 | **Cấu trúc facet đã viết** ở `docs/api/search.md` §3.1 (category/brand/price bucket cố định/attribute động) là **thiết kế theo suy luận từ Penpot** (`Search / Filter sidebar`, `Chip / 256GB`, `Brand / Apple`), **chưa được Product xác nhận**. Count dùng semantics "multi-select facet" (post_filter). | Nếu Product muốn cấu trúc khác (VD không cần count, hoặc bucket giá động theo phân phối dữ liệu), phải đổi contract trước khi FE code filter sidebar. | Product owner |
| 8 | **Đã chốt 2026-10-14:** Cấu trúc facet (category con trực tiếp D4, bucket giá cố định D9, multi-select post_filter D10), suggest QUERY từ title PUBLISHED (D11–D13), timeout trả 504 SEARCH_TIMEOUT (D5), cơ chế lock active job duy nhất và luồng INDEX_COPY/cancel reindex (D16, D17, D18, D21). | Đã đồng bộ vào spec API, LLD và DB. | Product owner / Techlead |
