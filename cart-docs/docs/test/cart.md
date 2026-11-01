# Test Plan — Cart Service

> Nguồn: `docs/lld/cart.md` · `docs/api/cart.md` · `docs/db/cart.md` · Code: chưa có repo (`ecommerce-taca/cart-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc, Testcontainers (MongoDB 8)

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers MongoDB 8 replica set (transaction §3.1a DB); TestClock để điều khiển thời gian TTL 24h, kiểm tra indexes. |
| Dependency doubles | Product detail `GET /products/{productId}` (`skus[].sku_id`, `price.sale_price`, `attributes`, product inactive, `404`, SKU không thuộc product, 5xx, timeout); Inventory availability đủ/thiếu/timeout; fake HTTP server kiểm concurrency và hydrate budget. Verify status/DTO theo contract thực tế trước khi viết mapper. |
| Data | Seed ở `docs/db/cart.md` §6. |
| Actors | Buyer owner, buyer khác, thiếu JWT/context, `X-User-ID` giả mạo/tự inject, direct service caller, internal caller có/không/sai token; Gateway private routes. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Cart | Add/update/delete, giỏ rỗng, SKU bị tắt, Product/Inventory timeout | mfe-buyer hiển thị đúng; quantity 1–999; `availability_status` đúng và fallback snapshot; cart không giữ stock. |
| Lưu lâu dài | Đăng xuất/đổi thiết bị, quay lại sau nhiều tuần | Giỏ còn nguyên. |
| Sau đặt hàng | Đặt 2 trong 3 item; buyer đổi quantity item đang checkout | Item chưa đặt vẫn còn; item đã sửa sau snapshot không bị xoá nhầm (`skipped_modified_item_ids`). |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| CART-API-01 | `GET /cart` buyer có/không giỏ | `200`; chưa có giỏ thì `items=[]`, `cart_id=null`. |
| CART-API-02 | `POST /cart/items` valid/invalid/SKU trùng/dòng 101 | `201` cộng dồn quantity; `400`; `409 CART_TOO_MANY_ITEMS`. Không tin `user_id` body. |
| CART-API-03 | `PUT/DELETE /cart/items/{itemId}` item của buyer khác | `404 CART_ITEM_NOT_FOUND`. |
| CART-API-04 | `GET /cart` khi Product timeout/Inventory timeout/giá Product đổi | `200` fallback snapshot; Product timeout → `STALE` (kể cả Inventory lỗi), Product hợp lệ + Inventory timeout → `UNKNOWN`. Giá mới chỉ ở response, `line_total = unit_price_snapshot` response `* quantity`; không ghi DB/tăng version. |
| CART-API-05 | `GET /internal/carts/{id}` thiếu/sai token | `401 CART_UNAUTHENTICATED`. |
| CART-API-06 | `GET /internal/carts/{id}?item_ids={id1},{missing}` có item không tồn tại | `200`, item thiếu ở `missing_item_ids`, item tồn tại có `item_revision`; `?item_ids=` rỗng → `400`. |
| CART-API-07 | `checked-out` G1 → G2 → gọi lại G1 (kèm `expected_item_revisions`) | Không xoá ngoài item có revision khớp; replay G1 `removed_item_ids=[]`, item đã sửa giữ nguyên, `200`. |
| CART-API-08 | Không có TTL | Document không bị xoá sau 30+ ngày (kiểm không có TTL index). |
| CART-API-09 | `POST /cart/items` gửi 2 lần cùng `Idempotency-Key`; đối chiếu POST không key | Cùng key chỉ cộng 1 lần; replay trả HTTP `201` và body gốc gồm `meta.request_id`; cùng key khác body trong 24h → `409 CART_IDEMPOTENCY_CONFLICT`. Không key thì hai request độc lập cộng hai lần, không auto-retry sau unknown result. |
| CART-API-10 | `GET /internal/carts/{id}` không có `item_ids` | Trả mọi item, mỗi item có `item_revision`; giỏ chưa có → `items=[]`, `cart_id=null`. |
| CART-API-11 | 21 lần thêm cùng SKU với key K1…K21 (mỗi lần +1), rồi gửi lại K1 trong 24h | Quantity giữ 21; K1 trả response gốc (không có giới hạn số key). Hai request đồng thời cùng key → chỉ cộng 1 lần. |
| CART-API-12 | `GET /cart` giỏ có SKU đã bị seller tắt | SKU không có trong `skus[]` → item `available=false`, `availability_status=PRODUCT_UNAVAILABLE`, không lỗi. |
| CART-API-13 | Hai request cùng key đồng thời | Chỉ một mutation được commit; request còn lại replay HTTP 201/body gốc. |
| CART-API-14 | Hai request khác key/no-key đồng thời cùng cart | Không mất cập nhật; CAS retry bounded hoặc `409 CART_VERSION_CONFLICT`, state cuối nhất quán. |
| CART-API-15 | Reuse key sau 24h khi TTL chưa dọn | Key hết hạn được claim lại; không false 409; TTL/delete race vẫn chỉ có một mutation cho key mới. |
| CART-API-16 | POST Product 404/inactive/SKU sai hoặc timeout/5xx; GET cache Product vừa cũ | POST validate bằng lần đọc Product mới, không dùng cache GET 30s; 409 với unavailable, 503 với dependency down; cart và key không bị ghi partial. |
| CART-API-17 | GET hydrate chạy đồng thời PUT/DELETE | GET chỉ dựng response; `version`, `updated_at`, `item_revision` trong DB không bị đổi bởi GET; mutation không bị ghi đè. |
| CART-API-18 | Buyer đổi qty từ 2→5 trong lúc Checkout place | checked-out với revision cũ không xoá item qty 5; `skipped_modified_item_ids` chứa item; không tự trừ số lượng. |
| CART-API-19 | checked-out thiếu/map revision sai, hoặc cart_id không thuộc buyer | Thiếu/sai revision → 400; sai cart/buyer → 404; không xoá dữ liệu. |
| CART-API-20 | `GET /internal/carts/{buyerUserId}` với `?item_ids=` rỗng, trùng ID, >100 ID | 400 cho rỗng/>100, dedupe giữ thứ tự cho ID trùng. |
| CART-API-21 | GET giỏ 100 product khác nhau; Product/Inventory chậm | Tối đa 10 concurrent Product calls, Product cache 30s; Inventory không fan-out không giới hạn, dùng budget còn lại; hydrate tổng ≤3s + HTTP overhead hợp lý; `200` fallback. DTO và cách batch/fan-out Inventory kiểm theo contract thực tế. |
| CART-API-22 | Product invalid/timeout phối hợp Inventory đủ/thiếu/timeout | Product invalid → `PRODUCT_UNAVAILABLE` ưu tiên hơn Inventory; Product timeout → `STALE`; Product valid + Inventory đủ/thiếu/timeout → `AVAILABLE`/`OUT_OF_STOCK`/`UNKNOWN`; `available` hint không dùng chốt order. |
| CART-API-23 | Header `X-User-ID` giả mạo, thiếu actor context, token nội bộ sai | Gateway strip/inject; direct public access bị chặn; thiếu/sai internal token `401`. |
| CART-API-24 | Mongo transaction/write conflict/unknown commit/DB outage | Không partial commit cart/key; known conflict chưa commit → retry bounded; keyed POST unknown commit → kiểm key/hash trước replay, không được cộng đôi; không thể xác minh → `503 CART_STORAGE_UNAVAILABLE`; no-key POST mất ACK → không tự retry phép cộng; DB down → `503`. |
| CART-API-25 | SKU capped 999/no-op PUT; DELETE sau đó re-add | No-op không tăng revision/version; capped POST có key vẫn lưu response idempotency 201; add lại tạo `item_id` mới, stale checked-out không xoá item mới. |

## 4. Unit test

⏳ Repo chưa có test class; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `CartService` | Quantity bounds, SKU trùng, cap 999/no-op, tối đa 100 dòng, scope buyer, CAS `version`, `item_revision` tăng đúng lúc, DELETE + re-add ID mới; thử PUT đến muộn từ thiết bị khác (không có client expected_version, giới hạn được document); không có reservation semantics. |
| `CartIdempotencyService` | SHA-256 canonical body, replay, key expired vs TTL chưa xoá, conditional reclaim, concurrent same key, capped no-op vẫn lưu response, unknown commit: verify key/hash trước khi quyết định retry. |
| `ProductCatalogClient` / `InventoryClient` | Mapping SKU thuộc product, status, GET cache 30s nhưng POST validate fresh, Product concurrency 10; Inventory bounded fan-out và dùng hydrate budget còn lại; ưu tiên `PRODUCT_UNAVAILABLE`/`STALE` trước Inventory status, `line_total` từ giá response, response-only hydrate. |
| `InternalCartService` | Filter buyer/cart_id, expected item revisions, missing/modified/removed classification, no-op cleanup, CAS retry. |
| `InternalTokenFilter` | Constant-time compare (fixed-length digest), thiếu/sai token, không leak giá trị, deny direct public callers. |
| `Observability` | Field log bắt buộc, không log token/nội dung giỏ. |

## 5. Contract/security/resilience

- Contract Cart→Product: `GET /products/{productId}` theo `product-catalog-docs/docs/api/product-catalog.md` §3.1; map giá từ `skus[].price.sale_price`, nhãn từ `skus[].attributes`; SKU vắng hoặc product `404`/inactive → `PRODUCT_UNAVAILABLE`; timeout dùng snapshot cũ (`STALE`). Max 10 lời gọi song song, cache `product_id` 30s, timeout per-call 2s và toàn hydrate 3s. **Không** dùng batch `GET /products?product_ids=` (không có `skus[]`). Field active phải verify contract Product thực tế.
- Contract Cart→Inventory: availability read-only theo `inventory-docs/docs/api/inventory.md` #1; khi thiếu quantity → `OUT_OF_STOCK`, timeout → `UNKNOWN`. Request/response DTO chưa nằm trong bộ Cart docs, phải verify trước implement adapter.
- Contract Checkout→Cart: §3.5/§3.6 **đã thay đổi** so với `checkout-docs/docs/lld/checkout.md` §3.3: internal GET thêm `item_revision`, checked-out bắt buộc `expected_item_revisions`; phải cập nhật caller/contract tests phía Checkout trước khi tích hợp, không khẳng định sẵn tương thích.
- Security: IDOR item/cart, token nội bộ compare constant-time, Gateway strip/inject actor header, deny direct access, private `/internal/**`, không log token/cart content; kiểm tra public ingress và network policy.
- Load: `GET /cart` lúc sale; update đồng thời cùng giỏ → retry/CAS hoặc `CART_VERSION_CONFLICT`, không lost update. Kiểm tra Product fan-out ≤10, Inventory bounded fan-out (ngưỡng theo contract đã xác minh), hydrate tổng 3s, Mongo transaction fail/retry và expiry reclaim key. Verify các P0 Product/Inventory/Gateway/Checkout bằng contract và môi trường thực tế trước integration; chưa coi là pass khi thiếu nguồn đối chiếu.

## 6. Tiêu chí pass/phát hành

Mọi case §3 pass; không mất item dưới ghi đồng thời/Checkout cleanup; replay key không cộng đôi kể cả expiry chưa dọn; GET hydrate không ghi DB và fallback trong timeout budget; không log nội dung giỏ/token; `/internal/**` không truy cập qua Gateway; Product/Inventory/Checkout contract đã được đối chiếu trước integration; MongoDB replica set + indexes sẵn sàng.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | ID test đổi từ `O-API-01..03` (Order-Commerce) sang `CART-API-*`. | Trace với test cũ qua bảng này. | QA |
| 2 | Bổ sung tests cho thay đổi contract `expected_item_revisions` của Checkout. | Caller/LLD Checkout hiện chưa được xác minh trong bộ tài liệu này. | Checkout + QA |
| 3 | Product active field, Inventory DTO, Gateway ingress chưa có nguồn đính kèm. | Phải dùng contract thực tế làm fixture; không suy đoán schema. | Product/Inventory/Gateway owner |
