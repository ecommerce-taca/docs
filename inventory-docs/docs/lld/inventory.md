# LLD — Inventory Service

> Nguồn: `HighLevelDesign/HLD.md` · `FrontendDesign/Taca.penpot.zip` · Code: `ecommerce-taca/inventory-service` @ `develop` (`92c1454`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai nghiệp vụ — repo mới có khung: health probe, error/response envelope, actor-context filter, `AuthorizationPolicy`, JSON log, metrics, trace propagation, Liquibase baseline. Chưa có bảng, endpoint nghiệp vụ, outbox hay consumer.
> Tech stack: Java 25 · Spring Boot 4.0.6 (web, jdbc, actuator) · MySQL 8.4 · Liquibase · Spring Kafka; mô hình đã chốt: SKU atomic reservation + ledger

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Sở hữu available/reserved stock theo SKU, atomic reserve/commit/release/expire, adjustment và immutable movement ledger. |
| Source of truth | Inventory là source of truth duy nhất cho stock; Product chỉ giữ display projection. |
| Accuracy | Không oversell; quantity integer không âm; lock/version/idempotency ở cùng transaction. |
| Database | MySQL 8.4 `inventorydb`; optimistic locking (version). |
| Không thuộc service | Product content/price, cart/order state, payment, shipment, KYC, search index. |
| Warehouse | Baseline một inventory balance/SKU; multi-warehouse chưa bật trong v1. |

### 1.2 Boundary

```text
checkout-service ──reserve / release (chưa có đơn)──► Inventory
order-service ────commit / release──────────────► Inventory
Seller/Admin ──stock adjustment/read──► Inventory
Inventory ──Kafka outbox──► Product projection, Order reconciliation
Product Catalog ──SKU lifecycle event──► Inventory SKU registration/disable
```

- Chỉ Inventory được mutate `qty_available`, `qty_reserved` và movement ledger.
- Product/Order không được đọc trực tiếp `inventorydb`; dùng API/event.
- Reservation phải gắn `reservation_id`, `order_intent_id`, `idempotency_key`, expiry và caller trace.
- Không có đủ stock là business conflict `409`, không phải lỗi hệ thống.

### 1.3 Mapping HLD/Penpot

| Nguồn | Requirement | Quyết định |
|---|---|---|
| HLD Inventory | `qtyAvailable`, `qtyReserved`, optimistic locking | Dùng SKU balance + optimistic locking (version). |
| mfe-buyer (Cart/Checkout) | Check inventory before checkout | Availability read-only rồi atomic reserve khi place. |
| mfe-seller (Dashboard) | Manage own products and stock | Seller scoped adjustment/read; audit reason bắt buộc. |
| mfe-catalog (Product Detail) | Còn hàng/hết hàng | Product consume stock snapshot event, không gọi deduction. |
| Order lifecycle | Paid/cancel/expire | Commit/release reservation idempotent theo order intent. |

## 2. Cấu trúc bên trong

### 2.1 Module Spring Boot

Đã có trong code (✅):

```text
com.taca.inventory
├── InventoryApplication
├── common/api/          # ApiResponse {data, meta{request_id,page,size,total,total_pages}}
├── common/error/        # ErrorCode, ErrorEnvelope, GlobalExceptionHandler, InventoryException
├── observability/       # HealthController, HealthProbeService
│   ├── logging/         # HttpLoggingFilter, JsonStructuredLogFormatter, SensitiveDataMasker
│   ├── metrics/         # InventoryMetrics (inventory.operation.duration, inventory.dependency.duration), BoundedMetricTags
│   └── trace/           # TraceContextFilter, TraceContext, KafkaTracePropagator
└── security/            # ActorContext, ActorContextFilter, AuthorizationPolicy, SecurityProperties
```

Thiết kế (⏳ chưa có):

```text
com.taca.inventory
├── balance/             # SKU balance and lock/version
├── reservation/         # reserve/commit/release/expire
├── movement/            # immutable stock ledger
├── adjustment/          # seller/admin correction workflow
├── sku-projection/      # Product SKU lifecycle reference
├── outbox/              # stock/reservation events
├── security/            # shop scope, admin permission, idempotency
└── observability/       # shared JSON log/trace/metric contract
```

| Thành phần | Trách nhiệm | Ràng buộc |
|---|---|---|
| `InventoryController` | Availability/reservation/internal contract | Không expose arbitrary delta cho buyer/order. |
| `SellerInventoryController` | Seller stock read/adjust | Actor shop scope; reason/audit; không đổi SKU shop khác. |
| `ReservationService` | Atomic reserve/commit/release/expire | Idempotent command, lock balance và reservation cùng transaction. |
| `BalanceService` | Read/update qty/version | `available ≥0`, `reserved ≥0`; version compare-and-set. |
| `MovementService` | Ledger append-only | Mọi delta phải có reason/ref/actor; không update/delete movement. |
| `ExpiryJob` | Release expired reservation | Batch bounded, idempotent, advisory lock khi chạy nhiều instance. |
| `OutboxPublisher` | Publish stock snapshot/reservation event | Transactional outbox, retry 3/backoff 2s/DLQ. |

### 2.2 Chuẩn observability dùng chung

- Log JSON stdout với field: `timestamp`, `level`, `service`, `env`, `version`, `event`, `trace_id`, `span_id`, `request_id`, `route`, `method`, `status_code`, `duration_ms`.
- REST/Kafka propagate W3C `traceparent`/`tracestate` và `X-Request-ID`; reservation log thêm `reservation_id`/`order_intent_id` được allowlist.
- Không log Authorization, token, payment data, raw address/phone/email, full order payload hoặc secret; quantity/idempotency payload chỉ log theo safe summary.
- Metrics không label bằng `sku_id`, `order_id`, `user_id`; dùng `operation`, `result`, `reason`, `service_dependency` bounded.
- `/health/live` chỉ process; `/health/ready` kiểm MySQL/Kafka/config. DB lock timeout/deadlock là metric/alert, không log raw SQL bind values.
- Error envelope `{error:{code,message,details,trace_id}}`; audit event chứa actor/shop/reason nhưng không chứa secret.

## 3. Luồng xử lý

### 3.1 Đăng ký SKU

1. Consume `sku.created`/`sku.updated`/`sku.status_changed` từ Product (tên event theo `product-catalog-docs/docs/lld/product-catalog.md` §6.2).
2. Tạo `inventory_items` cho SKU mới với quantity 0 nếu chưa có.
3. Disable/Archive SKU không hard-delete balance/ledger; không cho reserve SKU disabled.
4. Event duplicate/out-of-order bỏ qua theo source version.

### 3.2 Availability check

- `GET` trả `available_qty`, `reserved_qty` display cho internal Order/Seller policy.
- Read-only, không tạo reservation và không đảm bảo quantity giữ đến place.
- Product nhận snapshot event có `source_version`; Search/Product không được suy diễn deduction từ snapshot.

### 3.3 Atomic reserve

```text
BEGIN
  read inventory_items current version and quantity
  validate SKU active, request idempotency, expiry and quantity
  if qty_available < requested → rollback, STOCK_INSUFFICIENT
  update inventory_items set qty_available -= q, qty_reserved += q, version += 1 where sku_id = ID and version = current_version
  if affected_rows == 0 → rollback/retry, CONCURRENCY_CONFLICT
  insert/update reservation RESERVED
  insert movement RESERVE (delta available=-q, reserved=+q)
  insert outbox `inventory.reservation.created` + `inventory.stock_snapshot.updated`
COMMIT
```

Ràng buộc:

- Cùng `idempotency_key` + caller scope + payload giống nhau trả cùng result; payload khác trả conflict.
- Sắp xếp tăng dần `sku_id` khi update nhiều SKU trong cùng transaction để giảm deadlock.
- Không gọi payment/product trong DB transaction.
- Nếu bất kỳ SKU nào fail, toàn bộ reservation request fail; không partial reserve.

### 3.4 Commit/release/expire

| Operation | Atomic change | Trigger |
|---|---|---|
| `COMMIT` | `qty_reserved -= q`; reservation `COMMITTED`; movement commit | `order-service` gọi khi đơn được chốt: với VNPAY là lúc nhận `payment.succeeded`, với **COD là ngay khi tạo đơn từ checkout** (không chờ tiền). Inventory không tự suy ra thời điểm này. |
| `RELEASE` | `qty_reserved -= q`, `qty_available += q`; status `RELEASED`; movement release | Order cancel/payment fail. |
| `RELEASE_FROM_COMMITTED` | `qty_available += q`; status `RELEASED`; movement `release_committed` | Order huỷ sau commit (VNPAY) — không chạm `qty_reserved` vì COMMIT đã trừ. |
| `EXPIRE` | Tương tự RELEASE; status `EXPIRED` | TTL job sau `expires_at`. |
| `ADJUSTMENT` | Available delta theo policy; movement adjustment | Seller/admin stock correction, reason bắt buộc. |

Commit/release/expire cùng reservation đã hoàn tất trả idempotent success; không append movement lần hai.

### 3.5 Seller adjustment

- Seller chỉ adjust SKU thuộc shop scope và SKU active/allowed status.
- Adjustment request phải có absolute target hoặc signed delta, reason và idempotency key; baseline dùng delta integer bounded.
- Admin có thể correction với permission + step-up; mọi change audit append-only.
- Không cho seller sửa `qty_reserved` hoặc xóa movement.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---|---|---|
| `QTY_MIN` | 0 | — | Integer, không âm. |
| `QTY_MAX_PER_SKU` | 999999999 | — | Chống overflow/nhập nhầm. |
| `RESERVATION_TTL` | 15 phút | — | Align Order checkout. |
| `RESERVATION_MAX_ITEMS` | 100 | — | SKU/request. |
| `ADJUSTMENT_MAX_DELTA` | 999999999 | — | Bounded request. |
| `DB_LOCK_TIMEOUT` | 3s | — | Deadlock/lock conflict map 503 hoặc retry policy. |
| `IDEMPOTENCY_RETENTION` | 24h | — | Command result snapshot. |
| `EXPIRY_BATCH_SIZE` | 500 | — | Mỗi job batch. |
| `EXPIRY_INTERVAL` | 30s | — | Job trigger; phải idempotent multi-instance. |
| `OUTBOX_RETRY_COUNT` | 3 | — | Backoff 2s rồi DLQ. |
| `STOCK_SNAPSHOT_EVENT` | per successful mutation | — | Product projection only. |
| `INTERNAL_TIMEOUT` | 5s | — | REST client; không kéo dài DB lock. |
| `TIMESTAMP_FORMAT` | UTC ISO-8601 | — | DB DATETIME(6), API/event UTC. |

Cấu hình đã có trong `application.yml` (✅):

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---|---|---|
| Port | `8080` | `server.port` | Ghi cứng trong yaml; `SERVER_PORT` vẫn ghi đè được nhờ relaxed binding của Spring. |
| Datasource | `jdbc:mysql://localhost:3306/inventorydb`, user `inventory` | `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD` | |
| Kafka | `localhost:9092`, group `inventory-service-group`, `auto-offset-reset: earliest` | `SPRING_KAFKA_BOOTSTRAP_SERVERS` | |
| Liquibase | `classpath:db/changelog/db.changelog-master.yaml` | `spring.liquibase.*` | |
| Internal token | (không default) | `inventory.security.internal-token` | |
| Hikari connection timeout | 2000 ms | set trong `HealthProbeService` | |

Các hằng số ở bảng trên là thiết kế ⏳.

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `InventoryItemStatus` | `ACTIVE`, `DISABLED`, `ARCHIVED`. |
| `ReservationStatus` | `RESERVED`, `COMMITTED`, `RELEASED`, `EXPIRED`. |
| `MovementReason` | `INITIALIZE`, `RESERVE`, `COMMIT`, `RELEASE`, `RELEASE_COMMITTED`, `EXPIRE`, `ADJUSTMENT`, `RETURN`, `DAMAGE`. |
| `StockAdjustmentSource` | `SELLER`, `ADMIN`, `IMPORT`, `SYSTEM`. |
| `CommandResult` | `APPLIED`, `IDEMPOTENT_REPLAY`, `REJECTED`, `CONFLICT`. |

Invariant: `qty_available ≥ 0`, `qty_reserved ≥ 0`; reservation quantity không vượt reserved balance; tổng movement/audit phải reconcile được với balance. Các bước trả hàng (`RELEASE`, `RELEASE_FROM_COMMITTED`, `EXPIRE`) đều phải giữ quantity **không âm** sau khi cập nhật.

## 6. Event phát ra / lắng nghe

### 6.1 Event phát ra

⏳ Chưa có producer/outbox; chỉ có `KafkaTracePropagator` để inject/extract `traceparent`, `tracestate`, `X-Request-ID` vào Kafka header.

| Topic | Event | Payload chính |
|---|---|---|
| `inventory.events.v1` | `inventory.stock_snapshot.updated` | `sku_id`, `product_id`, `available_qty`, `reserved_qty`, `committed_qty`, `source_version`, `as_of` (tên field theo consumer `InventoryProjectionService` của Product Catalog) |
| `inventory.events.v1` | `inventory.reservation.created` | reservation/order_intent, SKU quantities, expires_at |
| `inventory.events.v1` | `inventory.reservation.committed` | reservation/order, committed_at |
| `inventory.events.v1` | `inventory.reservation.released` | reservation/order/reason |
| `inventory.events.v1` | `inventory.reservation.expired` | reservation/order/expired_at |
| `inventory.events.v1` | `inventory.adjusted` | sku, delta, reason, actor, movement_id |

### 6.2 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Product Catalog | `sku.created`, `sku.updated`, `sku.status_changed` | Register/update/disable SKU projection; không lấy giá/product content. |
| `order-service` | `order.cancelled` (reconciliation) | Release reservation nếu command callback bị mất; idempotent. |
| Payment-Wallet | *(không consume)* | Commit/release do **`order-service`** gọi qua API command (`POST /internal/inventory/reservations/{reservationId}/commit|release`); Payment-Wallet không mutate stock. |

### 6.3 Mock contract — reserve response

Response `201` dùng envelope chung `{data, meta:{request_id}}` của Inventory (`ApiResponse`, đã có code) — giống `docs/api/inventory.md` §3.2 và cách `checkout-service` đọc (`checkout-docs/docs/lld/checkout.md` §6.1). `trace_id` không nằm trong body thành công (chỉ có trong error envelope và header `traceparent`); `Idempotency-Key` là header request, không echo trong body.

```json
{
  "data": {
    "reservation_id": "reservation-01912f82",
    "status": "RESERVED",
    "expires_at": "2026-08-30T09:15:00Z",
    "items": [{"sku_id": "sku-01912f33", "quantity": 2, "available_after": 5}]
  },
  "meta": { "request_id": "01912f80-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`available_after` chỉ trả cho caller internal cần thiết; Product public không nhận trực tiếp response này.

## 7. Mã lỗi

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `INVENTORY_INVALID_INPUT` | 400 | Quantity/idempotency/field sai. |
| `INVENTORY_UNAUTHENTICATED` | 401 | Thiếu service/user auth context. |
| `INVENTORY_FORBIDDEN` | 403 | Sai shop scope/permission. |
| `INVENTORY_SKU_NOT_FOUND` | 404 | SKU chưa register/không tồn tại. |
| `INVENTORY_SKU_DISABLED` | 409 | SKU không cho reserve/adjust theo state. |
| `INVENTORY_STOCK_INSUFFICIENT` | 409 | Available quantity không đủ. |
| `INVENTORY_RESERVATION_NOT_FOUND` | 404 | Reservation không tồn tại/scope sai. |
| `INVENTORY_RESERVATION_STATE_INVALID` | 409 | Commit/release sai state. |
| `INVENTORY_RESERVATION_EXPIRED` | 409 | Reservation đã hết TTL. |
| `INVENTORY_IDEMPOTENCY_CONFLICT` | 409 | Cùng key nhưng payload khác. |
| `INVENTORY_CONCURRENCY_CONFLICT` | 409 | Version/lock conflict sau retry policy. |
| `INVENTORY_ADJUSTMENT_INVALID` | 400/409 | ⏳ Chưa có trong enum `ErrorCode`. Delta/reason sai (400) hoặc state không cho phép (409). |
| `INVENTORY_DEPENDENCY_UNAVAILABLE` | 503 | Kafka/config dependency không sẵn sàng. |
| `INVENTORY_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Inventory dùng SKU-level, một balance/SKU, không multi-warehouse trong v1. | Nếu thêm warehouse, cần allocation/reservation theo location và schema/API mới. | Inventory/Product owner |
| 2 | Backend Spring Boot/Java 25 + MySQL 8.4 theo HLD baseline. | Ảnh hưởng lock/migration/deployment. | Tech lead |
| 3 | Reservation TTL 15 phút align Order; exact timeout/payment flow chưa có HLD contract. | Ảnh hưởng abandoned checkout và stock availability. | Order/Payment owner |
| 4 | Seller adjustment dùng signed delta; chưa có cycle count/import UX chi tiết trong Penpot. | Có thể cần bulk import, approval hoặc absolute target flow. | Seller/Product owner |
| 5 | Commit/release do **`order-service` gọi trực tiếp qua API command** (đã chốt 2026-09-18, NC28; reserve do `checkout-service` từ khi tách Order-Commerce 2026-11-01) — Payment-Wallet không mutate stock và không có saga Payment/Order cho commit; event `inventory.reservation.*` chỉ dùng để reconcile khi mất response. | Ảnh hưởng duplicate commit/release và reconciliation job. | Order owner |
| 6 | `inventory.stock_snapshot.updated` là mock contract đã bổ sung để Product hiển thị. | Cần chốt topic/schema registry/source version ở platform. | Platform owner |
| 7 | MySQL optimistic lock + idempotency đủ baseline exactness; multi-region active-active chưa thuộc v1. | Nếu cần multi-region, phải có single-writer/partition strategy. | Architecture |
