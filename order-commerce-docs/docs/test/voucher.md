# Test Plan — Order-Commerce · Voucher

> Thuộc service Order-Commerce — chuẩn bị, contract/security chung và tiêu chí phát hành ở `docs/test/order-commerce.md` · Cập nhật: `2026-10-31`
> LLD: `docs/lld/voucher.md` · API: `docs/api/voucher.md` · DB: `docs/db/voucher.md`

## 1. Chuẩn bị

Theo `docs/test/order-commerce.md` §1.

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Voucher | Platform/shop, expired, limit, wrong scope | Discount/validation đúng; redemption atomic. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| O-API-04 | `GET /vouchers/validate` valid/expired/wrong scope | `200 valid=false` hoặc correct voucher error; preview không redeem. |
| O-API-04a | `GET /vouchers` không JWT | `200`, danh sách voucher `ACTIVE`; `eligible=null`; không lộ voucher `DRAFT`/`ARCHIVED`. |
| O-API-04b | `GET /vouchers?cart_id=` có JWT | `eligible`/`discount_amount` tính đúng theo giỏ; voucher hết lượt trả `eligible=false` + `ineligible_reason=USAGE_LIMIT_REACHED`. |
| O-API-04c | `GET /vouchers?shop_id=` và `?product_id=` | Chỉ trả voucher áp được cho shop/sản phẩm đó; không rò voucher shop khác. |
| O-API-04d | `GET /vouchers` gọi lặp | Không tăng `used_count`, không tạo redemption — endpoint read-only. |
| O-API-04e | `GET /vouchers/redemptions` | Chỉ redemption của chính buyer; `meta.total_saved` là tổng toàn bộ filter, loại order đã huỷ/hoàn. |
| O-API-18 | Seller voucher CRUD platform attempt | Shop voucher works; PLATFORM creation denied. |
| O-API-19 | `POST/PUT/DELETE /admin/vouchers` với/không `VOUCHER_MANAGE` | Có quyền → tạo/sửa platform voucher; không quyền → `403`. `scope` chỉ nhận `PLATFORM`/`FREESHIP`; gửi `SHOP` → `400`. |
| O-API-20 | Body voucher gửi kèm field audience/segment | Field lạ bị bỏ qua/`400`; v1 không hỗ trợ targeting. |

### 3.1 Huỷ một phần — tính lại voucher

| ID | Case | Kỳ vọng |
|---|---|---|
| O-VCAN-01 | Nhóm 2 đơn, huỷ 1 đơn, phần còn lại **vẫn đủ** `min_order` | Voucher giữ nguyên; platform discount phân bổ lại toàn bộ cho đơn còn lại; ghi audit `grand_total` trước/sau |
| O-VCAN-02 | Huỷ 1 đơn, phần còn lại **không đủ** `min_order`, đơn COD chưa thu tiền | Voucher bị gỡ, `used_count` nhả về, `grand_total` mới cao hơn, buyer trả theo số mới |
| O-VCAN-03 | Như trên nhưng VNPAY **đã capture** | **Không** truy thu buyer; ghi `VOUCHER_ADJUSTMENT_ABSORBED` vào audit |
| O-VCAN-04 | Huỷ toàn bộ nhóm | Nhả tất cả voucher; `used_count` về đúng giá trị trước checkout |
| O-VCAN-05 | Huỷ rồi gọi lại cùng idempotency key | Không nhả `used_count` hai lần |

## 4. Unit test

| Module | Cases |
|---|---|
| `VoucherScopeResolver` | Gom mã theo `scope`, phát hiện trùng scope, bỏ qua mã không áp được mà không chặn đơn. |
| `VoucherService` | Scope/time/usage, percent 1–100, fixed value, idempotent redemption. |

## 5. Contract/security/resilience

- Security: voucher abuse (dò mã — mã sai trả `200 valid=false`, không `404`), seller không tạo `PLATFORM`/`FREESHIP`, admin cần `VOUCHER_MANAGE` + 2FA.
- Golden case tính tiền có voucher (`O-CALC-*`) ở `docs/test/checkout.md` §3.2.

## 6. Tiêu chí pass/phát hành

Theo `docs/test/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/test/order-commerce.md` §7.
