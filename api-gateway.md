# LLD — API Gateway Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: Node.js + NestJS · REST/HTTP · Redis cho distributed rate limit · JWT RS256/JWKS · internal REST qua domain/IP nội bộ

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Làm entry point duy nhất cho client; route `/api/v1/**`; kiểm tra CORS và request size; validate JWT; áp dụng route-level role gate; rate limit; timeout/retry/circuit breaker; chuẩn hóa request ID, lỗi và telemetry. |
| Client | Buyer Web, Seller Center, Admin Console và các client tương lai dùng HTTPS. Các màn hình Penpot không gọi trực tiếp domain/IP của service. |
| Upstream | `auth-user`, `product-catalog`, `search`, `order-commerce`, `inventory`, `payment-wallet`, `shipment`, `rating-comment`, `notification`, `message` qua REST và địa chỉ nội bộ cấu hình bằng environment. |
| Nguồn dữ liệu | Không có domain database. Redis chỉ lưu counter/rate-limit key có TTL; không lưu user, product, order, token hoặc business state. Chi tiết được ghi ở `docs/db/api-gateway.md` với trạng thái N/A. |
| Xác thực | Auth-user ký access JWT bằng RS256. Gateway tải public key từ JWKS của auth-user và validate local; không gọi auth-user cho mỗi request. |
| Phân quyền | Gateway kiểm tra token và role/permission tối thiểu theo route. Service đích vẫn kiểm tra ownership, scope, trạng thái nghiệp vụ và quyền cuối cùng. |
| Không thuộc service | User/profile, KYC, product/SPU/SKU, search index, cart, checkout, order, inventory, payment, wallet, shipment, review, voucher, notification content và message content. |
| Không phát sinh business event | Gateway v1 không phát Kafka business event. Gateway chỉ ghi log/metric/trace và gọi REST adapter cần thiết. |

### 1.2 Nguyên tắc boundary

```text
Client
  │ HTTPS
  ▼
API Gateway
  ├─ CORS / request limit / request-id
  ├─ JWT RS256 validation từ JWKS cache
  ├─ route-level role gate + Redis rate limit
  └─ REST proxy qua internal domain/IP
       ├─ auth-user
       ├─ product-catalog / search
       ├─ order-commerce / inventory
       ├─ payment-wallet / shipment
       └─ rating-comment / notification / message
```

- Client không được biết hoặc truy cập trực tiếp các internal URL.
- Gateway strip mọi `X-User-*`, `X-Auth-*`, `X-Request-ID` do client tự gửi trước khi tạo context mới.
- Gateway có thể forward `Authorization: Bearer <access-token>` để service đích tự validate lại chữ ký; các header context do Gateway tạo chỉ là tối ưu, không thay thế business authorization.
- Gateway ưu tiên internal DNS ổn định. IP nội bộ chỉ là giá trị triển khai trong environment, không hard-code trong source.
- Mọi timestamp trong log/response gateway dùng UTC và ISO-8601; UI tự hiển thị theo timezone người dùng.

### 1.3 Mapping với HLD và Penpot

| Nguồn | Quan sát | Quyết định trong Gateway |
|---|---|---|
| HLD — API Gateway | Nêu authentication, authorization, rate limit, routing ở mức trách nhiệm. | Bổ sung pipeline, route registry, policy, timeout và error contract trong LLD này. |
| Penpot — Buyer | Home, search, category, shop, product detail, cart, checkout, orders, account, voucher, favorite, review. | Public GET đi qua route public; cart/checkout/order/account và action cá nhân yêu cầu JWT. |
| Penpot — Seller | Dashboard, product SPU/SKU, order, voucher, finance, settings, onboarding. | Route seller yêu cầu JWT + role/scope; quyền publish/withdraw do product/payment service kiểm tra theo KYC event. |
| Penpot — Admin | KYC, catalog, finance, order/dispute, voucher, user/role, settings; nhiều admin role và 2FA. | Gateway gate role/permission sơ bộ; auth-user kiểm tra RBAC và step-up 2FA ở mutation nhạy cảm. |
| Penpot — Messaging | Buyer ↔ seller và support escalation. | V1 expose REST conversation/message route; realtime WebSocket/SSE chưa thuộc scope vì HLD chưa chốt. |
| Penpot — States | Loading, empty, error, offline và retry. | Gateway trả error envelope ổn định, status rõ ràng, `traceId` để frontend hiển thị/retry phù hợp. |

## 2. Cấu trúc bên trong

### 2.1 Module trong NestJS

```text
src/
├── app.module.ts
├── config/                 # environment schema, fail-fast validation
├── routing/                # static route registry, upstream resolver
├── auth/                   # JWKS cache, JWT validator, route policy
├── rate-limit/             # Redis counter, key builder, 429 response
├── proxy/                  # internal REST client, timeout/retry/circuit
├── security/               # CORS, body/header limit, trusted headers
├── request-context/        # request-id, actor context, trace propagation
├── errors/                 # gateway/upstream error mapper
├── health/                 # liveness/readiness/dependency checks
└── observability/          # structured log, metric, trace, redaction
```

| Module | Trách nhiệm | Ràng buộc |
|---|---|---|
| `ConfigModule` | Đọc và validate environment khi process khởi động. | Thiếu `JWT_ISSUER`, `JWKS_URL`, Redis URL hoặc service URL bắt buộc thì readiness fail; không dùng giá trị production mặc định. |
| `RoutingModule` | Match method + path prefix với upstream; chọn policy timeout/retry. | Route static; không đọc route từ database; route không match trả 404 trước khi gọi service. |
| `JwtModule` | Cache JWKS, validate signature/issuer/audience/exp/nbf, tạo actor context. | Chỉ chấp nhận thuật toán `RS256`; không fallback sang `none`, HS256 hoặc key client gửi lên. |
| `RateLimitModule` | Tạo key theo IP/user/route và cập nhật Redis counter có TTL. | Lỗi Redis trên route protected/public phải fail-closed theo policy đã cấu hình, không tự chuyển sang counter local trong production. |
| `ProxyModule` | Forward request tới internal REST service, giới hạn timeout, retry request an toàn, circuit breaker. | Không retry POST/PATCH/PUT/DELETE mặc định; không tự thay đổi body hoặc business response. |
| `SecurityModule` | CORS allowlist, header/body limit, strip spoofed headers, trusted proxy. | Không dùng `Access-Control-Allow-Origin: *` cùng credentials; không log raw token/password/PII. |
| `ErrorModule` | Map lỗi gateway/upstream thành envelope `{code,message,traceId,details}`. | Giữ HTTP semantics; không expose stack trace, internal host, SQL hoặc secrets. |
| `HealthModule` | Liveness, readiness và kiểm tra dependency tối thiểu. | Liveness không phụ thuộc Redis/upstream; readiness kiểm tra config, JWKS và Redis theo deployment policy. |
| `ObservabilityModule` | Log JSON, metrics latency/status/rate-limit/circuit, trace propagation. | Mọi log request phải có `traceId`/`requestId`; chỉ log allowlist field. |

### 2.2 Request pipeline

```text
1. Nhận HTTPS request
2. Tạo/kiểm tra request-id và trace context
3. CORS + method/path/body/header limit
4. Match static route
5. Rate limit theo route và actor key
6. Nếu protected: validate JWT từ JWKS cache
7. Nếu role-gated: kiểm tra role/permission tối thiểu của route
8. Strip header giả mạo, gắn actor context và forward headers
9. Proxy tới internal service với timeout/circuit/retry
10. Map upstream response/error, ghi telemetry và trả client
```

### 2.3 Route registry v1

> Đây là route family để định tuyến, không thay thế API spec. Endpoint chi tiết, schema và ownership sẽ nằm ở tài liệu API của từng service.

| Route family | Upstream | Exposure mặc định | Timeout | Retry |
|---|---|---|---:|---|
| `/api/v1/auth/**` | `auth-user` | Public tùy endpoint; signout/2FA protected | 5s | Chỉ GET public nếu có |
| `/api/v1/users/**`, `/api/v1/addresses/**` | `auth-user` | Authenticated | 5s | Không retry mutation |
| `/api/v1/seller/onboarding/**`, `/api/v1/admin/users/**`, `/api/v1/admin/shops/**` | `auth-user` | Role-gated | 5s | Không retry mutation |
| `/api/v1/products/**`, `/api/v1/categories/**`, `/api/v1/seller/products/**` | `product-catalog` | GET public; seller/admin mutation role-gated | GET 5s, mutation 5s | GET tối đa 1 lần |
| `/api/v1/search/**` | `search` | GET public | 5s | GET tối đa 1 lần |
| `/api/v1/cart/**`, `/api/v1/checkout/**`, `/api/v1/orders/**`, `/api/v1/vouchers/**` | `order-commerce` | Authenticated; seller/admin route theo endpoint | 5s; checkout 10s | Chỉ GET; mutation dùng idempotency ở service |
| `/api/v1/inventory/**` | `inventory` | Seller/admin role-gated | 5s | GET tối đa 1 lần |
| `/api/v1/payments/**`, `/api/v1/wallet/**`, `/api/v1/payouts/**`, `/api/v1/refunds/**` | `payment-wallet` | Authenticated hoặc admin/seller role | 10s | Không retry mutation |
| `/api/v1/shipments/**` | `shipment` | Buyer/seller/admin theo endpoint | 10s | GET tối đa 1 lần |
| `/api/v1/reviews/**`, `/api/v1/comments/**` | `rating-comment` | GET public; create/update authenticated | 5s | GET tối đa 1 lần |
| `/api/v1/notifications/**` | `notification` | Authenticated | 5s | GET tối đa 1 lần |
| `/api/v1/conversations/**`, `/api/v1/messages/**`, `/api/v1/support/**` | `message` | Authenticated; support/admin route role-gated | 5s | GET tối đa 1 lần |

Quy tắc route:

- Các endpoint public cụ thể phải khai báo rõ trong route policy; không mặc định toàn bộ `GET` là public.
- `/health/live`, `/health/ready` và `/metrics` là endpoint vận hành, không expose qua public API prefix nếu chưa có ingress policy riêng.
- Internal service không được expose route quản trị database, actuator/debug hoặc endpoint bypass authorization.
- `POST /checkout`, `POST /payments`, `POST /orders` và action tương tự không được tự retry ở Gateway; idempotency key và duplicate protection thuộc service sở hữu nghiệp vụ.

### 2.4 JWT và actor context

Gateway nhận `Authorization: Bearer <access-token>` và validate các claim sau:

| Claim | Bắt buộc | Quy tắc |
|---|---:|---|
| `iss` | Có | Bằng `JWT_ISSUER` cấu hình cho auth-user. |
| `aud` | Có | Chứa audience của marketplace API. |
| `sub` | Có | UUID của user; dùng làm `X-User-ID` do Gateway tạo. |
| `exp` | Có | Phải lớn hơn thời điểm hiện tại; access token baseline 15 phút theo auth-user. |
| `iat` | Có | Không được ở tương lai vượt clock skew cho phép. |
| `jti` | Khuyến nghị | Dùng trace/security audit, không blacklist access token ở Gateway v1. |
| `roles` | Có với protected route | Array role đã được auth-user cấp: buyer/seller/staff/admin role. |
| `permissions` | Tùy | Chỉ dùng cho coarse route gate; service vẫn kiểm tra lại. |
| `shop_id` hoặc shop scope | Tùy | Forward để service tối ưu filter; không dùng một mình để kết luận ownership. |
| `email_verified` | Tùy | Route seller onboarding/publish có thể gate sơ bộ; auth-user/service là nơi quyết định cuối. |

JWKS cache:

1. Khởi động: lấy JWKS qua REST và validate issuer/audience config.
2. Request bình thường: validate local bằng `kid` trong cache.
3. Gặp `kid` mới: refresh JWKS một lần có lock, sau đó thử validate lại.
4. JWKS endpoint lỗi: không dùng key cũ quá `JWKS_MAX_STALE`; protected route trả lỗi service unavailable, không bypass xác thực.

### 2.5 Header context giữa Gateway và service

| Header | Nguồn | Quy tắc |
|---|---|---|
| `X-Request-ID` | Gateway | Giữ giá trị hợp lệ từ client nếu policy cho phép hoặc tạo UUIDv7 mới; tối đa 64 ký tự. |
| `X-Trace-ID` | Tracing layer | Propagate W3C trace context; không dùng làm authorization. |
| `X-User-ID` | JWT `sub` | Gateway strip header đầu vào và set lại sau khi validate. |
| `X-User-Roles` | JWT `roles` | Giá trị serialize an toàn; service không được tin nếu request không đi từ trusted Gateway network. |
| `X-User-Permissions` | JWT `permissions` | Chỉ là coarse context, không thay thế policy service. |
| `X-User-Shop-Scope` | JWT shop scope | Có thể rỗng hoặc nhiều shop; service phải kiểm tra record thật. |
| `X-Auth-Method` | Gateway | Giá trị v1 `jwt`; không cho client tự set. |
| `X-Forwarded-For` | Trusted proxy chain | Chỉ lấy client IP từ proxy đã khai báo; không tin chuỗi header tùy ý. |
| `Authorization` | Client access token | Forward nội bộ nếu service cần validate JWT lại; không ghi vào log. |

### 2.6 Upstream URL và network contract

| Biến cấu hình | Ví dụ mock | Bắt buộc |
|---|---|---:|
| `AUTH_USER_BASE_URL` | `http://auth-user.internal:8080` | Có |
| `PRODUCT_CATALOG_BASE_URL` | `http://product-catalog.internal:8080` | Có |
| `SEARCH_BASE_URL` | `http://search.internal:8080` | Có |
| `ORDER_COMMERCE_BASE_URL` | `http://order-commerce.internal:8080` | Có |
| `INVENTORY_BASE_URL` | `http://inventory.internal:8080` | Có |
| `PAYMENT_WALLET_BASE_URL` | `http://payment-wallet.internal:8080` | Có |
| `SHIPMENT_BASE_URL` | `http://shipment.internal:8080` | Có |
| `RATING_COMMENT_BASE_URL` | `http://rating-comment.internal:8080` | Có |
| `NOTIFICATION_BASE_URL` | `http://notification.internal:8080` | Có |
| `MESSAGE_BASE_URL` | `http://message.internal:8080` | Có |

- Các giá trị trên là tên biến và mock host, không phải địa chỉ production.
- Route registry fail-fast nếu URL sai scheme, thiếu host/port hoặc trỏ ra public network khi deployment policy cấm.
- Internal REST response phải có `X-Request-ID` hoặc `traceId` để Gateway map log; chi tiết mock contract ở mục 6.

## 3. Luồng xử lý

### 3.1 Public request — ví dụ `GET /api/v1/products`

```text
1. Client gửi request qua HTTPS
2. Gateway validate CORS, method, header/body size và tạo request-id
3. Match route → product-catalog; rate limit theo client IP
4. Không yêu cầu JWT; không inject actor identity
5. Proxy GET với timeout 5s, cho phép retry tối đa 1 lần nếu lỗi connect/reset
6. Trả status/body hợp lệ của product-catalog; map lỗi nếu upstream timeout/unavailable
```

Ràng buộc:

- Public không có nghĩa là bỏ qua rate limit.
- Không cache business response tại Gateway v1; caching/search result thuộc service hoặc CDN được chốt riêng.
- Nếu client gửi token hợp lệ vào public route, Gateway có thể tạo actor context để service hỗ trợ personalized response; route policy phải khai báo rõ việc này.

### 3.2 Protected request — ví dụ `GET /api/v1/users/me`

```text
1. Gateway match route AUTHENTICATED
2. Kiểm tra Redis rate limit theo user_id sau khi decode token; trước đó vẫn có IP guard
3. Lấy JWKS cache theo kid và validate RS256, iss, aud, exp, nbf
4. Tạo actor context từ claim; strip header giả mạo
5. Forward request tới auth-user qua internal REST
6. auth-user kiểm tra ownership từ token.sub và trả response
7. Gateway giữ status/body contract, thêm request/trace metadata vào log
```

- Access token hết hạn không được tự refresh tại Gateway. Client gọi `/api/v1/auth/refresh` tới auth-user.
- Gateway không lưu refresh token, không blacklist access token và không đọc `userdb`.
- Signout gọi auth-user để revoke refresh token; access token hiện tại có thể còn hiệu lực tối đa TTL đã chốt ở auth-user.

### 3.3 Role-gated request — ví dụ `POST /api/v1/seller/products`

```text
1. Validate JWT và xác định roles/permissions coarse
2. Route policy yêu cầu SELLER hoặc SELLER_STAFF + permission phù hợp
3. Thiếu claim phù hợp → 403, không gọi product-catalog
4. Đủ coarse gate → forward token/context tới product-catalog
5. Product-catalog kiểm tra shop ownership, KYC gate, SPU/SKU rule và trạng thái nghiệp vụ
```

- Gateway không tự quyết định seller có được sở hữu `shop_id` trong body/path hay không.
- Gateway không duyệt sản phẩm. Theo quyết định sản phẩm hiện tại, không có product approval/censor workflow; KYC gate do service áp dụng dựa trên event auth-user.
- Admin route phải qua role/permission coarse ở Gateway và permission/2FA chi tiết ở auth-user hoặc service sở hữu mutation.

### 3.4 Rate limit

```text
1. Xác định route policy
2. Key ưu tiên: user_id nếu JWT hợp lệ; nếu chưa xác thực dùng client IP đã chuẩn hóa
3. Redis atomic increment + TTL theo fixed window v1
4. Nếu vượt limit → 429 + Retry-After + error envelope
5. Ghi metric route/status/limit bucket, không ghi token
```

Baseline đã chốt:

| Bucket | Limit | Key | Burst |
|---|---:|---|---:|
| Auth endpoint: signup/signin/OTP/reset | 10 req/phút | IP | 20 request ngắn hạn |
| Public API | 120 req/phút | IP | 20 request ngắn hạn |
| Authenticated API | 300 req/phút | `user_id` + route group | 20 request ngắn hạn |

- Thứ tự áp dụng: global IP guard → route bucket → user bucket nếu có JWT.
- Redis key mẫu: `rl:v1:{bucket}:{identity}:{route_group}:{window}`; identity phải hash nếu đưa vào log.
- V1 dùng fixed window đơn giản qua Redis atomic operation; sliding window là tối ưu sau nếu traffic thực tế yêu cầu.

### 3.5 Timeout, retry và circuit breaker

```text
Request → route policy
  ├─ connect timeout → 504/503, không retry mutation
  ├─ idempotent GET connect/reset → retry tối đa 1 lần
  ├─ response timeout → 504, không retry POST/PATCH/PUT/DELETE
  └─ circuit OPEN → trả 503 ngay, không gọi upstream
```

- Timeout mặc định: connect `2s`, read `5s`; checkout/payment/shipment có thể dùng read `10s`.
- Retry chỉ cho GET/HEAD và request được route policy đánh dấu idempotent; không retry dựa trên status 4xx.
- Circuit mở sau `5` lỗi upstream trong cửa sổ `30s`, giữ OPEN `30s`, sau đó cho phép một probe HALF_OPEN.
- Gateway không tự thêm idempotency key cho write request; client/service sở hữu nghiệp vụ phải làm việc đó.

### 3.6 Upstream lỗi hoặc không sẵn sàng

| Tình huống | Gateway xử lý | HTTP trả client |
|---|---|---:|
| Upstream trả 4xx có error envelope hợp lệ | Giữ code/status/message an toàn; gắn `traceId` nếu thiếu | Giữ status upstream |
| Upstream trả 5xx | Không expose nội dung nội bộ; log sanitized; áp circuit counter | `502` hoặc `503` |
| Connect timeout | Không retry mutation; metric timeout | `504` |
| Read timeout | Map error chuẩn; circuit counter | `504` |
| Circuit OPEN | Không gọi upstream | `503` |
| Upstream trả body không đúng JSON contract | Log schema violation, không trả raw body | `502` |
| Redis rate-limit unavailable | Protected route fail-closed; health/readiness báo degraded | `503` |
| JWKS unavailable/không có key phù hợp | Không bypass validation | `503` cho protected route |

### 3.7 Request ID, trace và log

```text
Client X-Request-ID hợp lệ?
  ├─ Có → validate length/charset rồi giữ lại
  └─ Không → tạo UUIDv7
        ↓
Forward X-Request-ID + W3C trace context tới upstream
        ↓
Log start/end với method, route template, status, latency, upstream, outcome
```

- Không log request body mặc định.
- Redact `Authorization`, refresh token, OTP, password, TOTP secret, KYC data, bank data và message attachment metadata nhạy cảm.
- IP và user ID chỉ dùng ở mức cần thiết cho security/audit; policy retention cụ thể thuộc vận hành.
- Error response luôn có `traceId`; frontend dùng để hiển thị lỗi hoặc gửi support.

### 3.8 Upload và message attachment

- Gateway v1 giới hạn JSON request ở `1 MiB`.
- KYC document và message attachment không upload bytes lớn qua Gateway; service tạo signed URL/mock upload contract với object storage rồi client upload trực tiếp.
- Gateway chỉ proxy metadata/complete request có kích thước nhỏ và vẫn áp JWT/rate limit.
- Nếu sau này bắt buộc multipart qua Gateway, phải tạo route policy riêng: content type allowlist, file size, virus scan, timeout và không retry.

## 4. Hằng số & cấu hình

| Tên | Giá trị baseline | Đơn vị | Ghi chú |
|---|---:|---|---|
| `API_PREFIX` | `/api/v1` | path | Tất cả public API route. |
| `JWT_ALGORITHM` | `RS256` | algorithm | Chỉ verify signature, không ký token tại Gateway. |
| `JWT_ACCESS_TOKEN_TTL` | `900` | giây | Tham chiếu auth-user; Gateway không tự phát token. |
| `JWT_JWKS_CACHE_TTL` | `10` | phút | Cache public key theo `kid`. |
| `JWT_JWKS_MAX_STALE` | `30` | phút | Quá thời gian này mà JWKS không refresh được thì fail-closed. |
| `JWT_CLOCK_SKEW` | `30` | giây | Cho `iat`, `nbf`, `exp`; cần đồng bộ NTP. |
| `REDIS_RATE_LIMIT_STORE` | `true` | boolean | Distributed counter; không dùng local fallback production. |
| `REDIS_COMMAND_TIMEOUT` | `500` | ms | Timeout cho increment/read rate-limit. |
| `AUTH_RATE_LIMIT` | `10` | req/phút/IP | Signup/signin/OTP/reset baseline. |
| `PUBLIC_RATE_LIMIT` | `120` | req/phút/IP | Public API baseline. |
| `AUTHENTICATED_RATE_LIMIT` | `300` | req/phút/user | Authenticated API baseline. |
| `RATE_LIMIT_BURST` | `20` | request | Burst nhỏ trong fixed-window adapter. |
| `UPSTREAM_CONNECT_TIMEOUT` | `2` | giây | Tất cả internal REST route. |
| `UPSTREAM_READ_TIMEOUT_DEFAULT` | `5` | giây | Route thông thường. |
| `UPSTREAM_READ_TIMEOUT_LONG` | `10` | giây | Checkout/payment/shipment khi route policy cần. |
| `UPSTREAM_RETRY_MAX` | `1` | lần | Chỉ GET/HEAD hoặc idempotent route. |
| `UPSTREAM_RETRY_BACKOFF` | `100` | ms | Không retry 4xx. |
| `CIRCUIT_FAILURE_THRESHOLD` | `5` | lỗi/30s | Tính theo upstream + route group. |
| `CIRCUIT_WINDOW` | `30` | giây | Cửa sổ đếm lỗi. |
| `CIRCUIT_OPEN_DURATION` | `30` | giây | Sau đó cho một probe HALF_OPEN. |
| `MAX_JSON_BODY_SIZE` | `1` | MiB | Không áp cho signed-url upload bytes. |
| `MAX_HEADER_SIZE` | `16` | KiB | Vượt quá trả 413/431 tùy adapter HTTP. |
| `REQUEST_ID_MAX_LENGTH` | `64` | ký tự | Chỉ charset an toàn `[A-Za-z0-9._:-]`. |
| `CORS_ALLOW_CREDENTIALS` | `false` | boolean | Baseline Bearer header; nếu dùng HttpOnly cookie phải chốt lại. |
| `CORS_ALLOWED_ORIGINS` | environment allowlist | origin | Không wildcard trong production. |
| `HEALTH_LIVE_PATH` | `/health/live` | path | Không phụ thuộc upstream. |
| `HEALTH_READY_PATH` | `/health/ready` | path | Kiểm tra config/JWKS/Redis theo policy. |
| `METRICS_PATH` | `/metrics` | path | Chỉ internal/observability network. |
| `LOG_BODY_ENABLED` | `false` | boolean | Không log body production. |
| `TIMESTAMP_STORAGE` | `UTC` | timezone | Log/response ISO-8601. |

## 5. Enum & trạng thái

### 5.1 `RouteAccess`

| Giá trị | Ý nghĩa | Điều kiện |
|---|---|---|
| `PUBLIC` | Không cần JWT | Vẫn qua CORS, rate limit và route policy. |
| `AUTHENTICATED` | Cần access JWT hợp lệ | User status/verification chi tiết do auth-user/service kiểm tra. |
| `ROLE_GATED` | Cần JWT và role/permission coarse | Service đích kiểm tra scope/ownership/2FA cuối cùng. |
| `INTERNAL_ONLY` | Chỉ internal/ops network | Không expose qua client ingress. |

### 5.2 `CircuitState`

| Giá trị | Ý nghĩa | Chuyển sang |
|---|---|---|
| `CLOSED` | Cho request đi qua và đếm lỗi | `OPEN` khi đủ threshold |
| `OPEN` | Chặn gọi upstream trong thời gian bảo vệ | `HALF_OPEN` sau `CIRCUIT_OPEN_DURATION` |
| `HALF_OPEN` | Cho một probe kiểm tra phục hồi | `CLOSED` nếu thành công; `OPEN` nếu thất bại |

```text
CLOSED ──5 failures/30s──► OPEN ──30s──► HALF_OPEN
  ▲                                  ├─success──► CLOSED
  └──────────────────────────────────└─failure──► OPEN
```

### 5.3 `JwksAvailability`

| Giá trị | Ý nghĩa | Request protected |
|---|---|---|
| `AVAILABLE` | JWKS cache mới và có key phù hợp | Validate bình thường |
| `REFRESHING` | Đang refresh một lần theo `kid` mới | Request chờ trong timeout ngắn, không tạo refresh storm |
| `STALE` | Cache cũ nhưng còn trong `MAX_STALE` | Có thể validate key đã biết; ghi warning metric |
| `UNAVAILABLE` | Không có key hợp lệ hoặc quá stale | Fail-closed, trả `503` |

### 5.4 `GatewayOutcome`

| Giá trị | Ý nghĩa |
|---|---|
| `SUCCESS` | Upstream trả response hợp lệ. |
| `CLIENT_ERROR` | Request/auth/permission/rate-limit không hợp lệ. |
| `UPSTREAM_ERROR` | Upstream trả lỗi hoặc body sai contract. |
| `TIMEOUT` | Connect/read timeout. |
| `CIRCUIT_BLOCKED` | Request bị chặn do circuit OPEN. |
| `GATEWAY_ERROR` | Lỗi nội bộ Gateway/config/adapter. |

### 5.5 Quyền theo trạng thái

| Tình trạng | Gateway được làm | Gateway không được làm |
|---|---|---|
| Public route | Route, rate limit, response mapping | Tự suy đoán user/ownership. |
| JWT hợp lệ | Forward actor context và token | Tự sửa role, shop, KYC hoặc order state. |
| JWT hết hạn/sai | Dừng request protected, trả 401 | Gọi refresh hoặc cấp token mới. |
| Thiếu role coarse | Trả 403 trước upstream | Bypass bằng `shop_id` trong body/path. |
| Upstream down | Trả 503/504, ghi telemetry | Trả dữ liệu cache không được chốt. |
| Redis down | Fail-closed theo route policy | Tự chuyển sang in-memory production. |

## 6. Event phát ra / lắng nghe

### 6.1 Business event

| Loại | V1 |
|---|---|
| Event phát ra Kafka | Không có. Gateway không sở hữu aggregate/business state. |
| Event lắng nghe Kafka | Không có. Route/auth policy lấy qua config và JWKS REST, không qua event. |
| Audit/security log | Có, qua structured log/telemetry; không phải business event. |

### 6.2 Integration contracts

| Tích hợp | Giao thức | Ownership | Trạng thái |
|---|---|---|---|
| Auth-user JWKS | REST `GET /.well-known/jwks.json` | Auth-user | Mock contract bổ sung, cần API spec xác nhận. |
| Internal service proxy | REST/HTTP | Từng domain service | Route family và error envelope là contract baseline. |
| Distributed rate limit | Redis atomic counter + TTL | Gateway infrastructure | Config contract; không phải domain database. |
| Metrics/traces/log sink | OpenTelemetry-compatible exporter | Platform/ops | Endpoint thật chưa có trong HLD; dùng mock adapter. |

### 6.3 Mock contract — JWKS từ auth-user

Request:

```http
GET /.well-known/jwks.json
Host: auth-user.internal:8080
Accept: application/json
```

Mock response `200`:

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "auth-user-key-2026-01",
      "use": "sig",
      "alg": "RS256",
      "n": "base64url-modulus-from-auth-user",
      "e": "AQAB"
    }
  ]
}
```

Ràng buộc:

- `kid` phải khớp header JWT; key cũ có thể được giữ trong thời gian rotation overlap.
- Gateway không chấp nhận private key hoặc key do client gửi.
- `issuer`, `audience`, rotation schedule và endpoint path thật vẫn cần auth-user API spec xác nhận.

### 6.4 Mock contract — internal upstream error

Upstream error tối thiểu:

```json
{
  "code": "ORDER_STOCK_UNAVAILABLE",
  "message": "Sản phẩm không còn đủ tồn kho.",
  "details": {
    "item_id": "item-01912f31"
  },
  "traceId": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
}
```

Gateway xử lý:

- Giữ `code` nghiệp vụ đã allowlist và `message` an toàn nếu upstream trả status 4xx.
- Xóa `details` nhạy cảm hoặc nội dung có internal host/stack trace.
- Nếu upstream không có `traceId`, dùng `X-Request-ID` của Gateway.
- 5xx/timeout không trả raw body; map sang error code Gateway.

### 6.5 Mock contract — Gateway error envelope

```json
{
  "code": "GATEWAY_UPSTREAM_TIMEOUT",
  "message": "Hệ thống đang phản hồi chậm. Vui lòng thử lại sau.",
  "traceId": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
  "details": []
}
```

- `code` là English stable code để frontend xử lý.
- `message` là tiếng Việt thân thiện cho người dùng.
- `details` chỉ chứa field được allowlist; không trả stack trace hoặc URL nội bộ.

### 6.6 Mock contract — health/readiness

`GET /health/live` response `200`:

```json
{
  "status": "UP",
  "service": "api-gateway",
  "time": "2026-08-30T09:00:00Z"
}
```

`GET /health/ready` response `200` hoặc `503`:

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
  "traceId": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
}
```

Readiness body không được chứa secret, internal IP công khai ra client hoặc stack trace.

## 7. Mã lỗi

| Mã | HTTP | Khi nào xảy ra | Thông điệp cho người dùng |
|---|---:|---|---|
| `GATEWAY_INVALID_REQUEST` | 400 | Method, path parameter, header hoặc request format không hợp lệ | `Yêu cầu chưa đúng định dạng.` |
| `GATEWAY_ROUTE_NOT_FOUND` | 404 | Không có route family phù hợp | `Không tìm thấy đường dẫn yêu cầu.` |
| `GATEWAY_AUTH_REQUIRED` | 401 | Protected route thiếu Bearer token | `Vui lòng đăng nhập để tiếp tục.` |
| `GATEWAY_TOKEN_INVALID` | 401 | JWT sai signature, issuer, audience hoặc format | `Phiên đăng nhập không hợp lệ.` |
| `GATEWAY_TOKEN_EXPIRED` | 401 | JWT hết hạn | `Phiên đăng nhập đã hết hạn.` |
| `GATEWAY_PERMISSION_DENIED` | 403 | Không đạt role/permission coarse của route | `Bạn không có quyền thực hiện thao tác này.` |
| `GATEWAY_CORS_DENIED` | 403 | Origin không nằm trong allowlist | `Nguồn truy cập không được phép.` |
| `GATEWAY_RATE_LIMITED` | 429 | Vượt bucket rate limit | `Bạn thao tác quá nhanh. Vui lòng thử lại sau.` |
| `GATEWAY_REQUEST_TOO_LARGE` | 413 | Body vượt 1 MiB hoặc header vượt giới hạn | `Dữ liệu gửi lên vượt quá dung lượng cho phép.` |
| `GATEWAY_JWKS_UNAVAILABLE` | 503 | Không thể lấy key xác thực hợp lệ | `Hệ thống xác thực đang tạm thời gián đoạn.` |
| `GATEWAY_UPSTREAM_TIMEOUT` | 504 | Internal service connect/read timeout | `Hệ thống đang phản hồi chậm. Vui lòng thử lại sau.` |
| `GATEWAY_UPSTREAM_UNAVAILABLE` | 503 | Upstream down hoặc circuit OPEN | `Dịch vụ đang tạm thời không khả dụng.` |
| `GATEWAY_UPSTREAM_BAD_RESPONSE` | 502 | Upstream response sai contract | `Hệ thống vừa gặp lỗi. Vui lòng thử lại sau.` |
| `GATEWAY_REDIS_UNAVAILABLE` | 503 | Không thể áp rate limit phân tán | `Hệ thống đang tạm thời không khả dụng.` |
| `GATEWAY_CONFIG_INVALID` | 503 | Config runtime thiếu/sai khiến Gateway chưa ready | `Hệ thống chưa sẵn sàng.` |
| `GATEWAY_INTERNAL_ERROR` | 500 | Lỗi chưa phân loại tại Gateway | `Hệ thống đang bận. Vui lòng thử lại.` |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Framework là Node.js + NestJS; exact Node.js LTS major, NestJS version và HTTP adapter chưa chốt. | Ảnh hưởng Docker image, proxy adapter, dependency security và performance baseline. | Tech lead |
| 2 | Gateway route qua REST tới internal domain/IP bằng environment tĩnh; không dùng Consul/Eureka và không có route database. | Ảnh hưởng deployment, failover và cách thay đổi route. | Tech lead/DevOps |
| 3 | Auth-user cung cấp JWKS `GET /.well-known/jwks.json`, issuer/audience và RS256 key rotation; contract hiện là mock. | Không thể validate JWT hoặc xử lý key rotation đúng nếu endpoint/claim khác. | Auth-user owner |
| 4 | Client gửi Bearer access token; refresh token transport (JSON response, HttpOnly cookie hay mobile secure storage) chưa chốt. | Ảnh hưởng CORS credentials, CSRF policy và frontend interceptor. | Frontend + Security |
| 5 | Redis dùng chung cho rate limit; topology HA, password/TLS, eviction policy và failure policy chưa chốt. | Ảnh hưởng availability và việc fail-closed khi Redis lỗi. | DevOps |
| 6 | CORS allowlist thật cho Buyer Web, Seller Center và Admin Console chưa được cung cấp. | Nếu cấu hình sai, frontend bị chặn hoặc vô tình mở public origin. | Frontend/DevOps |
| 7 | Internal service có được truy cập trực tiếp bằng IP hay bắt buộc mTLS/network policy chưa chốt. | Nếu header context bị tin tuyệt đối, client có thể bypass qua đường nội bộ. | Security/DevOps |
| 8 | Message v1 hiện dùng REST; WebSocket/SSE cho realtime conversation chưa có trong HLD/Penpot contract. | Nếu cần realtime, phải bổ sung gateway protocol, connection auth và timeout khác. | Product + frontend |
| 9 | Exact ownership của một số route như `/shops/**`, `/vouchers/**`, `/notifications/**` cần align trong API spec từng service. | Route nhầm upstream gây duplicate API hoặc sai source of truth. | Backend leads |
| 10 | Observability backend/exporter và retention chưa được chỉ định; LLD chỉ chuẩn hóa adapter/field. | Ảnh hưởng dashboard, alert, trace sampling và chi phí lưu log. | Platform/DevOps |
| 11 | V1 không cache business response và không tự phát business event. | Nếu cần CDN/cache hoặc audit event qua Kafka, cần thêm module và contract. | Architecture owner |
| 12 | Error envelope `{code,message,traceId,details}` là mock contract áp dụng thống nhất cho upstream. | Nếu service trả format khác, Gateway phải duy trì adapter riêng cho từng service. | Backend leads |
