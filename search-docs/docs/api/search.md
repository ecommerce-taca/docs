# API Spec — Search Service

> Nguồn: `docs/lld/search.md` · `docs/db/search.md` · Code: `ecommerce-taca/search-service` @ `develop` (`13d36e3`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ✅ Đã triển khai — search, suggest, readiness, admin status/reindex/cancel, Kafka consumer 5 topic, DLQ; nhánh `main` mới chỉ có README, code nằm ở `develop`.
> Base path: `/api/v1` (trừ `GET /health/live` và `GET /metrics` không có prefix) · Port mặc định `3000` · External access qua API Gateway

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Public auth | Anonymous được search; vẫn qua Gateway CORS/rate-limit. |
| Request ID | `X-Request-ID` tối đa 64 ký tự; Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate` qua REST; Kafka headers với event (do Gateway/service propagate, client không gửi). |
| Actor context | Đọc `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`, `X-MFA-Step-Up` do Gateway inject (client không gửi được — Gateway strip). Admin route yêu cầu `X-User-Roles` chứa `SEARCH_ADMIN` (`RolesGuard`); step-up yêu cầu `X-MFA-Step-Up` = `true` hoặc `1` (`StepUpGuard`); thiếu → `403 SEARCH_ADMIN_FORBIDDEN`. Service echo `X-Request-ID` ở response header. |
| Timestamp | ISO-8601 UTC. |
| Pagination | `page=1`, `size=20`, max 100. |
| Query | Keyword max 200; suggest prefix max 100; field/sort/facet allowlist. |
| Money | Integer VND; range không float. |
| Response | `{data,meta}`; `meta` có `request_id`, với search thêm `page,size,total,total_pages,index_as_of` và `facets` (khi `include_facets=true`). |
| Validation | `ValidationPipe` `whitelist + forbidNonWhitelisted + transform`: query/body có field lạ → `400 SEARCH_INVALID_INPUT`. |
| Error | `{error:{code,message,details,trace_id}}`, không stack trace/DSL/token. |
| Log | JSON field chuẩn: `timestamp,level,service,env,version,event,trace_id,span_id,request_id,route,method,status_code,duration_ms`. |
| Redaction | Không log raw query nếu có PII, Authorization, full event payload, user ID làm metric label. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 1 | `GET /api/v1/products/search` | Public | Full-text/filter/facet/sort product. | ✅ |
| 2 | `GET /api/v1/search/suggest` | Public | Autocomplete. | ✅ |
| 3 | `GET /api/v1/search/health` | Internal/ops | Index/consumer readiness. | ✅ |
| 3a | `GET /health/live` | Internal/ops | Liveness process-only (Gateway active healthcheck gọi). | ✅ |
| 3b | `GET /metrics` | Internal/ops | Prometheus metrics (`prom-client`), không envelope. | ✅ |
| 4 | `GET /api/v1/admin/search/status` | `SEARCH_ADMIN` | Consumer lag/index status. | ✅ |
| 5 | `POST /api/v1/admin/search/reindex` | `SEARCH_ADMIN` + step-up (header `X-MFA-Step-Up`) | Start versioned reindex. | ✅ |
| 6 | `GET /api/v1/admin/search/reindex/{jobId}` | `SEARCH_ADMIN` | Reindex job status. | ✅ |
| 7 | `POST /api/v1/admin/search/reindex/{jobId}/cancel` | `SEARCH_ADMIN` + step-up (header `X-MFA-Step-Up`) | Cancel running reindex job. | ✅ |

Search không có endpoint CRUD product, stock reserve/deduct hoặc arbitrary Elasticsearch query.

## 3. Chi tiết endpoint

### 3.1 `GET /products/search`

Query: `q?`, `category_id?`, `shop_id?`, `brand?` (lặp được, ví dụ `brand=A&brand=B`), `attribute[key]?` (tối đa 10 key, key ≤64 ký tự, tối đa 20 giá trị/key, mỗi giá trị ≤100 ký tự), `min_price?`, `max_price?`, `sort` (`relevance`, `newest`, `price_asc`, `price_desc`, `rating_desc`), `page`, `size`, `include_facets?` (bool, mặc định `true`).

Response `200`:

```json
{
  "data": [{
    "product_id": "product-01912f31",
    "title": "Áo khoác cotton",
    "slug": "ao-khoac-cotton",
    "shop": {"shop_id": "shop-01912f30", "name": "Taca Shop"},
    "price": {"amount": 249000, "currency": "VND"},
    "cover_url": "https://cdn.example/signed",
    "rating_avg": 4.5
  }],
  "meta": {
    "facets": {
      "category": [
        { "category_id": "category-01912f21", "name": "Điện thoại", "count": 128 },
        { "category_id": "category-01912f22", "name": "Laptop", "count": 64 }
      ],
      "brand": [
        { "value": "Apple", "count": 42 },
        { "value": "Samsung", "count": 30 }
      ],
      "price": [
        { "min": 0, "max": 5000000, "count": 80 },
        { "min": 5000000, "max": 20000000, "count": 45 },
        { "min": 20000000, "max": null, "count": 6 }
      ],
      "attributes": {
        "capacity_liter": [
          { "value": "256GB", "count": 30 },
          { "value": "512GB", "count": 18 }
        ],
        "color": [
          { "value": "Titan đen", "count": 22 },
          { "value": "Titan tự nhiên", "count": 15 }
        ]
      }
    },
    "page": 1, "size": 20, "total": 1, "total_pages": 1, "request_id": "01912f90", "index_as_of": "2026-08-30T09:00:00Z"
  }
}
```

Ràng buộc: chỉ `visibility_status=PUBLISHED`; empty result `200 data=[]`; `q` optional để browse category; Search không hứa stock realtime hay giá checkout cuối.

Quy tắc sort và validation:

- Sort mặc định: `relevance` khi có `q`, `newest` khi không có `q`. `newest` sắp xếp theo `created_at desc`. Mọi kiểu sort đều kết thúc bằng tie-break `product_id asc` để đảm bảo phân trang ổn định (deterministic pagination).
- `min_price`/`max_price` phải là số nguyên không âm dạng chữ số (`^\d+$`), tối đa `1_000_000_000_000`; `page × size` không được vượt `max_result_window` 10000.
- Tham số out-of-range bị từ chối với lỗi `400`, tuyệt đối không clamp: `size > 100`, `page < 1` → `400 SEARCH_INVALID_INPUT`; `q > 200` ký tự → `400 SEARCH_QUERY_TOO_LONG`; tham số bắt buộc bị thiếu hoặc rỗng → `400 SEARCH_INVALID_INPUT`.

`facets` phục vụ `Search / Filter sidebar` trên Penpot (đếm số bên cạnh mỗi lựa chọn category/brand/giá/thuộc tính). Quy tắc:

- `count` trong mỗi facet là số kết quả **nếu áp thêm điều kiện đó lên trên các filter khác đang chọn** (kiểu "count after AND, before this facet" — chuẩn multi-select facet của Elasticsearch, dùng `post_filter` cho facet đang tính để nó không tự loại chính nó). Không tính `count` chỉ trên toàn tập chưa lọc — như vậy chọn thêm filter sẽ không bao giờ làm số ở facet khác giảm về 0 một cách khó hiểu.
- `category` facet chỉ hiển thị danh mục con trực tiếp (direct children): khi không có filter category → trả các danh mục gốc (root categories); khi có filter `category_id=X` → trả các danh mục con trực tiếp của X. Tên danh mục (`name`) lấy từ projection `search-categories` (db §3.4). Chỉ trả các category có `count > 0`.
- `attributes` chỉ trả các key có `is_variant_dimension=true` từ Product Catalog (`capacity_liter`, `color`...), **động theo category đang lọc** — category khác nhau có bộ attribute facet khác nhau, không có danh sách cố định.
- `price` chia bucket cố định 3 khoảng theo baseline trên (`[0, 5000000)`, `[5000000, 20000000)`, `[20000000, ∞)`); bucket cuối `max: null` nghĩa là không giới hạn trên.
- `include_facets=false` bỏ qua tính `facets` và khoá `meta.facets` bị lược bỏ hoàn toàn (omitted) trong response (giảm tải cho autocomplete-as-you-type, chỉ cần `data`), mặc định `true` cho trang search chính.
- Facet **không** tính lại khi chỉ đổi `page`/`sort` — client cache facet từ lần gọi đầu, chỉ tính lại facet khi filter/query đổi. Đây là tối ưu ở tầng client, Search server luôn trả facet khi `include_facets=true` bất kể `page`.

### 3.2 `GET /search/suggest`

Query: `prefix` bắt buộc max 100, `limit` default 10/max 10, optional `category_id`. Response `200`:

```json
{
  "data": [
    { "text": "iphone 16 pro max", "type": "QUERY" },
    { "text": "iPhone 16 Pro Max 256GB", "type": "PRODUCT", "product_id": "product-01912f31" },
    { "text": "iPhone", "type": "BRAND" }
  ],
  "meta": { "request_id": "01912f92-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`type` ∈ `QUERY` (gợi ý từ khoá phổ biến) `| PRODUCT` (gợi ý thẳng vào 1 sản phẩm, kèm `product_id` để deep-link) `| BRAND`. Không lưu search history v1; prefix không được log raw.

Quy tắc suggest:

- Validation out-of-range bị từ chối, không clamp: `limit > 10` → `400 SEARCH_INVALID_INPUT`; `prefix > 100` ký tự → `400 SEARCH_QUERY_TOO_LONG`; thiếu hoặc rỗng `prefix` → `400 SEARCH_INVALID_INPUT`.
- Nguồn gợi ý `QUERY`: sinh từ `title` của các sản phẩm `PUBLISHED` khớp với prefix (cụm từ chuẩn hóa = `title` được cắt ở ranh giới từ ngay sau prefix, tối đa 6 từ), xếp hạng theo số lượng sản phẩm khớp. Không lưu hay sử dụng search history.
- Thứ tự và dedupe: Trong `data`, thứ tự ưu tiên là `QUERY` trước, đến `PRODUCT`, rồi đến `BRAND`. Loại bỏ trùng lặp không phân biệt hoa thường và dấu tiếng Việt (case/diacritic-insensitively) giữa các type, giữ lại bản ghi xuất hiện đầu tiên; tổng số phần tử ≤ `limit`.
- Giới hạn theo category: Khi có `category_id`, phạm vi gợi ý của cả 3 type (`QUERY`, `PRODUCT`, `BRAND`) bị giới hạn trong các sản phẩm có `category_ids` chứa category đó.

### 3.3 `GET /api/v1/search/health`

Internal response `200`:

```json
{"data":{"elasticsearch":"UP","alias":"products-read","consumer":"RUNNING","event_lag_seconds":2},"meta":{"request_id":"01912f91"}}
```

Quy tắc health check:
- `consumer` ∈ `RUNNING`/`PAUSED`/`DEGRADED`/`DLQ`; khi chưa có consumer status provider (ví dụ `KAFKA_CONSUMER_ENABLED=false`) trả `PAUSED` và `event_lag_seconds: 0`.
- Elasticsearch được coi là `UP` khi `cluster.health` (timeout 1s) trả `green` hoặc `yellow`.
- Trả `200` khi Elasticsearch `UP` và alias `products-read` tồn tại, ngay cả khi consumer ở trạng thái `DEGRADED` hoặc lag > 60s (tình trạng vẫn được báo trong response body qua `consumer` và `event_lag_seconds`).
- Trả `503` chỉ khi Elasticsearch down (`503 SEARCH_UNAVAILABLE`) hoặc alias bị thiếu / chưa sẵn sàng (`503 SEARCH_INDEX_NOT_READY`), dùng chuẩn error envelope `{error:{code,message,details,trace_id}}`.

### 3.3a `GET /health/live`

Response `200` (có envelope):

```json
{"data":{"status":"UP","service":"search-service","time":"2026-10-31T09:00:00.000Z"},"meta":{"request_id":"01912f91"}}
```

Không kiểm Elasticsearch/Kafka.

### 3.4 `GET /admin/search/status`

Query optional `index`, `consumer_group` (chỉ ký tự `[a-zA-Z0-9_.-]`, sai → `400 SEARCH_INVALID_INPUT`). Response `200`:

```json
{
  "data": {
    "alias": "products-read",
    "active_index": "products-v1-20260815",
    "mapping_version": 1,
    "document_count": 48213,
    "last_event_at": "2026-08-30T08:59:58Z",
    "consumer_lag": { "product.events.v1": 0, "sku.events.v1": 2 },
    "dlq_count": 0,
    "reindex_state": "IDLE"
  },
  "meta": { "request_id": "01912fb7-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`reindex_state` phản ánh `state` của job reindex gần nhất, dùng đúng enum `search-reindex-jobs.state` ở `docs/db/search.md` §3.3: `REQUESTED`/`RUNNING`/`VALIDATING`/`SWAPPED`/`FAILED`/`CANCELLED`, cộng thêm giá trị ảo `IDLE` khi chưa từng có job reindex nào (không phải giá trị lưu trong `search-reindex-jobs.state`, chỉ dùng ở response này). Không trả raw event/query.

### 3.5 `POST /admin/search/reindex`

Request:

```json
{"source":"INDEX_COPY","mapping_version":1,"reason":"mapping update"}
```

Validation (`ReindexRequestDto`): `mapping_version` bắt buộc, số nguyên dương và phải bằng `MAPPING_VERSION` hiện tại của service (= `1`); `reason` bắt buộc, được trim, 3–200 ký tự; field khác (ví dụ `target_index`, `state`) → `400 SEARCH_INVALID_INPUT`.

`source` ∈ `INDEX_COPY` | `KAFKA_REPLAY`, mặc định là `INDEX_COPY`:
- `INDEX_COPY`: Chạy Elasticsearch `_reindex` từ index đứng sau alias `products-read` sang index versioned mới (mặc định, dùng cho thay đổi mapping/settings).
- `KAFKA_REPLAY`: Replay các topic từ earliest offsets với consumer group riêng `search-reindex-<jobId>`.

Target index được sinh tự động theo quy tắc `products-v{mapping_version}-{yyyyMMddHHmmss}` (độ chính xác theo giây, ví dụ: `products-v1-20261014120000`).

Response `202` `{data:{job_id,state:"REQUESTED",target_index:"products-v1-20261014120000"},meta}`.

Ràng buộc:
- Yêu cầu quyền `SEARCH_ADMIN` + header `X-MFA-Step-Up: true|1` (Gateway xác thực step-up, Search kiểm tra header; nếu thiếu hoặc không hợp lệ trả `403 SEARCH_ADMIN_FORBIDDEN`).
- Không cho client chọn arbitrary target index.
- Chỉ duy nhất 1 job được active (`REQUESTED`, `RUNNING`, `VALIDATING`) tại một thời điểm bất kể mapping version; nếu đang có job active, trả `409 SEARCH_REINDEX_CONFLICT`.

### 3.6 `GET /admin/search/reindex/{jobId}`

Response `200`:

```json
{
  "data": {
    "job_id": "job-01912fb8",
    "state": "RUNNING",
    "target_index": "products-v1-20261014120000",
    "mapping_version": 1,
    "processed_count": 32000,
    "failed_count": 4,
    "total_estimate": 48213,
    "checkpoint": "product-01912f9a",
    "started_at": "2026-10-14T11:55:00Z",
    "updated_at": "2026-10-14T12:00:00Z",
    "error_code": null
  },
  "meta": { "request_id": "01912fb9-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`state` ∈ `REQUESTED`/`RUNNING`/`VALIDATING`/`SWAPPED`/`FAILED`/`CANCELLED` (khớp `search-reindex-jobs.state` ở `docs/db/search.md` §3.3).

Quy tắc:
- `total_estimate`: Số lượng document của index active tại thời điểm job bắt đầu (nullable nếu chưa có active index).
- `error_code`: Chỉ có giá trị khi `state=FAILED`, null khi không có lỗi. Allowlist cố định gồm: `REINDEX_GATE_COUNT_MISMATCH`, `REINDEX_GATE_VERSION_REGRESSION`, `REINDEX_GATE_SMOKE_FAILED`, `REINDEX_SOURCE_UNAVAILABLE`, `REINDEX_TIMEOUT`, `REINDEX_WORKER_ERROR`. Tuyệt đối không trả raw exception/stack trace.
- `finished_at` chỉ xuất hiện khi job đã kết thúc (`SWAPPED`/`FAILED`/`CANCELLED`).
- Job không tồn tại: `404 SEARCH_REINDEX_NOT_FOUND`.

### 3.7 `POST /admin/search/reindex/{jobId}/cancel`

Yêu cầu quyền `SEARCH_ADMIN` kèm xác thực bước cao (header `X-MFA-Step-Up: true|1`; thiếu hoặc không hợp lệ trả `403 SEARCH_ADMIN_FORBIDDEN`).

Chỉ cho phép huỷ khi job đang ở trạng thái `REQUESTED`, `RUNNING`, hoặc `VALIDATING`. Nếu job ở trạng thái khác (`SWAPPED`, `FAILED`, `CANCELLED`), trả `409 SEARCH_REINDEX_CONFLICT`. Nếu không tìm thấy job, trả `404 SEARCH_REINDEX_NOT_FOUND`.

Khi huỷ thành công, target index đang tạo dở sẽ bị xoá, alias `products-read` được giữ nguyên vẹn không đổi, giải phóng active lock, và job chuyển sang `CANCELLED`.

Response `200`:

```json
{
  "data": {
    "job_id": "job-01912fb8",
    "state": "CANCELLED",
    "target_index": "products-v1-20261014120000",
    "mapping_version": 1,
    "processed_count": 32000,
    "failed_count": 0,
    "total_estimate": 48213,
    "checkpoint": "product-01912f9a",
    "started_at": "2026-10-14T11:55:00Z",
    "updated_at": "2026-10-14T12:00:00Z",
    "error_code": null
  },
  "meta": { "request_id": "01912fba-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

## 4. Mã lỗi chung

Nguồn: `src/common/errors/error-codes.ts` (`ERROR_HTTP_STATUS_MAP`, `ERROR_MESSAGES_MAP`); message trả nguyên văn tiếng Việt.

| Mã | HTTP | Thông điệp |
|---|---:|---|
| `SEARCH_INVALID_INPUT` | 400 | `Tham số tìm kiếm chưa hợp lệ.` |
| `SEARCH_QUERY_TOO_LONG` | 400 | `Từ khóa tìm kiếm quá dài.` |
| `SEARCH_UNAVAILABLE` | 503 | `Hệ thống tìm kiếm tạm thời không khả dụng.` |
| `SEARCH_TIMEOUT` | 504 | `Tìm kiếm mất quá nhiều thời gian.` |
| `SEARCH_INDEX_NOT_READY` | 503 | `Dữ liệu tìm kiếm đang được khởi tạo.` |
| `SEARCH_ADMIN_FORBIDDEN` | 403 | `Bạn không có quyền quản trị tìm kiếm.` |
| `SEARCH_REINDEX_CONFLICT` | 409 | `Đang có tiến trình đồng bộ tìm kiếm.` |
| `SEARCH_REINDEX_NOT_FOUND` | 404 | `Không tìm thấy tiến trình đồng bộ.` |
| `SEARCH_EVENT_INVALID` | 400/internal | `Sự kiện đồng bộ không hợp lệ.` |
| `SEARCH_EVENT_VERSION_OLD` | 200/internal | `Sự kiện cũ bị bỏ qua, ghi metric.` (chỉ nội bộ consumer, không trả qua REST) |
| `SEARCH_INTERNAL_ERROR` | 500 | `Hệ thống đang bận. Vui lòng thử lại sau.` |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Public search path là `/products/search`; Gateway đã chốt map cả `/products/search` (exact) lẫn `/search/**` về search — xem `api-gateway-docs/docs/lld/api-gateway.md` §2.3. (Đã chốt 2026-09-18.) | Ảnh hưởng route registry và `mfe-buyer`. | API Gateway owner |
| 2 | `PUBLISHED` là projection của Product `ACTIVE`. | Nếu Product đổi lifecycle, mapping/filter phải đổi. | Product owner |
| 3 | Search không trả stock — không consume event stock; khả năng mua do Cart/Checkout kiểm tra. | Tránh checkout dùng Search làm source stock. | Order/Inventory owner |
| 4 | `sort=rating_desc` dùng `rating_avg` đồng bộ từ `rating.aggregate.updated` của `rating-comment`; chỉ cần chốt tên topic + schema registry. | Nếu chưa chốt topic, `rating_desc` trả `rating_avg` cũ/thiếu cho tới khi đồng bộ. | Search + Rating owner |
| 5 | Exact analyzer/synonym/relevance chưa chốt. | Cần benchmark query tiếng Việt trước go-live. | Search owner |
| 6 | **Đã chốt 2026-10-14:** Cấu trúc facet category direct children (D4), sort default + tie-break (D6), out-of-range input bị từ chối 400 không clamp (D8), QUERY suggest từ title sản phẩm PUBLISHED (D11–D13), /search/health degrade 200 (D14), cancel reindex (D17), reindex source INDEX_COPY/KAFKA_REPLAY (D16), target index precision giây (D18), và error_code allowlist (D19). | Đã hoàn thiện spec và test contract. | Techlead / Product owner |
