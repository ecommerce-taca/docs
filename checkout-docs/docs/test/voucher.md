# Test Plan — Checkout Service · Voucher

> Nguồn: `docs/lld/voucher.md` · `docs/api/voucher.md` · `docs/db/voucher.md` · Code: chưa có repo (`ecommerce-taca/checkout-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc, Testcontainers (MySQL 8.4, Kafka)

Phần **Voucher** của Checkout Service. Chuẩn bị, tiêu chí phát hành và giả định chung ở `docs/test/checkout.md`.

## 1. Chuẩn bị

Theo `docs/test/checkout.md` §1.

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Voucher | Platform/shop/freeship, hết hạn, hết lượt, sai scope | Giảm giá/validate đúng; redemption atomic. |
| Huỷ một phần | Huỷ 1 đơn trong nhóm 2 shop | Đơn còn lại hiển thị số tiền mới đúng `docs/lld/voucher.md` §3.3. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| VCH-API-01 | `GET /vouchers/validate` valid/expired/wrong scope | `200 valid=false` + `reason`; không giữ lượt. |
| VCH-API-02 | `GET /vouchers` không JWT | `200`, chỉ `ACTIVE`; `eligible=null`; không lộ `DRAFT`/`ARCHIVED`. |
| VCH-API-03 | `GET /vouchers?cart_id=` có JWT | `eligible`/`discount_amount` đúng theo giỏ (đọc qua `cart-service`); hết lượt → `USAGE_LIMIT_REACHED`. |
| VCH-API-04 | `GET /vouchers?shop_id=` / `?product_id=` | Chỉ voucher áp được; không rò voucher shop khác. |
| VCH-API-05 | `GET /vouchers` gọi lặp | Không tăng `used_count`, không tạo redemption. |
| VCH-API-06 | `GET /vouchers/redemptions` | Chỉ của buyer; `total_saved` tính trên toàn filter, loại `RELEASED`. |
| VCH-API-07 | Seller CRUD; seller thử tạo `PLATFORM` | Shop voucher OK; `scope` luôn `SHOP`; trùng code → `409 CHECKOUT_VOUCHER_CODE_EXISTS`; đổi code khi `used_count>0` → `409 CHECKOUT_VOUCHER_CODE_LOCKED`. |
| VCH-API-08 | `POST/PUT/DELETE /admin/vouchers` có/không `VOUCHER_MANAGE`, có/không step-up | Có quyền → OK; không → `403 CHECKOUT_FORBIDDEN`. `scope=SHOP` → `400`. |
| VCH-API-09 | Body có field audience/segment | Bị bỏ qua hoặc `400`; v1 không targeting. |
| VCH-API-10 | `validate?code=SAVE10` khi 2 shop trong giỏ cùng có `SAVE10` | `valid=false`, `reason=AMBIGUOUS_CODE`, `candidates[]` 2 phần tử; thêm `shop_id` → đúng 1 voucher kèm `voucher_id`. `POST /checkout` với `voucher_ids` của cả hai → mỗi mã áp đúng shop của nó. |
| VCH-API-11 | `GET /vouchers?status=EXPIRED` | Chỉ voucher hết hạn ≤30 ngày; `eligible=false`, `ineligible_reason=EXPIRED`; không lộ `DRAFT`/`ARCHIVED`. |
| VCH-API-12 | Tạo voucher `scope=FREESHIP` + `PERCENT` 50%, `max_discount_amount` 20k, ship 60k | Tạo OK; checkout giảm ship `20.000`. `scope=SHOP` + `FREESHIP` → `400 CHECKOUT_VOUCHER_INVALID`. |

### 3.1 Vòng đời lượt dùng và tính lại khi huỷ

| ID | Case | Kỳ vọng |
|---|---|---|
| VCH-RECALC-01 | Nhóm 2 đơn, huỷ 1, phần còn lại **vẫn đủ** `min_order` | Voucher giữ; đơn còn lại **giữ nguyên** số tiền (không phân bổ lại); redemption `discount_amount` trừ phần của đơn huỷ; `absorbed_amount = 0`, `adjustment = NONE`. |
| VCH-RECALC-02 | Huỷ 1 đơn, phần còn lại **không đủ** `min_order`, đơn còn lại COD | Ví dụ `docs/api/checkout.md` §3.4: đơn còn lại giữ `grand_total` 388.889; `absorbed_amount` 111.111; `VOUCHER_ADJUSTMENT_ABSORBED`; redemption `TACA200K` 200.000 → 111.111, `FREESHIPMAX` 60.000 → 30.000; `used_count` không đổi. |
| VCH-RECALC-03 | Như trên nhưng `payment_captured=true` (VNPAY) | Kết quả giống hệt VCH-RECALC-02 (cờ chỉ ghi audit). |
| VCH-RECALC-04 | Huỷ cả nhóm | Mọi voucher `RELEASED`; `used_count` về giá trị trước checkout. |
| VCH-RECALC-05 | Gọi lại recalculate cùng `Idempotency-Key` | Cùng response, không nhả `used_count` hai lần. |
| VCH-RECALC-06 | Đơn bị huỷ có mã `SHOP` | Mã đó `RELEASED`, mã của shop khác không đổi. |
| VCH-RECALC-07 | Đơn còn lại có 2 dòng thuế suất khác nhau (10% và 5%) | `items[]` trả đủ 2 dòng với `discount_allocated`/`tax_amount` **không đổi** so với lúc place (đọc từ `checkout_group_items`); `Σ items[].tax_amount = amounts.tax`. |
| VCH-RECALC-08 | COD: so `cod_amount` Shipment, `amount` Payment và `grand_total` sau recalculate | Cả ba bằng nhau (388.889 trong ví dụ §3.4); Payment capture không lệch khi `shipment.delivered`. |
| VCH-HOLD-01 | 2 buyer đồng thời dùng voucher còn 1 lượt | Đúng 1 người `HELD`; người kia `409 CHECKOUT_VOUCHER_LIMIT_REACHED`. |
| VCH-HOLD-02 | Reserve kho thất bại sau khi giữ lượt | Redemption `RELEASED`, `used_count` trả lại. |

## 4. Unit test

⏳ Repo chưa có test class; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `VoucherScopeResolver` | Gom theo `scope`, phát hiện trùng slot, bỏ qua mã không áp được mà không chặn đơn. |
| `VoucherService` | Scope/time/usage, percent 1–100, fixed, vòng đời `HELD/REDEEMED/RELEASED`, giới hạn mỗi buyer. |
| `RecalculationService` | Các nhánh §3.3, dùng chung `CheckoutCalculator`. |

## 5. Contract/security/resilience

- Security: dò mã (mã sai trả `200 valid=false`, không `404`); seller không tạo `PLATFORM`/`FREESHIP`; admin cần `VOUCHER_MANAGE` + 2FA.
- Contract Order→Checkout recalculate: body/response `docs/api/checkout.md` §3.4 khớp `order-docs/docs/lld/order.md` §3.3.
- Golden tính tiền có voucher (`CO-CALC-*`) ở `docs/test/checkout.md` §3.2.

## 6. Tiêu chí pass/phát hành

Theo `docs/test/checkout.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/test/checkout.md` §7.
