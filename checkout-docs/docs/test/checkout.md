# Test Plan — Checkout Service

> Nguồn: `docs/lld/checkout.md` · `docs/api/checkout.md` · `docs/db/checkout.md` · Code: chưa có repo (`ecommerce-taca/checkout-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc, Testcontainers (MySQL 8.4, Kafka), WireMock cho service phụ thuộc

Tài liệu này là **tổng quan** test của Checkout Service và case checkout. Case voucher ở `docs/test/voucher.md`.

| Tài liệu | Case ID | ID cũ (Order-Commerce) |
|---|---|---|
| `checkout.md` | `CO-API-01..09`, `CO-INT-01..04`, `CO-CALC-01..15` | `O-API-05..10`, `O-API-21`, `O-INT-02`, `O-INT-06`, `O-CALC-01..15` |
| `voucher.md` | `VCH-API-01..09`, `VCH-RECALC-01..06`, `VCH-HOLD-01..02` | `O-API-04..04e`, `O-API-18..20`, `O-VCAN-01..05` |

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers. |
| DB | MySQL 8.4 InnoDB `checkoutdb`; Kafka test broker. |
| Dependency doubles | `cart-service` (giỏ, `checked-out`), `order-service` (`POST /internal/orders` thành công/4xx/timeout), Product (giá, `tax_rate_bps`), Inventory (reserve/release), Shipment quote, Auth address. |
| Data | Seed `docs/db/checkout.md` §6.2. |
| Actors | Buyer owner/khác, seller owner/khác shop, admin có/không `VOUCHER_MANAGE`, internal caller có/không token. |
| Observability | Assert field log, `X-Request-ID`, W3C context, Kafka correlation. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Checkout | Preview, giá đổi, hết hàng, giỏ nhiều shop | Warnings rõ; preview không tạo gì; place split đúng. |
| Payment method | VNPAY và COD | Response khác nhau đúng như `docs/api/checkout.md` §3.3. |
| Giỏ sau đặt | Đặt một phần giỏ | Item đã đặt biến khỏi giỏ (cart-service). |
| MFE states | Loading, empty, error, timeout, retry | Retry cùng key không tạo đơn trùng. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| CO-API-01 | `GET /checkout/shipping-fee` valid / address của user khác / Shipment down | `200`; `403 CHECKOUT_FORBIDDEN`; `503 CHECKOUT_DEPENDENCY_UNAVAILABLE`. |
| CO-API-02 | `POST /checkout/preview` valid | Total + warnings; không gọi reserve, không gọi `order-service`, không ghi DB. |
| CO-API-03 | `POST /checkout` VNPAY nhiều shop | `201`, group + đơn con, `PENDING_PAYMENT`, `payment_next_action` từ `order-service`. |
| CO-API-04 | `POST /checkout` hết hàng / giá đổi | `409`; không gọi `order-service`; voucher `RELEASED`; group `FAILED`. |
| CO-API-05 | Cùng key cùng payload | Cùng response; `order-service` chỉ nhận 1 lần tạo (key = `checkout_group_id`). |
| CO-API-06 | Cùng key khác payload | `409 CHECKOUT_IDEMPOTENCY_CONFLICT`. |
| CO-API-07 | `POST /checkout` `payment_method=COD` | `201`, `status=CONFIRMED`, `payment_status=PENDING_COD`, `payment_next_action.type=NONE`, `reservation_expires_at=null`. |
| CO-API-08 | Giỏ không còn item hợp lệ (`missing_item_ids` hết) | `409 CHECKOUT_CART_EMPTY`. |
| CO-API-09 | `POST /internal/checkout-groups/{id}/recalculate` thiếu token / group không tồn tại | `401`; `404 CHECKOUT_GROUP_NOT_FOUND`. |

### 3.1 Saga/reliability

| ID | Case | Expected |
|---|---|---|
| CO-INT-01 | Reserve thành công, `order-service` trả 4xx | Release reservation, nhả voucher, group `FAILED`, không đơn mồ côi. |
| CO-INT-02 | `order-service` timeout, đơn thực ra đã tạo | Group `RECONCILING`; job thấy đơn qua `GET /internal/orders?checkout_group_id=` → `ORDERS_CREATED`, redemption `REDEEMED`. |
| CO-INT-03 | `order-service` timeout, đơn không tạo, quá TTL | Release reservation + voucher, group `FAILED`. |
| CO-INT-04 | `cart-service` down sau khi tạo đơn | Response `201` bình thường; job `checked-out` retry tới khi xong. |

### 3.2 Golden test phép tính tiền *(bắt buộc — đây là nghiệp vụ lõi)*

Mọi case dưới đây là **số cụ thể**, không phải mô tả. Cài đặt sai một quy tắc làm tròn là lệch đối soát. Công thức ở `docs/lld/checkout.md` §3.2.

| ID | Giỏ hàng + voucher | Kỳ vọng |
|---|---|---|
| CO-CALC-01 | Shop A 800k, Shop B 400k, ship 30k/shop · `APPLE300K` (SHOP A, FIXED 300k) + `TACA200K` (PLATFORM, FIXED 200k) + `FREESHIPMAX` (scope `FREESHIP`, `discount_type=FREESHIP` 60k) | A: `grand_total=388.889` · B: `311.111` · tổng `700.000`. Phân bổ platform A=111.111, B=88.889 |
| CO-CALC-02 | Kiểm bất biến phân bổ | `Σ platform_alloc_s == platform_discount` **tuyệt đối**, không sai 1 VND, với 100 tổ hợp giỏ ngẫu nhiên |
| CO-CALC-03 | Phần dư chia không hết (3 shop, platform 100k, base chia lẻ) | Phần dư 1–2 VND rơi vào shop có phần thập phân lớn nhất; bằng nhau thì `shop_id` nhỏ hơn. Chạy 2 lần ra **cùng** kết quả |
| CO-CALC-04 | `PERCENT` 8% có `max_discount_amount=1.000.000`, đơn 50 triệu | Giảm đúng `1.000.000`, **không** phải 4.000.000 |
| CO-CALC-05 | `PERCENT` 8% không trần (`max_discount_amount=null`), đơn 50 triệu | Giảm `4.000.000` |
| CO-CALC-06 | `FIXED` 300k áp lên shop có subtotal 200k | Giảm `200.000`, không âm; `grand_total_s >= 0` |
| CO-CALC-07 | `FREESHIP` 100k, tổng ship 60k | Giảm `60.000`, không phải 100k; ship không âm |
| CO-CALC-08 | Thứ tự áp dụng | Platform `min_order` kiểm trên `Σ(subtotal_s − shop_discount_s)`, **không** trên subtotal gốc. Voucher platform `min_order=1tr` + giỏ 1.2tr + shop voucher 300k → base 900k → **không** đủ điều kiện |
| CO-CALC-09 | Thuế 10%, tiền hàng sau giảm 1.062.000 | `tax = 96.545` (làm tròn nửa lên); `grand_total` **không đổi** khi thuế thay đổi |
| CO-CALC-10 | Hai item khác thuế suất (10% và 5%) trong cùng đơn | Thuế tính **theo từng dòng** rồi cộng, không dùng thuế suất trung bình |
| CO-CALC-11 | `tax_rate_bps` thiếu trong response Product | Coi như `0`, ghi log cảnh báo, **không** chặn checkout |
| CO-CALC-12 | Preview vs place cùng giỏ, cùng voucher, giá không đổi | `grand_total` khớp **chính xác** giữa hai endpoint |
| CO-CALC-13 | Gửi 2 mã cùng `scope=PLATFORM` | `400 CHECKOUT_VOUCHER_SLOT_CONFLICT`; không tạo đơn |
| CO-CALC-14 | Gửi mã `SHOP` của shop không có trong giỏ | Bỏ qua mã đó, trả `warning VOUCHER_NOT_APPLICABLE`, **vẫn** đặt hàng thành công |
| CO-CALC-15 | Giỏ 3 shop, 3 mã `SHOP` khác nhau | Mỗi mã chỉ ăn vào shop của nó; không rò sang shop khác |

## 4. Unit test

⏳ Repo chưa có test class; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `CheckoutCalculator` | VND integer, shipping, tax, discount, overflow. **Cấm float.** Thứ tự shop→platform→freeship; phân bổ phần dư lớn nhất; `Σ alloc_s == tổng discount`; trần `goods_s ≥ 1`. |
| `TaxExtractor` | Tách VAT theo `tax_rate_bps` từng dòng; làm tròn nửa lên; không đổi `grand_total`. |
| `CheckoutSaga` | Mỗi bước compensation; idempotent; trạng thái group đúng. |
| `InventoryClient`, `OrderClient`, `CartClient` | Timeout/error mapping, idempotency header. |
| `Observability` | Field log bắt buộc, không log token/address/payload. |

## 5. Contract/security/resilience

- Contract Checkout→Product: SKU active, giá integer VND, `tax_rate_bps`.
- Contract Checkout→Inventory: reserve all-or-nothing, TTL, release idempotent.
- Contract Checkout→Order: body `POST /internal/orders` khớp `order-docs/docs/api/order.md` §3.7; `order-service` không tính lại tiền.
- Contract Checkout→Cart: `cart-docs/docs/api/cart.md` §3.5–3.6.
- Security: IDOR buyer/seller, giả `shop_id`/`user_id`, replay, dò mã voucher, token nội bộ.
- Load: checkout đồng thời cùng SKU/cùng voucher không vượt stock hay `usage_limit`.

## 6. Tiêu chí pass/phát hành

Không đơn/reservation/lượt voucher trùng dưới retry hoặc đồng thời; mọi golden `CO-CALC-*` khớp từng đồng; saga compensation pass; log redaction pass.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Reserve trước, tạo đơn qua `order-service` sau; Payment do `order-service`. | Đổi flow thì đổi test saga. | Order/Payment/Inventory |
| 2 | Carrier quote thật chưa có (đang dùng mock). | Ảnh hưởng độ chính xác phí ship. | Shipment owner |
