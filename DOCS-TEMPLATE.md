# Docs template — chuẩn chung cho mọi service

> Áp dụng cho `docs/<service>-docs/docs/{api,db,lld,test}/<service>.md`. Mỗi service có đúng 4 file; tên file = tên service (kebab-case).
> Mục tiêu: mọi doc cùng cấu trúc, cùng thuật ngữ, và **khớp code đã triển khai**. Khi code và doc khác nhau, code là nguồn đúng; doc được sửa theo code, còn phần thiết kế chưa code thì giữ nhưng đánh dấu ⏳.

## 1. Header bắt buộc

Mọi file mở đầu bằng tiêu đề + 3 dòng blockquote:

```markdown
# <Loại doc> — <Tên Service>

> Nguồn: <file doc liên quan> · Code: `ecommerce-taca/<repo>` @ `<branch>` (`<short-sha>`) · Cập nhật: `YYYY-MM-DD`
> Trạng thái triển khai: <✅ Đã triển khai | 🟡 Triển khai một phần | ⏳ Chưa triển khai> — <một câu tóm tắt phạm vi đã có code>
> <Dòng baseline riêng của loại doc: Base path (API) · Engine/DB (DB) · Tech stack (LLD) · Môi trường (Test)>
```

`<Loại doc>` dùng đúng một trong: `API Spec`, `Database`, `LLD`, `Test Plan`.

## 2. Ký hiệu trạng thái triển khai

| Ký hiệu | Nghĩa | Dùng ở |
|---|---|---|
| ✅ | Đã có code trên nhánh ghi ở header, hành vi trong doc đã đối chiếu với code. | Cột `Trạng thái` trong bảng endpoint/bảng/event/test. |
| 🟡 | Có code nhưng chưa đủ so với thiết kế (ví dụ adapter fail-closed, consumer tắt mặc định). Ghi rõ phần thiếu. | Như trên. |
| ⏳ | Thiết kế, chưa có code. Giữ nguyên để làm spec cho lần triển khai sau. | Như trên. |

## 3. Cấu trúc section cố định

### 3.1 API Spec

1. `## 1. Quy ước API chung` — bảng `Mục | Quy định` (Auth, Request ID/trace, Actor context, Timestamp, Idempotency, Response/error, Pagination, Log).
2. `## 2. Danh sách endpoint` — bảng `# | Method + path | Quyền | Mục đích | Trạng thái`.
3. `## 3. Chi tiết endpoint` — mỗi endpoint một `### 3.x`, có request/response JSON lấy theo DTO thật.
4. `## 4. Mã lỗi chung` — bảng `Mã | HTTP | Ý nghĩa`; chỉ liệt kê mã code thực sự trả (mã thiết kế chưa code thì gắn ⏳).
5. `## 5. Giả định & câu hỏi mở`.

### 3.2 Database

1. `## 1. Quy ước chung` · 2. `## 2. Quan hệ` (mermaid `erDiagram`) · 3. `## 3. Chi tiết bảng` (MongoDB: `Chi tiết collection`; Elasticsearch: `Chi tiết index`) · 4. `## 4. Index` · 5. `## 5. Enum và rules` · 6. `## 6. Migration và seed` (liệt kê file migration thật: Flyway `V00x__*.sql`, Liquibase changeset, Mongo init script…) · 7. `## 7. Giả định & câu hỏi mở`.

### 3.3 LLD

1. `## 1. Phạm vi` · 2. `## 2. Cấu trúc bên trong` (cây package thật trong repo) · 3. `## 3. Luồng xử lý` · 4. `## 4. Hằng số & cấu hình` (cột `Tên | Giá trị mặc định | Biến môi trường/property | Ghi chú`) · 5. `## 5. Enum & trạng thái` · 6. `## 6. Event phát ra / lắng nghe` · 7. `## 7. Mã lỗi` · 8. `## 8. Giả định & câu hỏi mở`.

### 3.4 Test Plan

1. `## 1. Chuẩn bị` · 2. `## 2. Manual QA` · 3. `## 3. API integration` · 4. `## 4. Unit test` (liệt kê test class thật đã có + test còn thiếu) · 5. `## 5. Contract/security/resilience` · 6. `## 6. Tiêu chí pass/phát hành` · 7. `## 7. Giả định & câu hỏi mở`.

## 4. Thuật ngữ và quy ước dùng chung

| Chủ đề | Quy ước |
|---|---|
| Ngôn ngữ | Văn bản tiếng Việt; tên field/enum/topic/mã lỗi giữ nguyên tiếng Anh như trong code. |
| Field JSON | `snake_case`, đúng tên trong DTO. |
| Envelope thành công | `{ "data": …, "meta": { "request_id": … } }`; có phân trang thì `meta` thêm `page, size, total, total_pages` (ghi rõ nếu service khác). |
| Envelope lỗi | `{ "error": { "code", "message", "details", "trace_id" } }`. |
| Mã lỗi | `<SERVICE>_<NGUYÊN_NHÂN>` viết hoa (ví dụ `SHIPMENT_NOT_FOUND`). |
| Header actor (Gateway inject) | `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`; client không tự gửi được. |
| Header nội bộ | `X-Internal-Service-Token` cho service-to-service (ghi rõ nếu service dùng cơ chế khác). |
| Request/trace | `X-Request-ID`, W3C `traceparent`/`tracestate`. |
| Idempotency | Header `Idempotency-Key`; ghi rõ độ dài tối đa và thời gian giữ theo code. |
| Thời gian | ISO-8601 UTC trong API/event; DB lưu UTC. |
| Health | `GET /health/live`, `GET /health/ready` (ghi rõ nếu path khác, ví dụ nằm dưới `/api/v1`). |
| Topic Kafka | `<domain>.events.v1`, DLQ `<domain>.events.dlq.v1` (ghi đúng tên trong config). |
| Event envelope | `event_id, event_type, schema_version, occurred_at, aggregate_type, aggregate_id, payload` (+ `correlation_id`, `causation_id` nếu có). |

## 5. Quy trình cập nhật khi code thay đổi

1. Ghi nhánh + commit đã đối chiếu vào header.
2. Đối chiếu theo thứ tự: controller/route → DTO → mã lỗi → migration/schema → enum/state machine → config → producer/consumer Kafka → test class.
3. Hành vi code khác thiết kế: sửa doc theo code; nếu code có vẻ sai so với yêu cầu nghiệp vụ, ghi vào `Giả định & câu hỏi mở` thay vì giữ thiết kế cũ.
4. Cập nhật bảng tổng hợp trạng thái ở `README.md`.
