# LLD — Auth User Service

> Nguồn: `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Tech stack: Java 25 · Spring Boot (pin version trong repository BOM) · MySQL 8.4 (`userdb`) · Kafka · S3/MinIO

## 1. Phạm vi

| Mục | Nội dung |
|---|---|
| Trách nhiệm chính | Sign up/sign in/sign out, JWT access/refresh token, email/phone verification, password reset, buyer profile, address book, seller onboarding, shop/KYC metadata, RBAC và admin 2FA, **product favorites (wishlist)**, **follow shop**, **public shop profile**. |
| Nguồn dữ liệu chính | MySQL 8.4 database riêng `userdb`; không tạo cross-service foreign key. Các service khác chỉ giữ ID tham chiếu và nhận event. |
| Tài liệu liên quan | Chi tiết cột, kiểu dữ liệu, constraint và migration nằm ở `docs/db/auth-user.md`; request/response đầy đủ nằm ở `docs/api/auth-user.md`. |
| Không thuộc service này | Product/SPU/SKU, category, inventory, cart, order, voucher, payment, wallet, shipment, review, message content và product moderation. Favorites/follow chỉ lưu **reference ID** (`product_id`, `shop_id`); nội dung sản phẩm/thẻ sản phẩm do Product Catalog cung cấp. |
| Phụ thuộc vào | `api-gateway`, Kafka, Notification Service, S3/MinIO private bucket và hệ thống tạo email/SMS của Notification Service. |
| Được gọi bởi | Các module Micro-Frontends (MFE) (`mfe-buyer`, `mfe-seller`, `mfe-admin`, `mfe-shell`) qua API Gateway; API Gateway gọi trực tiếp cho auth/profile; các domain service dùng event hoặc ID, không đọc `userdb`. |
| Chính sách seller | Tạo shop và vào Seller Center không đồng nghĩa KYC approved. Không duyệt từng sản phẩm; KYC chỉ điều khiển quyền publish và withdraw theo trạng thái shop. |

### Màn hình và luồng phụ thuộc

| Màn hình trong Penpot | Luồng chính | Quyền |
|---|---|---|
| `Overlay / Sign in` | Sign in bằng email hoặc phone, forgot password, 2FA challenge | Public → unauthenticated / Admin step-up |
| `Taca Buyer / Account`, `Account / Profile` | Xem/cập nhật profile, thêm/sửa/xóa address | Buyer đã đăng nhập |
| `Taca Buyer / Favorite Products`, `Favorites / Main`, `Favorite product / …`, `Account icon / Sản phẩm yêu thích` | Xem danh sách sản phẩm yêu thích, thêm/bỏ yêu thích, tìm trong danh sách | Buyer đã đăng nhập |
| `Taca Buyer / Shop`, `Shop / Hero`, `CTA / + Theo dõi`, `PDP / Shop card` | Xem hồ sơ shop công khai, theo dõi/bỏ theo dõi shop, số người theo dõi | Xem: public; theo dõi: buyer đã đăng nhập |
| `Overlay / Seller onboarding` | Hồ sơ → KYC → kho/vận chuyển → ngân hàng → sản phẩm đầu tiên | Buyer đã đăng nhập, có seller onboarding |
| `State Mobile / Permission KYC` | Hiển thị KYC thiếu/hết hạn và hành động bổ sung hồ sơ | Seller |
| `Taca Admin / Shops / KYC`, `Overlay / KYC review` | Queue và quyết định KYC | `RISK_MANAGER`, `SUPER_ADMIN` |
| `Taca Admin / Users / Roles`, `Overlay / Admin role editor` | Quản lý admin role/permission và audit | `SUPER_ADMIN`, role được cấp quyền |
| Seller Center `Settings` | Thông tin shop, kho, vận chuyển, tài khoản ngân hàng, thông báo | Seller / seller staff theo permission |

### Boundary với các service khác

```text
Client
  ↓ HTTPS
API Gateway ── JWT signature/expiry + rate limit ──► Auth User Service
                                                        ├─► MySQL userdb
                                                        ├─► Outbox → Kafka
                                                        ├─► Notification Service (async command)
                                                        └─► S3/MinIO (KYC document metadata + private object)

Product / Payment / Order / Shipment / Message
  └─ chỉ nhận user_id, shop_id và event; không join hoặc tạo FK sang userdb
```

## 2. Cấu trúc bên trong

### 2.1 Kiến trúc module

```text
HTTP
  → Authentication / Authorization Filter
  → Controller
  → Application Service
       ├→ Domain policy: user, profile, address, shop, KYC, RBAC
       ├→ Repository → MySQL userdb
       ├→ Token adapter → JWT + refresh-token rotation
       ├→ Object storage port → S3/MinIO
       ├→ Notification port → Kafka command
       └→ Outbox publisher → Kafka topic
```

| Thành phần | Trách nhiệm | Ghi chú |
|---|---|---|
| `AuthController` | Sign up, sign in, refresh, sign out, verification, password reset, 2FA, JWKS key set (`/.well-known/jwks.json`) | Không trả password hoặc raw token persistence ra ngoài service. |
| `UserProfileController` | `GET/PUT /users/me` | Email đổi qua flow verification riêng; profile update không được tự đổi role. |
| `AddressController` | CRUD address của user hiện tại | Mọi mutation phải kiểm tra `user_id` từ token, không tin `user_id` trong body. |
| `SellerOnboardingController` | Tạo/cập nhật shop, KYC, warehouse, bank metadata | S3/MinIO chỉ lưu object; metadata và trạng thái thuộc `auth-user`. |
| `AdminAccessController` | KYC review, user status, role/permission, audit query | Bắt buộc permission và step-up 2FA cho destructive action. |
| `FavoriteController` | `GET/POST /users/me/favorites`, `DELETE /users/me/favorites/{productId}`, `GET /users/me/favorites/contains` | Chỉ thao tác favorites của `token.sub`; chỉ lưu `product_id`, không validate product tồn tại (Product Catalog là nguồn); giới hạn `MAX_FAVORITES_PER_USER`. |
| `ShopFollowController` | `POST/DELETE /shops/{shopId}/follow`, `GET /users/me/following`, `GET /shops/{shopId}/followers/count` | Follow theo `token.sub`; chỉ lưu `shop_id`; không cho follow shop `deleted`/không tồn tại (dùng local shop projection). |
| `PublicShopController` | `GET /shops/{shopId}` (public) | Trả hồ sơ shop công khai (identity + verified badge + follower count). Không trả bank/tax raw; rating/product count do Product Catalog phục vụ. |
| `TokenService` | Hash/rotate/revoke refresh token, tạo JWT access token | Refresh token lưu hash, không lưu plaintext. |
| `KycPolicy` | Kiểm tra trạng thái KYC và quyền publish/withdraw | Product/Payment service phải consume event và áp dụng local gate. |
| `RbacPolicy` | Resolve role → permission → scope | Seller staff luôn có `shop_id` scope; admin có system scope. |
| `OutboxPublisher` | Publish event sau transaction commit | Retry 3 lần, cách nhau 2 giây, sau đó đưa vào dead-letter topic. |
| `ScheduledJobs` | Mở khóa login hết hạn, đánh dấu KYC document expired, dọn token hết hạn | Job chạy idempotent; lịch cụ thể ở mục 4. |

### 2.2 Bảng dữ liệu chính

Chi tiết đầy đủ sẽ nằm ở `docs/db/auth-user.md`; LLD chỉ chốt ownership và các cột nghiệp vụ chính.

| Bảng | Cột chính | Index cần có |
|---|---|---|
| `users` | `id`, `email`, `phone`, `password_hash`, `full_name`, `date_of_birth`, `status`, `email_verified_at`, `phone_verified_at` | Unique normalized email; unique normalized phone khi khác `NULL`; status; created time |
| `addresses` | `id`, `user_id`, `recipient`, `phone`, `line1`, `ward`, `district`, `province`, `is_default`, `deleted_at` | `(user_id, is_default)`, `(user_id, deleted_at)` |
| `favorites` | `id`, `user_id`, `product_id`, `created_at` | Unique `(user_id, product_id)`, `(user_id, created_at)` |
| `shop_follows` | `id`, `user_id`, `shop_id`, `created_at` | Unique `(user_id, shop_id)`, `(shop_id, created_at)`, `(user_id, created_at)` |
| `shops` | `id`, `owner_user_id`, `name`, `slug`, `business_name`, `tax_code`, `status`, `warehouse_snapshot`, `bank_account_snapshot` | Unique owner; unique slug; status; tax code |
| `refresh_tokens` | `id`, `user_id`, `token_hash`, `family_id`, `expires_at`, `revoked_at`, `revoke_reason` | User + active token; family; expiry |
| `verification_tokens` | `id`, `user_id`, `channel`, `purpose`, `token_hash`, `expires_at`, `used_at`, `attempt_count` | `(user_id, purpose, channel, used_at)`, expiry |
| `password_reset_tokens` | `id`, `user_id`, `token_hash`, `expires_at`, `used_at` | User + active token; expiry |
| `roles` / `permissions` | Role key, permission key, scope type | Unique keys |
| `user_roles` | `user_id`, `role_id`, `shop_id`, `granted_by`, `granted_at`, `revoked_at` | User + active role; shop scope |
| `shop_staff` | `shop_id`, `user_id`, `staff_role`, `status` | Unique active `(shop_id, user_id)` |
| `kyc_cases` | `id`, `shop_id`, `status`, `submitted_at`, `reviewed_by`, `reviewed_at`, `decision_reason`, `expires_at` | Shop + current case; status; SLA queue |
| `kyc_documents` | `id`, `kyc_case_id`, `document_type`, `object_key`, `content_type`, `size_bytes`, `sha256`, `status` | Case + document type; checksum |
| `two_factor_credentials` | `user_id`, encrypted TOTP secret, `enabled_at`, `disabled_at`, recovery-code hashes | One active credential/user |
| `login_attempts` | `user_id/identifier_hash`, `succeeded`, `occurred_at`, `ip_hash` | Identifier + occurred time |
| `audit_logs` | `actor_user_id`, `action`, `target_type`, `target_id`, `metadata`, `occurred_at` | Target; actor; occurred time |
| `outbox_events` | `event_id`, `aggregate_type`, `aggregate_id`, `event_type`, `payload`, `published_at`, `attempt_count` | Published flag/time; aggregate key |

### 2.3 Quy ước định danh và ownership

- ID domain dùng UUIDv7 dạng canonical string ở API; cách lưu vật lý sẽ được chốt trong Database document.
- `user_id`, `shop_id`, `order_id`, `product_id` ở service khác chỉ là reference ID, không phải MySQL FK.
- `auth-user` là source of truth cho `users`, `shops`, role assignment, KYC status và verification state.
- Product Service là source of truth cho product status; nó chỉ dùng `shop.kyc.*` event để gate publish.
- Payment/Wallet Service là source of truth cho withdraw; nó chỉ dùng `shop.kyc.*` event để gate withdraw.

## 3. Luồng xử lý

### 3.1 Sign up — `POST /api/v1/auth/signup` — trigger: `Overlay / Sign in` → `Đăng ký ngay`

```text
1 Client gửi full_name, email, password, phone?
  ↓
2 Gateway rate limit → Auth validate format, uniqueness, password policy
  └─ sai → 400/409, không ghi user
  ↓ hợp lệ
3 Transaction tạo users(status=ACTIVE, email_verified=false), role BUYER
  ├→ tạo verification_tokens(email, expires=24h)
  └→ ghi outbox user.created + notification.email_verification.requested
  ↓ commit
4 Trả 201 user_id + access token 15 phút + refresh token 30 ngày
  └─ token có email_verified=false; seller onboarding bị chặn cho tới khi verify email
```

| Bước | Điều kiện lỗi | Xử lý |
|---|---|---|
| Validate | `full_name` rỗng/>120 ký tự, email sai format, phone sai E.164, password không đạt | `400 AUTH_INVALID_INPUT`; giữ nguyên input an toàn ở client, không ghi password. |
| Unique | Email đã tồn tại hoặc phone đã gắn user khác | `409 AUTH_EMAIL_EXISTS` hoặc `409 AUTH_PHONE_EXISTS`. |
| Publish event | Kafka tạm lỗi sau khi transaction commit | Outbox retry; response signup vẫn thành công, không tạo user lần hai. |

### 3.2 Sign in — `POST /api/v1/auth/signin` — trigger: `Overlay / Sign in`

```text
1 Client gửi identifier=email hoặc verified phone + password
  ↓
2 Normalize identifier; tìm user; kiểm tra status và lock_until
  └─ không hợp lệ/không tìm thấy → cùng một lỗi 401, không tiết lộ account tồn tại
  ↓
3 Verify Argon2id password
  ├─ sai: ghi login_attempt (theo IP + identifier hash)
  │    └─ 3 lần sai trên IP → kích hoạt CAPTCHA challenge
  │    └─ 5 lần sai trong 15 phút → LOCKED 15 phút (cho phép tự mở khóa tức thì qua email unlock link)
  └─ đúng: reset failure counter
  ↓
4 Nếu user có role admin và 2FA enabled → trả MFA_REQUIRED + challenge_id, chưa cấp access token
  ↓ nếu không cần 2FA hoặc đã verify
5 Tạo access token TTL 15 phút + refresh token mới; refresh token cũ của cùng rotation family bị revoke
```

| Điều kiện | Kết quả |
|---|---|
| Email chưa verify | Được đăng nhập buyer; seller onboarding và các action yêu cầu verified email trả `403 AUTH_EMAIL_NOT_VERIFIED`. |
| Phone chưa verify | Không được dùng phone làm identifier; yêu cầu verify OTP trước. |
| `status=LOCKED` và `lock_until` chưa hết | `423 AUTH_ACCOUNT_LOCKED`, trả thời điểm retry theo ISO-8601 kèm tùy chọn gửi email mở khóa tài khoản. |
| `status=SUSPENDED` hoặc `DELETED` | `403 AUTH_ACCOUNT_SUSPENDED`; không cấp token. |
| Admin thiếu 2FA | `401 AUTH_MFA_REQUIRED` với `challenge_id`, TTL challenge 5 phút. |

### 3.3 Email/phone verification — `POST /api/v1/auth/email/verify`, `POST /api/v1/auth/phone/request-otp`, `POST /api/v1/auth/phone/verify-otp`

```text
Email:
  token → hash → tìm verification_tokens → kiểm tra purpose/expiry/used_at
  → set email_verified_at → mark token used → publish user.email_verified

Phone:
  request-otp → tạo challenge 5 phút → Notification command channel=SMS
  verify-otp → tối đa 5 lần → set phone_verified_at → mark challenge used
```

- Verification token chỉ dùng một lần; resend token cũ bị revoke.
- Không trả token raw trong response của signup; link/token chỉ đi qua Notification Service.
- Email verification được phép resend tối đa 3 lần trong 1 giờ cho mỗi user.
- Phone OTP không được log plaintext; chỉ lưu hash và attempt counter.

### 3.4 Refresh token — `POST /api/v1/auth/refresh`

```text
1 Client gửi refresh token hiện tại
  ↓
2 Hash và tìm token; kiểm tra expiry, revoked_at, user status
  └─ sai/hết hạn/reuse → 401; nếu phát hiện reuse thì revoke cả family
  ↓
3 Revoke token hiện tại, tạo token mới cùng family
  ↓
4 Trả access token 15 phút + refresh token mới 30 ngày
```

- Refresh rotation là bắt buộc.
- Một token đã dùng lại sau rotation được xem là token theft; revoke toàn bộ family và ghi audit security event.
- Không dùng refresh token đã revoke để khôi phục session.

### 3.5 Sign out — `POST /api/v1/auth/signout` — response `204`

1. Gateway chuyển `user_id`, `session_id` và refresh token fingerprint.
2. Auth-user revoke refresh token hiện tại; nếu request chỉ rõ `all_sessions=true` và user đã re-authenticate thì revoke toàn bộ family của user.
3. Access token hiện tại không cần blacklist; nó hết hiệu lực tối đa sau 15 phút.
4. Ghi `audit_logs.action=AUTH_SIGNOUT` nhưng không ghi token.

### 3.6 Forgot/reset password — `POST /api/v1/auth/password/forgot`, `POST /api/v1/auth/password/reset`

```text
forgot:
  identifier → luôn trả 202 dù account tồn tại hay không
  → nếu tồn tại: tạo password_reset_token TTL 30 phút
  → outbox notification.password_reset.requested

reset:
  token + new_password → validate token/password
  → update password_hash → revoke refresh-token families
  → publish user.password_changed → 204
```

- Không tiết lộ email/phone có tồn tại ở `forgot`.
- Reset password không tự đăng nhập user.
- Tất cả access token mới sau reset vẫn yêu cầu sign in lại.

### 3.7 Buyer profile — `GET/PUT /api/v1/users/me` — trigger: `Account / Profile`

```text
GET  → đọc profile theo subject trong JWT
PUT  → validate full_name, phone?, date_of_birth?
     → nếu phone thay đổi: clear phone_verified_at và yêu cầu OTP lại
     → nếu email thay đổi: từ chối ở endpoint này, dùng email-change flow riêng
     → update users + outbox user.updated
```

| Field | Quy tắc |
|---|---|
| `full_name` | Bắt buộc, 1–120 Unicode characters, trim hai đầu. |
| `email` | Read-only trên profile; email change phải verify địa chỉ mới. |
| `phone` | Nullable; E.164; unique khi có giá trị; đổi số thì trạng thái phone trở về unverified. |
| `date_of_birth` | Nullable; ngày lịch ISO-8601 `YYYY-MM-DD`; không nhận timestamp. |
| Role/status | Không cho client sửa qua profile endpoint. |

### 3.8 Address book — `GET/POST /api/v1/users/me/addresses`, `PUT/DELETE /api/v1/users/me/addresses/{id}`

```text
1 Xác thực JWT và ownership (address.user_id == token.sub)
2 Validate recipient, phone, line1, ward, district, province
3 Transaction ghi address
  ├─ nếu is_default=true → clear default address cũ cùng user
  └─ nếu address đầu tiên → tự động is_default=true
4 DELETE chỉ soft delete; nếu xóa default → chọn address còn sống gần nhất làm default
```

- Tối đa 20 address còn sống/user.
- Order Service lưu `address_snapshot` riêng độc lập tại thời điểm đặt hàng/checkout; việc soft delete địa chỉ tại Auth User không ảnh hưởng đến đơn hàng đang xử lý.
- `DELETE` id không thuộc user trả `404`, không trả `403` để tránh lộ resource.
- Thay đổi address không sửa `address_snapshot` của order đã đặt.

### 3.9 Seller registration và onboarding — `POST /api/v1/users/register-seller`

```text
1 Buyer đã đăng nhập và email verified gửi shop info tối thiểu
  ↓
2 Kiểm tra user chưa có shop active và tax_code chưa trùng
  └─ sai → 403/409
  ↓
3 Tạo shops(status=DRAFT), gán role SELLER, tạo seller onboarding record
  ↓
4 Publish shop.created; mở Seller Center ở trạng thái onboarding
```

Các endpoint bổ sung để khớp flow 5 bước trong Penpot:

| Endpoint | Mục đích |
|---|---|
| `GET /api/v1/seller/onboarding` | Trả current step, completion và blockers. |
| `PUT /api/v1/seller/onboarding/profile` | Tên shop, business name, tax code, mô tả, logo metadata. |
| `PUT /api/v1/seller/onboarding/warehouse` | Tên kho, contact, địa chỉ kho, carrier preference, COD flag. |
| `POST /api/v1/seller/onboarding/kyc/documents/presign` | Lấy presigned upload URL cho KYC document. |
| `POST /api/v1/seller/onboarding/kyc/submit` | Chốt bộ hồ sơ để chuyển `DRAFT → PENDING`. |
| `PUT /api/v1/seller/onboarding/bank` | Lưu bank account snapshot và xác nhận chủ tài khoản. |
| `GET/PUT /api/v1/seller/shop` | Xem/cập nhật business profile sau onboarding. |

Quy tắc quyền:

- `DRAFT`: seller được chỉnh hồ sơ, kho, ngân hàng và upload tài liệu.
- `PENDING`, `NEEDS_INFO`, `REJECTED`, `EXPIRED`: seller vẫn xem/chỉnh hồ sơ và quản lý draft; Product Service không cho publish; Payment Service không cho withdraw.
- `APPROVED`: Product Service cho publish và Payment Service cho withdraw nếu các điều kiện domain khác cũng đạt.
- Không có product-level admin approval trong flow này.

### 3.10 KYC review — `GET /api/v1/admin/shops/kyc`, `GET /api/v1/admin/shops/{shopId}/kyc`, `POST /api/v1/admin/shops/{shopId}/kyc/review`

```text
1 Admin mở queue → kiểm tra permission KYC_READ + 2FA session còn hiệu lực
2 Admin xem metadata hồ sơ và private signed URL ngắn hạn của documents
3 Admin chọn APPROVE | NEEDS_INFO | REJECT
4 Transaction cập nhật kyc_cases + shops.status + audit_logs
5 Outbox phát shop.kyc.approved / needs_info / rejected
```

- Chỉ `RISK_MANAGER` và `SUPER_ADMIN` được quyết định KYC.
- `CATALOG_ADMIN` được xem tín hiệu sản phẩm nhưng không được quyết định KYC.
- Quyết định phải có `reason` tối thiểu 10 và tối đa 1.000 ký tự khi `NEEDS_INFO` hoặc `REJECT`.
- Document `EXPIRED` chuyển shop sang `EXPIRED` bằng scheduled job; seller phải nộp lại hồ sơ.
- KYC document không public; signed URL chỉ có TTL 10 phút và chỉ cấp cho admin có quyền.

### 3.11 RBAC và admin 2FA — `PATCH /api/v1/admin/users/{userId}/roles`, `POST /api/v1/auth/2fa/setup`, `POST /api/v1/auth/2fa/verify`

Role baseline:

| Role | Scope | Mục đích |
|---|---|---|
| `BUYER` | User | Mua hàng, profile, address, review eligibility. |
| `SELLER` | User + owned shop | Quản lý shop, sản phẩm, đơn và tài chính theo service tương ứng. |
| `SELLER_STAFF` | One shop | Nhân viên shop, permission scope theo shop. |
| `SUPER_ADMIN` | System | Toàn quyền, bắt buộc 2FA. |
| `RISK_MANAGER` | System | KYC, risk và dispute scope được cấp. |
| `CATALOG_ADMIN` | System | Catalog/SPU/SKU moderation scope. |
| `FINANCE_OPS` | System | Settlement, fee, tax và payout scope. |
| `SUPPORT_VIEWER` | System | Read-only support/messaging scope. |

```text
Admin action
  → Gateway kiểm tra JWT permission
  → Auth-user kiểm tra role scope + 2FA step-up token TTL 5 phút
  ├─ thiếu permission/2FA → 403/401, không ghi mutation
  └─ hợp lệ → transaction mutation + audit log + event nếu cần
```

- Role assignment không cho user tự cấp role cho chính mình.
- Mọi role change, KYC decision, account suspend và shop suspend phải có audit log.
- `SUPPORT_VIEWER` không được đọc password hash, refresh token, TOTP secret hoặc raw KYC document ngoài signed URL được cấp quyền.

### 3.12 Mock contract — Notification Service

HLD chỉ mô tả Notification Service gửi email; Penpot yêu cầu email verification, password reset và phone OTP. Auth-user dùng event contract dưới đây, không gọi provider email/SMS trực tiếp.

Topic: `notification.commands.v1`

```json
{
  "event_id": "01912f1e-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "command_type": "AUTH_VERIFICATION_REQUESTED",
  "occurred_at": "2026-08-30T09:00:00Z",
  "dedupe_key": "email-verification:user-01912f1e:token-01912f2a",
  "user_id": "01912f1e-7a1b-7c12-9c55-8b1c34a6d921",
  "channel": "EMAIL",
  "recipient": "minhanh@example.com",
  "template": "auth-email-verification-v1",
  "data": {
    "display_name": "Nguyen Minh Anh",
    "verification_token": "opaque-token-not-logged",
    "expires_at": "2026-08-31T09:00:00Z"
  }
}
```

Contract rules:

- `channel`: `EMAIL` hoặc `SMS`.
- `command_type`: `AUTH_VERIFICATION_REQUESTED`, `PASSWORD_RESET_REQUESTED`, `PHONE_OTP_REQUESTED`.
- Notification Service trả kết quả async bằng event `notification.delivered.v1` hoặc `notification.failed.v1` với `dedupe_key` giữ nguyên.
- Auth-user không rollback user khi notification provider lỗi; user có thể resend.
- Notification Service phải mask recipient trong log và không log token/OTP raw.

### 3.13 Mock contract — Object Storage cho KYC document

Object storage chưa có contract trong HLD; dùng adapter nội bộ để có thể thay S3/MinIO mà không đổi domain logic.

`POST /internal/v1/storage/presign`

```json
{
  "purpose": "KYC_DOCUMENT",
  "owner_type": "KYC_CASE",
  "owner_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
  "file_name": "business-registration.pdf",
  "content_type": "application/pdf",
  "size_bytes": 5242880,
  "sha256": "hex-64-character-checksum"
}
```

Mock response:

```json
{
  "object_key": "private/kyc/01912f31/document-01912f32.pdf",
  "upload_url": "https://storage.example/mock-upload/opaque",
  "expires_at": "2026-08-30T09:10:00Z",
  "required_headers": {
    "Content-Type": "application/pdf",
    "x-content-sha256": "hex-64-character-checksum"
  }
}
```

Sau khi upload thành công, client gọi `POST /api/v1/seller/onboarding/kyc/documents/complete`; auth-user kiểm tra object metadata, checksum, content type và lưu `kyc_documents`. Không cho client tự gửi `object_key` của hồ sơ khác.

### 3.14 Product favorites (wishlist) — `GET/POST /api/v1/users/me/favorites`, `DELETE /api/v1/users/me/favorites/{productId}`, `GET /api/v1/users/me/favorites/contains`

```text
POST:
  1 Xác thực JWT; lấy user_id = token.sub
  2 Validate product_id là UUID hợp lệ (không gọi Product Catalog để check tồn tại)
  3 Đếm favorites còn sống của user; nếu ≥ MAX_FAVORITES_PER_USER → 409 FAVORITE_LIMIT_REACHED
  4 INSERT ... ON DUPLICATE: nếu đã có (user_id, product_id) → trả 200 idempotent (không lỗi)
  5 Trả 201 { product_id, created_at }

GET (list):
  - Trả trang product_id + created_at, sort created_at desc; client/BFF hydrate thẻ sản phẩm qua Product Catalog batch API
  - Query "q" (tìm trong favorites) KHÔNG do auth-user xử lý: auth-user không có product title; frontend lọc client-side hoặc BFF gọi Product Catalog

GET /contains?product_ids=a,b,c (tối đa 100):
  - Trả { product_id: boolean } để render trạng thái tim trên card/PDP

DELETE /{productId}:
  - Xóa cứng row (favorites không cần audit/soft delete); không tồn tại → 204 idempotent
```

Ràng buộc:

- `favorites` chỉ lưu reference `product_id`; auth-user **không** validate product tồn tại/visible và **không** đồng bộ khi product bị archive/block — frontend/BFF tự lọc khi hydrate.
- Không phát sự kiện bắt buộc; tùy chọn phát `user.favorite.added/removed` cho analytics (xem §6.2).
- Không có thao tác bulk trong v1.

### 3.15 Follow shop — `POST/DELETE /api/v1/shops/{shopId}/follow`, `GET /api/v1/users/me/following`, `GET /api/v1/shops/{shopId}/followers/count`

```text
POST /shops/{shopId}/follow:
  1 Xác thực JWT; user_id = token.sub
  2 Kiểm shop tồn tại và status != DELETED bằng bảng shops local (đã là source of truth ở auth-user)
     └─ không có/deleted → 404 SHOP_NOT_FOUND
  3 Đếm shop_follows của user; ≥ MAX_FOLLOWED_SHOPS_PER_USER → 409 SHOP_FOLLOW_LIMIT_REACHED
  4 INSERT ON DUPLICATE → idempotent
  5 Trả 201 { shop_id, followed_at }

DELETE /shops/{shopId}/follow → 204 idempotent

GET /users/me/following → trang shop_id + shop name/slug/logo_url snapshot + followed_at

GET /shops/{shopId}/followers/count (public) → { shop_id, follower_count }
  - count cache được (eventual); hiển thị "X người theo dõi" trên Shop hero
```

Ràng buộc:

- Follow shop là quan hệ nhẹ; không cấp quyền gì cho user, chỉ dùng hiển thị và (tương lai) voucher targeting/notification cho follower.
- `follower_count` không cần realtime chính xác tuyệt đối; có thể tính bằng `COUNT(*)` với index hoặc counter cache.
- Tùy chọn phát `shop.followed/unfollowed` cho Notification/analytics (xem §6.2).

### 3.16 Public shop profile — `GET /api/v1/shops/{shopId}`

```text
1 Không bắt buộc JWT (public)
2 Load shops theo id; status = DELETED → 404
3 Trả các field công khai:
   id, name, slug, logo_url (resolve từ logo_object_key), description,
   is_verified (kyc_status == APPROVED), status, follower_count, created_at
4 KHÔNG trả: business_name, tax_code, bank_*, warehouse_snapshot, owner_user_id
```

- `rating_avg`, `product_count`, `sales_count` **không** thuộc response này — Product Catalog phục vụ (đã sở hữu `shop_snapshots` + product aggregate). Frontend Shop hero ghép từ hai nguồn.
- Endpoint này đáp ứng HLD mục 10 (`GET /api/v1/shops/{shopId} -> public shop profile`).

## 4. Hằng số & cấu hình

| Tên | Giá trị | Đơn vị | Ghi chú |
|---|---:|---|---|
| `ACCESS_TOKEN_TTL` | 900 | giây | JWT access token, 15 phút. |
| `REFRESH_TOKEN_TTL` | 30 | ngày | Rotation bắt buộc. |
| `REFRESH_TOKEN_ROTATION` | `true` | boolean | Token cũ bị revoke sau mỗi refresh. |
| `PASSWORD_HASH_ALGORITHM` | `Argon2id` | algorithm | Không lưu raw password. |
| `PASSWORD_MIN_LENGTH` | 12 | ký tự | `PASSWORD_MAX_LENGTH=72`. |
| `FULL_NAME_MAX_LENGTH` | 120 | Unicode characters | Trim hai đầu. |
| `EMAIL_MAX_LENGTH` | 254 | ký tự | Lowercase normalization. |
| `PHONE_FORMAT` | E.164 | format | Phone chỉ unique khi non-null. |
| `LOGIN_FAILURE_LIMIT` | 5 | lần | Trong `LOGIN_FAILURE_WINDOW`. |
| `LOGIN_FAILURE_WINDOW` | 15 | phút | Theo user/identifier hash. |
| `ACCOUNT_LOCK_DURATION` | 15 | phút | Tự mở khóa sau TTL nếu không bị suspend. |
| `MAX_ADDRESSES_PER_USER` | 20 | address | Chỉ tính record chưa soft delete. |
| `MAX_FAVORITES_PER_USER` | 500 | product | Vượt trả `409 FAVORITE_LIMIT_REACHED`. |
| `MAX_FOLLOWED_SHOPS_PER_USER` | 1000 | shop | Vượt trả `409 SHOP_FOLLOW_LIMIT_REACHED`. |
| `FAVORITES_CONTAINS_MAX_IDS` | 100 | product_id/request | Batch check trạng thái tim. |
| `PAGE_SIZE_DEFAULT` | 20 | record | `PAGE_SIZE_MAX=100`. |
| `EMAIL_VERIFICATION_TTL` | 24 | giờ | Token dùng một lần. |
| `PHONE_OTP_TTL` | 5 | phút | `PHONE_OTP_MAX_ATTEMPTS=5`. |
| `PASSWORD_RESET_TTL` | 30 | phút | Reset token dùng một lần. |
| `EMAIL_RESEND_LIMIT` | 3 | lần/giờ | Theo user. |
| `MFA_CHALLENGE_TTL` | 5 | phút | Step-up token cho admin. |
| `TOTP_STEP` | 30 | giây | Chuẩn TOTP. |
| `KYC_MAX_FILE_SIZE` | 10 | MiB/file | PDF/JPG/PNG trong v1. |
| `KYC_MAX_FILES_PER_CASE` | 10 | file/case | Object private. |
| `SIGNED_URL_TTL` | 10 | phút | KYC review/download. |
| `INTERNAL_REQUEST_TIMEOUT` | 5 | giây | Storage/Notification adapter. |
| `KAFKA_RETRY_COUNT` | 3 | lần | Cách nhau `KAFKA_RETRY_BACKOFF=2s`. |
| `KAFKA_RETRY_BACKOFF` | 2 | giây | Sau đó dead-letter. |
| `KYC_EXPIRY_JOB` | `00:05` | UTC mỗi ngày | Đánh dấu document/case expired idempotent. |
| `AUDIT_RETENTION` | 365 | ngày | Không lưu secret/OTP/token raw. |
| `TIMESTAMP_STORAGE` | UTC | timezone | API trả ISO-8601; UI hiển thị user timezone. |

## 5. Enum & trạng thái

### 5.1 `UserStatus`

| Giá trị | Ý nghĩa | Chuyển được sang |
|---|---|---|
| `ACTIVE` | Account hoạt động bình thường. | `LOCKED`, `SUSPENDED`, `DELETED` |
| `LOCKED` | Tạm khóa do login failure. | `ACTIVE`, `SUSPENDED` |
| `SUSPENDED` | Bị admin khóa theo policy. | `ACTIVE`, `DELETED` |
| `DELETED` | Soft-deleted, không được login. | Không chuyển ngược trong v1 |

```text
ACTIVE ──5 failures/15m──► LOCKED ──lock_until reached──► ACTIVE
   │                              │
   ├─admin suspend───────────────┘
   └─user deletion──────────────────────────────────────► DELETED
```

### 5.2 `ShopStatus`, `KycStatus` và KYC gate

- **`ShopStatus`** (vòng đời gian hàng): `DRAFT`, `ACTIVE`, `SUSPENDED`, `DELETED`.
- **`KycStatus`** (tiến trình xét duyệt hồ sơ): `DRAFT`, `PENDING`, `NEEDS_INFO`, `APPROVED`, `REJECTED`, `EXPIRED`, `SUSPENDED`.

| `KycStatus` | Ý nghĩa KYC | `ShopStatus` tương ứng | Publish product | Withdraw |
|---|---|---|---:|---:|
| `DRAFT` | Chưa submit KYC. | `DRAFT` | Không | Không |
| `PENDING` | Đang chờ review. | `DRAFT` | Không | Không |
| `NEEDS_INFO` | Admin yêu cầu bổ sung. | `DRAFT` | Không | Không |
| `APPROVED` | KYC đạt. | `ACTIVE` | Có | Có |
| `REJECTED` | Hồ sơ bị từ chối, được phép resubmit. | `DRAFT` | Không | Không |
| `EXPIRED` | Tài liệu/case hết hạn. | `DRAFT` / `ACTIVE` (chặn action mới) | Không | Không |
| `SUSPENDED` | Khóa theo risk/policy. | `SUSPENDED` | Không | Không |

```text
KYC Workflow:
DRAFT ──submit──► PENDING
PENDING ──approve──► APPROVED (Shops.status chuyển ACTIVE)
PENDING ──need-info──► NEEDS_INFO ──resubmit──► PENDING
PENDING ──reject──► REJECTED ──resubmit──► PENDING
APPROVED ──document expiry──► EXPIRED ──resubmit──► PENDING
APPROVED ──risk action──► SUSPENDED ──admin restore──► APPROVED
```

### 5.3 Role và permission baseline

| Nhóm | Giá trị |
|---|---|
| User role | `BUYER`, `SELLER`, `SELLER_STAFF` |
| Admin role | `SUPER_ADMIN`, `RISK_MANAGER`, `CATALOG_ADMIN`, `FINANCE_OPS`, `SUPPORT_VIEWER` |
| KYC permission | `KYC_READ`, `KYC_DECIDE`, `KYC_REQUEST_INFO` |
| User permission | `USER_READ`, `USER_SUSPEND`, `ROLE_READ`, `ROLE_ASSIGN` |
| Seller scope permission | `SHOP_READ`, `SHOP_UPDATE`, `SELLER_STAFF_MANAGE` |
| Search admin permission | `SEARCH_ADMIN` (search reindex/diagnostics) |
| Commerce admin permission | `VOUCHER_MANAGE` (order-commerce platform voucher CRUD) |
| Finance admin scope | Role `FINANCE_OPS` (đã có ở §5.3 bảng Admin role) là scope cho payment-wallet `/admin/**`: fee/tax config, settlement, finance summary, reconciliation |
| Step-up marker | `MFA_REQUIRED`, `MFA_VERIFIED` |

> Phạm vi Admin đã chốt (`System_Overview.md` §6.3): **không có microservice admin riêng trong v1**; mỗi màn admin do service sở hữu dữ liệu phục vụ qua `/api/v1/admin/**`. Permission cross-service (`SEARCH_ADMIN`, `VOUCHER_MANAGE`, `FINANCE_OPS`, …) do `auth-user` cấp qua RBAC nhưng **được enforce tại service sở hữu tài nguyên** (Gateway chỉ coarse-gate theo role admin). Product Catalog moderation gate bằng role `CATALOG_ADMIN`/`SUPER_ADMIN`. `auth-user` tự phục vụ Shops/KYC + Users/Roles qua `AdminAccessController`. `dispute`/`campaign` là service v1.1 — khi có sẽ bổ sung role/permission tương ứng.

### 5.4 Token và verification state

| Enum | Giá trị hợp lệ |
|---|---|
| Refresh token | `ACTIVE`, `REVOKED`, `EXPIRED`, `REUSE_DETECTED` |
| Verification token | `ACTIVE`, `USED`, `EXPIRED`, `REVOKED` |
| KYC document | `UPLOADING`, `UPLOADED`, `VERIFIED`, `REJECTED`, `EXPIRED`, `DELETED` |
| 2FA | `DISABLED`, `ENROLLING`, `ENABLED`, `RESET_REQUIRED` |

### 5.5 Quyền theo chuyển trạng thái

| Chuyển trạng thái | Ai được phép | Điều kiện |
|---|---|---|
| `DRAFT → PENDING` | Shop owner | Email verified, đủ hồ sơ bắt buộc, có ít nhất 1 document hợp lệ. |
| `PENDING → APPROVED` | `RISK_MANAGER`, `SUPER_ADMIN` | Đã review toàn bộ document và có decision audit. |
| `PENDING → NEEDS_INFO` | `RISK_MANAGER`, `SUPER_ADMIN` | Có reason 10–1.000 ký tự. |
| `PENDING → REJECTED` | `RISK_MANAGER`, `SUPER_ADMIN` | Có reason và 2FA step-up. |
| `APPROVED → SUSPENDED` | `RISK_MANAGER`, `SUPER_ADMIN` | Policy/risk case tồn tại, ghi audit. |
| `ACTIVE → SUSPENDED` | Admin có `USER_SUSPEND` | Có reason, revoke refresh sessions, phát event `user.status_changed` và đẩy `revoked_user_id` vào Redis chung của Gateway để block JWT ngay. |
| Role assignment | `SUPER_ADMIN` hoặc permission được cấp | Không tự cấp quyền cao hơn quyền của actor. |

## 6. Event phát ra / lắng nghe

### 6.1 Envelope chung

```json
{
  "event_id": "UUIDv7",
  "schema_version": 1,
  "event_type": "shop.kyc.approved",
  "occurred_at": "2026-08-30T09:00:00Z",
  "aggregate_type": "SHOP",
  "aggregate_id": "shop-id",
  "actor_user_id": "admin-user-id-or-null",
  "payload": {}
}
```

### 6.2 Event phát ra

| Topic | Event type | Payload chính | Khi nào | Key |
|---|---|---|---|---|
| `user.events.v1` | `user.created` | `user_id`, `email`, `status`, `roles` | Signup commit thành công | `user_id` |
| `user.events.v1` | `user.updated` | `user_id`, changed fields, `updated_at` | Profile update | `user_id` |
| `user.events.v1` | `user.email_verified` | `user_id`, `verified_at` | Verify thành công | `user_id` |
| `user.events.v1` | `user.status_changed` | `user_id`, `old_status`, `new_status`, `reason` | Lock/suspend/restore/delete | `user_id` |
| `user.events.v1` | `user.role_changed` | `user_id`, `role`, `shop_id`, `action` | Grant/revoke role | `user_id` |
| `shop.events.v1` | `shop.created` | `shop_id`, `owner_user_id`, `status` | Register seller thành công | `shop_id` |
| `shop.events.v1` | `shop.updated` | `shop_id`, changed fields trong allowlist snapshot (`name`, `slug`, `logo_media_id`, `description`), `updated_at`, `version` | Seller cập nhật shop profile | `shop_id` |
| `shop.events.v1` | `shop.status_changed` | `shop_id`, `old_status`, `new_status`, `reason`, `changed_at` | Shop chuyển `DRAFT/ACTIVE/SUSPENDED/CLOSED` (gồm admin suspend/restore) | `shop_id` |
| `shop.events.v1` | `shop.kyc.submitted` | `shop_id`, `kyc_case_id`, document types | Submit KYC | `shop_id` |
| `shop.events.v1` | `shop.kyc.approved` | `shop_id`, `kyc_case_id`, `approved_at` | Admin approve | `shop_id` |
| `shop.events.v1` | `shop.kyc.needs_info` | `shop_id`, `kyc_case_id`, `reason` | Cần bổ sung | `shop_id` |
| `shop.events.v1` | `shop.kyc.rejected` | `shop_id`, `kyc_case_id`, `reason` | Admin reject | `shop_id` |
| `shop.events.v1` | `shop.kyc.expired` | `shop_id`, `kyc_case_id`, `expired_documents` | Scheduled job | `shop_id` |
| `notification.commands.v1` | `AUTH_VERIFICATION_REQUESTED` | channel, recipient, template, data, dedupe key | Signup/resend | `user_id` |
| `notification.commands.v1` | `PASSWORD_RESET_REQUESTED` | channel, recipient, template, reset token data | Forgot password | `user_id` |
| `notification.commands.v1` | `PHONE_OTP_REQUESTED` | channel, recipient, OTP challenge data | Phone verification | `user_id` |
| `user.events.v1` | `user.favorite.added` / `user.favorite.removed` *(optional v1)* | `user_id`, `product_id` | Thêm/bỏ favorite | `user_id` |
| `shop.events.v1` | `shop.followed` / `shop.unfollowed` *(optional v1)* | `shop_id`, `user_id`, `follower_count` | Follow/unfollow shop | `shop_id` |

> `user.favorite.*` và `shop.*followed` là event tùy chọn cho analytics/notification tương lai (VD: shop có sản phẩm mới → thông báo follower). Không có consumer bắt buộc trong v1; nếu không bật thì bỏ khỏi outbox.

> **Không có event `shop.kyc.suspended`.** Việc đình chỉ shop (do risk action hoặc admin) phát `shop.status_changed` với `new_status=SUSPENDED`; KYC case chỉ phát `submitted/approved/needs_info/rejected/expired`. Consumer (Product Catalog gate publish, Payment-Wallet gate payout, Message đóng participant) phải bắt `shop.status_changed`, không chờ event KYC suspended.

### 6.3 Event lắng nghe

| Nguồn | Event | Xử lý |
|---|---|---|
| Business domain services | Không lắng nghe event nghiệp vụ trong v1 | Auth-user là source of truth; mutation user/shop đi qua API của service này. |
| Object storage adapter | Callback/HTTP complete | Xác minh object metadata và cập nhật `kyc_documents`; request phải idempotent theo object checksum. |

### 6.4 Reliability và retry

- Ghi domain mutation và `outbox_events` trong cùng transaction.
- Publisher lấy event theo `created_at`, publish theo partition key (`user_id` hoặc `shop_id`).
- Retry tối đa 3 lần, backoff 2 giây; lỗi tiếp tục vào `auth-user.events.dlq.v1`.
- Consumer phải deduplicate theo `event_id`; không xử lý lại event đã hoàn tất.
- Event không chứa password hash, refresh token, OTP raw, TOTP secret, bank account đầy đủ hoặc KYC document bytes.

## 7. Mã lỗi

| Mã | HTTP | Khi nào xảy ra | Thông điệp cho người dùng |
|---|---:|---|---|
| `AUTH_INVALID_INPUT` | 400 | Payload hoặc format không hợp lệ | `Thông tin đăng ký chưa đúng.` |
| `AUTH_EMAIL_EXISTS` | 409 | Email đã được sử dụng | `Email đã được sử dụng.` |
| `AUTH_PHONE_EXISTS` | 409 | Phone đã được sử dụng | `Số điện thoại đã được sử dụng.` |
| `AUTH_INVALID_CREDENTIALS` | 401 | Identifier/password không đúng | `Email/số điện thoại hoặc mật khẩu không đúng.` |
| `AUTH_ACCOUNT_LOCKED` | 423 | Account đang bị lock tạm thời | `Tài khoản đang tạm khóa. Vui lòng thử lại sau.` |
| `AUTH_ACCOUNT_SUSPENDED` | 403 | Account suspended/deleted | `Tài khoản hiện không thể sử dụng.` |
| `AUTH_EMAIL_NOT_VERIFIED` | 403 | Action yêu cầu email verified | `Vui lòng xác thực email trước.` |
| `AUTH_PHONE_NOT_VERIFIED` | 403 | Dùng phone chưa verified để login | `Vui lòng xác thực số điện thoại trước.` |
| `AUTH_TOKEN_INVALID` | 401 | Access/refresh token sai | `Phiên đăng nhập không hợp lệ.` |
| `AUTH_TOKEN_EXPIRED` | 401 | Token hết hạn | `Phiên đăng nhập đã hết hạn.` |
| `AUTH_REFRESH_REUSED` | 401 | Phát hiện refresh token reuse | `Phiên đăng nhập đã bị thu hồi. Vui lòng đăng nhập lại.` |
| `AUTH_VERIFICATION_INVALID` | 400 | Token/OTP sai, hết hạn hoặc đã dùng | `Mã xác thực không hợp lệ hoặc đã hết hạn.` |
| `AUTH_OTP_ATTEMPTS_EXCEEDED` | 429 | Vượt quá 5 lần thử OTP | `Bạn đã thử quá số lần cho phép.` |
| `AUTH_RESET_INVALID` | 400 | Password reset token không hợp lệ | `Liên kết đặt lại mật khẩu không hợp lệ.` |
| `AUTH_MFA_REQUIRED` | 401 | Admin cần 2FA step-up | `Vui lòng xác thực 2FA.` |
| `AUTH_MFA_INVALID` | 401 | TOTP/recovery code sai | `Mã 2FA không đúng.` |
| `RBAC_PERMISSION_DENIED` | 403 | Thiếu permission hoặc scope | `Bạn không có quyền thực hiện thao tác này.` |
| `RBAC_MFA_REQUIRED` | 428 | Destructive action chưa step-up | `Vui lòng xác thực lại trước khi tiếp tục.` |
| `PROFILE_INVALID` | 400 | Profile field sai format/length | `Thông tin hồ sơ chưa đúng.` |
| `ADDRESS_NOT_FOUND` | 404 | Address không thuộc user hoặc không tồn tại | `Không tìm thấy địa chỉ.` |
| `ADDRESS_LIMIT_REACHED` | 409 | Đã có 20 address còn sống | `Bạn đã đạt giới hạn số địa chỉ.` |
| `ADDRESS_DEFAULT_REQUIRED` | 409 | Mutation làm user không còn default address | `Cần có một địa chỉ mặc định.` |
| `SHOP_ALREADY_EXISTS` | 409 | User đã có shop active/onboarding | `Tài khoản đã có hồ sơ người bán.` |
| `SHOP_INVALID_STATE` | 409 | Action không hợp lệ với shop status | `Trạng thái gian hàng không cho phép thao tác này.` |
| `KYC_REQUIRED` | 403 | Chưa đạt điều kiện KYC | `Vui lòng hoàn tất xác minh gian hàng.` |
| `KYC_DOCUMENT_INVALID` | 400 | Sai type/checksum/object metadata | `Tài liệu không hợp lệ.` |
| `KYC_DOCUMENT_TOO_LARGE` | 413 | File vượt 10 MiB | `Tài liệu vượt quá dung lượng cho phép.` |
| `KYC_DECISION_INVALID` | 400 | Thiếu reason hoặc decision không hợp lệ | `Quyết định KYC chưa đầy đủ.` |
| `FAVORITE_LIMIT_REACHED` | 409 | Đã đạt `MAX_FAVORITES_PER_USER` | `Bạn đã đạt giới hạn số sản phẩm yêu thích.` |
| `SHOP_FOLLOW_LIMIT_REACHED` | 409 | Đã đạt `MAX_FOLLOWED_SHOPS_PER_USER` | `Bạn đã đạt giới hạn số shop theo dõi.` |
| `RATE_LIMITED` | 429 | Vượt request limit | `Bạn thao tác quá nhanh. Vui lòng thử lại sau.` |
| `INTERNAL_ERROR` | 500 | Lỗi chưa phân loại | `Hệ thống đang bận. Vui lòng thử lại.` |

## 8. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Spring Boot chưa có minor/patch version; LLD chỉ khóa Java 25 và framework family. | Ảnh hưởng dependency, security library và deployment image. | Tech lead |
| 2 | LLD dùng MySQL 8.4, UUIDv7 và lưu vật lý UUID sẽ được chốt trong Database document. | Ảnh hưởng migration, index size và cross-service ID format. | Backend lead |
| 3 | Profile UI có field email nhưng email đang được xem là read-only; email change phải đi flow verification riêng. | Nếu email editable trực tiếp, cần thêm API, token và migration audit. | Product owner |
| 4 | KYC document dùng S3/MinIO private bucket, tối đa 10 MiB/file, lưu audit 365 ngày. | Ảnh hưởng storage cost, compliance và upload UX. | Product + Security |
| 5 | `SELLER` role được gán khi tạo shop `DRAFT`; publish/withdraw chỉ mở khi shop `APPROVED`. | Ảnh hưởng Seller Center access và permission mapping giữa services. | Product owner |
| 6 | Seller staff được hỗ trợ ở RBAC baseline nhưng giới hạn số staff, invitation flow và staff UI chưa có trong HLD/Penpot. | Cần bổ sung schema/API nếu triển khai staff management ở v1. | Product owner |
| 7 | Phone login/OTP dùng Notification Service channel `SMS`; provider thật chưa được chọn, hiện chỉ có mock contract. | Ảnh hưởng chi phí, deliverability và retry policy. | Tech lead |
| 8 | Admin 2FA dùng TOTP, step-up TTL 5 phút; recovery-code policy chưa mô tả trong HLD. | Ảnh hưởng account recovery và support operation. | Security owner |
| 9 | Product moderation không thuộc `auth-user`; KYC event là gate duy nhất từ user/shop side. | Nếu cần duyệt từng sản phẩm, Product Service phải có workflow riêng. | Product owner |
| 12 | Phạm vi Admin đã chốt (`System_Overview.md` §6.3): v1 **không** tách microservice admin. `auth-user` phục vụ các màn Shops/KYC, Users/Roles, Administrator roles, Admin role editor qua `AdminAccessController` (`/api/v1/admin/users/**`, `/api/v1/admin/shops/**`), gác `RISK_MANAGER`/`SUPER_ADMIN` + 2FA. `dispute`/`campaign` là service v1.1. | Nếu sau này gộp admin thành service riêng phải chuyển ownership KYC/role. | Architecture owner |
| 10 | Favorites (wishlist) và Follow shop có trong Frontend Design nhưng **không có trong HLD**; đặt tại `auth-user` (bảng `favorites`, `shop_follows`) vì là dữ liệu cá nhân của user, tương tự `addresses`. Chỉ lưu reference ID, không đồng bộ vòng đời product/shop. | Nếu khối lượng lớn hoặc cần feed/notification follower, có thể tách service `engagement` riêng sau này. | Product owner |
| 11 | `GET /api/v1/shops/{shopId}` (public shop profile, HLD mục 10) đặt tại `auth-user` — chỉ trả identity + verified badge + `follower_count`. `rating_avg`/`product_count` do Product Catalog phục vụ; Frontend Shop hero ghép hai nguồn. | Nếu muốn một endpoint hợp nhất, cần chọn service tổng hợp hoặc BFF. | Product owner + Architecture |
