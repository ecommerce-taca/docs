# Test Plan — Shipment Service

> Nguồn: `docs/api/shipment.md` · `docs/db/shipment.md` · `docs/lld/shipment.md` · Code: `ecommerce-taca/shipment-service` @ `develop` (`a375a41`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — có 16 test class (unit + MySQL Testcontainers); chưa có MVC/API integration test và carrier contract test.
> Môi trường: JUnit 5, Testcontainers MySQL 8.4, Kafka (local `compose.yaml`), `scripts/smoke-test.ps1`

## 1. Chuẩn bị

| Thành phần | Fixture |
|---|---|
| Runtime | Java 25, Maven (`mvn test`), Docker cho Testcontainers. |
| Carrier | `MOCK` (`APP_ENV=local|test|staging`, `SHIPMENT_MOCK_ENABLED=true`); `GHN` để kiểm fail-closed. |
| Secret | `SHIPMENT_INTERNAL_TOKEN`, `SHIPMENT_OPS_TOKEN`, `SHIPMENT_WEBHOOK_MOCK_SECRET`; IP allowlist `127.0.0.1,::1`. |
| Actor | Buyer chủ đơn/khác, seller đúng/sai shop, `shipment:read:any`, internal Order, operator, carrier webhook. |
| Data | Shipment `MOCK` ở mỗi trạng thái; webhook trùng/cũ hơn; idempotency key còn hạn. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Quote | `MOCK` hợp lệ; `GHN`; carrier lạ | `MOCK` trả fee/ETA; `GHN` `503`; lạ `400`. |
| So sánh carrier | Gọi carrier-quotes | 3 dòng `GHN/SPX/J&T`, đều `available=false` với code hiện tại; không có `MOCK`. |
| Create | `MOCK` hợp lệ, gọi lại cùng key, gọi lại khác body | `201`; `200` cùng shipment; `409 SHIPMENT_IDEMPOTENCY_CONFLICT`. |
| Tracking | Buyer/seller xem timeline | Đúng scope; không có address trong response. |
| Webhook | Hợp lệ / trùng / cũ hơn / sai chữ ký | Áp dụng một lần / `duplicate` / `ignored` / `400`. |
| Cancel | Trước và sau pickup, sai version | `200` / `409` / `409`. |

## 3. API integration

| ID | Endpoint/case | Expected | Trạng thái |
|---|---|---|---|
| SH-API-01 | `GET /orders/{orderId}/shipment` chủ đơn / user khác / chưa có shipment | `200` / `403 SHIPMENT_FORBIDDEN` / `404 SHIPMENT_NOT_FOUND`. | ⏳ |
| SH-API-02 | Seller tracking: shop đúng / sai / có `shipment:read:any` / role `BUYER` | `200` / `403` / `200` / `403`. | ⏳ |
| SH-API-03 | Thiếu `X-User-ID`/`X-User-Roles` | `401 SHIPMENT_UNAUTHENTICATED`. | ✅ (`TrustedContextFilterTest`) |
| SH-API-04 | `/internal/**` thiếu/sai `X-Internal-Service-Token` | `401 SHIPMENT_UNAUTHENTICATED`. | ✅ (`TrustedContextFilterTest`) |
| SH-API-05 | `/internal/ops/**` thiếu operator token/ID/role | `403 SHIPMENT_FORBIDDEN`. | ✅ (`TrustedContextFilterTest`) |
| SH-API-06 | Quote snapshot >16 KiB / items rỗng / >100 | `400 SHIPMENT_INVALID_INPUT`. | 🟡 (`QuoteServiceTest`) |
| SH-API-07 | Quote `MOCK` khi `APP_ENV=production` | `503 SHIPMENT_CARRIER_UNAVAILABLE`. | 🟡 (`QuoteServiceTest`) |
| SH-API-08 | Create `MOCK` hợp lệ | `201`, `status=CREATED`, outbox `shipment.created`. | ⏳ |
| SH-API-09 | Create currency khác quote / currency sai định dạng | `400 SHIPMENT_CURRENCY_MISMATCH` / `400 SHIPMENT_INVALID_INPUT`. | ✅ (`ShipmentCurrencyTest`) |
| SH-API-10 | Create `cod_amount>0` với `payment_method≠COD` | `400 SHIPMENT_INVALID_INPUT`. | ⏳ |
| SH-API-11 | Create carrier timeout | `202`, `PENDING_RECONCILIATION`, outbox `shipment.status_changed`. | ⏳ |
| SH-API-12 | Create trùng `(order_id, shop_id)` với key khác | `200` shipment cũ. | ⏳ |
| SH-API-13 | Cancel `CREATED` / `PICKED_UP` / version lệch / ID sai | `200` / `409` / `409` / `404`. | ⏳ |
| SH-API-14 | Webhook hợp lệ / trùng cùng hash / trùng khác hash / IP lạ / chữ ký sai | applied / duplicate / `400` / `400` / `400`. | ✅ (`ShipmentWebhookServiceTest`, `WebhookSignatureVerifierTest`) |
| SH-API-15 | Webhook event cũ hơn / transition sai / status lạ / tracking không có | `200 ignored`, state không đổi. | ✅ (`ShipmentWebhookServiceTest`) |
| SH-API-16 | Ops replay `DEAD` / trạng thái khác | `200 RETRY_WAIT` / `409`. | ⏳ |
| SH-API-17 | `/health/ready` DB down / thiếu topic | `503` với body tương ứng. | ⏳ |

## 4. Unit test

Đã có trong `src/test/java`:

| Test class | Phạm vi |
|---|---|
| `ShipmentApplicationTests` | Context load. |
| `ShipmentStateMachineTest` | Transition cho phép/từ chối. |
| `UuidV7Test` | Version/variant UUIDv7. |
| `GhnCarrierAdapterTest` | GHN fail-closed. |
| `QuoteServiceTest` | Quote/compare, MOCK theo môi trường. |
| `ShipmentCurrencyTest` | Currency bắt buộc, mismatch. |
| `ShipmentReconciliationJobTest` | Đối soát thành công. |
| `ShipmentWebhookServiceTest` | Dedupe, stale, unknown status. |
| `WebhookSignatureVerifierTest` | HMAC, IP allowlist. |
| `TrustedContextFilterTest` | Internal/ops/actor header. |
| `OutboxClaimServiceTest`, `OutboxClaimServiceMySqlTest` | Claim, lease, `SKIP LOCKED` (MySQL). |
| `OutboxEventEntityTest`, `OutboxPublisherTest` | Retry, DLQ, lease fencing. |
| `KafkaProducerSafetyGuardTest` | Kiểm cấu hình producer. |
| `ShipmentFeeMigrationMySqlTest` | Migration `V005`/`V006` không backfill. |

Còn thiếu: MVC test cho envelope/validation/status code của từng endpoint (SH-API-01, 02, 08, 10–13, 16, 17), idempotency create/cancel end-to-end, consumer `order.cancelled` và create command.

## 5. Contract/security/resilience

| ID | Case | Expected |
|---|---|---|
| SH-INT-01 | Payload `shipment.delivered`/`shipment.failed` có `buyer.user_id`, `buyer.email` | Notification render được email. |
| SH-INT-02 | `shipment.created` từ đối soát | Consumer chịu được thiếu field phí. |
| SH-SEC-01 | Header actor giả khi gọi thẳng service | Chỉ an toàn khi network chặn truy cập trực tiếp. |
| SH-SEC-02 | Log scan | Không có token/secret/signature/address/raw webhook. |
| SH-RES-01 | DB/Kafka/carrier lỗi | Không tạo shipment trùng; outbox giữ event; `PENDING_RECONCILIATION`. |
| SH-RES-02 | Hai publisher đồng thời | Không publish vượt thứ tự trong cùng aggregate; dedupe theo `event_id`. |

## 6. Tiêu chí pass/phát hành

- `mvn test` xanh trên toolchain Java 25 + Docker.
- Toàn bộ SH-API ở §3 chuyển sang ✅ trước khi tích hợp Order-Commerce.
- Có adapter carrier thật (ít nhất GHN) + contract test trước production.
- Không lỗi bảo mật mức cao ở SH-SEC.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Chỉ `MOCK` test được end-to-end. | Không kiểm được carrier thật. | Shipment/DevOps |
| 2 | Chữ ký webhook carrier thật chưa có spec. | Test bảo mật webhook chỉ ở mức HMAC chung. | Security |
| 3 | Multi-package, return/exchange chưa v1. | Cần test matrix mới. | Product |
