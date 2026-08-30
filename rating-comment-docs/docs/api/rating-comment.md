# API Spec — Rating & Comment Service

> Nguồn: `docs/lld/rating-comment.md` · `docs/db/rating-comment.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · NestJS/MongoDB · Verified delivered, không pre-moderation

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Public read anonymous; create/update buyer JWT; seller reply seller scope. |
| Eligibility | Server verify delivered order item; client không set `is_verified_purchase`. |
| Request ID/trace | `X-Request-ID`, W3C `traceparent`/`tracestate`; error `trace_id`. |
| Time | ISO-8601 UTC; page 1/size 20/max100. |
| Response/error | `{data,meta}` / `{error:{code,message,details,trace_id}}`. |
| Log | JSON field chuẩn; không log comment raw, media signed URL, token, PII. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /products/{productId}/reviews` | Public | Review list + aggregate. |
| 2 | `POST /products/{productId}/reviews` | Buyer | Tạo verified review. |
| 3 | `PATCH /reviews/{reviewId}` | Buyer owner | Sửa review trong policy window. |
| 4 | `DELETE /reviews/{reviewId}` | Buyer owner | Soft delete. |
| 5 | `POST /reviews/{reviewId}/media/upload-url` | Buyer owner | Signed upload. |
| 6 | `POST /reviews/{reviewId}/media/complete` | Buyer owner | Complete metadata. |
| 7 | `POST /seller/reviews/{reviewId}/reply` | Seller | Seller reply. |
| 8 | `PATCH /seller/reviews/{reviewId}/reply` | Seller | Sửa reply. |
| 9 | `DELETE /seller/reviews/{reviewId}/reply` | Seller | Soft delete reply. |
| 10 | `GET /health/live` | Ops | Liveness. |
| 11 | `GET /health/ready` | Ops | Readiness. |

## 3. Chi tiết endpoint

### 3.1 `GET /products/{productId}/reviews`

Query `rating?` (1–5), `with_media?` (bool), `page`, `size` (max 100), `sort` (`newest` | `rating_desc`). Review `DELETED`/`HIDDEN` không xuất hiện public.

```json
{
  "data": {
    "aggregate": {
      "avg": 4.6,
      "count": 128,
      "distribution": { "1": 2, "2": 3, "3": 8, "4": 30, "5": 85 }
    },
    "reviews": [
      {
        "review_id": "rv-01912fd0",
        "product_id": "product-01912f31",
        "sku_id": "sku-01912f33",
        "variant_label": "Đen / 65W",
        "rating": 5,
        "comment": "Sạc nhanh, đóng gói kỹ.",
        "media": [
          { "media_id": "rm-1", "url": "https://cdn.taca.vn/r/rm-1.webp", "type": "IMAGE", "status": "READY" }
        ],
        "author": { "display_name": "Nguyễn V. A", "avatar_url": null },
        "is_verified_purchase": true,
        "seller_reply": {
          "content": "Cảm ơn bạn đã ủng hộ shop ạ!",
          "replied_at": "2026-08-28T02:00:00Z"
        },
        "created_at": "2026-08-27T10:00:00Z",
        "edited_at": null
      }
    ]
  },
  "meta": { "request_id": "01912fd1-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 128 }
}
```

`author.display_name` đã được rút gọn (`Nguyễn V. A`) — **không** trả `user_id`, email, phone. Media `SCANNING` bị loại khỏi response public.

### 3.2 `POST /products/{productId}/reviews`

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `order_id` | string | Có | Order phải `DELIVERED` và thuộc buyer trong token |
| `sku_id` | string | Có | Phải nằm trong order item của `order_id` |
| `rating` | int | Có | 1–5 |
| `comment` | string | Không | ≤ 5.000 ký tự |
| `media_ids` | string[] | Không | ≤ 6, phải `READY` |

```json
{
  "data": {
    "review_id": "rv-01912fd2",
    "product_id": "product-01912f31",
    "sku_id": "sku-01912f33",
    "rating": 5,
    "comment": "Sạc nhanh, đóng gói kỹ.",
    "status": "PUBLISHED",
    "is_verified_purchase": true,
    "version": 1,
    "created_at": "2026-08-31T04:00:00Z",
    "aggregate": { "avg": 4.6, "count": 129, "distribution": { "1": 2, "2": 3, "3": 8, "4": 30, "5": 86 } }
  },
  "meta": { "request_id": "01912fd3-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Một buyer chỉ review **một lần cho mỗi `(order_id, sku_id)`** — trùng trả `409 REVIEW_ALREADY_EXISTS`. Order chưa `DELIVERED` → `409 REVIEW_NOT_ELIGIBLE`. Review **published ngay**, không có hàng đợi duyệt. Sau khi ghi, service phát `rating.aggregate.updated` để Product Catalog và Search cập nhật (§6 LLD).

### 3.3 Update/delete

- `PATCH /reviews/{id}` body `{version,rating,comment,media_ids[]}`; chỉ owner, trong edit window 30 ngày baseline; aggregate recompute.
- `DELETE /reviews/{id}` body `{version,reason?}`; soft delete, aggregate recompute, không xóa audit.

### 3.4 Media

`POST /reviews/{reviewId}/media/upload-url` response signed URL/private object; `POST /reviews/{reviewId}/media/complete` verify object key/checksum/type/size rồi `READY`. Media scan async trả `SCANNING`, chưa public.

### 3.5 Seller reply

`POST/PATCH /seller/reviews/{id}/reply` body `{content,version?}`; seller phải own shop của product, max 2.000 chars, một reply/review baseline. Seller không sửa rating/comment. DELETE soft delete reply.

### 3.6 Health

`/health/live` process-only; `/health/ready` MongoDB/Kafka/S3 config. Dependency fail trả `503` và error có trace ID.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `REVIEW_INVALID_INPUT` | 400 | Rating/comment/media sai. |
| `REVIEW_UNAUTHENTICATED` | 401 | Cần login. |
| `REVIEW_FORBIDDEN` | 403 | Sai buyer/seller scope. |
| `REVIEW_NOT_FOUND` | 404 | Review không tồn tại. |
| `REVIEW_NOT_ELIGIBLE` | 409 | Chưa delivered/không mua product. |
| `REVIEW_ALREADY_EXISTS` | 409 | Đã review order-product. |
| `REVIEW_EDIT_EXPIRED` | 409 | Hết edit window. |
| `REVIEW_MEDIA_INVALID` | 400 | Media sai. |
| `REVIEW_MEDIA_LIMIT_EXCEEDED` | 409 | Vượt 6 file. |
| `REVIEW_VERSION_CONFLICT` | 409 | Concurrent update. |
| `REVIEW_DEPENDENCY_UNAVAILABLE` | 503 | Order/Mongo/Kafka/S3 down. |
| `REVIEW_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Verified delivered và unique review/order-product/buyer. | Ảnh hưởng eligibility/unique index. | Order/Product owner |
| 2 | Không pre-moderation theo lựa chọn A. | Nếu cần censor, thêm moderation workflow. | Product owner |
| 3 | Edit window 30 ngày/delete policy là baseline. | Ảnh hưởng UX/aggregate. | Product owner |
| 4 | Delivered event/fallback Order contract chưa chốt. | Ảnh hưởng eventual eligibility. | Order owner |
| 5 | Media scan provider chưa chốt. | Ảnh hưởng SCANNING/READY. | Security/DevOps |
