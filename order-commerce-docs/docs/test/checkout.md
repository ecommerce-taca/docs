# Test Plan — Order-Commerce Service · Checkout

> Nguồn: `docs/lld/checkout.md` · `docs/api/checkout.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers (MySQL 8.4, Redis, Kafka)

Phần **Checkout** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/test/order-commerce.md`.

## 1. Chuẩn bị

Theo `docs/test/order-commerce.md` §1.

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Checkout | Preview, price changed, stock unavailable, multi-shop split | Warnings rõ; không tạo order ở preview; place split đúng. |
| Payment | VNPAY pending/success/fail, COD | Payment action đúng; Order không tự giả payment success. |
| MFE states | Loading, empty, error, timeout, retry | Idempotent retry không duplicate order. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| O-API-05 | `GET /checkout/shipping-fee` valid/address other user/dependency down | `200`, `403 ORDER_FORBIDDEN` (address của user khác), hoặc `503 ORDER_DEPENDENCY_UNAVAILABLE`. |
| O-API-06 | `POST /checkout/preview` valid | Total snapshot/warnings; không call reserve/charge. |
| O-API-07 | `POST /checkout` valid VNPAY multi-shop | `201`, group/order(s), reserve IDs, pending payment. |
| O-API-08 | `POST /checkout` stock fail/price changed | `409`, no partial order/reservation/voucher. |
| O-API-09 | `POST /checkout` same idempotency key same payload | Same response/no duplicate. |
| O-API-10 | Same key different payload | `409 ORDER_IDEMPOTENCY_CONFLICT`. |
| O-API-21 | `POST /checkout` `payment_method=COD` | `201` với `status=CONFIRMED`, `payment_status=PENDING_COD`, `payment_next_action.type=NONE`, `reservation_expires_at=null`. **Không** được trả `PENDING_PAYMENT`. |

### 3.1 Event/saga/reliability

| ID | Case | Expected |
|---|---|---|
| O-INT-02 | Inventory reserve success then MySQL failure | Compensation/reconcile; no orphan untracked order. |
| O-INT-06 | MySQL deadlock/Redis down | Bounded retry/failure; no duplicate checkout. |

### 3.2 Golden test phép tính tiền *(bắt buộc — đây là nghiệp vụ lõi)*

Mọi case dưới đây là **số cụ thể**, không phải mô tả. Cài đặt sai một quy tắc làm tròn là lệch đối soát. Công thức ở `docs/lld/checkout.md` §3.2.

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

## 4. Unit test

⏳ Repo chưa có test class nào; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `CheckoutCalculator` | VND integer totals, shipping, tax, discount, overflow/reconciliation. **Cấm float ở mọi bước trung gian.** Thứ tự shop→platform→freeship; phân bổ phần dư lớn nhất; bất biến `Σ alloc_s == tổng discount`. |
| `TaxExtractor` | Tách VAT từ giá đã gồm thuế theo `tax_rate_bps` từng dòng; làm tròn nửa lên; thuế không đổi `grand_total`. |
| `InventoryClient` | Timeout/error mapping, idempotency, reserve/commit/release command contract. |

## 5. Contract/security/resilience

- Contract Product→Order: active SKU, current integer VND price, unavailable/price-changed errors.
- Contract Order→Inventory: all-or-nothing reserve, reservation TTL, commit/release idempotency.
- Load: checkout concurrency cùng SKU không tạo reservation/order dư.

## 6. Tiêu chí pass/phát hành

Theo `docs/test/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/test/order-commerce.md` §7.
