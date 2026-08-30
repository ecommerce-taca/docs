# Test plan — API Gateway Service

> Nguồn: `docs/api/api-gateway.md` · `docs/lld/api-gateway.md` · `docs/db/api-gateway.md` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

### 1.1 Môi trường và fixture

| Mục | Nội dung |
|---|---|
| Môi trường | Local CI và staging; chạy tối thiểu 2 **node Kong** để kiểm distributed rate limit và hành vi healthcheck per-node. |
| Kong | Kong Gateway 3.x OSS, `database = off`; image có sẵn 5 custom plugin `taca-*` và `KONG_PLUGINS=bundled,taca-request-guard,taca-jwt,taca-rbac,taca-ws-guard,taca-error-envelope`. |
| Config | `kong.yaml` sinh bằng decK; CI chạy `deck validate` và `deck gateway diff` trước mọi `sync`. Có fixture config sai (thiếu plugin trên route admin, `retries` khác 0 trên Service write, Route catch-all) để kiểm pipeline **fail đúng**. |
| Custom plugin | Test đơn vị bằng `busted` cho từng plugin; chạy được độc lập không cần Kong đầy đủ. |
| Upstream mock | Mock `auth-user`, `product-catalog`, `search`, `order-commerce`, `inventory`, `payment-wallet`, `shipment`, `rating-comment`, `notification`, `message`. |
| Redis | Redis test riêng; có fixture allow, exhausted, timeout, unavailable; không dùng production key. |
| JWKS | RSA test key `kid=key-01`, key rotation `key-02`, invalid/expired/no-key fixture. |
| Config | Static route map qua internal domain mock; test route thiếu/sai URL/readiness fail. |
| Observability | Test exporter nhận JSON log/metric/trace; có log scanner để tìm token/PII. |
| Client | Curl/Postman cho HTTP; browser automation cho CORS/preflight; load tool cho concurrency. |
| Time | Fake clock hoặc time control để kiểm `10m` JWKS, `30m` stale, timeout/circuit windows. |

### 1.2 Tài khoản/token fixture

| Fixture | Claims/quyền | Mục đích |
|---|---|---|
| `public_client` | Không JWT | Public route/rate limit/CORS. |
| `buyer_token` | `sub`, `roles=[BUYER]`, valid issuer/audience | Authenticated buyer route. |
| `seller_token` | `SELLER`, `shop_id=shop-1` | Seller route/coarse gate. |
| `staff_token` | `SELLER_STAFF`, shop scope `shop-1` | Staff route/scope. |
| `risk_admin_token` | `RISK_MANAGER`, KYC permissions | Admin route. |
| `catalog_admin_token` | `CATALOG_ADMIN`, catalog permission | Product/admin route. |
| `finance_admin_token` | `FINANCE_OPS` | Payment/wallet route. |
| `expired_token` | `exp` trong quá khứ | 401 token expiry. |
| `wrong_issuer_token` | `iss` sai | 401 validation. |
| `wrong_audience_token` | `aud` sai | 401 validation. |
| `invalid_signature_token` | Ký bằng key không có trong JWKS | 401 validation. |
| `new_kid_token` | `kid=key-02` | JWKS refresh/rotation. |

## 2. Test thủ công (QA / nghiệm thu)

| Mã | Màn hình / Luồng | Tiền điều kiện | Các bước | Kết quả mong đợi | Mức |
|---|---|---|---|---|---|
| TC-GW-01 | Buyer browse product | Product mock UP | Mở Home/Product Detail/Search | Request đi qua Gateway tới đúng service; UI nhận body/pagination không đổi. | Cao |
| TC-GW-02 | Buyer profile/cart | Có buyer token | Mở Account/Cart/Checkout | Bearer được gửi; request không token bị 401; `traceId` xuất hiện khi lỗi. | Cao |
| TC-GW-03 | Seller product | Có seller token | Tạo/sửa/publish sản phẩm | Gateway cho qua coarse role; Product Service quyết định ownership/KYC/business state. | Cao |
| TC-GW-04 | Admin KYC | Có risk admin token + MFA | Mở queue/review KYC | Route đúng auth-user; thiếu role/MFA bị chặn trước upstream. | Cao |
| TC-GW-05 | CORS allowed origin | Origin trong allowlist | Gọi preflight và actual request | Preflight 204; response có allow-origin chính xác, không wildcard. | Cao |
| TC-GW-06 | CORS denied origin | Origin ngoài allowlist | Gọi preflight/request | Bị 403; không forward upstream. | Cao |
| TC-GW-07 | Loading/error state | Upstream delay/500 | Mở Product/Orders/Message | Frontend nhận timeout/503/error envelope; retry UI hoạt động theo method. | Cao |
| TC-GW-08 | Offline state | Tắt mạng client | Submit profile/checkout | UI báo offline; không tạo duplicate write khi mạng trở lại. | Cao |
| TC-GW-09 | Rate limit public | IP fixture | Gửi >120 request/phút | Request thứ vượt limit nhận 429, Retry-After; các IP khác không bị ảnh hưởng. | Cao |
| TC-GW-10 | Rate limit auth | User fixture | Gửi >300 request/phút | User bucket bị 429; bucket public/IP vẫn đúng. | Cao |
| TC-GW-11 | Upstream timeout | Mock read timeout | Mở route GET/checkout | Error message thân thiện, trace ID; không hiển thị internal host. | Cao |
| TC-GW-12 | Circuit breaker (Upstream healthcheck) | Mock upstream fail liên tiếp 5 lần | Gọi lặp route | Target bị eject, request trả 503 nhanh; active healthcheck đưa target trở lại sau `interval`. Lưu ý: Kong đếm **lỗi liên tiếp**, không theo cửa sổ 30s — kịch bản thất bại xen kẽ có thể không chạm ngưỡng (LLD §5.2). | Cao |
| TC-GW-13 | JWKS rotation | Có key-02 | Gửi token kid mới | Gateway refresh một lần, token hợp lệ; không tạo refresh storm. | Cao |
| TC-GW-14 | Spoofed headers | Client gửi X-User-ID/role | Gọi protected route | Header client bị strip; upstream nhận actor từ JWT thật. | Cao |
| TC-GW-15 | Request ID | Gửi ID hợp lệ/sai/quá dài | Gọi bất kỳ route | ID hợp lệ được propagate; sai được tạo ID mới hoặc 400 theo policy; log/response cùng trace. | TB |
| TC-GW-16 | Body limit | JSON 1MiB và 1MiB+1 | POST profile/checkout | Đúng limit được proxy; vượt limit trả 413; không gọi upstream. | Cao |
| TC-GW-17 | KYC/message upload | Signed URL flow | Upload metadata/complete | Gateway chỉ proxy request nhỏ; file bytes đi storage, không bị proxy qua Gateway. | TB |
| TC-GW-18 | Health/readiness | Tắt Redis/JWKS/mock upstream | Mở health endpoints | Liveness phản ánh process; readiness phản ánh dependency; không lộ secret/IP. | Cao |
| TC-GW-19 | Metrics/logging | Exporter hoạt động | Gọi public/protected/error | Metric route/status/outcome tăng; log có request ID; không có token/PII. | Cao |
| TC-GW-20 | Responsive/admin error | Browser desktop/mobile/tablet | Kiểm tra error states | Envelope map đúng UI, không overflow message/table; retry action đúng method. | TB |
| TC-GW-21 | Realtime chat handshake | Có buyer token | Mở `/ws/messages` với/không token, token hết hạn, sai origin | Có token hợp lệ → `101` và nhận message realtime; thiếu/hết hạn → `401` không upgrade; origin ngoài allowlist → `403`. | Cao |
| TC-GW-22 | Realtime chat reconnect | Đang có socket mở | Ngắt mạng client rồi kết nối lại | Gateway không tự reconnect; client reconnect + `conversation.sync` REST lấy message miss; không mất/không nhân đôi message. | Cao |

Mức: `Cao` (chặn phát hành) · `TB` · `Thấp`.

## 3. Test API (integration)

### 3.1 Health và CORS

| Mã | Endpoint | Input | HTTP | Response mong đợi | Ghi chú |
|---|---|---|---:|---|---|
| IT-GW-01 | `GET /health/live` | Process healthy | 200 | `status=UP`, service name/time | Không gọi Redis/upstream. |
| IT-GW-02 | `GET /health/live` | Redis down | 200 | Liveness vẫn UP | Process còn sống. |
| IT-GW-03 | `GET /health/ready` | Config/JWKS/Redis UP | 200 | Checks UP | Ready nhận traffic. |
| IT-GW-04 | `GET /health/ready` | Config invalid | 503 | `GATEWAY_CONFIG_INVALID` | Không expose secret. |
| IT-GW-05 | `GET /health/ready` | JWKS unavailable quá stale | 503 | `GATEWAY_JWKS_UNAVAILABLE` | Protected traffic fail-closed. |
| IT-GW-06 | `GET /health/ready` | Redis unavailable | 503 | `GATEWAY_REDIS_UNAVAILABLE` | Không local fallback. |
| IT-GW-07 | `GET /metrics` | Internal observability IP | 200 | OpenMetrics text | Labels không có PII. |
| IT-GW-08 | `GET /metrics` | Public client | 403/404 | Không expose metrics | Ingress policy. |
| IT-GW-09 | `OPTIONS /api/v1/products` | Allowed origin | 204 | CORS allow headers/methods | `Vary: Origin`. |
| IT-GW-10 | `OPTIONS /api/v1/products` | Denied origin | 403 | `GATEWAY_CORS_DENIED` | Không forward. |
| IT-GW-11 | `OPTIONS /api/v1/products` | Header ngoài allowlist | 403/400 | Preflight rejected | Không wildcard. |

### 3.2 Route và authentication

| Mã | Route | Input | HTTP | Response mong đợi | Ghi chú |
|---|---|---|---:|---|---|
| IT-GW-12 | `GET /api/v1/products` | Không JWT | 200/Upstream | Đúng product-catalog | Public rate limit. |
| IT-GW-13 | `GET /api/v1/search` | Không JWT | 200/Upstream | Đúng search | GET retry được. |
| IT-GW-14 | `GET /api/v1/users/me` | Không JWT | 401 | `GATEWAY_AUTH_REQUIRED` | Không gọi auth-user. |
| IT-GW-15 | `GET /api/v1/users/me` | Valid buyer JWT | 200 | Upstream response | Actor headers đúng. |
| IT-GW-16 | `POST /api/v1/seller/products` | Buyer JWT | 403 | `GATEWAY_PERMISSION_DENIED` | Không gọi Product Catalog. |
| IT-GW-17 | `POST /api/v1/seller/products` | Seller JWT | Upstream | Forward | Product kiểm ownership/KYC. |
| IT-GW-18 | `GET /api/v1/admin/shops/kyc` | Catalog admin thiếu KYC_READ | 403 | Permission denied | Coarse gate. |
| IT-GW-19 | `GET /api/v1/admin/shops/kyc` | Risk admin JWT | Upstream | Auth-user response | Step-up chỉ service yêu cầu. |
| IT-GW-20 | `POST /api/v1/payments` | Valid buyer/seller JWT | Upstream | Không retry | Payment idempotency ở service. |
| IT-GW-21 | `POST /api/v1/unknown` | Bất kỳ | 404 | `GATEWAY_ROUTE_NOT_FOUND` | Không internal call. |
| IT-GW-22 | Route family `/messages` | Valid JWT | Upstream | REST proxy đúng Message Service. |
| IT-GW-23 | `GET /ws/messages` handshake | Valid JWT ở `Sec-WebSocket-Protocol` | 101 | Tunnel mở tới Message Service; actor headers forward; token không xuất hiện trong log. |
| IT-GW-24 | `GET /ws/messages` thiếu token | Không token | 401 | `GATEWAY_AUTH_REQUIRED`; không mở tunnel. |
| IT-GW-25 | `GET /ws/messages` token hết hạn | `expired_token` | 401 | `GATEWAY_TOKEN_EXPIRED`; không upgrade. |
| IT-GW-26 | `GET /ws/messages` vượt connection cap | > `WS_MAX_CONNECTIONS_PER_USER` | 429 | `GATEWAY_RATE_LIMITED`; các connection cũ không bị đóng. |
| IT-GW-27 | `GET /ws/messages` khi Message circuit OPEN | Mock message down | 503 | `GATEWAY_UPSTREAM_UNAVAILABLE` ngay ở handshake. |
| IT-GW-28 | WS idle timeout | Socket mở, không frame | Close | Gateway đóng socket sau `WS_IDLE_TIMEOUT`; ghi `ws.close` với outcome. |
| IT-GW-29 | WS token qua query `?access_token=` | Valid token | 101 | Upgrade OK; query token bị redact trong access log/metric. |
| IT-GW-30 | `GET /api/v1/admin/settlements` | Finance admin JWT | Upstream | Route đúng `payment-wallet`; Gateway coarse-gate role admin, service enforce `FINANCE_OPS`. |
| IT-GW-31 | `PUT /api/v1/admin/fees` | Buyer/seller JWT | 403 | `GATEWAY_PERMISSION_DENIED`; không gọi payment-wallet. |
| IT-GW-32 | `GET /api/v1/admin/catalog/products` | Catalog admin JWT | Upstream | Route đúng `product-catalog`. |
| IT-GW-33 | `GET /api/v1/admin/dashboard` | Bất kỳ admin JWT | 404 | `GATEWAY_ROUTE_NOT_FOUND`; Gateway không có endpoint dashboard (tầng đọc do `mfe-admin`/BFF). |

### 3.2.1 Rủi ro riêng của Kong (bắt buộc, không có ở bản NestJS)

Các case dưới đây khóa những chỗ Kong có **hành vi mặc định khác** với thiết kế. Thiếu chúng thì lỗi chỉ lộ ra ở production.

| Mã | Tình huống | HTTP | Kết quả mong đợi |
|---|---|---:|---|
| IT-KONG-01 | Bất kỳ lỗi nào do Kong tự sinh (route không match, 413, 429) | — | Body **luôn** là `{error:{code,message,details,trace_id}}`; không bao giờ lọt body mặc định `{"message": ...}` của Kong. |
| IT-KONG-02 | Client gửi `X-User-ID` giả trên route authenticated | Upstream | Upstream nhận `X-User-ID` từ JWT, không phải giá trị client gửi; **và** bucket rate limit tính theo user thật (chứng minh `taca-request-guard` chạy trước `taca-jwt`). |
| IT-KONG-03 | Hai user khác nhau, một user gửi `X-User-ID` của user kia | 200/429 | Không chiếm được và không né được bucket của user kia. |
| IT-KONG-04 | `POST` tới route có upstream reset connection | 502/503/504 | Kong **không** retry; upstream chỉ nhận đúng 1 request (đếm ở mock). Khóa `Service.retries = 0` trên `*-write`. |
| IT-KONG-05 | `GET` tới route có upstream reset connection lần đầu | 200 | Retry đúng 1 lần trên `*-read`; không retry lần thứ hai. |
| IT-KONG-06 | Config có Service `*-write` với `retries` khác 0 | — | `deck validate`/lint CI **fail**; không sync được. |
| IT-KONG-07 | Config có Route `/admin/**` thiếu plugin `taca-rbac` | — | Lint CI **fail**; không sync được. |
| IT-KONG-08 | Config có Route catch-all `/api/v1` | — | Lint CI **fail**; Route catch-all vô hiệu hóa gate theo nhánh. |
| IT-KONG-09 | Redis down + `rate-limiting` | 503 | `GATEWAY_REDIS_UNAVAILABLE`. Khóa `fault_tolerant=false` — mặc định `true` của Kong sẽ cho request đi qua và làm case này fail. |
| IT-KONG-10 | Mọi target của một Upstream bị eject | 503 | `GATEWAY_UPSTREAM_UNAVAILABLE`; không lộ thông báo ring-balancer của Kong. |
| IT-KONG-11 | Upstream phục hồi sau khi bị eject | 200 | Active healthcheck đưa target trở lại trong khoảng `interval` đã cấu hình. |
| IT-KONG-12 | Gọi Admin API (`:8001`) từ mạng client | Không reachable | Admin API không expose qua ingress ở mọi môi trường. |
| IT-KONG-13 | Thứ tự plugin trong `access` | — | Assert thứ tự thực thi thật: `taca-request-guard` → `request-size-limiting` → `taca-jwt` → `taca-rbac` → `rate-limiting`. Test phải fail nếu ai đó đổi `PRIORITY`. |
| IT-KONG-14 | Origin ngoài allowlist | 403 | `GATEWAY_CORS_DENIED` — plugin `cors` của Kong không tự trả 403, case này khóa `taca-request-guard`. |
| IT-KONG-15 | `cors.config.origins` và `taca-request-guard.allowed_origins` lệch nhau | — | Lint CI **fail** (hai danh sách phải sinh từ một nguồn). |
| IT-KONG-16 | Route `/ws/messages` sau khi upgrade | — | `taca-error-envelope` không ghi gì vào connection đã upgrade; frame không bị hỏng. |
| IT-KONG-17 | Node Kong bị kill khi còn socket WS mở | — | Counter `ws:v1:conn:*` tự hết hạn theo TTL; user không bị khóa kết nối vĩnh viễn. |
| IT-KONG-18 | `lua_shared_dict taca_jwks` đầy | 503 | `GATEWAY_JWKS_UNAVAILABLE`; **không** bỏ qua verify. |

### 3.3 JWT/JWKS

| Mã | Tình huống | HTTP | Kết quả |
|---|---|---:|---|
| IT-JWT-01 | Signature hợp lệ, key cached | 200/upstream | Request protected được proxy. |
| IT-JWT-02 | `alg=none` | 401 | `GATEWAY_TOKEN_INVALID`. |
| IT-JWT-03 | `alg=HS256` | 401 | Không fallback algorithm. |
| IT-JWT-04 | Issuer sai | 401 | Không gọi upstream. |
| IT-JWT-05 | Audience sai | 401 | Không gọi upstream. |
| IT-JWT-06 | `exp` quá khứ | 401 | `GATEWAY_TOKEN_EXPIRED`. |
| IT-JWT-07 | `nbf` tương lai quá clock skew | 401 | Invalid token. |
| IT-JWT-08 | Kid mới + JWKS refresh 200 | 200/upstream | Validate lại sau refresh. |
| IT-JWT-09 | Kid mới + JWKS 500 | 503 | `GATEWAY_JWKS_UNAVAILABLE`, fail-closed. |
| IT-JWT-10 | Key cũ trong rotation overlap | 200/upstream | Không logout đột ngột trong overlap. |

### 3.4 Rate limit/proxy failure

| Mã | Tình huống | HTTP | Kết quả |
|---|---|---:|---|
| IT-RL-01 | Public request trong limit | 200/upstream | Remaining giảm đúng. |
| IT-RL-02 | Public request vượt 120/phút/IP | 429 | Retry-After/RateLimit headers. |
| IT-RL-03 | Auth endpoint vượt 10/phút/IP | 429 | Không gọi Auth User. |
| IT-RL-04 | Authenticated vượt 300/phút/user | 429 | User bucket độc lập IP khác. |
| IT-RL-05 | Redis INCR timeout | 503 | `GATEWAY_REDIS_UNAVAILABLE`; fail-closed. |
| IT-RL-06 | Upstream GET connect reset | Upstream/200 | Retry tối đa 1 lần. |
| IT-RL-07 | Upstream POST connect reset | 503/504 | Không retry/không duplicate write. |
| IT-RL-08 | Upstream 400 error envelope | 400 | Preserve allowlisted business code. |
| IT-RL-09 | Upstream 500 raw stack | 502/503 | Map sanitized Gateway error. |
| IT-RL-10 | Upstream read timeout | 504 | `GATEWAY_UPSTREAM_TIMEOUT`. |
| IT-RL-11 | Circuit opens after 5 failures | 503 | Không gọi upstream khi OPEN. |
| IT-RL-12 | Active healthcheck probe success/fail | 200/503 | Target trở lại `HEALTHY`/giữ `UNHEALTHY` đúng (tương đương HALF_OPEN → CLOSED/OPEN). |
| IT-RL-13 | Bucket authenticated với `limit_by=header` | 429 | Limit áp theo `X-User-ID` do `taca-jwt` đặt; hai user độc lập nhau; user không tự đổi được bucket bằng header gửi lên. Khóa giả định LLD §8 #17. |

## 4. Unit test và lint cấu hình

Sau khi chuyển sang Kong, phần "unit test" chia làm hai loại: **test Lua cho custom plugin** (`busted`) và **lint cho declarative config** — vì phần lớn hành vi giờ nằm ở cấu hình chứ không ở code.

### 4.1 Unit test custom plugin (`busted`)

| Plugin / hàm | Case cần phủ |
|---|---|
| `taca-request-guard` | Origin allowed/denied; preflight method/header; validate `X-Request-ID` (hợp lệ, sai charset, >64 ký tự) → giữ hoặc sinh mới; xóa `X-User-*`/`X-Auth-*`/`X-Forwarded-*` không tin cậy; không đọc body. |
| `taca-jwt` — verify | RS256 only, từ chối `alg=none`/HS256/key từ client; `iss`/`aud` sai; `exp`/`iat`/`nbf` với clock skew; thiếu claim bắt buộc; `kid` không tồn tại. |
| `taca-jwt` — JWKS cache | Cache hit; TTL refresh; stale trong/quá `max_stale`; refresh **một lần** dưới lock khi nhiều request cùng gặp `kid` mới; key rotation overlap; fetch failure → fail-closed; shared dict đầy → fail-closed. |
| `taca-jwt` — actor context | Claims → `X-User-ID`/`X-User-Roles`/`X-User-Permissions`/`X-User-Shop-Scope`/`X-Auth-Method`; shop scope rỗng; marker `revoked_user:{sub}` → từ chối. |
| `taca-jwt` — token source | Đọc token từ `Authorization`, từ `Sec-WebSocket-Protocol` (`bearer, <token>`), từ query `access_token`; thứ tự ưu tiên; token ở nhiều nguồn cùng lúc. |
| `taca-rbac` | Đủ/thiếu role; đủ/thiếu permission; route không khai báo yêu cầu; không suy luận ownership từ `shop_id` trong path/body; không tự quyết định 2FA. |
| `taca-ws-guard` | `INCR` ở `access`, `DECR` ở `log`; vượt cap → `429`; TTL an toàn khi `log` không chạy; Redis lỗi → fail-closed. |
| `taca-error-envelope` | `get_source()` = `exit`/`error` → envelope chuẩn; `service` + 4xx allowlist → giữ business code, thêm `trace_id` nếu thiếu; `service` + 5xx → sanitize; đủ 9 dòng ánh xạ lỗi native ở LLD §2.1.6; bỏ qua connection đã upgrade. |
| Redaction (dùng chung) | `Authorization`, password, OTP, KYC, bank, `?access_token=`, `Sec-WebSocket-Protocol` bearer bị che trong mọi log/metric/trace. |

### 4.2 Lint declarative config

Chạy trong CI, fail pipeline nếu vi phạm — đây là nơi thay thế phần lớn `RouteMatcher`/`RoutePolicy`/`RetryPolicy` của bản NestJS.

| Quy tắc | Vì sao |
|---|---|
| Mọi Kong Service `*-write` có `retries = 0` | Mặc định của Kong là `5`; sai là retry mutation. |
| Mọi Kong Service `*-read` có `retries` ≤ 1 | Giữ đúng policy "tối đa 1 lần". |
| Mọi Route `/api/v1/admin/**` có plugin `taca-rbac` | Thiếu là mất coarse gate admin. |
| Mọi Route protected có plugin `taca-jwt` | Thiếu là route protected thành public. |
| Không có Route catch-all (`/api/v1`, `/`) | Catch-all vô hiệu hóa gate theo nhánh. |
| Không có Route nào khớp `/internal/**` | `/internal/**` không được expose. |
| Không có Route `/ws/**` ngoài `/ws/messages` | Path WS duy nhất được upgrade. |
| `rate-limiting` có `policy=redis` và `fault_tolerant=false` | Mặc định `fault_tolerant=true` vi phạm policy fail-closed. |
| `cors.origins` và `taca-request-guard.allowed_origins` khớp nhau | Hai danh sách lệch tạo lỗ hổng hoặc chặn nhầm. |
| `strip_path = false` trên mọi Route business | Giữ path tới upstream không đổi. |
| Route WS không gắn plugin đọc/ghi body | Làm hỏng frame sau upgrade. |
| Không có giá trị secret hard-code trong `kong.yaml` | Secret đến từ env/Vault. |

## 5. Security, resilience và performance test

### 5.1 Security

| Mã | Kiểm tra | Kết quả bắt buộc |
|---|---|---|
| SEC-GW-01 | Bypass Gateway bằng public/internal URL từ client network | Internal service không reachable hoặc bị network policy chặn. |
| SEC-GW-02 | Spoof `X-User-ID`, `X-User-Roles`, `X-MFA-Step-Up` | Header client bị strip; không privilege escalation. |
| SEC-GW-03 | JWT algorithm confusion | Chỉ RS256 được chấp nhận. |
| SEC-GW-04 | Token replay sau expire | Protected request 401; không cache auth decision quá TTL. |
| SEC-GW-05 | CORS wildcard/credential | Không cho `*` với credentials; origin allowlist exact. |
| SEC-GW-06 | Oversized JSON/header | 413/431 trước upstream; không consume memory bất thường. |
| SEC-GW-07 | Internal error leakage | Không có stack trace, host, IP, SQL, secret trong response. |
| SEC-GW-08 | Log/event scan | Không có access/refresh token, password, OTP hoặc PII raw. |
| SEC-GW-09 | Rate-limit bypass qua forwarded IP | Chỉ tin trusted proxy chain; không tự set X-Forwarded-For. |
| SEC-GW-10 | Metrics exposure | Public client không đọc `/metrics`, labels không có PII. |
| SEC-GW-11 | WS handshake không auth | `/ws/messages` không token/token sai → 401, không mở tunnel; không có đường bypass qua `/ws/**` path khác. |
| SEC-GW-12 | WS token trong query log | `?access_token=` và subprotocol bearer bị redact trong access log/metric/trace. |
| SEC-GW-13 | WS sau revoke | User bị suspend/revoke → socket đang mở bị đóng theo Redis `revoked_user_id`; handshake mới bị từ chối. |
| SEC-GW-14 | Kong Admin API expose | `:8001` không reachable từ mạng client ở mọi môi trường; không có Route nào proxy tới nó. |
| SEC-GW-15 | Rò rỉ thông tin runtime của Kong | Response lỗi không chứa `{"message":...}` mặc định của Kong, tên Service/Upstream nội bộ, hay thông báo ring-balancer. |

### 5.2 Resilience/performance

| Mã | Tình huống | Kết quả mong đợi |
|---|---|---|
| RES-GW-01 | 2+ Gateway instances cùng rate-limit bucket | Tổng limit đúng toàn cluster, không theo từng instance. |
| RES-GW-02 | JWKS rotation concurrent | Một refresh in-flight; không tạo request storm. |
| RES-GW-03 | Upstream payment slow | Timeout 10s; không retry write; circuit metric tăng. |
| RES-GW-04 | Product/search GET transient reset | Chỉ retry tối đa 1; không nhân đôi request vượt policy. |
| RES-GW-05 | Redis outage | Protected/public request fail-closed; readiness degraded. |
| RES-GW-06 | Gateway restart | Config load lại; JWKS fetch; không có domain state mất. |
| RES-GW-07 | Large concurrent headers/body | Process không OOM; reject theo max limits. |
| RES-GW-08 | Route table invalid | Startup/readiness fail; không nhận traffic sai upstream. |
| RES-GW-09 | Load public catalog | Ghi p50/p95/p99, throughput, CPU/memory và error rate; target do team điền. |
| RES-GW-10 | Load protected API | JWT cache hit, Redis latency và upstream latency tách riêng; không có unbounded queue. |

## 6. Tiêu chí pass và phát hành

| Tiêu chí | Điều kiện đạt |
|---|---|
| Route coverage | Mọi route family trong API spec có test route đúng service và unknown route. |
| Auth coverage | Có case thiếu/sai/hết hạn/JWKS rotation/token algorithm/role gate. |
| Error coverage | Mọi Gateway error code có ít nhất một integration case. |
| Security gate | SEC-GW-01 đến SEC-GW-15 pass; không leak secret/PII; Admin API không expose. |
| Rate-limit gate | Public/auth/authenticated buckets đúng khi chạy nhiều node Kong; IT-RL-13 pass. |
| Retry gate | Không retry write; GET retry tối đa 1; trạng thái target Upstream đúng. |
| Kong gate | IT-KONG-01 đến IT-KONG-18 pass. Đặc biệt IT-KONG-01 (không lọt error format của Kong), IT-KONG-02/03 (thứ tự plugin), IT-KONG-04 (không retry mutation), IT-KONG-09 (fail-closed khi Redis lỗi) — bốn case này khóa những mặc định của Kong đi ngược thiết kế. |
| Config gate | Toàn bộ quy tắc lint §4.2 chạy trong CI và **fail đúng** với fixture config sai; `deck gateway diff` trên staging không có drift. |
| WebSocket gate | `/ws/messages` handshake auth/connection-cap/idle-timeout/redaction/circuit pass; không tự reconnect/không buffer frame. |
| Health gate | Liveness/readiness/metrics không lộ dữ liệu nhạy cảm và phản ánh dependency. |
| UI gate | Các Micro-Frontends (`mfe-buyer`, `mfe-seller`, `mfe-admin`) error/loading/offline state nhận đúng envelope/tracing. |
| Performance gate | Có baseline p50/p95/p99 và không có memory leak/unbounded request. |
| Release gate | Không còn defect Critical/High; route/config rollback được kiểm tra. |

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Test full business payload của từng service nằm ở test plan service đó; Gateway chỉ test proxy/security/transport contract. | Cần chạy integration suite liên service trước release toàn hệ thống. | Backend leads |
| 2 | Gateway dùng Redis distributed rate limit, không local fallback production. | Redis HA/outage policy cần được DevOps kiểm chứng trên staging. | DevOps |
| 3 | JWKS endpoint/issuer/audience hiện dùng mock contract từ auth-user. | Cần contract test với auth-user thật trước production. | Auth-user owner |
| 4 | Refresh token dùng Bearer JSON flow; cookie/CSRF chưa thuộc test hiện tại. | Nếu chuyển cookie, bổ sung browser security suite. | Frontend/Security |
| 5 | Message v1 dùng REST + WebSocket `/ws/messages`; test suite phủ handshake auth, connection cap, idle timeout, reconnect và redaction token. SSE không test vì không dùng trong v1. | Nếu Message Service đổi handshake/subprotocol, cập nhật IT-GW-23..29. | Product/frontend/Message owner |
| 6 | Timeout/healthcheck thresholds là baseline LLD; performance target/SLO chưa chốt. | Không dùng baseline này làm SLA chính thức nếu chưa có capacity test. | Architecture/DevOps |
| 7 | Gateway là Kong 3.x OSS + 5 custom Lua plugin; test suite giả định image đã build sẵn plugin và CI chạy được decK. | Nếu team chuyển sang Kong Enterprise, IT-KONG-01/14 (error envelope, CORS 403) và một phần §4.1 sẽ được plugin `exit-transformer`/`openid-connect` thay thế — phải viết lại test tương ứng. | Tech lead + DevOps |
| 8 | `PRIORITY` của custom plugin phải khớp phiên bản Kong đang chạy; IT-KONG-13 là test khóa hành vi này. | Nâng phiên bản Kong có thể đổi priority của plugin built-in và âm thầm đảo thứ tự — phải chạy lại IT-KONG-13 trong mọi lần nâng cấp. | Gateway owner |
| 9 | Tên metric Prometheus của Kong phụ thuộc phiên bản; test dashboard/alert phải đối chiếu với `/metrics` thật. | Alert viết theo tên metric cũ (`gateway_*`) sẽ im lặng không bao giờ kêu. | DevOps |
