# API Spec — Order-Commerce Service

> Nguồn: `docs/lld/order-commerce.md` · `docs/db/order-commerce.md` · HLD/Penpot · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · Spring Boot/Java 25 · Auth qua Gateway · `X-Request-ID`/W3C trace dùng chuẩn chung

## 1. Quy ước API chung

| Mục | Quy định |
|---|---|
| Auth | Cart/checkout/order cần JWT; buyer/seller scope kiểm tra tại service. |
| Request ID | `X-Request-ID` tối đa 64; Gateway tạo/propagate. |
| Trace | W3C `traceparent`/`tracestate`; propagate sang Product/Inventory/Payment/Shipment và Kafka. |
| Timestamp | ISO-8601 UTC; DB UTC `DATETIME(6)`. |
| Money | Integer VND, `currency=VND`; không float. |
| Pagination | page 1, size 20, max 100. |
| Idempotency | `Idempotency-Key` bắt buộc với checkout/cancel/fulfill/voucher mutation; giữ 24h. |
| Response | `{data,meta:{request_id}}`; error `{error:{code,message,details,trace_id}}`. |
| Log | JSON field chuẩn auth-user/Gateway; không log token/payment credential/full address/order payload. |
| Retry | Gateway không retry mutation; service dùng saga/idempotency. |

## 2. Danh sách endpoint

| # | Method + path | Quyền | Mục đích |
|---:|---|---|---|
| 1 | `GET /cart` | Buyer | Cart hiện tại. |
| 2 | `POST /cart/items` | Buyer | Thêm SKU. |
| 3 | `PUT /cart/items/{itemId}` | Buyer | Đổi quantity. |
| 4 | `DELETE /cart/items/{itemId}` | Buyer | Xóa item. |
| 5 | `GET /vouchers/validate` | Buyer | Preview voucher trên cart. |
| 6 | `GET /checkout/shipping-fee` | Buyer | Preview shipping fee. |
| 7 | `POST /checkout/preview` | Buyer | Tính preview tổng tiền. |
| 8 | `POST /checkout` | Buyer + Idempotency | Reserve Inventory và tạo order split. |
| 9 | `GET /orders` | Buyer | Own order list. |
| 10 | `GET /orders/{orderId}` | Buyer | Own order detail. |
| 11 | `GET /orders/{orderId}/status` | Buyer | Status timeline/projection. |
| 12 | `PATCH /orders/{orderId}/cancel` | Buyer | Cancel trước shipping. |
| 13 | `GET /orders/{orderId}/invoice` | Buyer | Invoice metadata/download URL. |
| 14 | `GET /seller/orders` | Seller/staff | Shop order list. |
| 15 | `GET /seller/orders/{orderId}` | Seller/staff | Shop order detail. |
| 16 | `PATCH /seller/orders/{orderId}/fulfill` | Seller/staff | Accept/ship flow. |
| 17 | `POST /seller/orders/{orderId}/cancel` | Seller/staff | Seller cancel theo policy. |
| 18 | `GET /seller/orders/{orderId}/invoice` | Seller/staff | Invoice. |
| 19 | `POST /seller/vouchers` | Seller | Tạo shop voucher. |
| 20 | `GET /seller/vouchers` | Seller | List voucher. |
| 21 | `PUT /seller/vouchers/{id}` | Seller | Update voucher. |
| 22 | `DELETE /seller/vouchers/{id}` | Seller | Soft archive/inactivate. |
| 23 | `POST /admin/vouchers` | Admin (`VOUCHER_MANAGE`) | Tạo platform voucher (`scope=PLATFORM`). |
| 24 | `GET /admin/vouchers` | Admin | List platform voucher + usage metric. |
| 25 | `PUT /admin/vouchers/{id}` | Admin | Update platform voucher. |
| 26 | `DELETE /admin/vouchers/{id}` | Admin | Soft archive/inactivate. |

## 3. Chi tiết endpoint

### 3.1 Cart endpoints

- `GET /cart`: trả cart Redis hiện tại; empty là `200 data.items=[]`.
- `POST /cart/items` request `{product_id,sku_id,quantity}`; validate Product active và quantity `1..999`; cart không reserve stock.
- `PUT /cart/items/{itemId}` request `{quantity}`; quantity `0` không dùng update, phải DELETE.
- `DELETE /cart/items/{itemId}` trả `204`; item khác user trả `404 ORDER_CART_ITEM_NOT_FOUND` theo anti-enumeration policy.

### 3.2 `GET /vouchers/validate?code=&cartId=`

Response `200` trả `valid`, discount, applicable shop/items, expires_at và warning. Đây là preview; redemption/usage limit phải kiểm tra lại trong checkout.

### 3.3 `GET /checkout/shipping-fee`

Query `from_shop_id`, `to_address_id`, `items[]`; response `200` `{shipping_fee,currency,estimate}`. Address phải thuộc buyer; carrier calculation là Shipment contract/mock. Dependency down trả `503 ORDER_DEPENDENCY_UNAVAILABLE`.

### 3.4 `POST /checkout/preview`

Request `{cart_id,selected_item_ids[],address_id,voucher_code?}`. Re-read Product price/status, Inventory availability read-only, validate voucher và shipping; response gồm `checkout_group_id`, item price snapshot preview, subtotal/discount/shipping/tax/grand_total, warnings và `expires_at`. Không reserve/charge.

### 3.5 `POST /checkout`

Headers: `Idempotency-Key` bắt buộc. Request:

```json
{
  "cart_id":"cart-01912f80",
  "selected_item_ids":["item-1"],
  "address_id":"address-1",
  "payment_method":"VNPAY",
  "voucher_code":"SHOP10"
}
```

Response `201`:

```json
{"data":{"checkout_group_id":"group-1","orders":[{"order_id":"order-1","order_number":"TC-20260830-0001","shop_id":"shop-1","status":"PENDING_PAYMENT","reservation_id":"reservation-1","payment_status":"PENDING"}],"payment_next_action":{"type":"VNPAY_QR","payment_id":"payment-1"},"reservation_expires_at":"2026-08-30T09:15:00Z"},"meta":{"request_id":"01912f81"}}
```

Order revalidates Product/price, calls Inventory atomic reserve, then commits local orders/items/invoice/outbox. Reserve fail không tạo order; retry cùng idempotency key trả cùng response.

### 3.6 Buyer order endpoints

- `GET /orders`: query status/page/size; chỉ buyer own orders.
- `GET /orders/{id}`: trả item/title/price/address/payment/shipment snapshot theo policy; không query lại giá Product.
- `GET /orders/{id}/status`: status + timeline từ Order/Shipment events.
- `PATCH /orders/{id}/cancel` headers Idempotency-Key, body `{reason,version}`; chỉ trước shipping, release Inventory/refund compensation theo payment state.
- `GET /orders/{id}/invoice`: `200` nếu issued; `409 ORDER_INVOICE_NOT_READY` nếu pending.

### 3.7 Seller order/voucher endpoints

- Seller list/detail filter theo `shop_id` auth context, không tin shop_id query/body.
- `PATCH /seller/orders/{id}/fulfill` body `{action:"ACCEPT"|"SHIP",tracking_code?}`; Shipment tạo tracking; seller không set `DELIVERED` trực tiếp.
- `POST /seller/orders/{id}/cancel` body `{reason,version}`; phải có policy/refund/stock release.
- Voucher create/update body gồm `code`, `scope=SHOP`, `discount_type` (`PERCENT`/`FIXED`), `value`, `min_order`, `usage_limit`, `starts_at`, `expires_at`; seller không tạo PLATFORM voucher. **Không có** field audience/targeting/segment trong v1 — các màn Penpot Seller "Voucher audience / Selected users / Import user IDs / Tab followers/VIP" ngoài phạm vi v1.
- Voucher DELETE là soft `INACTIVE/ARCHIVED`, không xóa redemption history.

### 3.8 Admin platform voucher (`/admin/vouchers`)

- Quyền: `VOUCHER_MANAGE` (do Auth User RBAC cấp) + step-up 2FA cho mutation; Gateway coarse-gate role admin, `order-commerce` enforce permission. v1 không có microservice admin riêng — admin console gọi trực tiếp route này (`System_Overview.md` §6.3).
- Body create/update: `code` (normalized unique), `scope=PLATFORM` (cố định), `discount_type`, `value`, `min_order`, `usage_limit`, `starts_at`, `expires_at`, `status`. Không có audience/targeting.
- `GET /admin/vouchers`: query `status`, `code`, `page`, `size`; trả voucher + `used_count`/`usage_limit`.
- `DELETE` là soft `INACTIVE/ARCHIVED`; redemption history giữ nguyên. Validation, usage increment và scope-check khi checkout dùng chung code path với shop voucher.

## 4. Mã lỗi chung

| Mã | HTTP | Ý nghĩa |
|---|---:|---|
| `ORDER_INVALID_INPUT` | 400 | Payload/query sai. |
| `ORDER_UNAUTHENTICATED` | 401 | Thiếu login. |
| `ORDER_FORBIDDEN` | 403 | Sai buyer/shop scope. |
| `ORDER_CART_EMPTY` | 409 | Không có item được chọn. |
| `ORDER_PRODUCT_UNAVAILABLE` | 409 | Product/SKU không active. |
| `ORDER_PRICE_CHANGED` | 409 | Giá snapshot cũ. |
| `ORDER_STOCK_UNAVAILABLE` | 409 | Inventory reserve fail. |
| `ORDER_RESERVATION_EXPIRED` | 409 | Reservation hết TTL. |
| `ORDER_VOUCHER_INVALID` | 400 | Voucher invalid/scope/time. |
| `ORDER_VOUCHER_LIMIT_REACHED` | 409 | Hết usage. |
| `ORDER_ADDRESS_INVALID` | 400 | Address không thuộc user/sai. |
| `ORDER_STATE_INVALID` | 409 | State transition sai. |
| `ORDER_CANCEL_NOT_ALLOWED` | 409 | Không còn được cancel. |
| `ORDER_IDEMPOTENCY_CONFLICT` | 409 | Key khác request hash. |
| `ORDER_DEPENDENCY_UNAVAILABLE` | 503 | Product/Inventory/Payment/Shipment down. |
| `ORDER_INVOICE_NOT_READY` | 409 | Invoice pending. |
| `ORDER_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | `POST /checkout` tạo order sau Inventory reserve, Payment charge tách service. | Ảnh hưởng saga/compensation/payment UX. | Order/Payment/Inventory owner |
| 2 | Multi-shop checkout split order theo shop nhưng chung `checkout_group_id`. | Ảnh hưởng invoice/shipment/voucher. | Product owner |
| 3 | Shipping fee endpoint dùng Shipment mock/quote; exact carrier contract chưa có. | Ảnh hưởng total/timeout. | Shipment owner |
| 4 | Voucher stacking/priority chưa chốt. | Ảnh hưởng discount calculation. | Product/Finance |
| 5 | Seller cancel/refund policy chưa đủ trong HLD. | Ảnh hưởng inventory/payment compensation. | Product/Finance |
| 6 | Voucher v1 chỉ theo mã, không audience/targeting; các màn Penpot Seller voucher targeting ngoài v1. | Nếu bật targeting cần schema `voucher_audiences` + follow shop. | Product owner |
| 7 | Platform voucher CRUD đặt tại `order-commerce` (`/admin/vouchers`, `VOUCHER_MANAGE`). | Nếu phạm vi Admin tách service riêng, service đó gọi vào contract này chứ không sở hữu voucher aggregate. | Architecture |
