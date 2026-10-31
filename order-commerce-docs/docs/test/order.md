# Test Plan — Order-Commerce Service · Order

> Nguồn: `docs/lld/order.md` · `docs/api/order.md` · `docs/db/order.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: ⏳ Chưa triển khai — repo code mới chỉ có `README.md`; toàn bộ nội dung là thiết kế, dùng làm spec cho lần triển khai đầu.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers (MySQL 8.4, Redis, Kafka)

Phần **Order** của Order-Commerce (một service, tài liệu tách theo nghiệp vụ). Quy ước chung, mã lỗi đầy đủ và giả định nằm ở `docs/test/order-commerce.md`.

## 1. Chuẩn bị

Theo `docs/test/order-commerce.md` §1.

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Orders | Buyer list/detail/status/cancel before/after shipping | Scope đúng; cancel sau shipping bị chặn. |
| Seller | Order list/fulfill/cancel/invoice | Chỉ shop own; không tự set delivered. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| O-API-11 | `GET /orders`, detail/status | Buyer only; snapshot immutable; no other buyer data. |
| O-API-12 | `PATCH /orders/{orderId}/cancel` before shipping | `200 CANCELLED`, Inventory release command/event. |
| O-API-13 | Cancel shipped/delivered/wrong version | `409 ORDER_CANCEL_NOT_ALLOWED/ORDER_STATE_INVALID`. |
| O-API-14 | Invoice pending/issued | `409 ORDER_INVOICE_NOT_READY` or issued metadata. |
| O-API-15 | Seller list/detail own/other shop | Correct shop scope; no cross-tenant leak. |
| O-API-16 | Seller fulfill accept/ship invalid transition | Correct state/tracking validation; Shipment dependency not bypassed. |
| O-API-17 | Seller cancel allowed/disallowed | Policy/compensation/idempotency correct. |
| O-API-22 | `GET /orders/{orderId}/status` đơn COD vừa đặt | Timeline bắt đầu bằng `CONFIRMED`, **không** có entry `PENDING_PAYMENT`. |

### 3.1 Event/saga/reliability

| ID | Case | Expected |
|---|---|---|
| O-INT-01 | Payment success event duplicate/out-of-order | One transition/commit; old event ignored. |
| O-INT-03 | Order cancel duplicate | One release/redemption release. |
| O-INT-04 | Shipment delivered event | Order `DELIVERED` only valid transition. |
| O-INT-05 | Kafka unavailable | Local transaction + outbox; retry/DLQ, not false success to consumers. |

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
| O-COD-08 | Huỷ đơn COD trước khi giao | Kho hoàn lại, **không** phát sinh refund (chưa từng thu tiền), `payment_status → FAILED` theo policy. |
| O-COD-09 | `shipment.failed` cho đơn COD | OrderStatus giữ `SHIPPED`, ghi timeline; không post ledger, không allocation, không payout. |
| O-COD-10 | So sánh hai nhánh cùng giỏ hàng | VNPAY và COD ra cùng `grand_total`; chỉ khác `status`, `payment_status` và thời điểm capture. |

## 4. Unit test

⏳ Repo chưa có test class nào; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `OrderStateMachine` | All valid/invalid transitions and cancel-before-shipping rule. **Hai điểm vào khác nhau**: `→ PENDING_PAYMENT` (trả trước) và `→ CONFIRMED` (COD). Khẳng định `PAID` không còn là trạng thái hợp lệ, và không transition nào ra khỏi `PENDING_PAYMENT` cho đơn COD. |
| `OrderMapper` | Product/address/shop/price immutable snapshots, redaction. |

## 5. Contract/security/resilience

- Contract Order→Payment/Shipment: event/status transition và compensation.
- Security: IDOR buyer/seller, forged shop/user IDs, replay/idempotency cancel/fulfill/bulk-accept.

## 6. Tiêu chí pass/phát hành

Theo `docs/test/order-commerce.md` §6.

## 7. Giả định & câu hỏi mở

Xem `docs/test/order-commerce.md` §7.
