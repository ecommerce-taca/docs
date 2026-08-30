# Test Plan — Order-Commerce Service

> Nguồn: `docs/lld/order-commerce.md` · `docs/db/order-commerce.md` · `docs/api/order-commerce.md` · Cập nhật: `2026-08-30`

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

| ID | Endpoint/case | Expected |
|---|---|---|
| O-API-01 | `GET /cart` buyer/empty | `200`, own cart; empty items hợp lệ. |
| O-API-02 | `POST /cart/items` valid/invalid/other user | `201` hoặc `400/403`; không trust user_id body. |
| O-API-03 | `PUT/DELETE /cart/items/{id}` | Update/delete own; wrong scope not found/forbidden. |
| O-API-04 | `GET /vouchers/validate` valid/expired/wrong scope | `200 valid=false` hoặc correct voucher error; preview không redeem. |
| O-API-04a | `GET /vouchers` không JWT | `200`, danh sách voucher `ACTIVE`; `eligible=null`; không lộ voucher `DRAFT`/`ARCHIVED`. |
| O-API-04b | `GET /vouchers?cart_id=` có JWT | `eligible`/`discount_amount` tính đúng theo giỏ; voucher hết lượt trả `eligible=false` + `ineligible_reason=USAGE_LIMIT_REACHED`. |
| O-API-04c | `GET /vouchers?shop_id=` và `?product_id=` | Chỉ trả voucher áp được cho shop/sản phẩm đó; không rò voucher shop khác. |
| O-API-04d | `GET /vouchers` gọi lặp | Không tăng `used_count`, không tạo redemption — endpoint read-only. |
| O-API-04e | `GET /vouchers/redemptions` | Chỉ redemption của chính buyer; `meta.total_saved` là tổng toàn bộ filter, loại order đã huỷ/hoàn. |
| O-API-05 | `GET /checkout/shipping-fee` valid/address other user/dependency down | `200`, `400 ORDER_ADDRESS_INVALID`, hoặc `503`. |
| O-API-06 | `POST /checkout/preview` valid | Total snapshot/warnings; không call reserve/charge. |
| O-API-07 | `POST /checkout` valid VNPAY multi-shop | `201`, group/order(s), reserve IDs, pending payment. |
| O-API-08 | `POST /checkout` stock fail/price changed | `409`, no partial order/reservation/voucher. |
| O-API-09 | `POST /checkout` same idempotency key same payload | Same response/no duplicate. |
| O-API-10 | Same key different payload | `409 ORDER_IDEMPOTENCY_CONFLICT`. |
| O-API-11 | `GET /orders`, detail/status | Buyer only; snapshot immutable; no other buyer data. |
| O-API-12 | `PATCH /orders/{id}/cancel` before shipping | `200 CANCELLED`, Inventory release command/event. |
| O-API-13 | Cancel shipped/delivered/wrong version | `409 ORDER_CANCEL_NOT_ALLOWED/ORDER_STATE_INVALID`. |
| O-API-14 | Invoice pending/issued | `409 ORDER_INVOICE_NOT_READY` or issued metadata. |
| O-API-15 | Seller list/detail own/other shop | Correct shop scope; no cross-tenant leak. |
| O-API-16 | Seller fulfill accept/ship invalid transition | Correct state/tracking validation; Shipment dependency not bypassed. |
| O-API-17 | Seller cancel allowed/disallowed | Policy/compensation/idempotency correct. |
| O-API-18 | Seller voucher CRUD platform attempt | Shop voucher works; PLATFORM creation denied. |
| O-API-19 | `POST/PUT/DELETE /admin/vouchers` với/không `VOUCHER_MANAGE` | Có quyền → tạo/sửa platform voucher; không quyền → `403`. `scope` bị ép `PLATFORM`. |
| O-API-20 | Body voucher gửi kèm field audience/segment | Field lạ bị bỏ qua/`400`; v1 không hỗ trợ targeting. |
| O-API-21 | `POST /checkout` `payment_method=COD` | `201` với `status=CONFIRMED`, `payment_status=PENDING_COD`, `payment_next_action.type=NONE`, `reservation_expires_at=null`. **Không** được trả `PENDING_PAYMENT`. |
| O-API-22 | `GET /orders/{id}/status` đơn COD vừa đặt | Timeline bắt đầu bằng `CONFIRMED`, **không** có entry `PENDING_PAYMENT`. |

### 3.1 Event/saga/reliability

| ID | Case | Expected |
|---|---|---|
| O-INT-01 | Payment success event duplicate/out-of-order | One transition/commit; old event ignored. |
| O-INT-02 | Inventory reserve success then MySQL failure | Compensation/reconcile; no orphan untracked order. |
| O-INT-03 | Order cancel duplicate | One release/redemption release. |
| O-INT-04 | Shipment delivered event | Order `DELIVERED` only valid transition. |
| O-INT-05 | Kafka unavailable | Local transaction + outbox; retry/DLQ, not false success to consumers. |
| O-INT-06 | MySQL deadlock/Redis down | Bounded retry/failure; no duplicate checkout. |

### 3.1a Golden test phép tính tiền *(bắt buộc — đây là nghiệp vụ lõi)*

Mọi case dưới đây là **số cụ thể**, không phải mô tả. Cài đặt sai một quy tắc làm tròn là lệch đối soát. Công thức ở `docs/lld/order-commerce.md` §3.2a.

| ID | Giỏ hàng + voucher | Kỳ vọng |
|---|---|---|
| O-CALC-01 | Shop A 800k, Shop B 400k, ship 30k/shop · `APPLE300K` (SHOP A, FIXED 300k) + `TACA200K` (PLATFORM, FIXED 200k) + `FREESHIPMAX` (FREESHIP, FIXED 60k) | A: `grand_total=388.889` · B: `311.111` · tổng `700.000`. Phân bổ platform A=111.111, B=88.889 |
| O-CALC-02 | Kiểm bất biến phân bổ | `Σ platform_alloc_s == platform_discount` **tuyệt đối**, không sai 1 VND, với 100 tổ hợp giỏ ngẫu nhiên |
| O-CALC-03 | Phần dư chia không hết (3 shop, platform 100k, base chia lẻ) | Phần dư 1–2 VND rơi vào shop có phần thập phân lớn nhất; bằng nhau thì `shop_id` nhỏ hơn. Chạy 2 lần ra **cùng** kết quả |
| O-CALC-04 | `PERCENT` 8% có `max_discount_amount=1.000.000`, đơn 50 triệu | Giảm đúng `1.000.000`, **không** phải 4.000.000 |
| O-CALC-05 | `PERCENT` 8% không trần (`max_discount_amount=null`), đơn 50 triệu | Giảm `4.000.000` |
| O-CALC-06 | `FIXED` 300k áp lên shop có subtotal 200k | Giảm `200.000`, không âm; `grand_total_s >= 0` |
| O-CALC-07 | `FREESHIP` 100k, tổng ship 60k | Giảm `60.000`, không phải 100k; ship không âm |
| O-CALC-08 | Thứ tự áp dụng | Platform `min_order` kiểm trên `Σ(subtotal_s − shop_discount_s)`, **không** trên subtotal gốc. Voucher platform `min_order=1tr` + giỏ 1.2tr + shop voucher 300k → base 900k → **không** đủ điều kiện |
| O-CALC-09 | Thuế 10%, tiền hàng sau giảm 1.062.000 | `tax = 96.545` (làm tròn nửa lên); `grand_total` **không đổi** khi thuế thay đổi |
| O-CALC-10 | Hai item khác thuế suất (10% và 5%) trong cùng đơn | Thuế tính **theo từng dòng** rồi cộng, không dùng thuế suất trung bình |
| O-CALC-11 | `tax_rate_bps` thiếu trong response Product | Coi như `0`, ghi log cảnh báo, **không** chặn checkout |
| O-CALC-12 | Preview vs place cùng giỏ, cùng voucher, giá không đổi | `grand_total` khớp **chính xác** giữa hai endpoint |
| O-CALC-13 | Gửi 2 mã cùng `scope=PLATFORM` | `400 ORDER_VOUCHER_SLOT_CONFLICT`; không tạo đơn |
| O-CALC-14 | Gửi mã `SHOP` của shop không có trong giỏ | Bỏ qua mã đó, trả `warning VOUCHER_NOT_APPLICABLE`, **vẫn** đặt hàng thành công |
| O-CALC-15 | Giỏ 3 shop, 3 mã `SHOP` khác nhau | Mỗi mã chỉ ăn vào shop của nó; không rò sang shop khác |

### 3.1b Huỷ một phần — tính lại voucher

| ID | Case | Kỳ vọng |
|---|---|---|
| O-VCAN-01 | Nhóm 2 đơn, huỷ 1 đơn, phần còn lại **vẫn đủ** `min_order` | Voucher giữ nguyên; platform discount phân bổ lại toàn bộ cho đơn còn lại; ghi audit `grand_total` trước/sau |
| O-VCAN-02 | Huỷ 1 đơn, phần còn lại **không đủ** `min_order`, đơn COD chưa thu tiền | Voucher bị gỡ, `used_count` nhả về, `grand_total` mới cao hơn, buyer trả theo số mới |
| O-VCAN-03 | Như trên nhưng VNPAY **đã capture** | **Không** truy thu buyer; ghi `VOUCHER_ADJUSTMENT_ABSORBED` vào audit |
| O-VCAN-04 | Huỷ toàn bộ nhóm | Nhả tất cả voucher; `used_count` về đúng giá trị trước checkout |
| O-VCAN-05 | Huỷ rồi gọi lại cùng idempotency key | Không nhả `used_count` hai lần |

### 3.2 Vòng đời COD xuyên service *(hồi quy — nhóm case này từng thiếu và đã để lọt lỗi deadlock)*

Đơn COD không có bước trả tiền trước, nên mọi giả định "đơn hợp lệ = đã thu tiền" đều sai. Các case dưới đây khoá lại điều đó.

| ID | Case | Expected |
|---|---|---|
| O-COD-01 | Đặt đơn COD, chạy job payment-timeout | Đơn **không** bị huỷ. Job chỉ quét `PENDING_PAYMENT`; đơn COD ở `CONFIRMED` nên nằm ngoài phạm vi quét. |
| O-COD-02 | Đặt đơn COD, chờ quá `INVENTORY_RESERVATION_TTL` (15 phút) | Kho **không** bị nhả. Reservation đã `COMMITTED` ngay tại checkout, không còn `RESERVED` để hết hạn. |
| O-COD-03 | Đơn COD xuất hiện ở hàng đợi seller "Chờ xác nhận" | Có mặt ngay sau checkout. Filter hàng đợi dùng `status=CONFIRMED`, không dùng `payment_status`. |
| O-COD-04 | Vòng đời đầy đủ COD | `CONFIRMED → PROCESSING → SHIPPED → DELIVERED` chạy trọn vẹn mà **không** cần bất kỳ event nào từ Payment-Wallet. |
| O-COD-05 | Sau `DELIVERED`, Payment phát `payment.succeeded` | `payment_status` `PENDING_COD → SUCCESS`; `OrderStatus` **giữ nguyên** `DELIVERED`, không transition thêm. |
| O-COD-06 | Thứ tự event `order.confirmed` vs `order.paid` cho COD | `order.confirmed` phát tại checkout; `order.paid` phát sau `DELIVERED`. Khoảng cách hai event = cả vòng đời giao hàng. |
| O-COD-07 | Notification nhận `order.confirmed` của đơn COD | Email `order-success-v1` + hoá đơn gửi **ngay lúc đặt đơn**, không chờ giao hàng. |
| O-COD-08 | Huỷ đơn COD trước khi giao | Kho hoàn lại, **không** phát sinh refund (chưa từng thu tiền), `payment_status → FAILED/CANCELLED` theo policy. |
| O-COD-09 | `shipment.failed` cho đơn COD | Không post ledger, không allocation, không payout cho seller. |
| O-COD-10 | So sánh hai nhánh cùng giỏ hàng | VNPAY và COD ra cùng `grand_total`; chỉ khác `status`, `payment_status` và thời điểm capture. |

## 4. Unit test

| Module | Cases |
|---|---|
| `CartService` | TTL, quantity bounds, user scope, no reservation semantics. |
| `CheckoutCalculator` | VND integer totals, shipping, tax, discount, overflow/reconciliation. **Cấm float ở mọi bước trung gian.** Thứ tự shop→platform→freeship; phân bổ phần dư lớn nhất; bất biến `Σ alloc_s == tổng discount`. |
| `VoucherScopeResolver` | Gom mã theo `scope`, phát hiện trùng scope, bỏ qua mã không áp được mà không chặn đơn. |
| `TaxExtractor` | Tách VAT từ giá đã gồm thuế theo `tax_rate_bps` từng dòng; làm tròn nửa lên; thuế không đổi `grand_total`. |
| `VoucherService` | Scope/time/usage, percent 1–100, fixed value, idempotent redemption. |
| `OrderStateMachine` | All valid/invalid transitions and cancel-before-shipping rule. **Hai điểm vào khác nhau**: `→ PENDING_PAYMENT` (trả trước) và `→ CONFIRMED` (COD). Khẳng định `PAID` không còn là trạng thái hợp lệ, và không transition nào ra khỏi `PENDING_PAYMENT` cho đơn COD. |
| `InventoryClient` | Timeout/error mapping, idempotency, reserve/commit/release command contract. |
| `OrderMapper` | Product/address/shop/price immutable snapshots, redaction. |
| `Observability` | Required JSON fields, trace propagation, no token/payment/address/order raw logs. |

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
| 3 | ~~Voucher stacking/tax chưa đầy đủ~~ → **Đã đóng**: 3 slot + thuế theo danh mục đã chốt, golden cases ở §3.1a/§3.1b. Còn lại: carrier quote policy thật (đang dùng mock). | Ảnh hưởng độ chính xác phí ship. | Shipment owner |
| 3b | Voucher v1 chỉ theo mã (không audience/targeting); platform voucher qua `/admin/vouchers`. | Nếu bật targeting cần thêm test audience/segment. | Product owner |
| 4 | PDF invoice provider chưa chốt. | Tạm test metadata/async state, chưa visual PDF. | Finance/DevOps |
