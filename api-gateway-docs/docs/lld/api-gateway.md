# LLD — API Gateway Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack đã chốt: **Kong Gateway 3.x OSS** (DB-less declarative, quản lý bằng decK trong Git) · plugin built-in + 5 custom Lua plugin (`taca-*`) · REST/HTTP · Redis cho distributed rate limit · JWT RS256/JWKS · internal REST qua domain/IP nội bộ
>
> **Lưu ý migration:** v1 trước đây thiết kế Gateway tự viết bằng Node.js + NestJS. Toàn bộ **contract đối ngoại giữ nguyên** (route family, error envelope, mã lỗi, header context, rate-limit baseline, WebSocket handshake) — chỉ thay đổi cơ chế thực thi. Xem §2.1 để biết yêu cầu nào do plugin built-in đáp ứng và yêu cầu nào bắt buộc phải viết custom plugin.

## 1. Phạm vi

### 1.1 Trách nhiệm và ranh giới

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Làm entry point duy nhất cho client; route `/api/v1/**` (HTTP) và `/ws/messages` (WebSocket upgrade); kiểm tra CORS và request size; validate JWT (HTTP header và WS handshake); áp dụng route-level role gate; rate limit; timeout/retry/circuit breaker; chuẩn hóa request ID, lỗi và telemetry. |
| Client | Micro-Frontends (MFE: `mfe-shell`, `mfe-catalog`, `mfe-buyer`, `mfe-seller`, `mfe-admin`) và các client tương lai dùng HTTPS. Các màn hình Penpot không gọi trực tiếp domain/IP của service. |
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
Kong Gateway (DB-less, kong.yml sinh bởi decK)
  ├─ taca-request-guard   → CORS origin allowlist, request-id, strip header giả mạo
  ├─ request-size-limiting → body/header limit
  ├─ taca-jwt             → JWKS cache + RS256 verify + actor headers
  ├─ taca-rbac            → coarse role/permission theo Route
  ├─ rate-limiting (redis)→ bucket auth/public/authenticated
  ├─ taca-error-envelope  → chuẩn hóa mọi lỗi về {error:{code,message,details,trace_id}}
  └─ Kong proxy core      → Service/Upstream + timeout + retries + healthcheck
       ├─ auth-user
       ├─ product-catalog / search
       ├─ order-commerce / inventory
       ├─ payment-wallet / shipment
       └─ rating-comment / notification / message

Ngoài request path:
  decK (Git) ──validate/diff/sync──► Kong config
  prometheus plugin ──► /metrics · http-log plugin ──► log sink · opentelemetry plugin ──► trace
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
| Penpot — Admin | KYC, catalog, finance, voucher, user/role, settings; nhiều admin role và 2FA. V1 **không** có microservice admin: mỗi màn route qua `/api/v1/admin/**` tới service sở hữu dữ liệu (§2.3, §8 #9). Dispute và Campaign là service v1.1 (`System_Overview.md` §6.3). | Gateway gate role/permission sơ bộ; service sở hữu enforce RBAC chi tiết và step-up 2FA ở mutation nhạy cảm. |
| Penpot — Messaging | Buyer ↔ seller và support escalation. | V1 expose REST conversation/message route **và** WebSocket `/ws/messages` cho realtime. Gateway validate JWT ở handshake, kiểm rate limit/connection cap rồi proxy TCP upgrade tới Message Service; không buffer/không retry message. SSE không dùng trong v1. |
| Penpot — States | Loading, empty, error, offline và retry. | Gateway trả error envelope ổn định, status rõ ràng, `traceId` để frontend hiển thị/retry phù hợp. |

## 2. Cấu trúc bên trong

### 2.1 Cấu trúc triển khai Kong

Kong chạy **DB-less** (`database = off`): toàn bộ Service/Route/Plugin/Upstream nằm trong một declarative config được sinh và kiểm tra bằng decK, version trong Git. Điều này giữ đúng nguyên tắc đã chốt ở §8 #2 — *route là static, không đọc từ database*.

```text
api-gateway/
├── kong/
│   ├── kong.conf                    # database=off, nginx tuning, lua_shared_dict
│   ├── deck/
│   │   ├── kong.yaml                # declarative config gốc (Service/Route/Upstream/Plugin)
│   │   ├── env/{dev,staging,prod}.yaml   # biến môi trường cho decK (upstream host, origin, TTL)
│   │   └── Makefile                 # deck validate / deck gateway diff / deck gateway sync
│   └── plugins/                     # custom Lua plugin, mỗi plugin một thư mục
│       ├── taca-request-guard/{handler.lua,schema.lua}
│       ├── taca-jwt/{handler.lua,schema.lua,jwks.lua}
│       ├── taca-rbac/{handler.lua,schema.lua}
│       ├── taca-ws-guard/{handler.lua,schema.lua}
│       └── taca-error-envelope/{handler.lua,schema.lua}
├── spec/                            # busted unit test cho từng plugin
└── Dockerfile                       # kong:3.x + COPY plugins + KONG_PLUGINS=bundled,taca-*
```

#### 2.1.1 Yêu cầu v1 → cơ chế Kong

Bảng này là **hợp đồng migration**: mỗi dòng là một yêu cầu đã chốt ở bản NestJS và cách Kong đáp ứng nó.

| Yêu cầu v1 | Cơ chế trong Kong | Loại |
|---|---|---|
| CORS response header, preflight | Plugin `cors` (`origins` allowlist, `credentials=false`, `max_age=600`) | Built-in |
| Từ chối origin ngoài allowlist bằng `403` | `taca-request-guard` — plugin `cors` **không** trả 403, xem §2.1.3 | **Custom** |
| Body ≤ 1 MiB | Plugin `request-size-limiting` (`allowed_payload_size=1`, `size_unit=megabytes`) | Built-in |
| Header limit 16 KiB | `kong.conf` → `nginx_http_large_client_header_buffers` | Kong core |
| `X-Request-ID` giữ/sinh mới | Plugin `correlation-id` (`header_name=X-Request-ID`, `generator=uuid`, `echo_downstream=true`) | Built-in |
| Validate charset/length của `X-Request-ID` client gửi | `taca-request-guard` | **Custom** |
| Strip `X-User-*`, `X-Auth-*` client gửi | `taca-request-guard` (phải chạy **trước** `taca-jwt`, xem §2.1.3) | **Custom** |
| JWKS fetch/cache/rotation + verify RS256 | `taca-jwt` — Kong OSS không có plugin JWKS | **Custom** |
| Kiểm `iss`/`aud`/`exp`/`iat`/`nbf`/`kid` | `taca-jwt` | **Custom** |
| Sinh `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`, `X-Auth-Method` | `taca-jwt` (`kong.service.request.set_header`) | **Custom** |
| Coarse role/permission gate theo route | `taca-rbac` (config per-Route) | **Custom** |
| Rate limit phân tán qua Redis | Plugin `rate-limiting` (`policy=redis`, `limit_by=consumer\|ip`, `fault_tolerant=false` để fail-closed) | Built-in |
| Timeout connect/read/write | `Service.connect_timeout` / `read_timeout` / `write_timeout` | Kong core |
| Retry chỉ cho GET/HEAD | Tách Kong Service read/write theo `Route.methods`, xem §2.1.4 | Kong core |
| Circuit breaker | `Upstream.healthchecks` passive + active, xem §5.2 | Kong core |
| Load balancing/connection pool tới upstream | `Upstream` + `targets` + keepalive của Kong | Kong core |
| Error envelope thống nhất | `taca-error-envelope` | **Custom** |
| WebSocket proxy `/ws/messages` | Kong proxy WS natively; timeout = `read_timeout` của Service `svc-message-ws` | Kong core |
| WS connection cap/user | `taca-ws-guard` (Redis `INCR` ở `access`, `DECR` ở `log`) | **Custom** |
| `/metrics` Prometheus | Plugin `prometheus` | Built-in |
| Structured JSON log + redaction | Plugin `http-log`/`file-log` + `custom_fields_by_lua` | Built-in |
| W3C trace propagation | Plugin `opentelemetry` (`header_type=w3c`) | Built-in |
| Liveness/readiness | Kong `/status` + Route nội bộ, xem §2.1.5 | Kong core |

**Kết luận migration:** phần lớn hạ tầng proxy (HTTP core, rate limit, CORS header, size limit, timeout, healthcheck, metrics, log, trace, WS tunnel) chuyển sang plugin đã được kiểm chứng của Kong. Phần **bắt buộc còn phải tự viết là 5 plugin Lua** ở bảng trên — chủ yếu vì Kong OSS không có JWKS và không có RBAC theo claim. Đây là đánh đổi cần biết trước: Kong **không** xóa hết custom code, nhưng thu hẹp nó từ "toàn bộ gateway" xuống "5 plugin có phạm vi hẹp, test được độc lập".

#### 2.1.2 Custom plugin

| Plugin | Phase | Trách nhiệm | Ràng buộc |
|---|---|---|---|
| `taca-request-guard` | `access` (ưu tiên cao nhất) | Kiểm `Origin` theo allowlist → `403 GATEWAY_CORS_DENIED`; validate `X-Request-ID` (`[A-Za-z0-9._:-]`, ≤64) → sai thì sinh mới; xóa mọi `X-User-*`, `X-Auth-*`, `X-Forwarded-*` không đến từ trusted proxy. | Phải chạy trước `taca-jwt`, nếu không sẽ xóa nhầm header do `taca-jwt` vừa set. Không đọc body. |
| `taca-jwt` | `access` | Lấy Bearer từ header / `Sec-WebSocket-Protocol` / query `access_token`; verify RS256 bằng JWKS cache trong `lua_shared_dict`; kiểm claim §2.4; kiểm marker `revoked_user:{sub}` trong Redis; set actor header. | Chỉ chấp nhận `alg=RS256`; không fallback `none`/HS256/key từ client. Refresh JWKS **một lần có lock** (`resty.lock`) khi gặp `kid` lạ. JWKS quá `JWT_JWKS_MAX_STALE` → `503`, không bypass. |
| `taca-rbac` | `access` (sau `taca-jwt`) | So `roles`/`permissions` trong actor context với `required_roles`/`required_any_permission` khai báo trên từng Route → `403 GATEWAY_PERMISSION_DENIED`. | Chỉ coarse gate. Không đọc body, không suy luận ownership từ `shop_id` trong path/body. Không tự quyết định 2FA. |
| `taca-ws-guard` | `access` + `log` | Chỉ gắn trên Route `/ws/messages`. `access`: `INCR ws:v1:conn:{user_hash}`, vượt `WS_MAX_CONNECTIONS_PER_USER` → `429`. `log`: `DECR` khi connection đóng. | Không parse/không sửa WebSocket frame. Không tự reconnect. Redis lỗi → fail-closed theo policy handshake. |
| `taca-error-envelope` | `header_filter` + `body_filter` | Dựa vào `kong.response.get_source()`: `exit`/`error` (lỗi do Kong/plugin sinh) → thay body bằng envelope chuẩn; `service` + 4xx → giữ nguyên business code đã allowlist, bổ sung `trace_id` nếu thiếu; `service` + 5xx → thay bằng `GATEWAY_UPSTREAM_UNAVAILABLE`/`GATEWAY_UPSTREAM_BAD_RESPONSE`. | Không bao giờ pass-through body 5xx của upstream. Không trả stack trace, internal host, SQL, secret. Bảng ánh xạ lỗi native của Kong ở §2.1.6. |

Mọi plugin đọc cấu hình từ `schema.lua` (khai báo trong decK), **không** hard-code giá trị môi trường; thiếu field bắt buộc thì `deck validate` fail trước khi sync.

#### 2.1.3 Thứ tự plugin

Kong thực thi plugin trong cùng phase theo `PRIORITY` giảm dần. Thứ tự **bắt buộc** trong `access`:

```text
taca-request-guard  →  request-size-limiting  →  taca-jwt  →  taca-rbac
                                                      ↓
                                              rate-limiting (redis)
                                                      ↓
                                              taca-ws-guard (chỉ /ws/messages)
```

- `taca-request-guard` phải cao hơn tất cả: nếu chạy sau `taca-jwt` nó sẽ xóa chính header actor vừa được sinh ra.
- `taca-jwt` phải chạy trước `rate-limiting` vì bucket authenticated dùng `limit_by=consumer` — cần danh tính đã xác thực. Bucket public/auth-endpoint dùng `limit_by=ip` nên không phụ thuộc thứ tự.
- `taca-rbac` phải chạy sau `taca-jwt` vì đọc `kong.ctx.shared.taca_actor` do plugin đó đặt.
- `taca-error-envelope` nằm ở `header_filter`/`body_filter` nên độc lập chuỗi trên; nó phải có priority **thấp nhất** trong hai phase đó để chạy sau cùng và bao được lỗi của mọi plugin khác.

> Giá trị `PRIORITY` cụ thể phải được chốt và **khóa bằng test** (§4 test plan) đối chiếu với priority của plugin built-in trong đúng phiên bản Kong đang dùng — không suy đoán từ tài liệu phiên bản khác. Anchor tham chiếu: `jwt` = 1005, `rate-limiting` = 901, `request-size-limiting` = 951, `correlation-id` = 1.

#### 2.1.4 Tách Service để kiểm soát retry

`retries` là thuộc tính của **Kong Service**, không phải Route — không thể cấu hình "chỉ retry GET" trên một Service duy nhất. Vì yêu cầu §3.5 cấm retry mutation, mỗi upstream được khai báo thành hai Kong Service trỏ về **cùng một Upstream**:

| Kong Service | Route gắn vào | `retries` | Dùng cho |
|---|---|---:|---|
| `svc-<name>-read` | `Route.methods = [GET, HEAD]` | `1` | Route idempotent |
| `svc-<name>-write` | `Route.methods = [POST, PUT, PATCH, DELETE]` | `0` | Mọi mutation |

- `Service.retries = 0` là **mặc định bắt buộc** cho mọi Service mới; giá trị mặc định `5` của Kong bị coi là sai cấu hình và phải bị chặn ở `deck validate`/review.
- Upstream có `retries=1` vẫn không được retry khi upstream đã trả response (chỉ retry lỗi connect/reset) — cấu hình qua `nginx_proxy_proxy_next_upstream = error timeout`.
- Service `svc-message-ws` luôn `retries = 0`: handshake WebSocket không được retry.

#### 2.1.5 Health endpoint

| Endpoint client/ops gọi | Nguồn thật | Ghi chú |
|---|---|---|
| `GET /health/live` | Kong `/status` trên admin/status listener, expose qua Route nội bộ | Chỉ phản ánh process Kong; không kiểm Redis/JWKS/upstream. |
| `GET /health/ready` | Route nội bộ + `taca-request-guard` chế độ `readiness` tổng hợp: config loaded, JWKS cache state, Redis ping, trạng thái healthcheck của Upstream | Trả `503` khi `config`/`jwks`/`redis` không `UP`. |
| Admin API (`:8001`) | **Không expose** ra ingress công khai trong mọi môi trường | DB-less nên Admin API chỉ read-only, nhưng vẫn lộ toàn bộ topology nếu mở. |

#### 2.1.6 Ánh xạ lỗi native của Kong sang mã lỗi hệ thống

`taca-error-envelope` phải chuyển mọi lỗi Kong tự sinh (mặc định là `{"message":"..."}`) sang envelope chuẩn:

| Lỗi native Kong | HTTP | Mã lỗi trả về |
|---|---:|---|
| `no Route matched with those values` | 404 | `GATEWAY_ROUTE_NOT_FOUND` |
| `rate-limiting` vượt bucket | 429 | `GATEWAY_RATE_LIMITED` |
| `request-size-limiting` vượt payload | 413 | `GATEWAY_REQUEST_TOO_LARGE` |
| `failure to get a peer from the ring-balancer` (mọi target unhealthy) | 503 | `GATEWAY_UPSTREAM_UNAVAILABLE` |
| Upstream connect/read timeout | 504 | `GATEWAY_UPSTREAM_TIMEOUT` |
| Upstream connection refused | 503 | `GATEWAY_UPSTREAM_UNAVAILABLE` |
| Upstream trả body không phải JSON hợp lệ | 502 | `GATEWAY_UPSTREAM_BAD_RESPONSE` |
| `rate-limiting` không kết nối được Redis (`fault_tolerant=false`) | 500 → map lại | `503 GATEWAY_REDIS_UNAVAILABLE` |
| Lỗi Lua chưa bắt trong bất kỳ plugin nào | 500 | `GATEWAY_INTERNAL_ERROR` |

Không có trường hợp nào được để lọt body mặc định `{"message": ...}` của Kong ra client — đó là contract break với frontend.

### 2.2 Request pipeline theo phase của Kong

```text
[router]        Match Route (host/path/method) → nếu không match: 404 GATEWAY_ROUTE_NOT_FOUND
   ▼
[access]  1. correlation-id            → giữ/sinh X-Request-ID
          2. taca-request-guard        → Origin allowlist, validate request-id, strip header giả mạo
          3. request-size-limiting     → body > 1 MiB → 413
          4. taca-jwt                  → RS256 + JWKS cache; set X-User-*; revoke check
          5. taca-rbac                 → coarse role/permission của Route
          6. rate-limiting (redis)     → bucket theo ip/consumer
          7. taca-ws-guard             → chỉ /ws/messages: connection cap
   ▼
[proxy]   Kong core → Upstream (healthcheck) → Service (timeout, retries)
   ▼
[header_filter / body_filter]
          taca-error-envelope          → chuẩn hóa error, sanitize 5xx
   ▼
[log]     http-log + prometheus + opentelemetry + taca-ws-guard (DECR connection)
```

So với bản NestJS, thứ tự nghiệp vụ giữ nguyên; khác biệt là bước 3 (`request-size-limiting`) chạy **sau** `taca-request-guard` thay vì trước, để request bị từ chối vì CORS không tốn chi phí đọc body.

### 2.3 Route registry v1

> Đây là route family để định tuyến, không thay thế API spec. Endpoint chi tiết, schema và ownership sẽ nằm ở tài liệu API của từng service.

Ánh xạ sang object của Kong: mỗi dòng dưới đây trở thành **một hoặc nhiều Kong Route** (`paths` + `methods`) trỏ tới `svc-<upstream>-read` hoặc `svc-<upstream>-write` (§2.1.4); cột `Timeout` là `read_timeout` của Service; cột `Retry` là `retries` của Service. Đặt tên theo quy ước `rt-<upstream>-<nhóm>-<read|write>`, ví dụ `rt-payment-admin-write`.

| Route family | Upstream | Exposure mặc định | Timeout | Retry |
|---|---|---|---:|---|
| `/api/v1/auth/**` | `auth-user` | Public tùy endpoint; signout/2FA protected | 5s | Chỉ GET public nếu có |
| `/api/v1/users/**` (gồm `/users/me/favorites/**`), `/api/v1/addresses/**` | `auth-user` | Authenticated | 5s | Không retry mutation |
| `/api/v1/seller/onboarding/**`, `/api/v1/seller/shop`, `/api/v1/admin/users/**`, `/api/v1/admin/shops/**` | `auth-user` | Role-gated | 5s | Không retry mutation |
| `/api/v1/shops/{id}`, `/api/v1/shops/{id}/follow` | `auth-user` | GET public (profile); follow authenticated | 5s | GET tối đa 1 lần |
| `/api/v1/products/**`, `/api/v1/categories/**`, `/api/v1/seller/products/**`, `/api/v1/admin/catalog/**`, `/api/v1/shops/{id}/products` | `product-catalog` | GET public; seller/admin mutation role-gated | GET 5s, mutation 5s | GET tối đa 1 lần |
| `/api/v1/search/**` | `search` | GET public | 5s | GET tối đa 1 lần |
| `/api/v1/cart/**`, `/api/v1/checkout/**`, `/api/v1/orders/**`, `/api/v1/vouchers/**`, `/api/v1/seller/vouchers/**`, `/api/v1/seller/orders/**`, `/api/v1/admin/vouchers/**` | `order-commerce` | Authenticated; seller/admin route theo endpoint | 5s; checkout 10s | Chỉ GET; mutation dùng idempotency ở service |
| `/api/v1/inventory/**`, `/api/v1/seller/inventory/**`, `/api/v1/admin/inventory/**` | `inventory` | Seller/admin role-gated; `/internal/**` không expose public | 5s | GET tối đa 1 lần |
| `/api/v1/payments/**`, `/api/v1/seller/wallet/**`, `/api/v1/seller/payouts/**`, `/api/v1/seller/revenue`, `/api/v1/seller/revenue/export`, `/api/v1/admin/payments/**`, `/api/v1/admin/fees/**`, `/api/v1/admin/taxes/**`, `/api/v1/admin/settlements/**`, `/api/v1/admin/finance/**` | `payment-wallet` | Authenticated hoặc admin/seller role; `/payments/webhook` không JWT | 10s | Không retry mutation |
| `/api/v1/orders/{id}/shipment`, `/api/v1/seller/orders/{id}/shipment`, `/api/v1/seller/orders/{id}/shipment/carriers`, `/api/v1/webhooks/shipping/**` | `shipment` | Buyer/seller theo endpoint; webhook carrier không JWT | 10s | GET tối đa 1 lần |
| `/api/v1/products/{id}/reviews`, `/api/v1/reviews/**`, `/api/v1/seller/reviews/**` | `rating-comment` | GET public; create/update/reply authenticated | 5s | GET tối đa 1 lần |
| `/api/v1/notifications/**` | `notification` | Authenticated | 5s | GET tối đa 1 lần |
| `/api/v1/conversations/**`, `/api/v1/messages/**`, `/api/v1/attachments/**` | `message` | Authenticated; conversation `type=SUPPORT` gate bằng participant/support scope | 5s | GET tối đa 1 lần |
| `/ws/messages` (HTTP `Upgrade: websocket`) | `message` | Authenticated (JWT ở handshake) | handshake `WS_HANDSHAKE_TIMEOUT`; sau đó idle theo `WS_IDLE_TIMEOUT` | Không retry; không buffer frame |

Quy tắc route:

- Các endpoint public cụ thể phải khai báo rõ trong route policy; không mặc định toàn bộ `GET` là public.
- `/health/live`, `/health/ready` và `/metrics` là endpoint vận hành, không expose qua public API prefix nếu chưa có ingress policy riêng.
- Internal service không được expose route quản trị database, actuator/debug hoặc endpoint bypass authorization. Các route `/internal/**` của Inventory/Shipment/Payment chỉ gọi service-to-service, không map ra public API prefix.
- `POST /checkout`, `POST /payments`, `POST /orders` và action tương tự không được tự retry ở Gateway; idempotency key và duplicate protection thuộc service sở hữu nghiệp vụ.
- `/ws/messages` là path WebSocket duy nhất được phép `Upgrade` trong v1; mọi path `/ws/**` khác trả `404`. Handshake bắt buộc JWT hợp lệ; token hết hạn giữa phiên xử lý theo `WS_IDLE_TIMEOUT`/policy, Gateway không tự refresh.
- `/api/v1/admin/**`: Gateway chỉ **coarse-gate** theo role admin (có bất kỳ admin role nào); permission chi tiết (`VOUCHER_MANAGE`, `CATALOG_ADMIN`, `FINANCE_OPS`, `RISK_MANAGER`…) và step-up 2FA do **service sở hữu dữ liệu** enforce. V1 không có microservice admin riêng: mỗi nhánh `/admin/**` route thẳng tới service chủ tương ứng (xem §8 #9, `System_Overview.md` §6.3). Admin Dashboard là tầng đọc tổng hợp (`mfe-admin` compose read-API hoặc BFF mỏng), Gateway không có endpoint dashboard riêng.
- **Không dùng catch-all Route.** Mỗi route family phải khai báo `paths` tường minh. Kong khớp Route theo độ dài prefix và `regex_priority`; một Route `/api/v1` duy nhất sẽ nuốt hết mọi request và vô hiệu hóa `taca-rbac` theo từng nhánh. Route không khai báo → `404`, đúng như policy hiện tại.
- **Không bật `strip_path` trừ khi có lý do rõ ràng.** Upstream service nhận nguyên `/api/v1/...`; đặt `strip_path = false` cho toàn bộ Route business để path tới service không đổi so với bản NestJS.
- **Route `/internal/**` không được khai báo trong Kong.** Không có Route nghĩa là Kong trả `404` — đây là lớp chặn thứ nhất; network policy vẫn là lớp chặn bắt buộc thứ hai.

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

JWKS cache (do `taca-jwt` quản lý, lưu trong `lua_shared_dict taca_jwks 10m` khai báo ở `kong.conf`):

1. Khởi động/lần dùng đầu: lấy JWKS qua REST và validate issuer/audience config.
2. Request bình thường: validate local bằng `kid` trong shared dict, không gọi auth-user.
3. Gặp `kid` mới: refresh JWKS **một lần có `resty.lock`** rồi thử validate lại — các request khác cùng worker chờ trên lock, không tạo refresh storm.
4. JWKS endpoint lỗi: không dùng key cũ quá `JWT_JWKS_MAX_STALE`; protected route trả `503 GATEWAY_JWKS_UNAVAILABLE`, không bypass xác thực.

Ràng buộc riêng của Kong:

- `lua_shared_dict` được chia sẻ giữa các nginx worker **trong cùng một node**, không chia sẻ giữa các node. Mỗi node Kong tự fetch JWKS — giống hệt mô hình "mỗi instance tự cache" của bản NestJS, nên `docs/db/api-gateway.md` §3.2 không đổi bản chất.
- Không dùng `kong.cache` cho JWKS trong DB-less mode: `kong.cache` gắn với vòng đời entity của config, không phù hợp cho dữ liệu có TTL độc lập lấy từ dịch vụ ngoài.
- Kích thước shared dict phải đủ cho số `kid` trong cửa sổ rotation overlap; hết bộ nhớ dict → plugin phải fail-closed (`503`), tuyệt đối không rơi về "bỏ qua verify".

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

Trong Kong, việc set/strip các header trên chia cho đúng hai plugin:

- `taca-request-guard` **xóa** `X-User-*`, `X-Auth-*` và `X-Forwarded-*` không đến từ trusted proxy chain.
- `taca-jwt` **set lại** `X-User-ID`, `X-User-Roles`, `X-User-Permissions`, `X-User-Shop-Scope`, `X-Auth-Method` bằng `kong.service.request.set_header()` — API này chỉ tác động lên request gửi tới upstream, không đổi request gốc, nên không có rủi ro phản hồi ngược về client.
- `X-Request-ID` do plugin `correlation-id` sinh; `taca-request-guard` chỉ validate giá trị client gửi và loại bỏ nếu sai charset/length.
- `X-Forwarded-For` chỉ tin khi `trusted_ips` trong `kong.conf` khai báo đúng dải proxy/ingress; để trống `trusted_ips` nghĩa là Kong không tin header và dùng IP kết nối trực tiếp — đây là cấu hình an toàn mặc định cho rate limit theo IP.

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
- Trong Kong, mỗi biến trở thành một **Upstream** (`name: up-auth-user`) có `targets`, và các Service `svc-auth-user-read`/`svc-auth-user-write` trỏ `host` vào tên Upstream đó. Không đặt hostname trực tiếp lên Service — làm vậy mất healthcheck và load balancing.
- decK thay biến môi trường khi render (`${AUTH_USER_BASE_URL}` trong `kong.yaml`, giá trị lấy từ `env/<môi trường>.yaml` hoặc `--set`); giá trị thật **không** commit vào Git.
- `deck validate` + `deck gateway diff` chạy trong CI trước khi `sync`: config sai scheme, thiếu host/port, `retries` khác 0 trên Service write, hoặc Route thiếu `taca-rbac` trên nhánh `/admin/**` đều phải fail pipeline. Đây là bản thay thế cho "fail-fast khi startup" của bản NestJS.
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
1. Route đã match → lấy instance plugin rate-limiting gắn trên Route/Service đó
2. Key: limit_by=ip cho bucket public/auth-endpoint;
        limit_by=header + header_name=X-User-ID cho bucket authenticated
3. policy=redis → atomic increment + expire trên Redis dùng chung mọi node Kong
4. Vượt limit → 429 + Retry-After + RateLimit-* headers
   → taca-error-envelope thay body thành GATEWAY_RATE_LIMITED
5. prometheus plugin ghi counter theo route/status; không ghi token/identity
```

Baseline đã chốt:

| Bucket | Limit | Key | Burst |
|---|---:|---|---:|
| Auth endpoint: signup/signin/OTP/reset | 10 req/phút | IP | 20 request ngắn hạn |
| Public API | 120 req/phút | IP | 20 request ngắn hạn |
| Authenticated API | 300 req/phút | `user_id` + route group | 20 request ngắn hạn |

- Thứ tự áp dụng: global IP guard → route bucket → user bucket nếu có JWT. Trong Kong, mỗi bucket là **một instance plugin `rate-limiting` riêng** gắn ở scope khác nhau (global cho IP guard, per-Route cho bucket còn lại); Kong cho phép nhiều instance cùng plugin ở các scope khác nhau cùng chạy.
- **Key Redis do plugin `rate-limiting` của Kong tự quản lý**, không còn dùng format `rl:v1:{bucket}:{identity}:{route_group}:{window}` như bản NestJS. Hệ quả: không được viết code/test/dashboard phụ thuộc vào format key này. Format cũ chỉ còn áp dụng cho key do custom plugin tự tạo (`ws:v1:conn:*`) — xem `docs/db/api-gateway.md` §3.1.
- Bucket authenticated dùng `limit_by=header` với `header_name=X-User-ID`. Header này **do `taca-jwt` đặt sau khi verify**, và `taca-request-guard` đã xóa mọi bản do client gửi ở bước trước — đây là lý do bắt buộc của thứ tự plugin ở §2.1.3. Nếu đảo thứ tự, client tự đặt `X-User-ID` sẽ chiếm được bucket của người khác hoặc né bucket của chính mình.
- `fault_tolerant = false` để Redis lỗi thì fail-closed (`503 GATEWAY_REDIS_UNAVAILABLE`). Mặc định của Kong là `true` (cho request đi qua khi Redis lỗi) — **giá trị mặc định này vi phạm policy §3.6 và phải bị chặn ở review/CI**.
- `policy = redis`, không dùng `local` (mỗi node đếm riêng, tổng limit sai gấp N lần số node) và không dùng `cluster` (yêu cầu database, không khả dụng ở DB-less).
- V1 dùng fixed window của plugin `rate-limiting`. Sliding window chính xác hơn nằm ở `rate-limiting-advanced` (Kong Enterprise) — nếu sau này cần, đó là một lý do nâng cấp license, xem §8 #14.

### 3.5 Timeout, retry và circuit breaker

```text
Request → route policy
  ├─ connect timeout → 504/503, không retry mutation
  ├─ idempotent GET connect/reset → retry tối đa 1 lần
  ├─ response timeout → 504, không retry POST/PATCH/PUT/DELETE
  └─ circuit OPEN → trả 503 ngay, không gọi upstream
```

- Timeout mặc định: `connect_timeout = 2000`, `read_timeout = 5000`, `write_timeout = 5000` (ms, trên Kong Service); checkout/payment/shipment dùng `read_timeout = 10000`.
- Retry chỉ cho GET/HEAD, thực hiện bằng cách tách Service `*-read` (`retries=1`) và `*-write` (`retries=0`) theo §2.1.4. Kong **không** phân biệt method khi retry, nên đây là cơ chế bắt buộc chứ không phải tùy chọn.
- `nginx_proxy_proxy_next_upstream = error timeout` — không thêm `http_500`/`non_idempotent`. Mặc định của nginx đã loại trừ request non-idempotent, nhưng phải khai báo tường minh để tránh phụ thuộc mặc định của phiên bản.
- **Circuit breaker được thực thi bằng `Upstream.healthchecks` của Kong**, không phải state machine riêng: passive healthcheck đếm lỗi và eject target (tương đương `OPEN`), active healthcheck probe định kỳ và đưa target trở lại (tương đương `HALF_OPEN` → `CLOSED`). Chi tiết cấu hình và ánh xạ trạng thái ở §5.2.
- Điểm khác biệt phải biết: healthcheck của Kong ở phạm vi **Upstream/target**, không phải theo `route_group` như thiết kế NestJS. Với topology hiện tại (mỗi service một Upstream), hiệu quả bảo vệ tương đương; nhưng khi mọi target của một Upstream bị eject, Kong trả lỗi ring-balancer → `taca-error-envelope` map sang `503 GATEWAY_UPSTREAM_UNAVAILABLE`.
- Kong lưu trạng thái healthcheck **theo từng node worker**, không chia sẻ giữa các node Kong. Hệ quả: khi upstream lỗi, mỗi node tự phát hiện; thời gian phát hiện toàn cụm không đồng thời. Đây là hành vi chấp nhận được ở v1 nhưng phải phản ánh vào alert (đừng cảnh báo "mất đồng bộ circuit state").
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

### 3.9 WebSocket upgrade — `GET /ws/messages`

```text
1. Client gửi HTTP GET /ws/messages với header Upgrade: websocket, Connection: Upgrade
   và access token qua Sec-WebSocket-Protocol (`bearer,<token>`) hoặc query `?access_token=`
2. Gateway kiểm CORS origin allowlist (WS cũng phải qua CORS/Origin check)
3. Rate limit handshake theo IP + user; kiểm WS_MAX_CONNECTIONS_PER_USER
4. Validate JWT RS256/JWKS như HTTP protected route (iss/aud/exp/nbf/kid)
   └─ sai/thiếu → trả 401 và KHÔNG upgrade
5. Strip header giả mạo, tạo actor context, gắn X-User-ID/X-Trace-ID
6. Mở TCP tunnel tới MESSAGE_BASE_URL, forward Upgrade request kèm actor headers
7. Sau khi upgrade: Gateway chỉ relay byte frame, không parse, không sửa, không buffer quá TCP window
8. Đóng socket khi: client/upstream đóng, quá WS_IDLE_TIMEOUT không có frame,
   Message Service báo lỗi, hoặc user bị revoke (theo Redis revoked_user_id của Gateway)
```

Ràng buộc:

- Gateway không giữ message history, không đảm bảo delivery; đó là trách nhiệm Message Service (REST cursor là source of truth khi reconnect).
- Không retry handshake và không tự reconnect socket; client tự reconnect và gọi `conversation.sync` qua REST.
- Handshake fail dùng cùng error envelope HTTP (`GATEWAY_AUTH_REQUIRED`, `GATEWAY_TOKEN_EXPIRED`, `GATEWAY_RATE_LIMITED`, `GATEWAY_UPSTREAM_UNAVAILABLE`).
- Circuit breaker cho `message` upstream áp cho cả REST và WS handshake; khi OPEN, handshake trả `503` ngay.
- Không log message frame; chỉ log sự kiện `ws.handshake`, `ws.open`, `ws.close` với `traceId`, `requestId`, `outcome`, `duration_ms` và connection count.

Cách Kong hiện thực luồng trên:

| Bước | Cơ chế Kong |
|---|---|
| Route `/ws/messages` | Kong Route thường (`protocols: [http, https]`, `paths: [/ws/messages]`, `methods: [GET]`) trỏ tới Service `svc-message-ws`. Kong tự nhận biết `Upgrade: websocket` và chuyển sang tunnel — không cần plugin đặc biệt. |
| Auth ở handshake | Plugin `access` chạy **trước** khi upgrade, nên `taca-jwt` hoạt động bình thường: đọc token từ `Sec-WebSocket-Protocol` hoặc query `access_token`, sai/thiếu → `kong.response.exit(401)` và không có `101`. |
| Connection cap | `taca-ws-guard`: `INCR` ở `access`, `DECR` ở `log`. Với WebSocket, phase `log` của Kong chạy khi **connection đóng**, không phải khi handshake xong — đúng ngữ nghĩa cần cho counter. |
| Idle timeout | `svc-message-ws.read_timeout = 1800000` ms, khớp `WEBSOCKET_IDLE_TIMEOUT` của Message Service. |
| Không retry | `svc-message-ws.retries = 0`. |
| Không buffer | Đây là hành vi mặc định của Kong với connection đã upgrade; **không** gắn bất kỳ plugin nào đọc/ghi body (`request-transformer`, `response-transformer`, `http-log` với `body`) lên Route này. |

Cảnh báo cấu hình: `taca-error-envelope` phải **bỏ qua** Route WebSocket sau khi đã upgrade — cố ghi body vào một connection đã chuyển sang chế độ tunnel sẽ làm hỏng frame. Plugin chỉ được tác động khi handshake thất bại (response HTTP thật, chưa upgrade).

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
| `WS_ALLOWED_PATH` | `/ws/messages` | path | Path WebSocket duy nhất được upgrade; path khác trả 404. |
| `WS_HANDSHAKE_TIMEOUT` | `5` | giây | Tối đa cho validate JWT + mở tunnel upstream. |
| `WS_IDLE_TIMEOUT` | `1800` | giây | Đồng bộ `WEBSOCKET_IDLE_TIMEOUT` của Message Service; không frame trong khoảng này thì đóng socket. |
| `WS_MAX_CONNECTIONS_PER_USER` | `10` | connection | Vượt trả `429` ở handshake; chống connection flood. |
| `WS_UPSTREAM` | `message` | service | Mọi WS chỉ proxy tới Message Service. |
| `LOG_BODY_ENABLED` | `false` | boolean | Không log body production. |
| `TIMESTAMP_STORAGE` | `UTC` | timezone | Log/response ISO-8601. |

### 4.1 Ánh xạ hằng số sang cấu hình Kong

Tên hằng số ở §4 là **ngôn ngữ chung của thiết kế**; bảng dưới cho biết mỗi hằng số thực sự nằm ở đâu trong Kong. Giá trị baseline không đổi khi migrate.

| Hằng số §4 | Vị trí thật trong Kong |
|---|---|
| `JWT_ALGORITHM`, `JWT_CLOCK_SKEW` | `taca-jwt.config.allowed_algs`, `taca-jwt.config.clock_skew` |
| `JWT_JWKS_CACHE_TTL`, `JWT_JWKS_MAX_STALE` | `taca-jwt.config.jwks_ttl`, `jwks_max_stale` + `lua_shared_dict taca_jwks` trong `kong.conf` |
| `AUTH_RATE_LIMIT` / `PUBLIC_RATE_LIMIT` / `AUTHENTICATED_RATE_LIMIT` | `rate-limiting.config.minute` trên từng instance plugin (scope Route/global) |
| `RATE_LIMIT_BURST` | Không có tương đương trực tiếp ở `rate-limiting` fixed-window; xem §8 #15 |
| `REDIS_COMMAND_TIMEOUT` | `rate-limiting.config.redis.timeout` (và cấu hình Redis của `taca-ws-guard`) |
| `UPSTREAM_CONNECT_TIMEOUT` | `Service.connect_timeout` |
| `UPSTREAM_READ_TIMEOUT_DEFAULT` / `_LONG` | `Service.read_timeout` (5000 / 10000 ms) |
| `UPSTREAM_RETRY_MAX` | `Service.retries` (`1` trên `*-read`, `0` trên `*-write`) |
| `UPSTREAM_RETRY_BACKOFF` | **Không cấu hình được** ở Kong OSS — nginx retry ngay, không backoff; xem §8 #15 |
| `CIRCUIT_FAILURE_THRESHOLD` | `Upstream.healthchecks.passive.unhealthy.http_failures` / `timeouts` |
| `CIRCUIT_WINDOW` | Không có cửa sổ thời gian ở passive healthcheck (đếm liên tiếp, không theo window); xem §5.2 và §8 #15 |
| `CIRCUIT_OPEN_DURATION` | `Upstream.healthchecks.active.healthy.interval` (chu kỳ probe đưa target trở lại) |
| `MAX_JSON_BODY_SIZE` | `request-size-limiting.config.allowed_payload_size` |
| `MAX_HEADER_SIZE` | `kong.conf` → `nginx_http_large_client_header_buffers` |
| `REQUEST_ID_MAX_LENGTH` | `taca-request-guard.config.request_id_max_length` |
| `CORS_ALLOWED_ORIGINS` | `cors.config.origins` **và** `taca-request-guard.config.allowed_origins` — hai nơi phải khớp, xem §8 #16 |
| `CORS_ALLOW_CREDENTIALS` | `cors.config.credentials` |
| `WS_ALLOWED_PATH` | `paths` của Route `rt-message-ws`; không có Route nào khác khớp `/ws/**` |
| `WS_HANDSHAKE_TIMEOUT` | `svc-message-ws.connect_timeout` |
| `WS_IDLE_TIMEOUT` | `svc-message-ws.read_timeout` |
| `WS_MAX_CONNECTIONS_PER_USER` | `taca-ws-guard.config.max_connections_per_user` |
| `METRICS_PATH` | Route nội bộ trỏ tới plugin `prometheus` |
| `HEALTH_LIVE_PATH` / `HEALTH_READY_PATH` | §2.1.5 |

Ba hằng số **không có tương đương native** (`RATE_LIMIT_BURST`, `UPSTREAM_RETRY_BACKOFF`, `CIRCUIT_WINDOW`) là các sai lệch thật của việc chuyển sang Kong OSS — đã ghi thành mục cần quyết định ở §8 #15, không được lặng lẽ bỏ qua.

## 5. Enum & trạng thái

### 5.1 `RouteAccess`

| Giá trị | Ý nghĩa | Điều kiện |
|---|---|---|
| `PUBLIC` | Không cần JWT | Vẫn qua CORS, rate limit và route policy. |
| `AUTHENTICATED` | Cần access JWT hợp lệ | User status/verification chi tiết do auth-user/service kiểm tra. |
| `ROLE_GATED` | Cần JWT và role/permission coarse | Service đích kiểm tra scope/ownership/2FA cuối cùng. |
| `WS_AUTHENTICATED` | WebSocket upgrade cần JWT hợp lệ ở handshake | Chỉ `/ws/messages`; sau upgrade Gateway chỉ relay frame, không kiểm từng message. |
| `INTERNAL_ONLY` | Chỉ internal/ops network | Không expose qua client ingress. |

### 5.2 `CircuitState` → trạng thái target của Kong Upstream

Kong không có object "circuit"; vai trò đó do **healthcheck trên Upstream** đảm nhiệm. Ba trạng thái logic của thiết kế ánh xạ như sau:

| `CircuitState` (thiết kế) | Trạng thái Kong | Ý nghĩa |
|---|---|---|
| `CLOSED` | Target `HEALTHY` | Request đi qua; passive healthcheck đếm lỗi trên traffic thật. |
| `OPEN` | Target `UNHEALTHY` (bị eject khỏi ring-balancer) | Kong không gửi request tới target. Nếu Upstream không còn target healthy nào → lỗi ring-balancer → `503 GATEWAY_UPSTREAM_UNAVAILABLE`. |
| `HALF_OPEN` | Active healthcheck probe | Kong chủ động gọi `healthchecks.active.healthy.http_path` theo `interval`; đủ số lần thành công thì target trở lại `HEALTHY`. |

Cấu hình tương ứng với baseline `5 lỗi / 30s`:

```yaml
upstreams:
  - name: up-order-commerce
    healthchecks:
      passive:                       # đếm trên traffic thật, không tốn request thừa
        unhealthy:
          http_failures: 5           # ~ CIRCUIT_FAILURE_THRESHOLD
          timeouts: 5
          tcp_failures: 5
      active:                        # đóng vai trò HALF_OPEN probe
        type: http
        http_path: /health/live
        healthy:
          interval: 30               # ~ CIRCUIT_OPEN_DURATION
          successes: 1
        unhealthy:
          interval: 10
          http_failures: 3
```

Ba khác biệt so với circuit breaker tự viết, **phải biết trước khi tin vào con số cũ**:

1. Passive healthcheck của Kong đếm **lỗi liên tiếp**, không đếm theo cửa sổ trượt `30s`. Traffic xen kẽ thành công/thất bại có thể không bao giờ chạm ngưỡng. Nếu cần ngữ nghĩa cửa sổ, phải dựa vào active healthcheck với `interval` ngắn.
2. Trạng thái healthcheck là **per-node**, không chia sẻ giữa các node Kong (§3.5).
3. Active healthcheck gọi `/health/live` của service — nghĩa là **mọi service upstream bắt buộc phải có endpoint đó** và endpoint phải nhẹ, không phụ thuộc database. Điều này đã đúng với 10 service hiện tại (`/health/live` process-only), nhưng giờ trở thành ràng buộc cứng chứ không còn là khuyến nghị.

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
| Distributed rate limit | Redis atomic counter + TTL, key do plugin `rate-limiting` quản lý | Gateway infrastructure | Config contract; không phải domain database. |
| Metrics/traces/log sink | OpenTelemetry-compatible exporter (plugin `opentelemetry` + `prometheus` + `http-log`) | Platform/ops | Endpoint thật chưa có trong HLD; dùng mock adapter. |
| Kong declarative config | `kong.yaml` + decK (`validate` → `diff` → `sync`) trong CI | Gateway owner + DevOps | Config là artifact được review như code; rollback = revert commit rồi `sync` lại. |
| Active healthcheck | REST `GET /health/live` của từng service | Từng domain service | **Ràng buộc mới**: mọi upstream phải có endpoint này, nhẹ và không phụ thuộc DB (§5.2). |

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
  "error": {
    "code": "ORDER_STOCK_UNAVAILABLE",
    "message": "Sản phẩm không còn đủ tồn kho.",
    "details": {
      "item_id": "item-01912f31"
    },
    "trace_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Gateway xử lý:

- Giữ `code` nghiệp vụ đã allowlist và `message` an toàn nếu upstream trả status 4xx.
- Xóa `details` nhạy cảm hoặc nội dung có internal host/stack trace.
- Nếu upstream không có `trace_id`, dùng `X-Request-ID` của Gateway.
- 5xx/timeout không trả raw body; map sang error code Gateway.

### 6.5 Mock contract — Gateway error envelope

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
  "trace_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
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
| 1 | Gateway là **Kong Gateway 3.x OSS** chạy DB-less, config bằng decK. Patch version cụ thể, base image và cách build custom plugin vào image chưa chốt. | Ảnh hưởng Docker image, priority của plugin built-in, dependency security và performance baseline. | Tech lead |
| 2 | Gateway route qua REST tới internal domain/IP bằng Upstream/target khai báo tĩnh trong decK; không dùng Consul/Eureka và không có route database. | Ảnh hưởng deployment, failover và cách thay đổi route. | Tech lead/DevOps |
| 3 | Auth-user cung cấp JWKS `GET /.well-known/jwks.json`, issuer/audience và RS256 key rotation. | Không thể validate JWT hoặc xử lý key rotation đúng nếu endpoint/claim khác. | Auth-user owner |
| 4 | Client gửi Bearer access token; refresh token transport (JSON response, HttpOnly cookie hay mobile secure storage) chưa chốt. | Ảnh hưởng CORS credentials, CSRF policy và frontend interceptor. | Frontend + Security |
| 5 | Redis dùng chung cho rate limit; topology HA, password/TLS, eviction policy và failure policy chưa chốt. | Ảnh hưởng availability và việc fail-closed khi Redis lỗi. | DevOps |
| 6 | CORS allowlist thật cho các ứng dụng Micro-Frontends (`mfe-shell`, `mfe-buyer`, `mfe-seller`, `mfe-admin`) chưa được cung cấp. | Nếu cấu hình sai, frontend bị chặn hoặc vô tình mở public origin. | Frontend/DevOps |
| 7 | Internal service có được truy cập trực tiếp bằng IP hay bắt buộc mTLS/network policy chưa chốt. | Nếu header context bị tin tuyệt đối, client có thể bypass qua đường nội bộ. | Security/DevOps |
| 8 | Message v1 dùng REST **và** WebSocket `/ws/messages`; Gateway validate JWT ở handshake, proxy TCP upgrade, không buffer/không retry/không tự reconnect. SSE không dùng trong v1. Subprotocol handshake (`Sec-WebSocket-Protocol`) cần Message Service xác nhận format token. | Nếu Message Service đổi handshake/subprotocol hoặc thêm SSE, phải cập nhật `ws-proxy` contract. | Product + frontend + Message owner |
| 9 | Route ownership đã chốt: `/api/v1/shops/{id}` + `/api/v1/shops/{id}/follow` → `auth-user`; `/api/v1/shops/{id}/products` → `product-catalog`; `/api/v1/vouchers/**` (buyer validate) + `/api/v1/seller/vouchers/**` + `/api/v1/admin/vouchers/**` → `order-commerce`; `/api/v1/seller/wallet/**` + `/api/v1/seller/payouts/**` + `/api/v1/seller/revenue` + `/api/v1/admin/fees/**` + `/api/v1/admin/taxes/**` + `/api/v1/admin/settlements/**` + `/api/v1/admin/finance/**` → `payment-wallet`; `/api/v1/admin/catalog/**` → `product-catalog`; `/api/v1/admin/users/**` + `/api/v1/admin/shops/**` → `auth-user`; `/api/v1/notifications/**` → `notification`. | Route nhầm upstream gây duplicate API hoặc sai source of truth. | Backend leads |
| 13 | Phạm vi Admin/back-office đã chốt (xem `System_Overview.md` §6.3): v1 **không thêm microservice**; mỗi màn admin đi qua `/api/v1/admin/**` trên service sở hữu dữ liệu. `dispute` và `campaign` là service riêng ở v1.1 — khi có, Gateway thêm route family mới `/api/v1/admin/disputes/**` và `/api/v1/admin/campaigns/**`. | Nếu sau này tách service admin gộp, phải thiết kế lại route + auth model. | Architecture owner |
| 10 | Observability backend/exporter và retention chưa được chỉ định; LLD chỉ chuẩn hóa adapter/field. | Ảnh hưởng dashboard, alert, trace sampling và chi phí lưu log. | Platform/DevOps |
| 11 | V1 không cache business response và không tự phát business event. | Nếu cần CDN/cache hoặc audit event qua Kafka, cần thêm module và contract. | Architecture owner |
| 12 | Error envelope `{error:{code,message,details,trace_id}}` là contract áp dụng thống nhất cho toàn bộ Gateway và upstream services. | Đảm bảo tính nhất quán trên toàn bộ hệ thống API. | Backend leads |
| 14 | Chọn **Kong OSS + 5 custom Lua plugin** thay vì Kong Enterprise/Konnect. Enterprise sẽ thay `taca-jwt` bằng plugin `openid-connect` và `taca-error-envelope` bằng `exit-transformer`, tức bỏ được 2/5 plugin tự viết, đổi lại là chi phí license. | Nếu team không có năng lực Lua hoặc không muốn tự bảo trì plugin auth, đây là quyết định phải đảo sớm — càng muộn càng tốn. | Tech lead + Architecture owner |
| 15 | Ba hằng số §4 **không có tương đương native** ở Kong OSS: `RATE_LIMIT_BURST` (fixed-window không có burst riêng), `UPSTREAM_RETRY_BACKOFF` (nginx retry ngay, không delay), `CIRCUIT_WINDOW` (passive healthcheck đếm lỗi liên tiếp, không theo cửa sổ thời gian). | Nếu vẫn coi baseline cũ là cam kết, hành vi thực tế sẽ khác tài liệu. Phải chọn: chấp nhận ngữ nghĩa của Kong, hoặc viết thêm plugin, hoặc nâng lên `rate-limiting-advanced`. | Architecture owner + DevOps |
| 16 | `CORS_ALLOWED_ORIGINS` bị khai báo ở **hai nơi** (`cors.config.origins` cho response header và `taca-request-guard` cho việc trả `403`) vì plugin `cors` của Kong không từ chối origin lạ bằng `403` mà chỉ bỏ header. | Hai danh sách lệch nhau sẽ tạo lỗ hổng hoặc chặn nhầm frontend. Cần sinh cả hai từ một nguồn duy nhất trong decK và có test đối chiếu. | Frontend/DevOps + Security |
| 17 | Bucket rate limit theo user dựa vào `limit_by=header` đọc `X-User-ID` do `taca-jwt` đặt. Hành vi này phụ thuộc thứ tự plugin và cách plugin `rate-limiting` đọc header trong đúng phiên bản Kong. | Nếu sai, hoặc rate limit theo user không hoạt động, hoặc client tự đặt header để chiếm/né bucket. **Bắt buộc có integration test khóa hành vi này** (IT-RL-04, IT-RL-13). | Gateway owner + Security |
| 18 | Custom plugin không được truy cập `kong.db` hay Admin API lúc runtime; mọi cấu hình đến từ `schema.lua`. | Giữ Gateway stateless và DB-less; vi phạm sẽ phá vỡ khả năng scale ngang và rollback bằng config. | Gateway owner |
