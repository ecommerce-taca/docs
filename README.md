# ecommerce-taca docs

Tài liệu thiết kế (HLD, Frontend) và tài liệu từng service (API Spec, Database, LLD, Test Plan). Mọi doc service theo chuẩn chung ở [`DOCS-TEMPLATE.md`](DOCS-TEMPLATE.md): header có repo/nhánh/commit đã đối chiếu, dòng trạng thái triển khai, bộ section cố định và thuật ngữ dùng chung.

## Trạng thái triển khai theo service

Đối chiếu ngày `2026-10-31`. ✅ đã có code và doc khớp code · 🟡 có code nhưng còn thiếu so với thiết kế · ⏳ chưa có code, doc là thiết kế.

| Service | Docs | Repo @ nhánh (commit) | Trạng thái | Ghi chú |
|---|---|---|---|---|
| Auth User | [`auth-user-docs`](auth-user-docs/docs) | `user-auth-service` @ `develop` (`7c268a0`) | 🟡 | Endpoint nghiệp vụ đủ; health chỉ có `/actuator/health` (`/liveness`, `/readiness` chỉ khi chạy trên K8s), chưa có `/health/live`; event favorite/follow chưa phát. |
| Product Catalog | [`product-catalog-docs`](product-catalog-docs/docs) | `Product-Catalog-Service` @ `develop` (`6572ece`) | 🟡 | Contract với Search/Auth-user đã khớp (search-service#27, Product-Catalog-Service#44); chưa phát `sku.updated`, `sku.status_changed`, `category.*`; admin chưa kiểm step-up. |
| Search | [`search-docs`](search-docs/docs) | `search-service` @ `develop` (`13d36e3`) | ✅ | Một số event consumer chờ chưa được Product Catalog phát (xem dòng trên). |
| Inventory | [`inventory-docs`](inventory-docs/docs) | `inventory-service` @ `develop` (`92c1454`) | ⏳ | Chỉ có scaffold (health, auth policy, Liquibase baseline). |
| Order Commerce | [`order-commerce-docs`](order-commerce-docs/docs) | `order-commerce-service` @ `main` (`bf512b5`) | ⏳ | Repo chỉ có `README.md`. Doc tách theo domain `cart`, `checkout`, `order`, `voucher`. |
| Payment Wallet | [`payment-wallet-docs`](payment-wallet-docs/docs) | `payment-wallet-service` @ `main` (`859b7b1`) | 🟡 | Payout API chưa có (ADR-003); outbox chỉ publish `payment.created`, `wallet.allocated`; event camelCase không có envelope `event_type`/`payload`, không có recipient. |
| Shipment | [`shipment-docs`](shipment-docs/docs) | `shipment-service` @ `develop` (`a375a41`) | 🟡 | Chỉ carrier `MOCK` chạy end-to-end; GHN fail-closed, SPX/J&T chưa có adapter. |
| Notification | [`notification-docs`](notification-docs/docs) | `notification-service` @ `main` (`6c6d9a2`) | 🟡 | Chưa xử lý `payment.expired`/`payment.refunded`; chưa có template `message-received-v1`. |
| Message | [`message-docs`](message-docs/docs) | `message-service` @ `main` (`bb00012`) | ⏳ | Repo chỉ có `README.md`. |
| Rating & Comment | [`rating-comment-docs`](rating-comment-docs/docs) | `rating-comment-service` @ `main` (`355daa3`) | ⏳ | Repo chỉ có `README.md`. |
| API Gateway | [`api-gateway-docs`](api-gateway-docs/docs) | chưa có repo | ⏳ | Thiết kế Kong 3.x DB-less. |
| Web (frontend) | [`FrontendDesign`](FrontendDesign) | `web-service` @ `develop` (`9619c4f`) | — | Ngoài phạm vi doc service. |

Contract giữa các service (Kafka topic/event, envelope, header, REST gọi chéo, health, env) và các lệch còn mở: [`CONTRACT-MATRIX.md`](CONTRACT-MATRIX.md).

Khi code thay đổi: cập nhật header của doc liên quan theo `DOCS-TEMPLATE.md` §5 và sửa dòng tương ứng ở bảng này.
