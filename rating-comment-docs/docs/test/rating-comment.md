# Test Plan — Rating & Comment Service

> Nguồn: `docs/lld/rating-comment.md` · `docs/db/rating-comment.md` · `docs/api/rating-comment.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture |
|---|---|
| Runtime | NestJS + Jest/Supertest, MongoDB/Kafka test containers. |
| Orders | Delivered eligible, pending/not delivered, wrong buyer/product, already reviewed. |
| Reviews | Rating 1–5, comment boundary, seller reply, deleted/hidden, aggregate distribution. |
| Media | READY/SCANNING/REJECTED, checksum/type/size/quota cases. |
| Actors | Buyer owner/other buyer, seller own/other shop, anonymous, admin future policy. |
| Telemetry | JSON logs/traces/metrics, request/trace propagation, raw comment/PII scan. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Product Detail (`mfe-catalog`) | Read list/aggregate/rating filter | Chỉ published; aggregate count/distribution đúng. |
| Buyer review (`mfe-buyer`) | Delivered order, dynamic media, rating | Create public ngay sau validation; không pre-approval. |
| Not eligible | Chưa delivered/không mua/duplicate | Message rõ, không tạo review. |
| Edit/delete | Own trong window/expired | Version/edit policy đúng; aggregate cập nhật. |
| Seller reply | Own product/other shop, create/edit/delete | Seller chỉ reply, không sửa buyer review. |
| UI states | Loading/empty/error/offline/media scanning | Retry không duplicate, media status rõ. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| R-API-01 | `GET /products/{id}/reviews` public/filter/pagination | `200`, only published, aggregate correct. |
| R-API-02 | Create delivered eligible review | `201 PUBLISHED`, verified true, aggregate/outbox. |
| R-API-03 | Create not delivered/wrong item/wrong buyer | `409 REVIEW_NOT_ELIGIBLE`, no write. |
| R-API-04 | Duplicate same order/product/buyer | `409 REVIEW_ALREADY_EXISTS`. |
| R-API-05 | Rating/comment boundary/invalid media | `400 REVIEW_INVALID_INPUT/MEDIA_INVALID`. |
| R-API-06 | Update own within window/wrong owner | Success or `403 REVIEW_FORBIDDEN`. |
| R-API-07 | Update after window/wrong version | `409 REVIEW_EDIT_EXPIRED/VERSION_CONFLICT`. |
| R-API-08 | Delete/repeat delete | Soft delete/ idempotent policy; aggregate once. |
| R-API-09 | Media presign/complete valid/wrong key/checksum | READY or invalid; no cross-review object. |
| R-API-10 | Seller reply own/other shop | Success or forbidden; one reply baseline. |
| R-API-11 | Reply edit/delete/version | State/version correct, buyer rating unchanged. |
| R-API-12 | Health live/ready | Correct Mongo/Kafka/S3 readiness. |

### 3.1 Contract/security/reliability

| ID | Case | Expected |
|---|---|---|
| R-INT-01 | Order delivered event duplicate/out-of-order | Eligibility projection idempotent. |
| R-INT-02 | Product archived/shop update | Review history retained; public policy respected. |
| R-SEC-01 | Client sets `is_verified_purchase=true` | Ignored; server eligibility required. |
| R-SEC-02 | XSS/NoSQL injection/comment log scan | Sanitize/reject; no raw comment/PII/token logs. |
| R-RES-01 | Kafka/S3/Mongo failure | No partial review/aggregate; outbox/retry. |

## 4. Unit test

| Module | Cases |
|---|---|
| `EligibilityService` | Delivered/order item/buyer/duplicate conditions. |
| `ReviewValidator` | Rating 1–5, length/sanitize/media quota. |
| `AggregateService` | Average, count, distribution, delete/update consistency. |
| `ReplyService` | Shop ownership, max length, one reply/version. |
| `MediaService` | Signed URL ownership, checksum, SCANNING/READY. |
| `Observability` | JSON fields, W3C/request propagation, redaction, bounded metrics. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Review verified delivered, no pre-moderation. | Nếu đổi policy phải thêm test queue/SLA. | Product owner |
| 2 | Edit window 30 ngày/delete policy baseline. | Ảnh hưởng time-based cases. | Product owner |
| 3 | Delivered event/fallback Order contract chưa chốt. | Ảnh hưởng eligibility integration. | Order owner |
| 4 | Media scan provider chưa chốt. | Ảnh hưởng SCANNING/READY tests. | Security/DevOps |
