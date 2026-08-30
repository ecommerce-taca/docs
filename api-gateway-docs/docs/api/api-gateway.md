# API — API Gateway Service

> Nguồn: `docs/lld/api-gateway.md` · `docs/db/api-gateway.md` · `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · Gateway không sở hữu business endpoint; các route business được proxy tới domain service
> Runtime: **Kong Gateway 3.x OSS** (DB-less + decK). Toàn bộ contract đối ngoại trong tài liệu này **không đổi** khi chuyển từ bản NestJS sang Kong — ngoại lệ duy nhất là tên metric ở §3.3.

## 1. Quy ước chung

### 1.1 HTTP, security và request context

| Mục | Quy định |
|---|---|
| Protocol | Client → Gateway qua HTTPS; Gateway → service qua internal REST domain/IP. |
| Content-Type | JSON UTF-8 cho API; Gateway không parse business schema ngoài security/body limit. |
| Auth header | `Authorization: Bearer <JWT>`; protected route yêu cầu access token RS256 hợp lệ. |
| JWT validation | Verify local từ JWKS Auth User; kiểm `iss`, `aud`, `sub`, `exp`, `iat`, `nbf`, `kid`, algorithm `RS256`. |
| Refresh | Gateway chỉ proxy `/auth/refresh`; không tự refresh và không lưu refresh token. |
| CORS | Origin phải nằm trong environment allowlist; không `*` khi credentials. Preflight trả 204. |
| Request ID | `X-Request-ID` tối đa 64 ký tự; Gateway giữ giá trị hợp lệ hoặc tạo UUIDv7. |
| Trace | Propagate W3C `traceparent` và `X-Request-ID` tới upstream. |
| Body limit | JSON tối đa `1 MiB`; file lớn dùng signed URL của domain service/object storage. |
| Pagination | Không biến đổi query/response; pagination contract thuộc domain service. |
| Error | Gateway-generated error dùng `{error:{code,message,details,trace_id}}`; upstream 4xx allowlist giữ business code an toàn. |
| Time | Error/health/metadata dùng ISO-8601 UTC. |
| Internal header | Gateway strip `X-User-*`, `X-Auth-*`, `X-Request-ID` giả từ client rồi tạo lại context. |
| Idempotency | Gateway không tự thêm key; write endpoint phải do service sở hữu nghiệp vụ xử lý. |

### 1.2 Response behavior

- Với successful upstream response, Gateway giữ HTTP status, content type, body và business headers được allowlist.
- Gateway có thể thêm `X-Request-ID`/trace metadata nhưng không đổi field business.
- Với Gateway error, response có format:

```json
{
  "error": {
    "code": "GATEWAY_UPSTREAM_TIMEOUT",
    "message": "Hệ thống đang phản hồi chậm. Vui lòng thử lại sau.",
    "details": [],
    "trace_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

- Không trả stack trace, internal hostname/IP, SQL, secret, password, token, KYC bytes hoặc raw upstream 5xx body.
- `trace_id` phải tồn tại ở mọi response lỗi; client dùng giá trị này khi báo sự cố.

### 1.3 Route family

| # | Method/path family | Upstream | Access mặc định | Timeout | Retry |
|---:|---|---|---|---:|---|
| 1 | `ANY /api/v1/auth/**` | `auth-user` | Exact endpoint policy: public/protected | 5s | GET/HEAD idempotent only |
| 2 | `ANY /api/v1/users/**` (gồm `/users/me/favorites/**`, `/users/me/following`), `/api/v1/addresses/**` | `auth-user` | Authenticated | 5s | Không retry mutation |
| 3 | `ANY /api/v1/seller/onboarding/**`, `/api/v1/seller/shop` | `auth-user` | Seller role-gated | 5s | Không retry mutation |
| 4 | `ANY /api/v1/admin/users/**`, `/api/v1/admin/shops/**` | `auth-user` | Admin permission-gated | 5s | Không retry mutation |
| 5 | `GET /api/v1/shops/{id}`, `ANY /api/v1/shops/{id}/follow` | `auth-user` | GET public; follow authenticated | 5s | GET tối đa 1 lần |
| 6 | `ANY /api/v1/products/**`, `/api/v1/categories/**`, `GET /api/v1/shops/{id}/products` | `product-catalog` | GET public; mutation role-gated | 5s | GET tối đa 1 lần |
| 7 | `ANY /api/v1/seller/products/**` | `product-catalog` | Seller/admin role-gated | 5s | Không retry mutation |
| 8 | `ANY /api/v1/admin/catalog/**` | `product-catalog` | Admin permission-gated | 5s | Không retry mutation |
| 9 | `ANY /api/v1/search/**`, `/api/v1/products/search`, `/api/v1/admin/search/**` | `search` | GET public; admin role-gated | 5s | GET tối đa 1 lần |
| 10 | `ANY /api/v1/cart/**`, `/api/v1/checkout/**`, `/api/v1/orders/**`, `/api/v1/vouchers/**` (trừ `GET /api/v1/vouchers`) | `order-commerce` | Authenticated | 5s; checkout 10s | Không retry write |
| 10a | `GET /api/v1/vouchers` | `order-commerce` | Public (auth tuỳ chọn) | 5s | GET tối đa 1 lần |
| 11 | `ANY /api/v1/seller/orders/**`, `/api/v1/seller/vouchers/**` | `order-commerce` | Seller role-gated | 5s | Không retry write |
| 12 | `ANY /api/v1/admin/vouchers/**` | `order-commerce` | Admin permission-gated | 5s | Không retry write |
| 13 | `ANY /api/v1/seller/inventory/**`, `/api/v1/admin/inventory/**` | `inventory` | Seller/admin role-gated (`/internal/**` không expose) | 5s | GET tối đa 1 lần |
| 14 | `ANY /api/v1/payments/**` (webhook không JWT), `/api/v1/seller/wallet/**`, `/api/v1/seller/payouts/**`, `GET /api/v1/seller/revenue`, `GET /api/v1/seller/revenue/export`, `/api/v1/admin/payments/**`, `/api/v1/admin/fees/**`, `/api/v1/admin/taxes/**`, `/api/v1/admin/settlements/**`, `/api/v1/admin/finance/**` | `payment-wallet` | Authenticated/seller/admin; `/payments/webhook` public+signature | 10s | Không retry write |
| 15 | `GET /api/v1/orders/{id}/shipment`, `GET /api/v1/seller/orders/{id}/shipment`, `GET /api/v1/seller/orders/{id}/shipment/carriers`, `POST /api/v1/webhooks/shipping/{carrier}` | `shipment` | Buyer/seller theo endpoint; webhook carrier không JWT | 10s | GET tối đa 1 lần |
| 16 | `GET/POST /api/v1/products/{id}/reviews`, `ANY /api/v1/reviews/**`, `/api/v1/seller/reviews/**` | `rating-comment` | GET public; write authenticated | 5s | GET tối đa 1 lần |
| 17 | `ANY /api/v1/notifications/**` | `notification` | Authenticated | 5s | GET tối đa 1 lần |
| 18 | `ANY /api/v1/conversations/**`, `/api/v1/messages/**`, `/api/v1/attachments/**` | `message` | Authenticated/admin theo endpoint | 5s | GET tối đa 1 lần |
| 19 | `GET /ws/messages` (`Upgrade: websocket`) | `message` | Authenticated (JWT ở handshake) | handshake 5s; idle 1800s | Không retry; không buffer frame |

Route match ưu tiên exact path policy → method/path policy → family policy. Không được mặc định mọi `GET` là public.

### 1.4 Access matrix baseline

| Route/action | Không JWT | JWT buyer | JWT seller/staff | JWT admin |
|---|---:|---:|---:|---:|
| Public catalog/search/detail GET, `GET /shops/{id}`, `GET /shops/{id}/followers/count`, `GET /vouchers` | Có | Có | Có | Có |
| Auth signup/signin/refresh/email verify/password reset | Có | Có | Có | Có |
| Profile/address/cart/checkout/order/**favorites**/**following** của user | Không | Có | Có | Có |
| Follow/unfollow shop (`/shops/{id}/follow`) | Không | Có | Có | Có |
| WebSocket `/ws/messages` (handshake) | Không | Có | Có | Có |
| Seller onboarding/shop/product/inventory | Không | Chỉ endpoint onboarding mở | Có, theo shop scope | Có, theo permission |
| KYC review/admin users/roles | Không | Không | Không | Có, theo permission + 2FA khi cần |
| Payment/wallet/payout/refund | Không | Buyer/seller theo endpoint | Seller theo shop | Finance/admin theo permission |
| Health/metrics | Không qua public ingress | Không | Không | Internal ops only |

Gateway chỉ thực hiện coarse gate trong bảng; service đích kiểm tra ownership, KYC, business state, amount, stock và 2FA cuối cùng.

## 2. Danh sách endpoint của Gateway

| # | Method | Path | Mục đích | Quyền | Response |
|---:|---|---|---|---|---|
| 1 | `GET` | `/health/live` | Liveness của process | Internal/ops | Gateway health `200` |
| 2 | `GET` | `/health/ready` | Readiness config/JWKS/Redis/upstream | Internal/ops | `200` hoặc `503` |
| 3 | `GET` | `/metrics` | Metrics Prometheus/OpenMetrics | Internal observability | `200` text |
| 4 | `OPTIONS` | `/api/v1/**` | CORS preflight | Public origin allowlist | `204` |
| 5 | `ANY` | `/api/v1/auth/**` | Proxy auth/verification/MFA | Theo exact endpoint | Upstream pass-through/mapped error |
| 6 | `ANY` | `/api/v1/users/**`, `/api/v1/addresses/**` | Proxy profile/address | Authenticated | Upstream pass-through/mapped error |
| 7 | `ANY` | `/api/v1/seller/onboarding/**` | Proxy seller onboarding/KYC | Seller | Upstream pass-through/mapped error |
| 8 | `ANY` | `/api/v1/admin/users/**`, `/api/v1/admin/shops/**` | Proxy admin user/KYC | Admin permission | Upstream pass-through/mapped error |
| 9 | `ANY` | `/api/v1/products/**`, `/api/v1/categories/**` | Proxy catalog public/admin | Method/role policy | Upstream pass-through/mapped error |
| 10 | `ANY` | `/api/v1/seller/products/**` | Proxy seller product CRUD/publish | Seller/admin | Upstream pass-through/mapped error |
| 11 | `ANY` | `/api/v1/search/**` | Proxy search | Public GET | Upstream pass-through/mapped error |
| 12 | `ANY` | `/api/v1/cart/**`, `/api/v1/checkout/**`, `/api/v1/orders/**`, `/api/v1/vouchers/**` | Proxy commerce | Authenticated | Upstream pass-through/mapped error |
| 12a | `GET` | `/api/v1/vouchers` | List voucher khả dụng (PDP/Shop voucher strip cho khách chưa đăng nhập) | Public; có JWT thì service tính thêm `eligible` | Upstream pass-through/mapped error |
| 13 | `ANY` | `/api/v1/inventory/**` | Proxy inventory | Seller/admin | Upstream pass-through/mapped error |
| 14 | `ANY` | `/api/v1/payments/**`, `/api/v1/wallet/**`, `/api/v1/payouts/**`, `/api/v1/refunds/**`, `/api/v1/admin/fees/**`, `/api/v1/admin/taxes/**`, `/api/v1/admin/settlements/**`, `/api/v1/admin/finance/**` | Proxy payment/wallet + admin finance | Authenticated/admin (`FINANCE_OPS` cho `/admin/**`) | Upstream pass-through/mapped error |
| 15 | `ANY` | `/api/v1/shipments/**` | Proxy shipment | Buyer/seller/admin | Upstream pass-through/mapped error |
| 16 | `ANY` | `/api/v1/products/{id}/reviews`, `/api/v1/reviews/**`, `/api/v1/seller/reviews/**` | Proxy rating/comment | Public GET/auth write | Upstream pass-through/mapped error |
| 17 | `ANY` | `/api/v1/notifications/**` | Proxy notification center | Authenticated | Upstream pass-through/mapped error |
| 18 | `ANY` | `/api/v1/conversations/**`, `/api/v1/messages/**`, `/api/v1/attachments/**` | Proxy messaging (gồm conversation `type=SUPPORT`) | Authenticated/admin | Upstream pass-through/mapped error |
| 19 | `GET` | `/ws/messages` | WebSocket upgrade cho realtime chat | Authenticated (JWT ở handshake) | `101 Switching Protocols` hoặc `401/429/503` |

## 3. Chi tiết endpoint và contract

### 3.1 `GET /health/live` — liveness

Quyền: Internal/ops only · Response: `200`

Response:

```json
{
  "status": "UP",
  "service": "api-gateway",
  "time": "2026-08-30T09:00:00Z"
}
```

Ràng buộc:

- Không kiểm tra Redis/JWKS/upstream; process còn nhận request thì liveness vẫn `UP`.
- Không trả environment variable, hostname hoặc internal IP.
- Nguồn thật là `/status` của Kong, được expose lại dưới path này qua một Route nội bộ. Admin API (`:8001`) **không** được expose ra ingress công khai ở bất kỳ môi trường nào.

Lỗi: `503 GATEWAY_CONFIG_INVALID` nếu process chưa bind/health module không sẵn sàng.

### 3.2 `GET /health/ready` — readiness

Quyền: Internal/ops only · Response: `200` khi dependency bắt buộc sẵn sàng, `503` khi không sẵn sàng/degraded theo deployment policy.

Response mock:

```json
{
  "status": "DEGRADED",
  "service": "api-gateway",
  "checks": {
    "config": "UP",
    "jwks": "UP",
    "redis": "UP",
    "upstreams": "DEGRADED"
  },
  "time": "2026-08-30T09:00:00Z",
  "trace_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
}
```

Ràng buộc:

- `config`, `jwks`, `redis` phải `UP` để instance nhận protected traffic.
- `upstreams=DEGRADED` có thể vẫn ready nếu route-specific failure policy cho phép; `kong_upstream_target_health` phải báo rõ service lỗi.
- Response `503` không chứa secret/internal address.
- Giá trị từng check lấy từ: `config` = declarative config đã load thành công; `jwks` = trạng thái `lua_shared_dict` của `taca-jwt`; `redis` = ping Redis dùng cho rate limit; `upstreams` = tổng hợp trạng thái target trong Upstream healthcheck. Vì healthcheck của Kong là **per-node**, hai node có thể trả `upstreams` khác nhau tại cùng một thời điểm — đây là hành vi đúng, không phải lỗi.

### 3.3 `GET /metrics` — Prometheus/OpenMetrics

Quyền: Internal observability only · Content-Type: `text/plain; version=0.0.4` · Response: `200`

Metrics do plugin `prometheus` của Kong sinh; **tên metric khác với bản NestJS** và dashboard/alert phải được viết lại theo bảng này.

| Nhu cầu quan sát | Metric Kong | Ghi chú |
|---|---|---|
| Request count theo route/status | `kong_http_requests_total` (label `service`, `route`, `code`) | Thay cho `gateway_http_requests_total`. |
| Latency tổng / upstream / phần Kong | `kong_request_latency_ms`, `kong_upstream_latency_ms`, `kong_kong_latency_ms` (histogram) | Tách được thời gian do Gateway gây ra và thời gian do service gây ra — chi tiết hơn bản cũ. |
| Băng thông | `kong_bandwidth_bytes` | — |
| Trạng thái "circuit" | `kong_upstream_target_health` (label `upstream`, `target`, `state`) | Thay cho `gateway_circuit_state`; `state` gồm `healthy`/`unhealthy`/`dns_error`. |
| Rate limit bị chặn | `kong_http_requests_total{code="429"}` | Không có metric riêng theo bucket; xem hạn chế bên dưới. |
| Bộ nhớ shared dict (JWKS cache) | `kong_memory_lua_shared_dict_bytes` | Cảnh báo sớm khi dict JWKS sắp đầy. |

Metric **không có tương đương native**, phải do custom plugin tự ghi vào `lua_shared_dict taca_metrics` và expose trên cùng route `/metrics`:

| Metric | Label allowlist | Nguồn |
|---|---|---|
| `taca_jwks_refresh_total` | `outcome` (`success`/`failure`/`stale`) | `taca-jwt` |
| `taca_rate_limit_total` | `bucket`, `outcome` | wrapper quanh `rate-limiting` |
| `taca_ws_connections` | — (gauge tổng, không theo user) | `taca-ws-guard` |

- Tên metric chính xác phụ thuộc phiên bản Kong (một số bản dùng hậu tố `_seconds` thay `_ms`). Phải **chốt theo bản đang chạy** khi dựng dashboard, không copy từ tài liệu phiên bản khác.
- Không đưa `user_id`, email, phone, raw IP, token, `kid` thô hoặc URL có PII vào label.
- Kong sinh label `route`/`service` theo **tên object trong decK**, nên quy ước đặt tên ở LLD §2.3 (`rt-<upstream>-<nhóm>-<read|write>`) trực tiếp quyết định chất lượng dashboard — đặt tên tùy tiện sẽ làm metric không đọc được.

### 3.4 `OPTIONS /api/v1/**` — CORS preflight

Quyền: Public origin allowlist · Response: `204`

Request headers được kiểm:

| Header | Quy tắc |
|---|---|
| `Origin` | Bắt buộc và phải nằm trong `CORS_ALLOWED_ORIGINS`. |
| `Access-Control-Request-Method` | Method phải được route policy cho phép. |
| `Access-Control-Request-Headers` | Chỉ allowlist `Authorization`, `Content-Type`, `X-Request-ID`, `Idempotency-Key` và headers đã chốt. |

Response headers baseline:

```http
Access-Control-Allow-Origin: https://buyer.example
Access-Control-Allow-Methods: GET,POST,PUT,PATCH,DELETE,OPTIONS
Access-Control-Allow-Headers: Authorization,Content-Type,X-Request-ID,Idempotency-Key
Access-Control-Max-Age: 600
Vary: Origin
```

Origin không hợp lệ trả `403 GATEWAY_CORS_DENIED`; không trả wildcard.

### 3.5 Proxy business route — contract chung

Gateway không định nghĩa request/response business schema. Payload chi tiết nằm ở API spec của service đích. Gateway contract bắt buộc:

| Thành phần | Quy tắc |
|---|---|
| Path/query | Giữ nguyên path/query sau khi strip public prefix theo route config; không tự đổi field. |
| Body | Giữ JSON/multipart metadata theo allowlist; reject vượt 1 MiB JSON. |
| Auth | Route protected validate JWT; public route không bắt buộc JWT. |
| Actor context | Set `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`; strip bản client gửi. |
| Token | Forward `Authorization` nội bộ nếu service cần validate lại; không log. |
| Request ID | Forward `X-Request-ID` và trace context. |
| Status success | Giữ nguyên status upstream. |
| Status 4xx | Giữ business error code nếu allowlist; bổ sung trace ID nếu thiếu. |
| Status 5xx | Map thành `GATEWAY_UPSTREAM_UNAVAILABLE` hoặc `GATEWAY_UPSTREAM_BAD_RESPONSE`; không trả raw body. |
| Timeout | Default connect 2s/read 5s; checkout/payment/shipment read 10s. |
| Retry | Chỉ GET/HEAD hoặc route explicitly idempotent; tối đa 1 lần. |
| Rate limit | Auth 10/phút/IP; public 120/phút/IP; authenticated 300/phút/user. |

### 3.6 Ví dụ public proxy — `GET /api/v1/products`

Request:

```http
GET /api/v1/products?page=1&size=20&category_id=cat-01912f31 HTTP/1.1
Host: api.example.com
X-Request-ID: 01912f60-7a1b-7c12-9c55-8b1c34a6d921
Accept: application/json
```

Gateway:

1. CORS/request limit/request ID.
2. Match `product-catalog`, public rate limit theo IP.
3. Không yêu cầu JWT.
4. Proxy internal `GET /api/v1/products?...` tới `PRODUCT_CATALOG_BASE_URL`.
5. Retry tối đa 1 lần nếu connect/reset; không retry 4xx.

Response thành công: giữ body/pagination của Product Catalog; thêm `X-Request-ID`.

### 3.7 Ví dụ protected proxy — `GET /api/v1/users/me`

Request:

```http
GET /api/v1/users/me HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJ...access-token
X-Request-ID: 01912f61-7a1b-7c12-9c55-8b1c34a6d921
```

Gateway:

1. Rate limit IP guard rồi validate JWT RS256/JWKS.
2. Kiểm `iss`, `aud`, `sub`, `exp`, `iat`, `nbf`, `kid`.
3. Strip client-supplied identity headers; set actor context từ claims.
4. Proxy tới `AUTH_USER_BASE_URL`.
5. Auth-user kiểm ownership/profile; Gateway giữ response.

Lỗi token trước khi gọi upstream:

```json
{
  "error": {
    "code": "GATEWAY_TOKEN_EXPIRED",
    "message": "Phiên đăng nhập đã hết hạn.",
    "details": [],
    "trace_id": "01912f62-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

### 3.8 Ví dụ role-gated proxy — `POST /api/v1/seller/products`

Gateway kiểm role/permission coarse `SELLER`/`SELLER_STAFF`, sau đó forward tới Product Catalog. Product Catalog vẫn kiểm `shop_id` ownership, KYC status, SPU/SKU validation và product state. Gateway không kiểm body business hoặc tự approve product.

Write request không được Gateway retry. Nếu client cần retry, phải gửi `Idempotency-Key` và Product Catalog xử lý duplicate protection.

### 3.9 Rate-limit headers và 429

Request vượt limit trả:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 12
RateLimit-Limit: 120
RateLimit-Remaining: 0
RateLimit-Reset: 12
Content-Type: application/json
```

```json
{
  "error": {
    "code": "GATEWAY_RATE_LIMITED",
    "message": "Bạn thao tác quá nhanh. Vui lòng thử lại sau.",
    "details": {
      "retry_after_seconds": 12
    },
    "trace_id": "01912f63-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

### 3.10 Timeout/circuit response

| Tình huống | HTTP/code |
|---|---|
| Upstream connect timeout | `504 GATEWAY_UPSTREAM_TIMEOUT` |
| Upstream read timeout | `504 GATEWAY_UPSTREAM_TIMEOUT` |
| Circuit OPEN | `503 GATEWAY_UPSTREAM_UNAVAILABLE` |
| Upstream connection refused/down | `503 GATEWAY_UPSTREAM_UNAVAILABLE` |
| Upstream body/schema không hợp lệ | `502 GATEWAY_UPSTREAM_BAD_RESPONSE` |
| JWKS unavailable quá stale | `503 GATEWAY_JWKS_UNAVAILABLE` |
| Redis rate-limit unavailable | `503 GATEWAY_REDIS_UNAVAILABLE` |

Không retry `POST/PATCH/PUT/DELETE` tự động trong bất kỳ tình huống nào.

### 3.11 Internal error contract mock

Gateway-generated error:

```json
{
  "error": {
    "code": "GATEWAY_UPSTREAM_UNAVAILABLE",
    "message": "Dịch vụ đang tạm thời không khả dụng.",
    "details": [],
    "trace_id": "01912f64-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Upstream 4xx allowlist error:

```json
{
  "error": {
    "code": "ORDER_STOCK_UNAVAILABLE",
    "message": "Sản phẩm không còn đủ tồn kho.",
    "details": {
      "item_id": "item-01912f31"
    },
    "trace_id": "01912f65-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Gateway chỉ giữ business `code`/message đã allowlist; nội dung 5xx không được pass-through.

### 3.12 `GET /ws/messages` — WebSocket upgrade

Quyền: Authenticated (JWT hợp lệ ở handshake) · Upstream: `message`

Request handshake:

```http
GET /ws/messages HTTP/1.1
Host: api.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: bearer, eyJ...access-token
Origin: https://buyer.example
```

- Access token gửi qua `Sec-WebSocket-Protocol` (`bearer, <token>`); cho phép query `?access_token=` cho client không set được subprotocol, nhưng token trong query phải được redact khỏi log.
- Gateway kiểm: CORS `Origin` allowlist → rate limit handshake (IP + user) → `WS_MAX_CONNECTIONS_PER_USER` → JWT RS256/JWKS (`iss/aud/exp/nbf/kid`).
- Thành công: `101 Switching Protocols`, Gateway mở tunnel tới `MESSAGE_BASE_URL` kèm `X-User-ID`, `X-User-Roles`, `X-Trace-ID`; sau đó chỉ relay frame.

Handshake lỗi (không upgrade), dùng error envelope HTTP:

| HTTP/code | Khi nào |
|---|---|
| `401 GATEWAY_AUTH_REQUIRED` | Thiếu token ở handshake. |
| `401 GATEWAY_TOKEN_INVALID` / `GATEWAY_TOKEN_EXPIRED` | Token sai chữ ký/issuer/audience hoặc hết hạn. |
| `403 GATEWAY_CORS_DENIED` | Origin không trong allowlist. |
| `429 GATEWAY_RATE_LIMITED` | Vượt rate limit handshake hoặc `WS_MAX_CONNECTIONS_PER_USER`. |
| `503 GATEWAY_UPSTREAM_UNAVAILABLE` | Message Service down hoặc circuit OPEN. |

Sau khi upgrade:

- Gateway không parse/sửa/buffer frame, không retry, không tự reconnect.
- Đóng socket khi: client/upstream đóng, quá `WS_IDLE_TIMEOUT` không có frame, hoặc user bị revoke (Redis `revoked_user_id`).
- Reconnect và đồng bộ message bị miss là trách nhiệm client (`conversation.sync` qua REST cursor của Message Service).

## 4. Bảng mã lỗi dùng chung

| Code | HTTP | Ý nghĩa | Thông điệp hiển thị |
|---|---:|---|---|
| `GATEWAY_INVALID_REQUEST` | 400 | Header/path/method/body format sai | `Yêu cầu chưa đúng định dạng.` |
| `GATEWAY_ROUTE_NOT_FOUND` | 404 | Không match static route | `Không tìm thấy đường dẫn yêu cầu.` |
| `GATEWAY_AUTH_REQUIRED` | 401 | Protected route thiếu Bearer | `Vui lòng đăng nhập để tiếp tục.` |
| `GATEWAY_TOKEN_INVALID` | 401 | JWT signature/issuer/audience/format sai | `Phiên đăng nhập không hợp lệ.` |
| `GATEWAY_TOKEN_EXPIRED` | 401 | JWT hết hạn | `Phiên đăng nhập đã hết hạn.` |
| `GATEWAY_PERMISSION_DENIED` | 403 | Thiếu role/permission coarse | `Bạn không có quyền thực hiện thao tác này.` |
| `GATEWAY_CORS_DENIED` | 403 | Origin không trong allowlist | `Nguồn truy cập không được phép.` |
| `GATEWAY_RATE_LIMITED` | 429 | Vượt rate limit | `Bạn thao tác quá nhanh. Vui lòng thử lại sau.` |
| `GATEWAY_REQUEST_TOO_LARGE` | 413 | Body/header vượt limit | `Dữ liệu gửi lên vượt quá dung lượng cho phép.` |
| `GATEWAY_JWKS_UNAVAILABLE` | 503 | Không có public key hợp lệ | `Hệ thống xác thực đang tạm thời gián đoạn.` |
| `GATEWAY_REDIS_UNAVAILABLE` | 503 | Không áp được distributed rate limit | `Hệ thống đang tạm thời không khả dụng.` |
| `GATEWAY_UPSTREAM_TIMEOUT` | 504 | Connect/read timeout | `Hệ thống đang phản hồi chậm. Vui lòng thử lại sau.` |
| `GATEWAY_UPSTREAM_UNAVAILABLE` | 503 | Upstream down/circuit OPEN | `Dịch vụ đang tạm thời không khả dụng.` |
| `GATEWAY_UPSTREAM_BAD_RESPONSE` | 502 | Upstream response sai contract | `Hệ thống vừa gặp lỗi. Vui lòng thử lại sau.` |
| `GATEWAY_CONFIG_INVALID` | 503 | Config thiếu/sai, readiness fail | `Hệ thống chưa sẵn sàng.` |
| `GATEWAY_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại | `Hệ thống đang bận. Vui lòng thử lại.` |

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Gateway proxy business schema và không duplicate API schema của service đích. | API spec từng service phải hoàn tất trước khi frontend code full integration. | Backend leads |
| 2 | Runtime là Kong Gateway 3.x OSS (DB-less + decK); patch version, base image và cách đóng gói custom plugin chưa chốt. | Ảnh hưởng priority của plugin built-in, tên metric Prometheus, security patch và Docker image. | Tech lead |
| 10 | Contract đối ngoại giữ nguyên khi migrate sang Kong: route family (§1.3), access matrix (§1.4), error envelope, bảng mã lỗi (§4), rate-limit headers (§3.9), WebSocket handshake (§3.12). Chỉ tên metric ở §3.3 thay đổi. | Nếu một mục nào đó lệch sau khi triển khai thật, frontend phải sửa theo — phải phát hiện bằng contract test trước khi release. | Backend leads + Frontend |
| 3 | `CORS_ALLOWED_ORIGINS` thật của các ứng dụng Micro-Frontends (`mfe-shell`, `mfe-buyer`, `mfe-seller`, `mfe-admin`) chưa cung cấp. | Frontend có thể bị block hoặc mở origin ngoài ý muốn. | Frontend/DevOps |
| 4 | Refresh token đang dùng JSON Bearer flow; nếu chuyển HttpOnly cookie phải bổ sung CSRF/CORS contract. | Ảnh hưởng browser auth và Gateway credentials policy. | Frontend/Security |
| 5 | Internal REST có bắt buộc TLS/mTLS hay chỉ network policy chưa chốt. | Nếu chỉ tin forwarded identity header, có rủi ro bypass khi service bị gọi trực tiếp. | Security/DevOps |
| 6 | WebSocket `/ws/messages` được hỗ trợ trong v1: Gateway validate JWT ở handshake (subprotocol `bearer, <token>` hoặc query), proxy TCP upgrade tới Message Service, không buffer/không retry. Format subprotocol cần Message Service xác nhận. | Nếu Message Service đổi handshake, cập nhật §3.12. SSE không dùng trong v1. | Product/frontend/Message owner |
| 7 | Route ownership đã chốt (xem §1.3): `/shops/{id}` + follow → auth-user; `/shops/{id}/products` → product-catalog; voucher buyer/seller/admin → order-commerce; `/seller/wallet` + `/seller/payouts` + `/seller/revenue` + `/admin/fees` + `/admin/taxes` + `/admin/settlements` + `/admin/finance` → payment-wallet; `/admin/catalog/**` → product-catalog; `/admin/users/**` + `/admin/shops/**` → auth-user; `/notifications/**` → notification. Upstream business error allowlist vẫn cần align trong API spec từng service. | Gateway map nhầm service hoặc expose error không nhất quán. | Backend leads |
| 9 | Phạm vi Admin/back-office đã chốt (`System_Overview.md` §6.3): v1 **không** có microservice admin riêng; mọi `/api/v1/admin/**` route thẳng tới service sở hữu dữ liệu, Gateway chỉ coarse-gate theo role admin, permission chi tiết + 2FA do service enforce. Admin Dashboard là tầng đọc tổng hợp (không có endpoint Gateway riêng). `dispute`/`campaign` là service v1.1 — khi có sẽ thêm family `/api/v1/admin/disputes/**`, `/api/v1/admin/campaigns/**`. | Nếu tách service admin gộp sau này phải thiết kế lại route/auth. | Architecture owner |
| 8 | `GET /health/ready` degraded rule cho upstream chưa có SLO; LLD dùng dependency baseline. | Ảnh hưởng autoscaling/rollout khi một domain service tạm down. | DevOps/Architecture |
