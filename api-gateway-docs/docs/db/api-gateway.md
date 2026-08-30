# Database — API Gateway Service

> Nguồn: `docs/lld/api-gateway.md` · `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Trạng thái: N/A — API Gateway không sở hữu domain database
> Runtime: **Kong Gateway 3.x OSS chạy DB-less** (`database = off`). Kong cũng **không** dùng PostgreSQL của riêng nó — cấu hình đến từ file declarative, nên kết luận "Gateway không có database" vẫn đúng nguyên vẹn sau khi migrate.

## 1. Quy ước chung

| Mục | Quy định |
|---|---|
| Domain database | Không có. Gateway là stateless edge service. |
| Kong datastore | Không có. `database = off` — Kong không tạo/dùng PostgreSQL. Mọi Service/Route/Plugin/Upstream nằm trong `kong.yaml` được decK sync. Hệ quả: không có migration của Kong, không có `kong migrations up`, và không thể tạo entity runtime qua Admin API. |
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

### 3.1 Redis key — không phải domain table

Sau khi chuyển sang Kong, các key trong Redis chia làm **hai nhóm có chủ sở hữu khác nhau**.

#### 3.1.1 Key do plugin `rate-limiting` của Kong quản lý

| Thành phần | Quy định |
|---|---|
| Format key | **Do Kong định nghĩa, không do chúng ta chọn.** Bản NestJS dùng `rl:v1:{bucket}:{identity_hash}:{route_group}:{window}` — format này **không còn hiệu lực**. |
| Value | Counter fixed-window do plugin quản lý. |
| Identity | `limit_by=ip` cho bucket public/auth-endpoint; `limit_by=header` + `header_name=X-User-ID` cho bucket authenticated. |
| Operation | Atomic increment + expire do plugin thực hiện. |
| Cấu hình bắt buộc | `policy=redis`, `fault_tolerant=false`, `redis.timeout=500`. |

> **Ràng buộc cho dev:** không viết code, test, script vận hành hay dashboard nào phụ thuộc vào format key rate limit. Nếu cần quan sát, dùng metric (`kong_http_requests_total{code="429"}`) chứ không scan Redis. Nếu cần một namespace riêng để tách môi trường, dùng `redis.database` khác nhau, không tự đặt prefix.

#### 3.1.2 Key do custom plugin `taca-*` quản lý

Đây là các key **chúng ta tự đặt tên và tự chịu trách nhiệm**, giữ nguyên quy ước cũ:

| Thành phần | Format | TTL | Nội dung |
|---|---|---:|---|
| WS connection counter | `ws:v1:conn:{user_id_hash}` | `INCR` ở phase `access`, `DECR` ở phase `log` (chạy khi connection đóng); kèm TTL an toàn để tự dọn nếu node chết | Số socket đang mở/user để enforce `WS_MAX_CONNECTIONS_PER_USER`. |
| Revoked user marker | `revoked_user:{user_id}` (dùng chung với HTTP) | 15 phút | Do Auth User đẩy vào khi suspend/revoke; `taca-jwt` từ chối handshake/request mới, `taca-ws-guard` đóng socket đang mở. |

TTL an toàn của connection counter là bắt buộc: nếu một node Kong bị kill, phase `log` không chạy và counter sẽ rò rỉ, dần dần khóa hết khả năng kết nối của người dùng đó.

Không dùng Redis key để xác nhận role, ownership, balance, order state hoặc inventory.

### 3.2 In-memory JWKS cache — không phải persistence

Lưu trong `lua_shared_dict taca_jwks` khai báo ở `kong.conf`, do plugin `taca-jwt` quản lý.

| Field | Kiểu | TTL/điều kiện |
|---|---|---|
| `kid` | string | Theo key rotation. |
| `public_key` | RSA public key (PEM/JWK serialize được vào shared dict) | Cache 10 phút; stale tối đa 30 phút. |
| `issuer`/`audience` | string | Đọc từ config plugin, không nhận từ client. |
| `loaded_at` | ISO timestamp | Dùng health/readiness. |

- `lua_shared_dict` chia sẻ giữa các nginx worker **trong cùng một node**, không giữa các node Kong. Mỗi node tự fetch JWKS — giống mô hình "mỗi instance tự cache" của bản NestJS, nên không có thay đổi về bản chất.
- Shared dict lưu được string/number, không lưu được Lua object phức tạp: plugin phải serialize key và parse lại, hoặc cache object đã parse trong `lua_resty_lrucache` cấp worker với shared dict làm lớp thứ hai.
- Dict đầy → plugin **fail-closed** (`503 GATEWAY_JWKS_UNAVAILABLE`), tuyệt đối không bỏ qua verify. Giám sát bằng `kong_memory_lua_shared_dict_bytes`.

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
| SQL/NoSQL migration | Không áp dụng. Kong chạy `database = off` nên cũng không có `kong migrations`. |
| Database seed | Không áp dụng. |
| Route/config seed | `kong.yaml` quản lý bằng decK trong Git, review như code. CI chạy `deck validate` → `deck gateway diff` → `deck gateway sync`. |
| Redis bootstrap | Không cần seed; key được tạo theo request và tự hết TTL. |
| JWKS bootstrap | Fetch từ auth-user khi startup/readiness; không seed key thủ công. |
| Rollback | Revert commit config rồi `deck gateway sync` lại, hoặc rollback image nếu lỗi nằm ở custom plugin. Không rollback database. |
| Drift detection | `deck gateway diff` chạy định kỳ trên môi trường thật; khác biệt so với Git là sự cố cấu hình phải điều tra, không được sync đè im lặng. |

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Gateway không có domain database; README/API project ghi Database là N/A. | Nếu sau này lưu dynamic route/audit trong DB, phải tạo schema và migration riêng. | Architecture owner |
| 2 | Redis là infrastructure state cho distributed rate limit, không phải source of truth. | Nếu Redis mất, cần giữ fail-closed hoặc có policy fallback được phê duyệt. | DevOps/Security |
| 3 | Route/config quản lý bằng decK + Git; cách nạp secret (Redis password, JWKS URL) qua env hay Kong Vault chưa chốt. | Ảnh hưởng rollout/rollback và secret rotation. | DevOps |
| 4 | JWKS cache chỉ trong `lua_shared_dict`; mỗi **node Kong** tự refresh cùng endpoint auth-user. | Restart node cần fetch lại key; cần auth-user availability/readiness. Scale nhiều node làm tăng tần suất gọi JWKS theo bội số node. | Auth-user owner |
| 5 | Format key rate limit thuộc plugin `rate-limiting` của Kong, không phải contract của team. | Mọi công cụ vận hành phụ thuộc format key cũ sẽ hỏng; phải quan sát bằng metric thay vì đọc Redis. | DevOps |
| 6 | Redis dùng chung cho plugin `rate-limiting` và cho custom plugin `taca-ws-guard`/revoke marker; có tách database/instance hay không chưa chốt. | Dùng chung một database làm khó phân tách quota, eviction và sự cố. | DevOps/Security |
