# LLD — Checkout Service · Voucher

> Nguồn: `docs/api/voucher.md` · `docs/db/voucher.md` · `docs/test/voucher.md` · Code: chưa có repo (`ecommerce-taca/checkout-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Tech stack đã chốt: Java 25 · Spring Boot · MySQL 8.4 `checkoutdb` · Kafka outbox

Phần **Voucher** của Checkout Service. Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `checkout.md`.

## 1. Phạm vi

| Mục | Nội dung |
|---|---|
| Trách nhiệm | Voucher discovery/validate cho buyer, lịch sử redemption, seller CRUD shop voucher, admin CRUD platform/freeship voucher, giữ/chốt/nhả lượt dùng trong place checkout, tính lại khi huỷ đơn con. |
| Module | `com.taca.checkout.voucher` — `VoucherService`, `VoucherScopeResolver`. |
| Ràng buộc | Atomic usage limit; redemption chỉ phát sinh trong `POST /checkout` (`checkout.md` §3.3). |
| Công thức tiền | Số tiền giảm và phân bổ theo shop tính ở `checkout.md` §3.2 — file này chỉ định nghĩa quy tắc voucher. |
| Vì sao ở Checkout | Voucher dính chặt công thức tiền và huỷ một phần; giữ chung để hold/redeem nằm trong cùng transaction MySQL với `checkout_groups` (quyết định 2026-11-01). |

## 2. Cấu trúc bên trong

Package `com.taca.checkout.voucher` trong cây module ở `checkout.md` §2. ⏳ Repo chưa có code.

## 3. Luồng xử lý

### 3.1 Discovery, validate và phạm vi v1

- Voucher scope `PLATFORM`, `SHOP` hoặc `FREESHIP`; kiểm scope trước khi split đơn.
- **Discovery vs validate là hai đường khác nhau.** `GET /vouchers` liệt kê voucher buyer đang dùng được (HLD #23 "choose ... voucher"), `GET /vouchers/validate` kiểm một mã cụ thể. Cả hai chỉ **đọc**, không giữ lượt, không tăng `used_count`; cùng bộ mã lý do không hợp lệ.
- `GET /vouchers` gọi được **không cần đăng nhập** (PDP/Shop voucher strip); không có JWT thì `eligible` trả `null`. Có JWT + `cart_id` mới tính `eligible`/`discount_amount` (Checkout đọc giỏ qua `cart-service`).
- `GET /vouchers/redemptions` đọc `voucher_redemptions` của buyer với `status = REDEEMED`; redemption đã `RELEASED` (đơn huỷ/voucher bị gỡ) không tính vào `total_saved`. Không cần đọc dữ liệu của `order-service`.
- Voucher v1 chỉ áp theo **mã** + `min_order`/thời gian/`usage_limit`; `PERCENT` 1–100, `FIXED` VND, `FREESHIP` VND hoặc phần trăm phí ship. Không có audience/targeting; các màn Penpot Seller "Voucher audience / Selected users / Tab followers / Import user IDs / Create targeted voucher" là **ngoài v1**.

### 3.2 Quy tắc cộng dồn voucher — mô hình 3 slot

Một lần checkout áp được **tối đa 3 loại voucher cùng lúc**, mỗi loại một mã:

| `scope` | Số mã | Áp lên | Ai tạo |
|---|---|---|---|
| `PLATFORM` | 1 cho cả giỏ | Tiền hàng, sau khi trừ shop voucher | Admin (`/admin/vouchers`) |
| `SHOP` | 1 **mỗi shop** | Tiền hàng của đúng shop đó | Seller (`/seller/vouchers`) |
| `FREESHIP` | 1 cho cả giỏ | Chỉ phí ship | Admin |

Ràng buộc:

- **Không** cộng dồn hai mã cùng `scope`. Gửi 2 mã `PLATFORM` → `400 CHECKOUT_VOUCHER_SLOT_CONFLICT`.
- Giỏ 3 shop có thể dùng 3 mã `SHOP` khác nhau — mỗi mã chỉ ăn vào shop của nó.
- Mã `SHOP` gửi kèm mà giỏ không có shop tương ứng → mã đó bị bỏ qua, trả `warning VOUCHER_NOT_APPLICABLE`, **không** chặn đặt hàng.
- Thứ tự áp dụng cố định (shop → platform → freeship) — xem `checkout.md` §3.2. Đổi thứ tự sẽ ra số khác vì `min_order` của platform tính trên phần đã trừ shop voucher.
- Mỗi mã sinh **một** bản ghi `voucher_redemptions` cho cả nhóm checkout; mã `SHOP` gắn với đúng một đơn con, mã `PLATFORM`/`FREESHIP` gắn với cả nhóm.
- Platform voucher (`scope=PLATFORM`) và freeship voucher (`scope=FREESHIP`) do admin tạo/sửa qua `/admin/vouchers` của `checkout-service`; seller chỉ tạo `scope=SHOP`.
- Usage increment/redemption idempotent; lượt dùng được giữ (`HELD`) trước khi reserve kho và nhả nếu không tạo được đơn (§3.4).

### 3.3 Huỷ một phần nhóm đơn — tính lại voucher

Khi một **đơn con** trong nhóm nhiều shop bị huỷ (buyer, seller, hoặc hệ thống), `order-service` gọi `POST /internal/checkout-groups/{checkoutGroupId}/recalculate` (`checkout.md` §3.4). Voucher `PLATFORM`/`FREESHIP` đã phân bổ cho cả nhóm phải tính lại; nếu không, phần giảm của đơn bị huỷ sẽ bốc hơi hoặc bị tính hai lần.

Quy tắc: **giữ voucher, tính lại trên phần còn lại.**

1. Mã `SHOP` của đơn bị huỷ được nhả (`RELEASED`, `used_count` trả lại) vì đơn của shop đó không còn.
2. Bỏ đơn bị huỷ ra, tính lại `base_platform` và `base_ship` trên các đơn còn lại.
3. Kiểm lại `min_order` của từng voucher `PLATFORM`/`FREESHIP` trên base mới.
4. **Còn đủ điều kiện** → phân bổ lại theo `checkout.md` §3.2. Đơn còn lại có thể được giảm **nhiều hơn** trước; phần chênh ghi là điều chỉnh giảm.
5. **Không còn đủ `min_order`** → voucher bị gỡ khỏi nhóm, redemption `RELEASED`, `used_count` trả lại, các đơn còn lại mất phần giảm đó. Chênh lệch buyer phải trả thêm:
   - Đơn **chưa thu tiền** (COD, hoặc VNPAY chưa capture): trả `grand_total` mới, buyer trả theo số mới.
   - Đơn **đã thu tiền** (VNPAY đã capture): **không** truy thu buyer. Trả `absorbed_amount`, sàn chịu, ghi `VOUCHER_ADJUSTMENT_ABSORBED` trong `voucher_adjustments`.
6. Huỷ **toàn bộ** nhóm → nhả tất cả voucher, `used_count` trả về nguyên trạng.

`order-service` gửi kèm cờ `payment_captured` của từng đơn còn lại (nó sở hữu payment projection); Checkout không tự suy ra trạng thái tiền. Mọi lần tính lại ghi `voucher_adjustments` kèm `grand_total` trước/sau để đối soát.

### 3.4 Vòng đời lượt dùng (redemption)

```text
POST /checkout bước 5:  HELD     (used_count += 1, cùng transaction với checkout_groups)
  reserve fail / tạo đơn fail / reconcile không thấy đơn → RELEASED (used_count -= 1)
  tạo đơn thành công:   REDEEMED (gắn order_id; mã PLATFORM/FREESHIP gắn đơn đầu tiên của nhóm làm tham chiếu)
recalculate gỡ voucher / huỷ đơn có mã SHOP / huỷ cả nhóm → RELEASED
```

`used_count` chỉ đổi cùng transaction với thay đổi trạng thái redemption, dùng `UPDATE … WHERE used_count < usage_limit` để chặn vượt giới hạn dưới tải đồng thời.

## 4. Hằng số & cấu hình

| Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú |
|---|---:|---|---|
| `VOUCHER_CODE_MAX_LENGTH` | 32 | — | Normalize uppercase/trim trước khi so. |
| `VOUCHER_EXPIRING_SOON_WINDOW` | 7 ngày | — | Filter `status=EXPIRING_SOON` của `GET /vouchers`. |
| `PERCENT` value | 1–100 | — | `max_discount_amount` `null` = không trần. |
| `VOUCHER_PER_USER_LIMIT` | 1 | — | Mỗi buyer dùng mỗi voucher tối đa 1 lần (đếm redemption `HELD`/`REDEEMED`). |

## 5. Enum & trạng thái

| Enum | Giá trị |
|---|---|
| `VoucherStatus` | `DRAFT`, `ACTIVE`, `INACTIVE`, `EXPIRED`, `ARCHIVED`. |
| `DiscountType` | `PERCENT`, `FIXED`, `FREESHIP`. `FREESHIP` chỉ áp lên phí ship; `PERCENT`/`FIXED` chỉ áp lên tiền hàng. |
| `VoucherScope` | `PLATFORM`, `SHOP`, `FREESHIP`. Vừa là phạm vi áp dụng vừa là **slot cộng dồn**: tối đa 1 mã mỗi scope (`SHOP` là 1 mã **mỗi shop**). |
| `RedemptionStatus` | `HELD`, `REDEEMED`, `RELEASED` (§3.4). |
| `AdjustmentType` | `REALLOCATED`, `VOUCHER_REMOVED`, `VOUCHER_ADJUSTMENT_ABSORBED`, `GROUP_CANCELLED`. |

## 6. Event phát ra / lắng nghe

| Topic | Event | Payload chính |
|---|---|---|
| `voucher.events.v1` | `voucher.redeemed` | `voucher_id`, `code`, `scope`, `shop_id?`, `checkout_group_id`, `order_id`, `buyer.user_id`, `discount_amount`, `redeemed_at` |
| `voucher.events.v1` | `voucher.released` | `voucher_id`, `code`, `scope`, `checkout_group_id`, `order_id?`, `buyer.user_id`, `reason` (`AdjustmentType` hoặc `CHECKOUT_FAILED`), `released_at` |

Envelope snake_case theo `CONTRACT-MATRIX.md` §8 (`event_id`, `event_type`, `schema_version`, `occurred_at`, `aggregate_type=VOUCHER`, `aggregate_id`, `payload`). Hiện chưa service nào consume; dùng cho analytics/ngân sách voucher.

## 7. Mã lỗi

⏳ Chưa có code. `CHECKOUT_VOUCHER_INVALID`, `CHECKOUT_VOUCHER_SLOT_CONFLICT`, `CHECKOUT_VOUCHER_LIMIT_REACHED`, `CHECKOUT_VOUCHER_CODE_EXISTS`, `CHECKOUT_VOUCHER_CODE_LOCKED`, `CHECKOUT_FORBIDDEN` — định nghĩa ở `checkout.md` §7.

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Voucher v1 chỉ theo mã + scope `PLATFORM`/`SHOP`/`FREESHIP`; **không** audience/targeting. | Bật targeting cần bảng `voucher_audiences` + quan hệ follow shop. | Product owner |
| 2 | Đã chốt (2026-09-18, giữ nguyên khi tách): v1 **không** có microservice admin. Platform voucher do service sở hữu voucher (nay là `checkout-service`) quản lý; admin console gọi trực tiếp `/api/v1/admin/vouchers/**` (`VOUCHER_MANAGE` + 2FA). | Nếu voucher chuyển service khác phải đổi event/redemption. | Product owner + Architecture |
| 3 | Mã `SHOP` của đơn bị huỷ được nhả lượt (§3.3 bước 1). Trước khi tách, doc chỉ ghi rõ cho trường hợp huỷ cả nhóm. | Nếu policy muốn giữ lượt khi seller huỷ, đổi bước 1. | Product owner |
| 4 | Giới hạn mỗi buyer 1 lần/voucher là baseline; có thể nới theo campaign. | Ảnh hưởng unique/kiểm tra redemption. | Product owner |
