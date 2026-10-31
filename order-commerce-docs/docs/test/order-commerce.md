# Test Plan — Order-Commerce Service

> Nguồn: `docs/lld/order-commerce.md` · `docs/db/order-commerce.md` · `docs/api/order-commerce.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers (MySQL 8.4, Redis, Kafka)

Tài liệu này là **tổng quan** test: chuẩn bị, manual QA, test chung, tiêu chí phát hành. Case chi tiết tách theo nghiệp vụ:

| Tài liệu | Case ID |
|---|---|
| `docs/test/cart.md` | `O-API-01..03` |
| `docs/test/checkout.md` | `O-API-05..10`, `O-API-21`, `O-INT-02`, `O-INT-06`, `O-CALC-01..15` |
| `docs/test/order.md` | `O-API-11..17`, `O-API-22`, `O-INT-01`, `O-INT-03..05`, `O-COD-01..10` |
| `docs/test/voucher.md` | `O-API-04..04e`, `O-API-18..20`, `O-VCAN-01..05` |

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers. |
| DB/cache | MySQL 8.4 InnoDB + Redis TTL 30 ngày; Kafka test broker. |
| Dependency doubles | Product price/status, Inventory availability/reserve/commit/release, Payment, Shipment, Auth. |
| Data | Cart empty/valid/expired, multi-shop cart, product price changed, vouchers platform/shop, orders mọi state, invoice pending/issued. |
| Actors | Buyer owner/other buyer, seller owner/staff/other shop, admin, expired JWT. |
| Observability | Capture JSON logs/traces/metrics; assert `X-Request-ID`, W3C context, Kafka correlation. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Cart | Add/update/delete, empty/expired, SKU disabled | mfe-buyer state đúng; quantity 1–999; cart không giữ stock. |
| Checkout | Preview, price changed, stock unavailable, multi-shop split | Warnings rõ; không tạo order ở preview; place split đúng. |
| Payment | VNPAY pending/success/fail, COD | Payment action đúng; Order không tự giả payment success. |
| Voucher | Platform/shop, expired, limit, wrong scope | Discount/validation đúng; redemption atomic. |
| Orders | Buyer list/detail/status/cancel before/after shipping | Scope đúng; cancel sau shipping bị chặn. |
| Seller | Order list/fulfill/cancel/invoice | Chỉ shop own; không tự set delivered. |
| MFE states | Loading, empty, error, timeout, retry | Idempotent retry không duplicate order. |

## 3. API integration

Case API/event theo nghiệp vụ nằm ở các file trên. ID giữ nguyên để trace.

## 4. Unit test

⏳ Repo chưa có test class nào; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `Observability` | Required JSON fields, trace propagation, no token/payment/address/order raw logs. |

Unit test nghiệp vụ: `CartService` (`cart.md`), `CheckoutCalculator`/`TaxExtractor`/`InventoryClient` (`checkout.md`), `OrderStateMachine`/`OrderMapper` (`order.md`), `VoucherService`/`VoucherScopeResolver` (`voucher.md`).

## 5. Contract/security/resilience

- Contract Product→Order: active SKU, current integer VND price, unavailable/price-changed errors.
- Contract Order→Inventory: all-or-nothing reserve, reservation TTL, commit/release idempotency.
- Contract Order→Payment/Shipment: event/status transition and compensation.
- Security: IDOR buyer/seller, forged shop/user IDs, replay/idempotency, voucher abuse, raw address/payment log scan.
- Load: cart/preview/list read load; checkout concurrency same SKU must not create excess reservations/orders.

## 6. Tiêu chí pass/phát hành

Không duplicate order/payment/reservation dưới retry/concurrency; mọi state transition, money reconciliation, scope/RBAC, outbox/replay, trace/log redaction pass; no test relies on Product as inventory ledger.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | MySQL 8.4 thay cho Postgres theo lựa chọn A. | Cần đổi Testcontainers/SQL dialect nếu architecture đổi. | Tech lead |
| 2 | Checkout reserve trước tạo order và Payment tách saga. | Cần điều chỉnh compensation test nếu flow đổi. | Order/Payment/Inventory |
| 3 | ~~Voucher stacking/tax chưa đầy đủ~~ → **Đã đóng**: 3 slot + thuế theo danh mục đã chốt, golden cases ở `docs/test/checkout.md` §3.2 và `docs/test/voucher.md` §3.1. Còn lại: carrier quote policy thật (đang dùng mock). | Ảnh hưởng độ chính xác phí ship. | Shipment owner |
| 3b | Voucher v1 chỉ theo mã (không audience/targeting); platform voucher qua `/admin/vouchers`. | Nếu bật targeting cần thêm test audience/segment. | Product owner |
| 4 | PDF invoice provider chưa chốt. | Tạm test metadata/async state, chưa visual PDF. | Finance/DevOps |
