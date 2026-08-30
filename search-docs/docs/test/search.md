# Test Plan — Search Service

> Nguồn: `docs/lld/search.md` · `docs/db/search.md` · `docs/api/search.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | NestJS + Jest/Supertest; Elasticsearch test container; Kafka test broker. |
| Index | `products-v1` + alias `products-read`; mapping dynamic attributes/vi analyzer. |
| Events | Product created/updated/published/unpublished/blocked/archived; duplicate và out-of-order version. |
| Data | Product published/hidden/deleted, category path, brand, price 1/0/max, attributes typed, rating mock. |
| Actors | Anonymous buyer, seller, admin `SEARCH_ADMIN`, admin thiếu step-up. |
| Observability | Capture JSON stdout, OpenTelemetry spans, `traceparent`, `X-Request-ID`, Kafka headers. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Buyer Search | Keyword tiếng Việt, empty keyword browse, price/category/brand filter | Kết quả relevance/facet đúng, pagination bounded. |
| Suggest | Prefix, no result, limit 10 | Không lag UI; không lộ search history/raw prefix trong log. |
| UI state | Loading, empty, ES unavailable, timeout/offline | Hiển thị trạng thái đúng, retry an toàn, có request/trace ID khi báo lỗi. |
| Product state | Product publish rồi block/unpublish/archive | Không còn trong public search sau event xử lý. |
| Dynamic attrs | Facet attribute không phải color/size | Facet key/value đúng, không hard-code. |
| Admin | Status/lag/reindex | Chỉ admin có quyền; job progress và error rõ ràng. |

## 3. API integration

| ID | Endpoint | Case | Expected |
|---|---|---|---|
| S-API-01 | `GET /products/search` | q + filter + facet hợp lệ | `200`, published docs, meta request/index time. |
| S-API-02 | `GET /products/search` | empty q browse category | `200 data=[]/list`, không coi empty là error. |
| S-API-03 | `GET /products/search` | query >200, size>100, sort/field lạ | `400 SEARCH_INVALID_INPUT/SEARCH_QUERY_TOO_LONG`. |
| S-API-04 | `GET /products/search` | Elasticsearch down/timeout | `503 SEARCH_UNAVAILABLE` hoặc `504 SEARCH_TIMEOUT`, error có trace_id. |
| S-API-05 | `GET /search/suggest` | prefix hợp lệ/limit boundary | `200`, max 10 suggestions. |
| S-API-06 | `GET /search/suggest` | prefix >100/DSL injection | `400`, không gọi arbitrary DSL. |
| S-API-07 | `GET /search/health` | alias/consumer healthy/degraded | Đúng `200/503`, không public expose nếu ingress sai. |
| S-API-08 | `GET /admin/search/status` | Admin/anonymous | Admin `200`; non-admin `403 SEARCH_ADMIN_FORBIDDEN`. |
| S-API-09 | `POST /admin/search/reindex` | Admin + step-up | `202`, job requested, target index server-generated. |
| S-API-10 | `POST /admin/search/reindex` | Job conflict/no step-up | `409 SEARCH_REINDEX_CONFLICT`/`403`. |
| S-API-11 | `GET /admin/search/reindex/{jobId}` | running/failed/not found | Đúng state/progress/`404 SEARCH_REINDEX_NOT_FOUND`. |

## 4. Unit test

| Module | Cases |
|---|---|
| `QueryParser` | Allowlist field/sort, price integer, bounds, attribute escaping, q length. |
| `ProductDocumentBuilder` | Product ACTIVE→PUBLISHED; blocked/archived→hidden; typed attrs; no stock deduction. |
| `EventConsumer` | Dedupe event ID, ignore older version, schema invalid, retry/DLQ. |
| `ReindexService` | Checkpoint/resume, validation, atomic alias swap, concurrent job conflict. |
| `SuggestMapper` | Prefix normalize, max 10, no raw history persistence. |
| `Observability` | JSON required fields, trace/request propagation, redaction, bounded metric labels. |

## 5. Contract/security/resilience

- Contract Product→Search: event envelope, schema version, price VND, visibility mapping, event ordering.
- Contract Gateway→Search: request ID, W3C trace, auth/rate-limit, error envelope.
- Security: arbitrary Elasticsearch DSL/script blocked; admin route RBAC; no query/token/PII in logs.
- Resilience: ES timeout, Kafka pause/restart, duplicate/out-of-order/replay, DLQ replay, alias rollback.
- Performance: p95 search/suggest within agreed SLA under bounded query and bulk indexing load; alert consumer lag >60s.

## 6. Tiêu chí pass/phát hành

All API cases, visibility/state transitions, event idempotency, security/redaction và reindex rollback pass; không có hidden product trong public result; OpenTelemetry/log scan không có secret/PII/raw query.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Exact Elasticsearch/analyzer/relevance benchmark chưa chốt. | Cần bổ sung golden query set. | Search owner |
| 2 | Event source dùng CDC (Debezium/Kafka). | Ảnh hưởng replay/recovery test. | Platform owner |
| 3 | Rating aggregate contract là mock. | Disable rating tests/feature nếu chưa có event thật. | Rating owner |
| 4 | Browser matrix/Penpot visual regression chưa chốt. | Bổ sung test matrix khi frontend chốt. | Frontend lead |
