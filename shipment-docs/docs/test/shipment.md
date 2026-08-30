# Test Plan — Shipment Service

> Nguồn: `docs/lld/shipment.md` · `docs/db/shipment.md` · `docs/api/shipment.md` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

| Thành phần | Fixture |
|---|---|
| Runtime | Spring Boot/Java 25, JUnit 5, MySQL/Kafka Testcontainers. |
| Carrier | GHN sandbox double + MOCK adapter; success/timeout/error/status map. |
| Data | One shop-order shipment, duplicate tracking, status timeline, webhook events. |
| Actors | Buyer own/other, seller own/other shop, internal Order, carrier webhook. |
| Telemetry | JSON log/W3C trace/X-Request-ID; secret/address/webhook redaction scan. |

## 2. Manual QA

| Khu vực | Kịch bản | Kỳ vọng |
|---|---|---|
| Quote | Valid address/items, carrier unavailable | Fee/estimate đúng; lỗi có retry rõ. |
| Seller fulfillment | Create GHN/MOCK, tracking display | Một shop-order một shipment; no duplicate. |
| Tracking | picked up/in transit/delivered/failed | Timeline đúng; UI không tự set delivered. |
| Webhook | Duplicate/out-of-order/invalid signature | Idempotent/ignored/rejected đúng. |
| Cancel | Before pickup/after pickup | Chỉ cancel hợp lệ; không refund trực tiếp. |
| UI states | Loading/empty/error/offline | Retry không tạo shipment mới. |

## 3. API integration

| ID | Endpoint/case | Expected |
|---|---|---|
| SH-API-01 | `GET /orders/{id}/shipment` own/other buyer | Own `200`; other denied/not found. |
| SH-API-02 | Seller tracking own/other shop | Correct shop scope. |
| SH-API-03 | Internal quote valid/invalid | `200` fee or `400 SHIPMENT_INVALID_INPUT`. |
| SH-API-04 | Create GHN/MOCK valid | `201 CREATED`, tracking/external ID, event/outbox. |
| SH-API-05 | Create duplicate/idempotency | Existing result / conflict for different payload. |
| SH-API-06 | Carrier timeout/error | `503/504`, pending reconciliation; no false tracking. |
| SH-API-07 | Cancel before/after pickup | Success or `409 SHIPMENT_STATE_INVALID`. |
| SH-API-08 | Webhook valid status | Timeline/state/outbox updated once. |
| SH-API-09 | Webhook bad signature/duplicate/old | Invalid rejected; duplicate ACK; old ignored. |
| SH-API-10 | Health live/ready | Correct dependency behavior. |

### 3.1 Contract/security/reliability

| ID | Case | Expected |
|---|---|---|
| SH-INT-01 | Shipment delivered event to Order | Order receives event; Shipment does not mutate payment. |
| SH-SEC-01 | Forged seller/order/carrier signature | Denied; no state change. |
| SH-SEC-02 | Log scan | No carrier secret/full address/raw webhook/token. |
| SH-RES-01 | DB/Kafka/carrier outage | Outbox/reconcile; no duplicate carrier create. |

## 4. Unit test

| Module | Cases |
|---|---|
| `StatusMapper` | GHN/MOCK statuses, unknown status, no backward transition. |
| `ShipmentStateMachine` | Created/picked/in transit/delivered/failed/cancelled. |
| `CarrierAdapter` | Timeout, retry rules, safe response mapping. |
| `WebhookService` | Signature, external event dedupe, skew, payload hash. |
| `Authorization` | Buyer/seller/internal scopes. |
| `Observability` | JSON fields, trace/request propagation, redaction. |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | GHN sandbox + MOCK baseline. | Production carrier contract test bổ sung. | Shipment/DevOps |
| 2 | One shop-order one shipment; multi-package chưa v1. | Cần thêm package test matrix. | Order owner |
| 3 | Webhook signature/IP policy chưa chốt. | Ảnh hưởng security acceptance. | Security |
| 4 | Return/exchange chưa v1. | Cần lifecycle/refund test sau. | Product/Finance |
