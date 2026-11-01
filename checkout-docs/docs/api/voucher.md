# API Spec — Checkout Service · Voucher

> Nguồn: `docs/lld/voucher.md` · `docs/db/voucher.md` · `docs/test/voucher.md` · Code: chưa có repo (`ecommerce-taca/checkout-service`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

Phần **Voucher** của Checkout Service. Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/api/checkout.md`.

## 1. Quy ước API chung

Theo `docs/api/checkout.md` §1 (base path `/api/v1`, auth qua Gateway, envelope `{data,meta}`, error `{error:{code,message,details,trace_id}}`, `Idempotency-Key` cho mutation, tiền integer VND).

## 2. Danh sách endpoint

Số thứ tự theo bảng tổng ở `docs/api/checkout.md` §2.

| # | Method + path | Quyền | Mục đích | Trạng thái |
|---:|---|---|---|---|
| 4 | `GET /vouchers/validate` | Buyer | Preview voucher trên giỏ (buyer đã biết mã). | ⏳ |
| 5 | `GET /vouchers` | **Public (auth tuỳ chọn)** | Danh sách voucher khả dụng; có JWT thì tính thêm `eligible`. | ⏳ |
| 6 | `GET /vouchers/redemptions` | Buyer | Lịch sử voucher đã dùng + tổng tiền đã tiết kiệm. | ⏳ |
| 7 | `POST /seller/vouchers` | Seller | Tạo shop voucher. | ⏳ |
| 8 | `GET /seller/vouchers` | Seller | List voucher. | ⏳ |
| 9 | `PUT /seller/vouchers/{voucherId}` | Seller | Update voucher. | ⏳ |
| 10 | `DELETE /seller/vouchers/{voucherId}` | Seller | Soft inactivate. | ⏳ |
| 11 | `POST /admin/vouchers` | Admin (`VOUCHER_MANAGE`) | Tạo platform/freeship voucher. | ⏳ |
| 12 | `GET /admin/vouchers` | Admin | List + usage metric. | ⏳ |
| 13 | `PUT /admin/vouchers/{voucherId}` | Admin | Update voucher. | ⏳ |
| 14 | `DELETE /admin/vouchers/{voucherId}` | Admin | Soft inactivate/archive. | ⏳ |

## 3. Chi tiết endpoint

### 3.1 `GET /vouchers/validate?code=&cart_id=`

Preview áp mã trên cart hiện tại. **Không** giữ chỗ voucher; `usage_limit`/redemption được kiểm tra lại (và tăng) trong `POST /checkout`.

| Query | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `code` | string | Có | 1–32 ký tự, normalize uppercase/trim trước khi so |
| `cart_id` | string (UUIDv7) | Có | Phải thuộc buyer trong token |

```json
{
  "data": {
    "valid": true,
    "code": "SHOP10",
    "scope": "SHOP",
    "shop_id": "shop-01912f31",
    "discount_type": "PERCENT",
    "value": 10,
    "discount_amount": 25000,
    "applicable_item_ids": ["item-1", "item-2"],
    "min_order": 200000,
    "expires_at": "2026-09-30T16:59:59Z",
    "warnings": []
  },
  "meta": { "request_id": "01912fa1-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Mã không hợp lệ vẫn trả `200` với `valid:false` + `reason` (`NOT_FOUND`/`EXPIRED`/`MIN_ORDER_NOT_MET`/`USAGE_LIMIT_REACHED`/`SCOPE_MISMATCH`/`ALREADY_USED`) để UI hiển thị — **không** dùng `404` cho mã sai (tránh dò mã).

### 3.2 `GET /vouchers` — danh sách voucher khả dụng

`validate` chỉ trả lời được khi buyer **đã biết mã**. HLD #23 yêu cầu buyer *"choose and apply voucher of system platform or shop"* — chọn từ danh sách. Endpoint này phục vụ mọi mặt hiển thị voucher: `Taca Buyer / My Vouchers`, `Overlay / Apply vouchers` ở checkout, `PDP / Voucher strip`, `Shop / Voucher strip`, `Mobile Buyer / Apply Voucher`.

| Query | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---|---|
| `scope` | enum | Không | `PLATFORM` \| `SHOP` \| `FREESHIP` (cùng enum `VoucherScope` mà checkout nhận); bỏ trống trả cả ba |
| `shop_id` | string (UUIDv7) | Không | Lọc voucher của một shop — dùng cho `Shop / Voucher strip` |
| `product_id` | string (UUIDv7) | Không | Voucher áp được cho sản phẩm — dùng cho `PDP / Voucher strip` |
| `cart_id` | string (UUIDv7) | Không | Có thì server tính `eligible` + `discount_amount` theo giỏ hiện tại |
| `status` | enum | Không | `AVAILABLE` (mặc định) \| `EXPIRING_SOON` \| `EXPIRED` |
| `page`, `size` | int | Không | `size` mặc định 20, max 100 |

```json
{
  "data": [
    {
      "code": "TACA200K",
      "scope": "PLATFORM",
      "shop_id": null,
      "title": "Giảm 200K cho đơn từ 2 triệu",
      "discount_type": "FIXED",
      "value": 200000,
      "min_order": 2000000,
      "starts_at": "2026-08-01T00:00:00Z",
      "expires_at": "2026-09-30T16:59:59Z",
      "usage_limit": 1000,
      "used_count": 812,
      "eligible": true,
      "ineligible_reason": null,
      "discount_amount": 200000
    },
    {
      "code": "TACA9SALE",
      "scope": "PLATFORM",
      "shop_id": null,
      "title": "Giảm 8% tối đa 1 triệu",
      "discount_type": "PERCENT",
      "value": 8,
      "min_order": 5000000,
      "starts_at": "2026-09-09T00:00:00Z",
      "expires_at": "2026-09-09T16:59:59Z",
      "usage_limit": 500,
      "used_count": 500,
      "eligible": false,
      "ineligible_reason": "USAGE_LIMIT_REACHED",
      "discount_amount": 0
    }
  ],
  "meta": { "request_id": "01912fa7-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 12, "total_pages": 1 }
}
```

Quy tắc:

- `eligible`/`discount_amount` **chỉ có nghĩa khi truyền `cart_id`**; không truyền thì `eligible` trả `null` và UI chỉ hiển thị điều kiện.
- `ineligible_reason` dùng đúng bộ mã của `validate`: `EXPIRED`, `MIN_ORDER_NOT_MET`, `USAGE_LIMIT_REACHED`, `SCOPE_MISMATCH`, `ALREADY_USED`. Đây là nguồn cho trạng thái `Voucher option / TACA9SALE disabled` trên Penpot.
- Chỉ trả voucher `status=ACTIVE` và còn trong cửa sổ thời gian; voucher `DRAFT`/`ARCHIVED` không bao giờ lộ ra buyer.
- Endpoint này **không** giữ chỗ và **không** tăng `used_count`; redemption vẫn chốt ở `POST /checkout`.
- `EXPIRING_SOON` = hết hạn trong 7 ngày; phục vụ metric `Voucher metric / Sắp hết hạn`.

> **v1 không có "lưu/đổi voucher".** Không có bảng sở hữu voucher theo user; "Voucher của tôi" = danh sách voucher buyer **đang đủ điều kiện dùng**, tính động. Nút `CTA / Đổi voucher` trên Penpot (thu thập voucher về ví cá nhân) ngoài phạm vi v1 (quyết định đã chốt 2026-09-18).

### 3.3 `GET /vouchers/redemptions` — voucher đã dùng

Phục vụ metric `Voucher metric / Đã tiết kiệm` và `Chip / Đã sử dụng 28` trên màn My Vouchers.

Query: `page`, `size` (max 100), `from?`, `to?` (ISO-8601).

```json
{
  "data": [
    {
      "code": "APPLE300K",
      "scope": "SHOP",
      "shop_id": "shop-01912f31",
      "order_id": "order-01912f91",
      "discount_amount": 300000,
      "redeemed_at": "2026-08-27T04:11:00Z"
    }
  ],
  "meta": {
    "request_id": "01912fa8-7a1b-7c12-9c55-8b1c34a6d921",
    "page": 1, "size": 20, "total": 28, "total_pages": 2,
    "total_saved": 1840000, "currency": "VND"
  }
}
```

`total_saved` là tổng `discount_amount` của **toàn bộ** redemption khớp filter, không phải chỉ trang hiện tại. Redemption đã `RELEASED` (đơn huỷ hoặc voucher bị gỡ khi huỷ một phần) không tính vào `total_saved`.

### 3.4 Seller voucher CRUD (`/seller/vouchers`)

`POST /seller/vouchers` request:

```json
{
  "code": "SHOP10",
  "discount_type": "PERCENT",
  "value": 10,
  "max_discount_amount": 100000,
  "min_order": 200000,
  "usage_limit": 500,
  "starts_at": "2026-09-01T00:00:00Z",
  "expires_at": "2026-09-30T16:59:59Z"
}
```

Response `201`:

```json
{
  "data": {
    "voucher_id": "vch-01912fc3",
    "code": "SHOP10",
    "scope": "SHOP",
    "shop_id": "shop-01912f31",
    "discount_type": "PERCENT",
    "value": 10,
    "max_discount_amount": 100000,
    "min_order": 200000,
    "usage_limit": 500,
    "used_count": 0,
    "starts_at": "2026-09-01T00:00:00Z",
    "expires_at": "2026-09-30T16:59:59Z",
    "status": "ACTIVE",
    "version": 1
  },
  "meta": { "request_id": "01912fc4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`scope` luôn `SHOP` (cố định, không nhận từ request) và `shop_id` lấy từ token — seller không tạo `PLATFORM`/`FREESHIP` voucher. `max_discount_amount` chỉ dùng với `PERCENT`, `null` = không trần. **Không có** field audience/targeting/segment trong v1 — các màn Penpot Seller "Voucher audience / Selected users / Import user IDs / Tab followers/VIP" ngoài phạm vi v1. Trùng `code` trong cùng shop → `409 CHECKOUT_VOUCHER_CODE_EXISTS`.

`GET /seller/vouchers` — query `status?`, `code?`, `page`, `size`:

```json
{
  "data": [
    { "voucher_id": "vch-01912fc3", "code": "SHOP10", "discount_type": "PERCENT", "value": 10, "used_count": 128, "usage_limit": 500, "status": "ACTIVE", "expires_at": "2026-09-30T16:59:59Z" }
  ],
  "meta": { "request_id": "01912fc5-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 4, "total_pages": 1 }
}
```

`PUT /seller/vouchers/{voucherId}` request cùng field với create cộng `version`; response `200` cùng shape với create. Không đổi `code` sau khi đã có redemption (tránh sai lệch lịch sử) — đổi `code` khi `used_count > 0` → `409 CHECKOUT_VOUCHER_CODE_LOCKED`.

`DELETE /seller/vouchers/{voucherId}` body `{version}`. Response `200`:

```json
{ "data": { "voucher_id": "vch-01912fc3", "status": "INACTIVE" }, "meta": { "request_id": "01912fc6-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Soft `INACTIVE`, không xóa redemption history.

### 3.5 Admin platform voucher (`/admin/vouchers`)

Quyền: `VOUCHER_MANAGE` (do Auth User RBAC cấp) + step-up 2FA cho mutation; Gateway coarse-gate role admin, `checkout-service` enforce permission. v1 không có microservice admin riêng — admin console gọi trực tiếp route này (quyết định admin-scope đã chốt 2026-09-18).

`POST /admin/vouchers` request:

```json
{
  "code": "TACA200K",
  "scope": "PLATFORM",
  "discount_type": "FIXED",
  "value": 200000,
  "min_order": 2000000,
  "usage_limit": 1000,
  "starts_at": "2026-09-01T00:00:00Z",
  "expires_at": "2026-09-30T16:59:59Z"
}
```

Response `201` cùng shape với `POST /seller/vouchers` nhưng `shop_id: null`. `scope` nhận `PLATFORM` hoặc `FREESHIP` (không nhận `SHOP` — admin không tạo voucher hộ shop). Voucher freeship là `scope=FREESHIP`, `discount_type` chỉ `PERCENT`/`FREESHIP`; nó chỉ trừ vào phí ship, không trừ tiền hàng — đây là mã kiểu `FREESHIPMAX` trên Penpot.

`GET /admin/vouchers` — query `status?`, `code?`, `scope?`, `page`, `size`:

```json
{
  "data": [
    { "voucher_id": "vch-01912fc7", "code": "TACA200K", "scope": "PLATFORM", "discount_type": "FIXED", "value": 200000, "used_count": 812, "usage_limit": 1000, "status": "ACTIVE", "expires_at": "2026-09-30T16:59:59Z" }
  ],
  "meta": { "request_id": "01912fc8-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 12, "total_pages": 1 }
}
```

`PUT /admin/vouchers/{voucherId}` cùng field với create cộng `version`, response `200` cùng shape. `DELETE /admin/vouchers/{voucherId}` body `{version}` → `200 { data: { voucher_id, status: "INACTIVE" }, meta }`, soft `INACTIVE/ARCHIVED`, redemption history giữ nguyên. Validation, usage increment và scope-check khi checkout dùng chung code path với shop voucher.

## 4. Mã lỗi chung

`CHECKOUT_INVALID_INPUT`, `CHECKOUT_FORBIDDEN`, `CHECKOUT_VOUCHER_INVALID`, `CHECKOUT_VOUCHER_CODE_EXISTS`, `CHECKOUT_VOUCHER_CODE_LOCKED`, `CHECKOUT_VOUCHER_LIMIT_REACHED`, `CHECKOUT_IDEMPOTENCY_CONFLICT` — định nghĩa ở `docs/lld/checkout.md` §7.

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Voucher v1 chỉ theo mã, không audience/targeting. | Bật targeting cần schema `voucher_audiences` + follow shop. | Product owner |
| 2 | Platform voucher CRUD ở `checkout-service` (`/admin/vouchers`, `VOUCHER_MANAGE`, seed ở auth-user `V025`). | Nếu Admin tách service riêng, service đó gọi vào contract này chứ không sở hữu voucher. | Architecture |
