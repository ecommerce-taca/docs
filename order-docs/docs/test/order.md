# Test Plan — Order Service

> Nguồn: `docs/lld/order.md` · `docs/api/order.md` · `docs/db/order.md` · Code: `ecommerce-taca/order-commerce-service` @ `main` (`bf512b5`, chỉ có `README.md`) · Cập nhật: `2026-11-01`
> Trạng thái triển khai: ⏳ Chưa triển khai — tách từ Order-Commerce ngày 2026-11-01; toàn bộ nội dung là thiết kế.
> Môi trường: Spring Boot/Java 25, JUnit 5, MockMvc, Testcontainers (MySQL 8.4, Kafka), WireMock cho service phụ thuộc

| Case ID | ID cũ (Order-Commerce) |
|---|---|
| `ORD-API-01..12` | `O-API-11..17`, `O-API-22` + case mới cho tạo đơn/façade |
| `ORD-INT-01..09` | `O-INT-01`, `O-INT-03..05` + case mới cho Payment/recalculate |
| `ORD-COD-01..10` | `O-COD-01..10` |

## 1. Chuẩn bị

| Thành phần | Fixture/baseline |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MockMvc/Testcontainers. |
| DB | MySQL 8.4 `orderdb`; Kafka test broker. |
| Dependency doubles | Payment-Wallet (`POST /api/v1/payments` đúng validation của code thật: `amount > 0`, `shipping_fee < amount`, method `VNPAY|COD`), Inventory commit/release, Shipment carrier-quotes/create, `checkout-service` recalculate. |
| Event fixtures | `payment.*` dạng record camelCase của Payment-Wallet; `shipment.*` envelope snake của Shipment; `inventory.reservation.*`. |
| Data | Seed `docs/db/order.md` §6.2. |
| Actors | Buyer owner/khác, seller owner/staff (`SELLER_STAFF`)/khác shop, internal caller có/không token. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Orders | Buyer list/detail/status/cancel trước/sau shipping | Scope đúng; cancel sau shipping bị chặn. |
| Seller | List/carriers/fulfill/cancel/invoice/export/bulk-accept | Chỉ shop own; không tự set delivered. |
| Huỷ một phần | Huỷ 1 đơn trong nhóm 2 shop | Đơn còn lại hiện số tiền mới sau recalculate. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| ORD-API-01 | `GET /orders`, detail, status | Buyer only; snapshot bất biến; đơn người khác → `404 ORDER_NOT_FOUND`. |
| ORD-API-02 | `PATCH /orders/{id}/cancel` trước shipping | `200 CANCELLED`; Inventory release; outbox `order.cancelled`; tạo `voucher_recalc_requests`. |
| ORD-API-03 | Cancel đã shipped / sai version | `409 ORDER_CANCEL_NOT_ALLOWED` / `409 ORDER_VERSION_CONFLICT`. |
| ORD-API-04 | Invoice pending/issued | `409 ORDER_INVOICE_NOT_READY` hoặc metadata. |
| ORD-API-05 | Seller list/detail own/other shop | Đúng shop scope; không rò dữ liệu. |
| ORD-API-06 | Seller fulfill accept/ship sai transition | `409 ORDER_STATE_INVALID`; SHIP gọi Shipment với `order_code`. |
| ORD-API-07 | Seller cancel được/không được | Policy/compensation/idempotency đúng. |
| ORD-API-08 | `GET /orders/{id}/status` đơn COD | Timeline bắt đầu bằng `CONFIRMED`, không có `PENDING_PAYMENT`. |
| ORD-API-09 | `POST /internal/orders` VNPAY | `201`; Payment nhận đúng `CreatePaymentRequest`; `payment_next_action.type=VNPAY_QR`. |
| ORD-API-10 | `POST /internal/orders` lặp cùng key / khác body | Cùng response, Payment chỉ gọi 1 lần / `409 ORDER_IDEMPOTENCY_CONFLICT`. |
| ORD-API-11 | `POST /internal/orders` sai bất biến tiền / thiếu token | `400 ORDER_INVALID_INPUT` / `401`. |
| ORD-API-12 | `GET /seller/orders/{id}/shipment/carriers` | Đơn `CONFIRMED`/`PROCESSING` chưa có shipment → shape Shipment; sai state → `409`. |

### 3.1 Event/saga/reliability

| ID | Case | Expected |
|---|---|---|
| ORD-INT-01 | `payment.succeeded` trùng/đến trễ | Một transition, một commit; event cũ bỏ qua (`processed_events`). |
| ORD-INT-02 | Huỷ đơn trùng | Một release, một recalculate. |
| ORD-INT-03 | `shipment.delivered` | `SHIPPED → DELIVERED`; phát `order.delivered` có `buyer`, `items[]`. |
| ORD-INT-04 | Kafka down | Transaction local + outbox; retry/DLQ. |
| ORD-INT-05 | Payment-Wallet down khi tạo đơn VNPAY | Đơn vẫn tạo, `payment_next_action.type=RETRY_LATER`; job retry cùng key. |
| ORD-INT-06 | `checkout-service` down khi recalculate | Huỷ đơn vẫn thành công; request `PENDING`, retry tới khi `DONE`. |
| ORD-INT-07 | Recalculate trả số mới cho đơn đã thu tiền | `grand_total` giữ nguyên, `absorbed_amount` lưu trên đơn, history `AMOUNT_ADJUSTED`; bất biến `grand_total = subtotal − discount + shipping_fee − freeship_discount − absorbed_amount` đúng. |
| ORD-INT-08 | Group 2 shop, huỷ đơn shop A | Chỉ release `reservation_id` của đơn A; reservation đơn B vẫn `RESERVED`/`COMMITTED`. |
| ORD-INT-09 | Recalculate trả `items[]` | `order_items.discount_allocated`/`tax_amount` cập nhật theo `order_item_id`; `Σ tax_amount = orders.tax`. |

### 3.2 Vòng đời COD xuyên service *(hồi quy — nhóm case này từng thiếu và đã để lọt lỗi deadlock)*

Đơn COD không có bước trả tiền trước, nên mọi giả định "đơn hợp lệ = đã thu tiền" đều sai. Các case dưới đây khoá lại điều đó.

| ID | Case | Expected |
|---|---|---|
| ORD-COD-01 | Đặt đơn COD, chạy job payment-timeout | Đơn **không** bị huỷ. Job chỉ quét `PENDING_PAYMENT`; đơn COD ở `CONFIRMED` nên nằm ngoài phạm vi quét. |
| ORD-COD-02 | Đặt đơn COD, chờ quá `INVENTORY_RESERVATION_TTL` (15 phút) | Kho **không** bị nhả. Reservation đã `COMMITTED` ngay khi `order-service` tạo đơn COD, không còn `RESERVED` để hết hạn. |
| ORD-COD-03 | Đơn COD xuất hiện ở hàng đợi seller "Chờ xác nhận" | Có mặt ngay sau checkout. Filter hàng đợi dùng `status=CONFIRMED`, không dùng `payment_status`. |
| ORD-COD-04 | Vòng đời đầy đủ COD | `CONFIRMED → PROCESSING → SHIPPED → DELIVERED` chạy trọn vẹn mà **không** cần bất kỳ event nào từ Payment-Wallet. |
| ORD-COD-05 | Sau `DELIVERED`, Payment phát `payment.succeeded` | `payment_status` `PENDING_COD → SUCCESS`; `OrderStatus` **giữ nguyên** `DELIVERED`, không transition thêm. |
| ORD-COD-06 | Thứ tự event `order.confirmed` vs `order.paid` cho COD | `order.confirmed` phát khi tạo đơn; `order.paid` phát sau `DELIVERED`. Khoảng cách hai event = cả vòng đời giao hàng. |
| ORD-COD-07 | Notification nhận `order.confirmed` của đơn COD | Email `order-success-v1` + hoá đơn gửi **ngay lúc đặt đơn**, không chờ giao hàng. |
| ORD-COD-08 | Huỷ đơn COD trước khi giao | Kho hoàn lại, **không** phát sinh refund (chưa từng thu tiền), `payment_status → FAILED` theo policy. |
| ORD-COD-09 | `shipment.failed` cho đơn COD | OrderStatus giữ `SHIPPED`, ghi timeline; không post ledger, không allocation, không payout. |
| ORD-COD-10 | So sánh hai nhánh cùng giỏ hàng | VNPAY và COD ra cùng `grand_total`; chỉ khác `status`, `payment_status` và thời điểm capture. |

## 4. Unit test

⏳ Repo chưa có test class; mọi module dưới đây là test còn thiếu.

| Module | Cases |
|---|---|
| `OrderStateMachine` | Mọi transition hợp lệ/không hợp lệ; hai điểm vào (`PENDING_PAYMENT` cho VNPAY, `CONFIRMED` cho COD); `PAID` không phải trạng thái. |
| `OrderCreationService` | Kiểm bất biến tiền nhận từ Checkout; không tự tính lại. |
| `PaymentRequestMapper` | Map đúng `CreatePaymentRequest`/`CreatePaymentOrderRequest` của Payment-Wallet. |
| `PaymentEventParser` | Parse record camelCase của Payment-Wallet (`eventId`, `paymentId`, `checkoutGroupId`, `orderIds`…). |
| `OrderMapper` | Snapshot bất biến, redaction. |

## 5. Contract/security/resilience

- Contract Order→Payment: body khớp code Payment (`payment-wallet-service@main`).
- Contract Order→Shipment: carrier-quotes, create (có `order_code`), cancel; consumer `order.cancelled` của Shipment.
- Contract Order→Notification: field top-level `order_code`, `display_name`, `amount`, `invoice_url` (`docs/lld/order.md` §6.2).
- Contract Checkout↔Order: `POST /internal/orders` và recalculate khớp `checkout-docs`.
- Security: IDOR buyer/seller, giả shop/user, replay cancel/fulfill/bulk-accept, token nội bộ.

## 6. Tiêu chí pass/phát hành

Không đơn/payment/commit trùng dưới retry hoặc đồng thời; mọi transition, bất biến tiền, scope, outbox/replay và log redaction pass; nhóm `ORD-COD-*` pass.

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Payment-Wallet chưa route `payment.succeeded/failed/expired/refunded` ra Kafka (C3). | Test end-to-end VNPAY chưa chạy được với Payment thật. | Payment owner |
| 2 | Chưa có API sửa `amount` của payment COD sau recalculate (`docs/lld/order.md` §8 mục 6). | Case ORD-INT-07 cho COD chưa có kỳ vọng phía Payment. | Payment owner + Product |
