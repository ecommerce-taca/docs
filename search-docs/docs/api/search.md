# API Spec — Search Service

> Nguồn: `docs/lld/search.md` · `docs/db/search.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · External access qua API Gateway · Error envelope và observability đồng nhất `auth-user`

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Public auth | Anonymous được search; vẫn qua Gateway CORS/rate-limit. |
| Request ID | `X-Request-ID` tối đa 64 ký tự; Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate` qua REST; Kafka headers với event. |
| Timestamp | ISO-8601 UTC. |
| Pagination | `page=1`, `size=20`, max 100. |
| Query | Keyword max 200; suggest prefix max 100; field/sort/facet allowlist. |
| Money | Integer VND; range không float. |
| Response | `{data,meta}`; `meta` có `request_id`, `index_as_of`, optional lag. |
| Error | `{error:{code,message,details,trace_id}}`, không stack trace/DSL/token. |
| Log | JSON field chuẩn: `timestamp,level,service,env,version,event,trace_id,span_id,request_id,route,method,status_code,duration_ms`. |
| Redaction | Không log raw query nếu có PII, Authorization, full event payload, user ID làm metric label. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /products/search` | Public | Full-text/filter/facet/sort product. |
| 2 | `GET /search/suggest` | Public | Autocomplete. |
| 3 | `GET /search/health` | Internal/ops | Index/consumer readiness. |
| 4 | `GET /admin/search/status` | `SEARCH_ADMIN` | Consumer lag/index status. |
| 5 | `POST /admin/search/reindex` | `SEARCH_ADMIN` + step-up | Start versioned reindex. |
| 6 | `GET /admin/search/reindex/{jobId}` | `SEARCH_ADMIN` | Reindex job status. |

Search không có endpoint CRUD product, stock reserve/deduct hoặc arbitrary Elasticsearch query.

## 3. Chi tiết endpoint

### 3.1 `GET /products/search`

Query: `q?`, `category_id?`, `shop_id?`, `brand?`, `attribute[key]?`, `min_price?`, `max_price?`, `sort` (`relevance`, `newest`, `price_asc`, `price_desc`, `rating_desc`), `page`, `size`.

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
    "rating_avg": 4.5,
    "stock_display": "IN_STOCK"
  }],
  "meta": {"page": 1, "size": 20, "total": 1, "total_pages": 1, "request_id": "01912f90", "index_as_of": "2026-08-30T09:00:00Z"}
}
```

Ràng buộc: chỉ `visibility_status=PUBLISHED`; empty result `200 data=[]`; `q` optional để browse category; Search không hứa stock realtime hay giá checkout cuối.

### 3.2 `GET /search/suggest`

Query: `prefix` bắt buộc max 100, `limit` default 10/max 10, optional `category_id`. Response `200` `{data:[{text,type}],meta}`. Không lưu search history v1; prefix không được log raw.

### 3.3 `GET /search/health`

Internal response `200`:

```json
{"data":{"elasticsearch":"UP","alias":"products-read","consumer":"RUNNING","event_lag_seconds":2},"meta":{"request_id":"01912f91"}}
```

Nếu index alias/cluster không sẵn sàng: `503 SEARCH_INDEX_NOT_READY`/`SEARCH_UNAVAILABLE`.

### 3.4 `GET /admin/search/status`

Query optional `index`, `consumer_group`. Trả alias, mapping version, document count, last event time, consumer lag, DLQ count và reindex state. Không trả raw event/query.

### 3.5 `POST /admin/search/reindex`

Request:

```json
{"source":"KAFKA_REPLAY","mapping_version":1,"reason":"mapping update"}
```

Response `202` `{data:{job_id,state:"REQUESTED",target_index:"products-v1-20260830"},meta}`. Chỉ `SEARCH_ADMIN` + step-up; không cho client chọn arbitrary target index; nếu job cùng mapping đang chạy trả `409 SEARCH_REINDEX_CONFLICT`.

### 3.6 `GET /admin/search/reindex/{jobId}`

Trả job state, processed/failed counts, checkpoint, timestamps và `error_code` đã redacted. Job không tồn tại: `404 SEARCH_REINDEX_NOT_FOUND`.

## 4. Mã lỗi chung

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
| `SEARCH_INTERNAL_ERROR` | 500 | `Hệ thống đang bận. Vui lòng thử lại sau.` |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Public search path là `/products/search`, Gateway có thể map từ `/search`. | Ảnh hưởng route registry và `mfe-buyer`. | API Gateway owner |
| 2 | `PUBLISHED` là projection của Product `ACTIVE`. | Nếu Product đổi lifecycle, mapping/filter phải đổi. | Product owner |
| 3 | `stock_display` chỉ là display field, không phải Inventory guarantee. | Tránh checkout dùng Search làm source stock. | Order/Inventory owner |
| 4 | `sort=rating_desc` dùng `rating_avg` đồng bộ từ `rating.aggregate.updated` của `rating-comment`; chỉ cần chốt tên topic + schema registry. | Nếu chưa chốt topic, `rating_desc` trả `rating_avg` cũ/thiếu cho tới khi đồng bộ. | Search + Rating owner |
| 5 | Exact analyzer/synonym/relevance chưa chốt. | Cần benchmark query tiếng Việt trước go-live. | Search owner |
