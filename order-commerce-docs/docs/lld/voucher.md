# LLD — Order-Commerce · Voucher

> Thuộc service Order-Commerce — tổng quan, quy ước chung và mã lỗi đầy đủ ở `docs/lld/order-commerce.md` · Cập nhật: `2026-10-31`
> API: `docs/api/voucher.md` · DB: `docs/db/voucher.md` · Test: `docs/test/voucher.md`

## 1. Phạm vi

| Mục | Nội dung |
|---|---|
| Trách nhiệm | Voucher discovery/validate cho buyer, lịch sử redemption, seller CRUD shop voucher, admin CRUD platform/freeship voucher, redemption tại checkout, tính lại khi huỷ một phần. |
| Module | `com.taca.order.voucher` — `VoucherService`. |
| Ràng buộc | Atomic usage limit; unique redemption theo voucher+user/order policy; redemption chỉ xảy ra trong `POST /checkout` (`checkout.md` §2.3). |
| Công thức tiền | Số tiền giảm và phân bổ theo shop tính ở `checkout.md` §2.2 — file này chỉ định nghĩa quy tắc voucher. |

## 2. Luồng xử lý

### 2.1 Discovery, validate và phạm vi v1

- Voucher scope `PLATFORM`, `SHOP` hoặc `FREESHIP`; một order item/shop phải kiểm scope trước split.
- **Discovery vs validate là hai đường khác nhau.** `GET /vouchers` liệt kê voucher buyer đang dùng được (HLD #23 "choose ... voucher"), `GET /vouchers/validate` kiểm một mã cụ thể buyer đã biết. Cả hai chỉ **đọc**, không giữ chỗ, không tăng `used_count`; redemption duy nhất xảy ra trong `POST /checkout`. Cùng dùng chung bộ mã lý do không hợp lệ để UI không phải map hai bảng.
- `GET /vouchers` cho phép gọi **không cần đăng nhập** (PDP/Shop voucher strip là màn public); khi không có JWT thì `eligible` trả `null`, chỉ hiển thị điều kiện. Có JWT + `cart_id` mới tính `eligible`/`discount_amount`.
- `GET /vouchers/redemptions` đọc từ `voucher_redemptions` join order để loại redemption của order đã huỷ/hoàn khỏi `total_saved`.
- Voucher v1 chỉ áp theo **mã** (`code`) + điều kiện `min_order`/thời gian/`usage_limit`; ràng buộc `PERCENT` 1–100, `FIXED` VND, `FREESHIP` VND hoặc phần trăm phí ship. Không có audience/targeting (selected users, segment, VIP, followers, import user IDs). Các màn Penpot Seller "Voucher audience / Selected users / Tab followers / Import user IDs / Create targeted voucher" là **ngoài phạm vi v1**; Penpot phần này cần đơn giản hoá về form tạo voucher theo mã. Nếu bật targeting sau này cần bảng `voucher_audiences` + quan hệ follow shop (chưa có trong v1).

### 2.2 Quy tắc cộng dồn voucher — mô hình 3 slot

Một lần checkout áp được **tối đa 3 loại voucher cùng lúc**, mỗi loại một mã:

| `scope` | Số mã | Áp lên | Ai tạo |
|---|---|---|---|
| `PLATFORM` | 1 cho cả giỏ | Tiền hàng, sau khi trừ shop voucher | Admin (`/admin/vouchers`) |
| `SHOP` | 1 **mỗi shop** | Tiền hàng của đúng shop đó | Seller (`/seller/vouchers`) |
| `FREESHIP` | 1 cho cả giỏ | Chỉ phí ship | Admin |

Ràng buộc:

- **Không** cộng dồn hai mã cùng `scope`. Gửi 2 mã `PLATFORM` → `400 ORDER_VOUCHER_SLOT_CONFLICT`.
- Giỏ 3 shop có thể dùng 3 mã `SHOP` khác nhau — mỗi mã chỉ ăn vào shop của nó.
- Mã `SHOP` gửi kèm mà giỏ không có shop tương ứng → mã đó bị bỏ qua, trả `warning VOUCHER_NOT_APPLICABLE`, **không** chặn đặt hàng.
- Thứ tự áp dụng cố định (shop → platform → freeship) — xem `checkout.md` §2.2. Đổi thứ tự sẽ ra số khác vì `min_order` của platform tính trên phần đã trừ shop voucher.
- Mỗi mã sinh **một** bản ghi `voucher_redemptions` riêng; một order có nhiều redemption là bình thường.
- Platform voucher (`scope=PLATFORM`) và freeship voucher (`scope=FREESHIP`) do admin tạo/sửa qua `order-commerce` (endpoint admin); seller chỉ tạo `scope=SHOP`.
- Usage increment/redemption idempotent; không giảm `used_count` nếu order không tạo được hoặc đã release theo policy.

### 2.3 Huỷ một phần nhóm đơn — tính lại voucher

Khi buyer huỷ **một đơn con** trong nhóm nhiều shop, voucher `PLATFORM`/`FREESHIP` đã phân bổ cho cả nhóm phải được tính lại; nếu không, phần giảm giá của đơn bị huỷ sẽ bốc hơi hoặc bị tính hai lần.

Quy tắc: **giữ voucher, tính lại trên phần còn lại.**

1. Bỏ đơn bị huỷ ra khỏi giỏ, tính lại `base_platform` và `base_ship` trên các đơn còn lại.
2. Kiểm lại `min_order` của từng voucher trên base mới.
3. **Còn đủ điều kiện** → phân bổ lại theo `checkout.md` §2.2. Đơn còn lại có thể được giảm **nhiều hơn** trước (vì cùng số tiền giảm chia cho ít đơn hơn); phần chênh này ghi nhận là điều chỉnh giảm.
4. **Không còn đủ `min_order`** → voucher bị gỡ khỏi nhóm, `used_count` được nhả về, các đơn còn lại mất phần giảm giá đó. Chênh lệch buyer phải trả thêm:
   - Đơn **chưa thu tiền** (COD, hoặc VNPAY chưa capture): cập nhật `grand_total` mới, buyer trả theo số mới.
   - Đơn **đã thu tiền** (VNPAY đã capture): **không** truy thu buyer. Phần chênh do sàn chịu, ghi nhận `VOUCHER_ADJUSTMENT_ABSORBED` trong audit. Truy thu sau khi đã trừ tiền là trải nghiệm không chấp nhận được và dễ thành khiếu nại.
5. Huỷ **toàn bộ** nhóm → nhả tất cả voucher, `used_count` trả về nguyên trạng.

Mọi lần tính lại đều ghi audit kèm `grand_total` trước/sau để đối soát truy được.

## 3. Hằng số & cấu hình

| Tên | Giá trị baseline | Ghi chú |
|---|---:|---|
| `VOUCHER_CODE_MAX_LENGTH` | 32 | Normalize uppercase/trim trước khi so. |
| `VOUCHER_EXPIRING_SOON_WINDOW` | 7 ngày | Filter `status=EXPIRING_SOON` của `GET /vouchers`. |
| `PERCENT` value | 1–100 | `max_discount_amount` `null` = không trần. |

## 4. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `VoucherStatus` | `DRAFT`, `ACTIVE`, `INACTIVE`, `EXPIRED`, `ARCHIVED`. |
| `DiscountType` | `PERCENT`, `FIXED`, `FREESHIP`. `FREESHIP` chỉ áp lên phí ship; `PERCENT`/`FIXED` chỉ áp lên tiền hàng. |
| `VoucherScope` | `PLATFORM`, `SHOP`, `FREESHIP`. Vừa là phạm vi áp dụng vừa là **slot cộng dồn**: mỗi checkout tối đa 1 mã mỗi scope (`SHOP` là 1 mã **mỗi shop**). |

## 5. Event phát ra

| Topic | Event | Payload chính |
|---|---|---|
| `voucher.events.v1` | `voucher.redeemed/released` | voucher/order/user/scope |

## 6. Mã lỗi

`ORDER_VOUCHER_INVALID`, `ORDER_VOUCHER_SLOT_CONFLICT`, `ORDER_VOUCHER_LIMIT_REACHED`, `ORDER_VOUCHER_CODE_EXISTS`, `ORDER_VOUCHER_CODE_LOCKED`, `ORDER_FORBIDDEN` — định nghĩa ở `order-commerce.md` §7.

## 7. Giả định & câu hỏi mở

Xem `order-commerce.md` §8 mục 9 (voucher v1 chỉ theo mã, không targeting) và 10 (platform voucher thuộc `order-commerce`).
