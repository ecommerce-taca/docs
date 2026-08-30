# Database — API Gateway Service

> Nguồn: `docs/lld/api-gateway.md` · `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Trạng thái: N/A — API Gateway không sở hữu domain database

## 1. Quy ước chung

| Mục | Quy định |
|---|---|
| Domain database | Không có. Gateway là stateless edge service. |
| Domain tables/collections | Không tạo `users`, `products`, `orders`, `payments` hoặc business table nào. |
| Configuration | Route, JWT issuer/audience, CORS, timeout và service URL lấy từ environment/config repository; không lưu trong database. |
| Ephemeral state | Redis chỉ lưu rate-limit counter + TTL; không lưu domain state, session, refresh token hoặc user profile. |
| JWKS cache | Cache public key trong memory của instance; có thể refresh qua REST từ auth-user. Không persist private key. |
| Logs/metrics/traces | Gửi qua observability pipeline; không tạo bảng audit domain tại Gateway. |
| Cross-service data | Gateway chỉ forward ID/claims/header context; không join hoặc tạo foreign key. |
| Secret | Redis credential, JWT config và internal service credential dùng secret manager/environment; không ghi vào source/DB. |

## 2. Quan hệ

```text
API Gateway
  ├── không sở hữu aggregate/domain record
  ├── Redis ── ephemeral rate-limit key + TTL
  ├── Auth User JWKS ── REST public-key cache
  └── Internal services ── REST proxy, không cross-service FK
```

| Quan hệ | Kiểu | Lưu ở đâu | Khi dependency mất |
|---|---|---|---|
| Gateway → Redis | Ephemeral key/value | Redis managed by platform | Protected/public request fail-closed theo policy; không local fallback production. |
| Gateway → Auth User JWKS | REST read-only | Memory cache từng instance | Không bypass JWT; protected request trả `503` khi key quá stale. |
| Gateway → domain services | REST proxy | Không lưu response business | Trả `502/503/504` theo upstream state; không trả dữ liệu cache chưa được chốt. |
| Gateway → observability | Log/metric/trace stream | Platform sink | Request vẫn chạy nếu exporter lỗi, nhưng readiness/alert ghi degraded. |

## 3. Chi tiết dữ liệu được dùng tạm thời

### 3.1 Redis rate-limit key — không phải domain table

| Thành phần | Kiểu/format | TTL | Nội dung |
|---|---|---:|---|
| Key | `rl:v1:{bucket}:{identity_hash}:{route_group}:{window}` | Theo fixed window | Identity hash + route bucket. |
| Value | Integer counter | 60 giây baseline | Số request trong window. |
| Identity | SHA-256/IP hoặc `user_id` | Không persist ngoài TTL | Không ghi raw identity vào log. |
| Operation | Atomic increment + expire | — | Chống race giữa nhiều Gateway instance. |

Không dùng Redis key để xác nhận role, ownership, balance, order state hoặc inventory.

### 3.2 In-memory JWKS cache — không phải persistence

| Field | Kiểu | TTL/điều kiện |
|---|---|---|
| `kid` | string | Theo key rotation. |
| `public_key` | RSA public key object | Cache 10 phút; stale tối đa 30 phút. |
| `issuer`/`audience` | string | Đọc từ config, không nhận từ client. |
| `loaded_at` | ISO timestamp | Dùng health/readiness. |

Private key không tồn tại trong Gateway runtime.

## 4. Index

Không có database index.

Redis chỉ cần:

- Atomic `INCR`/Lua script cho counter.
- TTL trên từng key.
- Không dùng `KEYS *` hoặc scan toàn bộ Redis trong request path.
- Metric theo prefix/bucket, không truy vấn business key.

## 5. Enum & quy tắc dữ liệu

### 5.1 State lưu ngoài database

| State | Giá trị |
|---|---|
| Route access | `PUBLIC`, `AUTHENTICATED`, `ROLE_GATED`, `INTERNAL_ONLY` |
| Circuit | `CLOSED`, `OPEN`, `HALF_OPEN` |
| JWKS availability | `AVAILABLE`, `REFRESHING`, `STALE`, `UNAVAILABLE` |
| Gateway outcome | `SUCCESS`, `CLIENT_ERROR`, `UPSTREAM_ERROR`, `TIMEOUT`, `CIRCUIT_BLOCKED`, `GATEWAY_ERROR` |

Các state này chỉ sống trong process/metrics và không phải record có tính bền vững.

### 5.2 Configuration rules

| Quy tắc | Kiểm ở đâu |
|---|---|
| Mọi service URL phải là internal domain/IP hợp lệ | Config validation khi startup/readiness. |
| JWT algorithm chỉ `RS256` | Config schema + JWT validator. |
| Không route public tới database/debug/actuator endpoint | Static route registry review + integration test. |
| Không nhận trusted user headers từ client | Security middleware. |
| Rate-limit key có TTL và không chứa raw PII | Redis adapter + log redaction test. |
| Không persist refresh token/access token | Code review + secret scan. |
| Không tạo cross-service FK | Database N/A policy; Gateway không có schema. |

## 6. Migration & seed

| Hạng mục | Trạng thái |
|---|---|
| SQL/NoSQL migration | Không áp dụng. |
| Database seed | Không áp dụng. |
| Route/config seed | Quản lý bằng versioned config/Git/env, review như code. |
| Redis bootstrap | Không cần seed; key được tạo theo request và tự hết TTL. |
| JWKS bootstrap | Fetch từ auth-user khi startup/readiness; không seed key thủ công. |
| Rollback | Rollback image/config version; không rollback database. |

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Gateway không có domain database; README/API project ghi Database là N/A. | Nếu sau này lưu dynamic route/audit trong DB, phải tạo schema và migration riêng. | Architecture owner |
| 2 | Redis là infrastructure state cho distributed rate limit, không phải source of truth. | Nếu Redis mất, cần giữ fail-closed hoặc có policy fallback được phê duyệt. | DevOps/Security |
| 3 | Route/config được quản lý qua environment/Git; exact config delivery tool chưa chốt. | Ảnh hưởng rollout/rollback và secret rotation. | DevOps |
| 4 | JWKS cache chỉ trong memory; mỗi instance tự refresh cùng endpoint auth-user. | Restart instance cần fetch lại key; cần auth-user availability/readiness. | Auth-user owner |
