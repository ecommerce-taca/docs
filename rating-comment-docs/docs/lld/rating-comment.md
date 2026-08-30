# LLD — Rating & Comment Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Node.js + NestJS · MongoDB/Mongoose · Kafka events · S3/MinIO media metadata

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Buyer product review/rating, verified purchase check, seller reply, review media metadata và rating aggregate. |
| Review eligibility | Chỉ buyer có order item đã `DELIVERED`; baseline một review cho mỗi `(order_id, product_id, buyer_user_id)`. |
| Public moderation | Không có pre-approval/moderation queue trong v1; vẫn có report/status policy sau này. |
| Source | Review document/aggregate thuộc service; Product/Search chỉ consume aggregate event/projection. |
| Không thuộc service | Order truth, delivery state, product content, KYC, payment, inventory, notification delivery. |
| Media | S3/MinIO giữ bytes; MongoDB giữ object key/checksum/status. |

### 1.2 Boundary

```text
Buyer ──Gateway──► Rating-Comment
Seller ──reply──► Rating-Comment
Rating-Comment ──REST/event──► Order verify delivered
Rating-Comment ──Kafka──► Product/Search/Notification
Rating media ──signed URL──► S3/MinIO
```

- Service phải verify order/product/buyer eligibility qua Order contract/event; không tin `is_verified_purchase` từ client.
- Một review create là idempotent theo buyer/order/product; update/delete policy không xóa audit/aggregate history.
- Rating aggregate update transactionally recompute hoặc atomic consistent write với review mutation.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Rating | reviews, aggregate, seller reply | Review Mongo document + aggregate by product. |
| Buyer Product Detail (`mfe-catalog` / `mfe-buyer`) | rating stars/list, write review after delivery | Public read + protected create/update. |
| Seller dashboard (`mfe-seller`) | View comments/ratings, seller reply | Shop ownership via product/shop reference and Order eligibility. |
| User choice | Verified delivered, no pre-approval | Review `PUBLISHED` sau validation, không queue censor. |

## 2. Cấu trúc bên trong

### 2.1 Module NestJS

```text
src/
├── review/               # review aggregate and state
├── eligibility/          # delivered/order-item verification
├── aggregate/            # rating average/count/distribution
├── reply/                # seller reply policy
├── media/                # signed upload/metadata
├── integration/          # Order/Product/Auth contracts
├── outbox/               # Kafka publisher
├── security/             # ownership, abuse limits, redaction
└── observability/        # shared telemetry contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `ReviewController` | Public list/create/update/delete | Public chỉ published; buyer scope từ JWT. |
| `EligibilityService` | Verify delivered order item | Không trust client flag; cache/event eventual phải có fallback policy. |
| `AggregateService` | Recompute avg/count/distribution | Rating 1–5; aggregate không âm/count mismatch. |
| `SellerReplyService` | Verify seller owns product/shop | Một reply/review baseline; edit history/audit. |
| `MediaService` | Signed URL + checksum metadata | Max files/size; media READY mới public. |
| `OutboxPublisher` | Review/aggregate events | Retry 3/backoff 2s/DLQ; consumer dedupe. |

### 2.2 Chuẩn observability dùng chung

- Structured JSON stdout field: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST/Kafka propagate W3C `traceparent`/`tracestate`, `X-Request-ID`; logs chỉ allowlist review/product/order/shop IDs.
- Không log comment raw, image bytes, signed URL, token, email/phone/address hoặc full order payload; metrics không label bằng user/review/comment text.
- `/health/live` process-only; `/health/ready` MongoDB/Kafka/S3 config; trace/error envelope giống auth-user.
- Audit/security events chứa actor/action/target/reason; redact PII and secrets.

## 3. Luồng xử lý

### 3.1 Create review

1. Buyer gửi `product_id`, `order_id`, rating, comment, media IDs.
2. Verify actor owns order, order status `DELIVERED`, order item contains product; reject duplicate review.
3. Validate rating 1–5, comment length, media READY/ownership.
4. Transaction create review `PUBLISHED`, recompute aggregate, write audit/outbox.
5. Return review and aggregate; Product/Search update eventually via event.

### 3.2 Update/delete

- Buyer chỉ update/delete review own theo time/policy; aggregate recompute trong transaction.
- Delete là soft `DELETED`, không mất audit; public list ẩn review deleted.
- Seller không được sửa rating/comment; chỉ reply.

### 3.3 Seller reply

1. Verify seller/shop owns product referenced by review.
2. Create/update one reply, validate length and redaction policy.
3. Emit `review.reply_updated`; không đổi buyer rating/aggregate.

### 3.4 Aggregate

- `avg` tính từ published non-deleted reviews, precision response 2 decimals.
- Distribution keys 1..5; count tổng bằng distribution sum.
- Duplicate/replayed mutation không làm count tăng hai lần.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `RATING_MIN/MAX` | 1/5 | Integer. |
| `COMMENT_MAX_LENGTH` | 5000 | Unicode characters. |
| `REPLY_MAX_LENGTH` | 2000 | Unicode characters. |
| `REVIEW_MEDIA_MAX_COUNT` | 6 | Per review. |
| `REVIEW_IMAGE_MAX_SIZE` | 10 MiB | JPG/PNG/WebP baseline. |
| `REVIEW_EDIT_WINDOW` | 30 ngày | Sau delivered, cần confirm. |
| `PAGE_SIZE_DEFAULT` | 20 | Max 100. |
| `OUTBOX_RETRY_COUNT` | 3 | Backoff 2s rồi DLQ. |
| `MEDIA_SIGNED_URL_TTL` | 10 phút | Private object. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | MongoDB Date. |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `ReviewStatus` | `PUBLISHED`, `DELETED`, `HIDDEN` (reserved/admin policy). |
| `ReplyStatus` | `PUBLISHED`, `DELETED`. |
| `MediaStatus` | `UPLOADING`, `SCANNING`, `READY`, `REJECTED`, `DELETED`. |
| `EligibilityStatus` | `ELIGIBLE`, `NOT_DELIVERED`, `NOT_ITEM_OWNER`, `ALREADY_REVIEWED`, `UNKNOWN`. |

V1 create eligible → published; delete → deleted; review không chuyển qua approval queue.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `rating.events.v1` | `review.created/updated/deleted` | review/product/shop/order/buyer-safe summary |
| `rating.events.v1` | `rating.aggregate.updated` | product, avg, count, distribution |
| `rating.events.v1` | `review.reply_updated` | review/product/shop/reply status |
| `notification.commands.v1` | `REVIEW_REQUESTED` (optional) | user/order/product template data |

### 6.2 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Order-Commerce | `order.delivered` | Cache/eligibility projection; không tự mark review. |
| Product Catalog | `product.archived`, `product.shop_snapshot_updated` | Cập nhật read metadata; review history vẫn giữ. |
| Auth User | `user.status_changed` | Chặn mutation account suspended theo policy. |

### 6.3 Mock contract — delivered eligibility

```json
{
  "event_id":"order-event-1",
  "event_type":"order.delivered",
  "aggregate_id":"order-1",
  "occurred_at":"2026-08-30T12:00:00Z",
  "payload":{"order_id":"order-1","buyer_user_id":"user-1","items":[{"product_id":"product-1","sku_id":"sku-1"}]}
}
```

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `REVIEW_INVALID_INPUT` | 400 | Rating/comment/media sai. |
| `REVIEW_UNAUTHENTICATED` | 401 | Create/reply cần login. |
| `REVIEW_FORBIDDEN` | 403 | Sai buyer/seller scope. |
| `REVIEW_NOT_FOUND` | 404 | Review không tồn tại/public. |
| `REVIEW_NOT_ELIGIBLE` | 409 | Chưa delivered/không phải item owner. |
| `REVIEW_ALREADY_EXISTS` | 409 | Đã review order-product. |
| `REVIEW_EDIT_EXPIRED` | 409 | Quá edit window. |
| `REVIEW_MEDIA_INVALID` | 400 | Media/checksum/type sai. |
| `REVIEW_MEDIA_LIMIT_EXCEEDED` | 409 | Vượt 6 file/review. |
| `REVIEW_VERSION_CONFLICT` | 409 | Concurrent update. |
| `REVIEW_DEPENDENCY_UNAVAILABLE` | 503 | Order/Mongo/Kafka/S3 unavailable. |
| `REVIEW_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Review chỉ được tạo sau `DELIVERED`, unique theo order+product+buyer. | Ảnh hưởng eligibility/API unique index. | Product/Order owner |
| 2 | Không có pre-moderation; `HIDDEN` chỉ reserved cho admin/report policy tương lai. | Nếu cần censor, phải thêm moderation workflow. | Product owner |
| 3 | Review edit window 30 ngày và delete policy chưa có HLD cụ thể. | Ảnh hưởng UX/audit/aggregate. | Product owner |
| 4 | Order delivered event là mock; fallback synchronous verify contract chưa chốt. | Ảnh hưởng eventual eligibility. | Order owner |
| 5 | Media scan provider chưa chốt; `SCANNING` có thể async. | Ảnh hưởng publish/public media. | Security/DevOps |
| 6 | `rating.aggregate.updated` (topic `rating.events.v1`, payload `product_id/avg/count/distribution`) được **Product Catalog** (cache `ratingAvg`) và **Search** (`rating_avg` cho sort `rating_desc`) consume. Cần chốt tên topic + schema registry với hai service này. | Nếu lệch topic/schema, rating hiển thị/sort ở PDP và Search bị cũ. | Rating + Search + Product owner |
