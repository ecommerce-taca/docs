# Tài liệu dự án Ecommerce Platform — Taca-style Marketplace

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`

| Service | LLD | Database | API | Task | Test |
|---|---|---|---|---|---|
| `api-gateway` | [✔](lld/api-gateway.md) | — | — | — | — |
| `auth-user` | [✔](lld/auth-user.md) | [✔](db/auth-user.md) | [✔](api/auth-user.md) | [✔](tasks/auth-user.md) | [✔](test/auth-user.md) |
| `product-catalog` | [✔](lld/product-catalog.md) | — | — | — | — |
| `search` | — | — | — | — | — |
| `order-commerce` | — | — | — | — | — |
| `inventory` | — | — | — | — | — |
| `payment-wallet` | — | — | — | — | — |
| `shipment` | — | — | — | — | — |
| `rating-comment` | — | — | — | — | — |
| `notification` | — | — | — | — | — |
| `message` | — | — | — | — | — |

## Thứ tự triển khai tài liệu

```text
LLD → Database → API spec → Task breakdown + Test plan
```

Mỗi service sẽ được hoàn tất theo thứ tự trên và chờ xác nhận trước khi chuyển sang loại tài liệu kế tiếp.
