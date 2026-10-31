# API Spec — Auth User Service

> Nguồn: `docs/lld/auth-user.md` · `docs/db/auth-user.md` · `docs/contracts/*`, `docs/google-signin-contract.md`, `docs/vietnam-location-api.md` trong service repo · Code: `ecommerce-taca/user-auth-service` @ `develop` (`7c268a0`) · Cập nhật: `2026-10-31`
> Trạng thái triển khai: 🟡 Triển khai một phần — toàn bộ endpoint nghiệp vụ ở §2 đã có controller; health chỉ có `/actuator/health` (chưa có `/health/live`, `/health/ready`); chưa có handler 500 chung.
> Base path: `/api/v1` (JWKS ở `/.well-known/jwks.json`) · External access: qua `api-gateway` · Port local `8081`

## 1. Quy ước API chung

### 1.1 HTTP và authentication

| Mục | Quy định |
|---|---|
| Base path | `/api/v1` |
| Content-Type | `application/json; charset=utf-8` |
| Auth header | `Authorization: Bearer <access-token>` cho protected endpoint. |
| Access token | JWT RS256, TTL 15 phút; Gateway và service validate `iss`, `aud`, `exp`, `sub`. |
| Refresh token | Opaque token TTL 30 ngày; gửi trong JSON body tới `/auth/refresh`; rotation bắt buộc. |
| MFA step-up | `X-MFA-Step-Up: <opaque-step-up-token>` cho admin mutation yêu cầu 2FA. Không log header. |
| Request ID | `X-Request-ID` tối đa 64 ký tự; Gateway tạo/propagate. |
| Timestamp | ISO-8601 UTC, ví dụ `2026-08-30T09:15:00Z`. |
| UUID | Canonical string, ví dụ `01912f31-7a1b-7c12-9c55-8b1c34a6d921`; lưu `BINARY(16)`. |
| Date | `YYYY-MM-DD`, ví dụ `1995-06-30`; không nhận timestamp cho date-of-birth. |
| Phone | E.164, ví dụ `+84901234567`. |
| Locale | V1 response message tiếng Việt; error `code` tiếng Anh ổn định. |
| Idempotency | Auth mutation dùng idempotency theo token/challenge; không yêu cầu client header cho signup v1. |
| Pagination | `page` bắt đầu từ 1, `size` mặc định 20, tối đa 100; response có `page,size,total,total_pages`. |
| Sort | `sort=<field>,asc|desc`; field ngoài allowlist trả 400. |
| File upload | Không upload bytes KYC qua API; dùng signed URL rồi gọi endpoint `complete`. |

### 1.2 Response envelope

Successful single resource:

```json
{
  "data": {},
  "meta": {
    "request_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Successful list:

```json
{
  "data": [],
  "meta": {
    "page": 1,
    "size": 20,
    "total": 42,
    "total_pages": 3,
    "request_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Error:

```json
{
  "error": {
    "code": "AUTH_INVALID_CREDENTIALS",
    "message": "Email/số điện thoại hoặc mật khẩu không đúng.",
    "details": [],
    "trace_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Ràng buộc:

- Không trả password hash, refresh token hash, OTP raw, TOTP secret, bank account raw hoặc KYC object bytes.
- Error response không tiết lộ account có tồn tại với các endpoint forgot password/signin.
- `details` chỉ chứa field/code được allowlist, ví dụ `field`, `expected`, `actual`; không chứa stack trace hoặc SQL.

### 1.3 Contract tích hợp (Kafka, object storage)

#### 1.3.1 Notification command

Auth-user publish command vào `notification.commands.v1` sau transaction:

```json
{
  "event_id": "01912f55-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "command_type": "PASSWORD_RESET_REQUESTED",
  "occurred_at": "2026-08-30T09:00:00Z",
  "dedupe_key": "password-reset:user-01912f31:token-01912f56",
  "user_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
  "channel": "EMAIL",
  "recipient": "minhanh@example.com",
  "template": "auth-password-reset-v1",
  "data": {
    "display_name": "Nguyễn Minh Anh",
    "reset_url": "https://taca.vn/reset-password?t=opaque-token-reference",
    "expires_in_minutes": 30
  }
}
```

#### 1.3.2 Object storage presign

Mock request/response được mô tả ở endpoint KYC presign. Adapter phải kiểm tra `owner_id`, `purpose`, `content_type`, `size_bytes`, `sha256`; không cho client dùng object key của case khác.

#### 1.3.3 Event downstream

Các event `user.created`, `user.updated`, `user.email_verified`, `user.status_changed`, `user.role_changed`, `shop.created`, `shop.updated`, `shop.status_changed`, `shop.kyc.*` dùng chung envelope:

```json
{
  "event_id": "01912fe2-7a1b-7c12-9c55-8b1c34a6d921",
  "schema_version": 1,
  "aggregate_type": "USER",
  "aggregate_id": "01912f10-7a1b-7c12-9c55-8b1c34a6d921",
  "event_type": "user.status_changed",
  "occurred_at": "2026-08-30T09:00:00Z",
  "payload": {
    "user_id": "01912f10-7a1b-7c12-9c55-8b1c34a6d921",
    "old_status": "ACTIVE",
    "new_status": "SUSPENDED",
    "reason": "Vi phạm chính sách nền tảng."
  }
}
```

Kafka header (không nằm trong payload) mang `traceparent`, `request_id` — theo quy ước chung của repo. Consumer dedupe theo `event_id`. Bảng field chi tiết từng event nằm ở `docs/lld/auth-user.md` §6.2.

## 2. Danh sách endpoint


| # | Method | Path | Mục đích | Quyền | Màn hình/consumer | Trạng thái |
|---:|---|---|---|---|---|---|
| 1 | `POST` | `/auth/signup` | Tạo buyer account | Public | Sign in overlay → đăng ký | ✅ |
| 2 | `POST` | `/auth/signin` | Đăng nhập email/phone | Public | Sign in overlay | ✅ |
| 3 | `POST` | `/auth/refresh` | Đổi refresh token | Public với refresh token | Client auth interceptor | ✅ |
| 4 | `POST` | `/auth/signout` | Revoke session/token family | Authenticated | Account/sign out | ✅ |
| 5 | `POST` | `/auth/email/verify` | Verify email | Public với token | Email link | ✅ |
| 6 | `POST` | `/auth/email/resend` | Gửi lại email verification | Authenticated | Sign in/onboarding | ✅ |
| 7 | `POST` | `/auth/phone/request-otp` | Gửi phone OTP | Authenticated | Profile/onboarding | ✅ |
| 8 | `POST` | `/auth/phone/verify-otp` | Verify phone OTP | Authenticated | Profile/onboarding | ✅ |
| 9 | `POST` | `/auth/password/forgot` | Khởi tạo reset password | Public | Forgot password | ✅ |
| 10 | `POST` | `/auth/password/reset` | Đặt password mới | Public với reset token | Reset password | ✅ |
| 11 | `POST` | `/auth/2fa/setup` | Tạo TOTP enrollment | Admin authenticated | Admin security settings | ✅ |
| 12 | `POST` | `/auth/2fa/verify` | Verify login/2FA enrollment | Challenge/authenticated | Admin login/step-up | ✅ |
| 13 | `GET` | `/users/me` | Đọc profile | Authenticated | Buyer Account/Profile | ✅ |
| 14 | `PUT` | `/users/me` | Cập nhật profile | Authenticated | Buyer Account/Profile | ✅ |
| 15 | `GET` | `/users/me/addresses` | Danh sách địa chỉ | Authenticated | Address book/checkout | ✅ |
| 16 | `POST` | `/users/me/addresses` | Thêm địa chỉ | Authenticated | Address book | ✅ |
| 17 | `PUT` | `/users/me/addresses/{addressId}` | Sửa địa chỉ | Authenticated | Address book | ✅ |
| 18 | `DELETE` | `/users/me/addresses/{addressId}` | Soft delete địa chỉ | Authenticated | Address book | ✅ |
| 19 | `POST` | `/users/register-seller` | Tạo shop/onboarding | Authenticated + email verified | Seller onboarding | ✅ |
| 20 | `GET` | `/seller/onboarding` | Đọc tiến độ onboarding | Seller | Seller onboarding | ✅ |
| 21 | `PUT` | `/seller/onboarding/profile` | Lưu hồ sơ shop | Seller | Step 1 — Hồ sơ | ✅ |
| 22 | `PUT` | `/seller/onboarding/warehouse` | Lưu kho/vận chuyển | Seller | Step 3 — Kho & vận chuyển | ✅ |
| 23 | `POST` | `/seller/onboarding/kyc/documents/presign` | Lấy signed upload URL | Seller | Step 2 — KYC | ✅ |
| 24 | `POST` | `/seller/onboarding/kyc/documents/complete` | Xác nhận upload document | Seller | Step 2 — KYC | ✅ |
| 25 | `POST` | `/seller/onboarding/kyc/submit` | Submit KYC case | Seller | Step 2 — KYC | ✅ |
| 26 | `PUT` | `/seller/onboarding/bank` | Lưu thông tin ngân hàng | Seller | Step 4 — Ngân hàng | ✅ |
| 27 | `GET` | `/seller/shop` | Đọc shop profile | Seller/staff | Seller Settings | ✅ |
| 28 | `PUT` | `/seller/shop` | Cập nhật shop profile | Seller | Seller Settings | ✅ |
| 29 | `GET` | `/admin/shops/kyc` | KYC queue | `KYC_READ` | Admin Shops/KYC | ✅ |
| 30 | `GET` | `/admin/shops/{shopId}/kyc` | KYC detail/document metadata | `KYC_READ` | KYC review overlay | ✅ |
| 31 | `POST` | `/admin/shops/{shopId}/kyc/review` | Approve/request info/reject | `KYC_DECIDE` | KYC review overlay | ✅ |
| 32 | `GET` | `/admin/users/{userId}/roles` | Đọc role/permission | `ROLE_READ` | Admin Users/Roles | ✅ |
| 33 | `PATCH` | `/admin/users/{userId}/roles` | Grant/revoke role | `ROLE_ASSIGN` + 2FA | Admin role editor | ✅ |
| 34 | `PATCH` | `/admin/users/{userId}/status` | Suspend/restore user | `USER_SUSPEND` + 2FA | Admin Users | ✅ |
| 35 | `GET` | `/admin/audit-logs` | Tra cứu audit | Role được cấp | Admin audit/support | ✅ |
| 36 | `GET` | `/.well-known/jwks.json` | Public JWKS key set cho Gateway và internal services | Public / Internal REST | API Gateway, internal clients | ✅ |
| 37 | `GET` | `/users/me/favorites` | Danh sách sản phẩm yêu thích (reference) | Authenticated | Favorite Products | ✅ |
| 38 | `POST` | `/users/me/favorites` | Thêm sản phẩm vào yêu thích | Authenticated | Product card/PDP | ✅ |
| 39 | `DELETE` | `/users/me/favorites/{productId}` | Bỏ yêu thích | Authenticated | Favorite Products/PDP | ✅ |
| 40 | `GET` | `/users/me/favorites/contains` | Batch kiểm tra trạng thái tim | Authenticated | Product list/PDP | ✅ |
| 41 | `POST` | `/shops/{shopId}/follow` | Theo dõi shop | Authenticated | Shop hero/PDP shop card | ✅ |
| 42 | `DELETE` | `/shops/{shopId}/follow` | Bỏ theo dõi shop | Authenticated | Shop hero | ✅ |
| 43 | `GET` | `/users/me/following` | Danh sách shop đang theo dõi | Authenticated | Account | ✅ |
| 44 | `GET` | `/shops/{shopId}/followers/count` | Số người theo dõi shop | Public | Shop hero | ✅ |
| 45 | `GET` | `/shops/{shopId}` | Hồ sơ shop công khai (HLD #10) | Public | `Taca Buyer / Shop` | ✅ |
| 46 | `GET` | `/health/live` | Liveness process-only (Gateway active healthcheck gọi — bắt buộc) | Internal/ops | Ops healthcheck | ⏳ Chỉ có `/actuator/health` |
| 47 | `GET` | `/health/ready` | Readiness: MySQL/Kafka/config bắt buộc | Internal/ops | Ops healthcheck | ⏳ Chỉ có `/actuator/health` |
| 48 | `POST` | `/auth/password/change` | Đổi mật khẩu khi đã đăng nhập | Authenticated (+ `X-MFA-Step-Up` nếu user bật 2FA) | Account/Security | ✅ |
| 49 | `POST` | `/auth/google` | Đăng nhập bằng Google ID token | Public | Sign in overlay | ✅ |
| 50 | `GET` | `/auth/google/authorize` | Bắt đầu authorization-code flow (redirect Google) | Public | Sign in overlay | ✅ |
| 51 | `GET` | `/auth/google/callback` | Callback Google, trả session như #49 | Public (state cookie) | Browser | ✅ |
| 52 | `POST` | `/auth/identities/google/link` | Liên kết Google vào tài khoản hiện có | Authenticated | Account/Security | ✅ |
| 53 | `DELETE` | `/seller/onboarding/kyc/documents/{documentId}` | Xoá document KYC chưa submit | Seller | Step 2 — KYC | ✅ |
| 54 | `GET` | `/locations/vn/provinces` | Danh mục tỉnh/thành VN | Public | Address form | ✅ |
| 55 | `GET` | `/locations/vn/provinces/{provinceCode}/wards` | Danh mục phường/xã theo tỉnh | Public | Address form | ✅ |
| 56 | `GET` | `/actuator/health` | Health Spring Boot (`show-details: never`) — path thật trong code | Public (permitAll) | Ops | ✅ |

## 3. Chi tiết endpoint

### 3.1 `POST /auth/signup` — tạo buyer account

Quyền: Public · Rate limit: `10 req/phút/IP` · Response: `201`

Request body:

| Field | Kiểu | Bắt buộc | Ràng buộc | Ví dụ |
|---|---|---:|---|---|
| `full_name` | string | Có | 1–120 Unicode characters, trim | `"Nguyễn Minh Anh"` |
| `email` | string | Có | 3–254 ký tự, email format | `"minhanh@example.com"` |
| `password` | string | Có | 12–72 ký tự; hash Argon2id | `"StrongPassword#2026"` |
| `phone` | string | Không | E.164; unique nếu có | `"+84901234567"` |

```json
{
  "full_name": "Nguyễn Minh Anh",
  "email": "minhanh@example.com",
  "password": "StrongPassword#2026",
  "phone": "+84901234567"
}
```

Response `201`:

```json
{
  "data": {
    "user": {
      "id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
      "full_name": "Nguyễn Minh Anh",
      "email": "minhanh@example.com",
      "email_verified": false,
      "phone": "+84901234567",
      "phone_verified": false,
      "roles": ["BUYER"],
      "status": "ACTIVE"
    },
    "tokens": {
      "token_type": "Bearer",
      "access_token": "eyJ...access-token",
      "expires_in": 900,
      "refresh_token": "opaque-refresh-token",
      "refresh_expires_in": 2592000
    },
    "verification": {
      "email_sent": true,
      "expires_at": "2026-08-31T09:00:00Z"
    }
  },
  "meta": {"request_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi:

| HTTP | Code | Khi nào |
|---:|---|---|
| 400 | `AUTH_INVALID_INPUT` | Sai field/format/password policy. |
| 409 | `AUTH_EMAIL_EXISTS` | `email_normalized` đã tồn tại. |
| 409 | `AUTH_PHONE_EXISTS` | Phone non-null đã tồn tại. |
| 429 | `RATE_LIMITED` | Vượt rate limit. |

### 3.2 `POST /auth/signin` — đăng nhập

Quyền: Public · Rate limit: `10 req/phút/IP` · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc | Ví dụ |
|---|---|---:|---|---|
| `identifier` | string | Có | Email hoặc phone đã normalized/verified | `"minhanh@example.com"` |
| `password` | string | Có | 12–72 ký tự | `"StrongPassword#2026"` |
| `remember_me` | boolean | Không | Default `true`; không đổi TTL v1 | `true` |

Response bình thường `200`: giống `tokens` của signup và trả `user` summary.

Response admin cần MFA `401`:

```json
{
  "error": {
    "code": "AUTH_MFA_REQUIRED",
    "message": "Vui lòng xác thực 2FA.",
    "details": {
      "challenge_id": "01912f40-7a1b-7c12-9c55-8b1c34a6d921",
      "expires_at": "2026-08-30T09:05:00Z",
      "methods": ["TOTP", "RECOVERY_CODE"]
    },
    "trace_id": "01912f41-7a1b-7c12-9c55-8b1c34a6d921"
  }
}
```

Lỗi:

| HTTP | Code | Khi nào |
|---:|---|---|
| 401 | `AUTH_INVALID_CREDENTIALS` | Identifier/password sai; không tiết lộ account. |
| 401 | `AUTH_MFA_REQUIRED` | Admin cần TOTP/recovery code. |
| 403 | `AUTH_PHONE_NOT_VERIFIED` | Dùng phone chưa verified làm identifier. |
| 403 | `AUTH_ACCOUNT_SUSPENDED` | Account suspended/deleted. |
| 423 | `AUTH_ACCOUNT_LOCKED` | 5 lần fail trong 15 phút, lock còn hiệu lực. |
| 429 | `RATE_LIMITED` | Vượt rate limit. |

### 3.3 `POST /auth/refresh` — refresh token rotation

Quyền: Public với refresh token · Response: `200`

Request:

```json
{
  "refresh_token": "opaque-refresh-token"
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `refresh_token` | string | Có | 43–512 ký tự opaque; không log. |

Response `200`: access token mới TTL 900 giây, refresh token mới TTL 30 ngày; token cũ bị revoke.

Lỗi:

| HTTP | Code | Khi nào |
|---:|---|---|
| 401 | `AUTH_TOKEN_INVALID` | Token sai/không tồn tại. |
| 401 | `AUTH_TOKEN_EXPIRED` | Token hết hạn. |
| 401 | `AUTH_REFRESH_REUSED` | Token đã revoked được dùng lại; revoke cả family. |
| 403 | `AUTH_ACCOUNT_SUSPENDED` | User không còn được login. |

### 3.4 `POST /auth/signout` — revoke session

Quyền: Authenticated · Response: `204` không body

Request:

```json
{
  "refresh_token": "opaque-refresh-token",
  "all_sessions": false
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `refresh_token` | string | Không | Nếu có, revoke family chứa token. |
| `all_sessions` | boolean | Không | Default `false`; `true` yêu cầu re-auth/step-up. |

Lỗi: `401 AUTH_TOKEN_INVALID`, `428 RBAC_MFA_REQUIRED` nếu `all_sessions=true` nhưng thiếu re-auth, `500 INTERNAL_ERROR` nếu lỗi ngoài dự kiến.

### 3.5 `POST /auth/email/verify` — verify email

Quyền: Public với opaque token · Response: `200`

Request:

```json
{
  "token": "opaque-email-verification-token"
}
```

Response:

```json
{
  "data": {
    "user_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
    "email_verified": true,
    "verified_at": "2026-08-30T09:20:00Z"
  },
  "meta": {"request_id": "01912f42-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `400 AUTH_VERIFICATION_INVALID` nếu sai/hết hạn/đã dùng; `404 AUTH_USER_NOT_FOUND` không được phân biệt với invalid token ở public response.

### 3.6 `POST /auth/email/resend` — gửi lại email verification

Quyền: Authenticated · Rate limit: tối đa 3 lần/giờ/user · Response: `202`

Request rỗng hoặc:

```json
{
  "purpose": "EMAIL_VERIFY"
}
```

Response:

```json
{
  "data": {
    "accepted": true,
    "expires_at": "2026-08-31T09:20:00Z"
  },
  "meta": {"request_id": "01912f43-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `400 AUTH_INVALID_INPUT`, `409 AUTH_VERIFICATION_ALREADY_COMPLETE`, `429 AUTH_RESEND_LIMIT_EXCEEDED`.

### 3.7 `POST /auth/phone/request-otp` — yêu cầu phone OTP

Quyền: Authenticated · Response: `202`

Request:

```json
{
  "phone": "+84901234567"
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `phone` | string | Có | E.164, unique nếu verify thành công. |

Response không chứa OTP raw:

```json
{
  "data": {
    "challenge_id": "01912f44-7a1b-7c12-9c55-8b1c34a6d921",
    "masked_phone": "+84******567",
    "expires_at": "2026-08-30T09:25:00Z",
    "max_attempts": 5
  },
  "meta": {"request_id": "01912f45-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `400 AUTH_INVALID_INPUT`, `409 AUTH_PHONE_EXISTS`, `429 AUTH_OTP_RATE_LIMITED`.

### 3.8 `POST /auth/phone/verify-otp` — verify phone

Quyền: Authenticated · Response: `200`

Request:

```json
{
  "challenge_id": "01912f44-7a1b-7c12-9c55-8b1c34a6d921",
  "otp": "123456"
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `challenge_id` | UUID string | Có | Challenge của user hiện tại. |
| `otp` | string | Có | Đúng 6 chữ số, tối đa 5 attempts. |

Response: `phone_verified=true`, `phone_verified_at`.

Lỗi: `400 AUTH_VERIFICATION_INVALID`, `429 AUTH_OTP_ATTEMPTS_EXCEEDED`, `409 AUTH_PHONE_EXISTS`.

### 3.9 `POST /auth/password/forgot` — khởi tạo reset

Quyền: Public · Response: `202` luôn cùng message an toàn

Request:

```json
{
  "identifier": "minhanh@example.com"
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `identifier` | string | Có | Email hoặc phone; trim, không log raw. |

Response:

```json
{
  "data": {
    "accepted": true,
    "message": "Nếu tài khoản tồn tại, hướng dẫn đặt lại mật khẩu sẽ được gửi."
  },
  "meta": {"request_id": "01912f46-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `400 AUTH_INVALID_INPUT`, `429 RATE_LIMITED`. Không trả `USER_NOT_FOUND`.

### 3.10 `POST /auth/password/reset` — đặt password mới

Quyền: Public với reset token · Response: `204`

Request:

```json
{
  "token": "opaque-password-reset-token",
  "new_password": "NewStrongPassword#2026"
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `token` | string | Có | One-time, TTL 30 phút. |
| `new_password` | string | Có | 12–72 ký tự, Argon2id sau khi validate. |

Lỗi: `400 AUTH_RESET_INVALID`, `400 AUTH_INVALID_INPUT`, `429 RATE_LIMITED`.

### 3.11 `POST /auth/2fa/setup` — tạo TOTP enrollment

Quyền: Admin authenticated · Response: `201`

Request rỗng.

Response `201`:

```json
{
  "data": {
    "setup_id": "01912f47-7a1b-7c12-9c55-8b1c34a6d921",
    "issuer": "Taca Marketplace",
    "account": "admin@example.com",
    "otpauth_uri": "otpauth://totp/Taca%20Marketplace:admin@example.com?...",
    "expires_at": "2026-08-30T09:05:00Z"
  },
  "meta": {"request_id": "01912f48-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Ràng buộc:

- Không trả `secret` plaintext riêng ngoài `otpauth_uri` bảo vệ bởi HTTPS; không log response.
- Setup chưa enabled cho tới khi verify TOTP.
- Nếu user đã có credential `ENABLED`, trả `409 AUTH_MFA_ALREADY_ENABLED`.

### 3.12 `POST /auth/2fa/verify` — login hoặc enrollment

Quyền: MFA challenge hoặc authenticated enrollment · Response: `200`

Request login:

```json
{
  "purpose": "LOGIN",
  "challenge_id": "01912f40-7a1b-7c12-9c55-8b1c34a6d921",
  "method": "TOTP",
  "code": "123456"
}
```

Request enrollment:

```json
{
  "purpose": "ENROLL",
  "setup_id": "01912f47-7a1b-7c12-9c55-8b1c34a6d921",
  "method": "TOTP",
  "code": "123456"
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `purpose` | enum | Có | `LOGIN`, `ENROLL`, `STEP_UP`. |
| `challenge_id` | UUID | Khi LOGIN/STEP_UP | Challenge đúng user. |
| `setup_id` | UUID | Khi ENROLL | Setup còn hạn. |
| `method` | enum | Có | `TOTP` hoặc `RECOVERY_CODE`. |
| `code` | string | Có | TOTP 6 số hoặc recovery code opaque. |

Response `LOGIN`: tokens mới. Response `ENROLL`: `status=ENABLED` và recovery codes chỉ trả đúng một lần.

Lỗi: `401 AUTH_MFA_INVALID`, `400 AUTH_MFA_CHALLENGE_EXPIRED`, `429 AUTH_MFA_ATTEMPTS_EXCEEDED`.

### 3.13 `GET /users/me` — đọc profile

Quyền: Authenticated · Response: `200`

Response:

```json
{
  "data": {
    "id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
    "full_name": "Nguyễn Minh Anh",
    "email": "minhanh@example.com",
    "email_verified": true,
    "phone": "+84901234567",
    "phone_verified": true,
    "date_of_birth": "1995-06-30",
    "roles": ["BUYER", "SELLER"],
    "status": "ACTIVE",
    "default_shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
    "created_at": "2026-08-30T09:00:00Z",
    "updated_at": "2026-08-30T09:20:00Z"
  },
  "meta": {"request_id": "01912f4a-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `401 AUTH_TOKEN_INVALID`, `404 AUTH_USER_NOT_FOUND` nếu subject không còn tồn tại.

### 3.14 `PUT /users/me` — cập nhật profile

Quyền: Authenticated · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `full_name` | string | Có | 1–120 Unicode characters. |
| `phone` | string/null | Không | E.164; đổi phone clear verification. |
| `date_of_birth` | date/null | Không | Không ở tương lai. |

`email`, `role`, `status`, `user_id` không được gửi hoặc sẽ bị từ chối.

```json
{
  "data": {
    "id": "01912f10-7a1b-7c12-9c55-8b1c34a6d921",
    "full_name": "Nguyễn Văn A",
    "phone": "+84901234567",
    "date_of_birth": "1995-05-20",
    "phone_verification_required": true,
    "updated_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "01912fd0-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

`phone_verification_required=true` khi phone vừa đổi — client phải điều hướng sang luồng verify OTP.

Lỗi: `400 PROFILE_INVALID`, `409 AUTH_PHONE_EXISTS`, `401 AUTH_TOKEN_INVALID`.

### 3.15 `GET /users/me/addresses` — list address

Quyền: Authenticated · Query: `page`, `size`, `sort=created_at,desc` · Response: `200`

Response item:

```json
{
  "id": "01912f4b-7a1b-7c12-9c55-8b1c34a6d921",
  "recipient": "Nguyễn Minh Anh",
  "phone": "+84901234567",
  "line1": "12 Nguyễn Huệ",
  "line2": "Tòa A, căn 501",
  "country_code": "VN",
  "province_code": "79",
  "province": "Hồ Chí Minh",
  "ward_code": "26740",
  "ward": "Bến Nghé",
  "district": null,
  "postal_code": "700000",
  "is_default": true,
  "created_at": "2026-08-30T09:10:00Z",
  "updated_at": "2026-08-30T09:10:00Z"
}
```

Lỗi: `400 AUTH_INVALID_INPUT` nếu query sai, `401 AUTH_TOKEN_INVALID`.

### 3.16 `POST /users/me/addresses` — thêm address

Quyền: Authenticated · Response: `201`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc | Ví dụ |
|---|---|---:|---|---|
| `recipient` | string | Có | 1–120 | `"Nguyễn Minh Anh"` |
| `phone` | string | Có | E.164 | `"+84901234567"` |
| `line1` | string | Có | 1–255 | `"12 Nguyễn Huệ"` |
| `line2` | string | Không | Tối đa 255 | `"Tòa A"` |
| `country_code` | string | Có | Chỉ `VN` | `"VN"` |
| `province_code` | string | Có | Mã trong `GET /locations/vn/provinces` | `"79"` |
| `ward_code` | string | Có | Phải thuộc `province_code` | `"26740"` |
| `postal_code` | string | Không | Tối đa 12 | `"700000"` |
| `is_default` | boolean | Không | Default `false`; address đầu tiên tự default | `true` |

Response `201` cùng shape với item ở `GET /users/me/addresses` (§3.15) — `id` mới sinh, `created_at`/`updated_at` bằng thời điểm tạo.

Client **không** gửi tên tỉnh/phường; service tự resolve tên từ catalog và lưu cả mã lẫn tên (mô hình 2 cấp, không còn `district`; cột `district` cũ chỉ còn cho bản ghi trước `V021`).

Lỗi: `400 PROFILE_INVALID`, `422 ADDRESS_LOCATION_INVALID`, `409 ADDRESS_LIMIT_REACHED`, `409 ADDRESS_DEFAULT_REQUIRED`.

### 3.17 `PUT /users/me/addresses/{addressId}` — sửa address

Quyền: Authenticated · Response: `200`

Request: cùng field với POST; `addressId` UUID string.

Ràng buộc:

- Chỉ update address có `user_id = JWT.sub` và `deleted_at IS NULL`.
- Đổi `is_default=true` clear default cũ trong cùng transaction.
- Không tin `user_id` trong body.

Response `200` cùng shape với `GET /users/me/addresses` item, `updated_at` mới.

Lỗi: `400 PROFILE_INVALID`, `404 ADDRESS_NOT_FOUND`, `409 ADDRESS_DEFAULT_REQUIRED`.

### 3.18 `DELETE /users/me/addresses/{addressId}` — soft delete

Quyền: Authenticated · Response: `204`

Ràng buộc:

- Không xóa cứng; chỉ set `deleted_at`.
- Nếu xóa default, tự động chọn address sống gần nhất theo `updated_at DESC` làm default mới.
- Order Service lưu `address_snapshot` riêng độc lập; soft-delete address tại Auth User không ảnh hưởng đơn hàng đã tạo hoặc phiên checkout đã clone snapshot.
- Address không thuộc user trả `404 ADDRESS_NOT_FOUND`, không trả 403.

Lỗi: `404 ADDRESS_NOT_FOUND`, `409 ADDRESS_DEFAULT_REQUIRED`.

### 3.19 `POST /users/register-seller` — tạo shop/onboarding

Quyền: Authenticated + email verified · Response: `201`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `name` | string | Có | 1–120; tên hiển thị shop. |
| `business_name` | string | Có | 1–200. |
| `tax_code` | string | Có | 10–14 chữ số sau normalize; unique. |
| `slug` | string | Không | 3–160, lowercase/hyphen; server generate nếu thiếu. |
| `description` | string | Không | Tối đa 2.000. |

Response:

```json
{
  "data": {
    "shop": {
      "id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
      "name": "Taca Home",
      "slug": "taca-home",
      "status": "DRAFT",
      "kyc_status": "DRAFT"
    },
    "onboarding": {
      "current_step": "PROFILE",
      "completed_steps": [],
      "blockers": []
    }
  },
  "meta": {"request_id": "01912f4c-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

`blockers` là mã gợi ý free-form cho FE (không phải enum chốt — nguồn lưu là `blockers_json` ở `db` §3.4); chỉ xuất hiện mã khi điều kiện tương ứng thật sự thiếu (VD `KYC_DOCUMENT_REQUIRED` khi chưa có document).

Lỗi: `403 AUTH_EMAIL_NOT_VERIFIED`, `409 SHOP_ALREADY_EXISTS`, `409 AUTH_TAX_CODE_EXISTS`, `400 PROFILE_INVALID`.

### 3.20 `GET /seller/onboarding` — tiến độ onboarding

Quyền: `SELLER`/owner · Response: `200`

Response:

```json
{
  "data": {
    "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
    "current_step": "KYC",
    "steps": [
      {"key": "PROFILE", "completed": true},
      {"key": "KYC", "completed": false},
      {"key": "WAREHOUSE", "completed": false},
      {"key": "BANK", "completed": false},
      {"key": "FIRST_PRODUCT", "completed": false}
    ],
    "shop_status": "DRAFT",
    "kyc_status": "DRAFT",
    "blockers": ["KYC_DOCUMENT_REQUIRED"]
  },
  "meta": {"request_id": "01912f4d-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `404 SHOP_NOT_FOUND`, `403 RBAC_PERMISSION_DENIED`.

`onboarding_status` trả ở các response step là enum **suy ra** từ `current_step` + cờ step (không phải cột lưu — xem `docs/db/auth-user.md` §5.1 `OnboardingStatus`): `NOT_STARTED` (chưa hoàn thành step nào), `IN_PROGRESS`, `READY_TO_SUBMIT` (đủ hồ sơ/KYC/bank, chờ seller submit), `COMPLETED`.

### 3.21 `PUT /seller/onboarding/profile` — step Hồ sơ

Quyền: Seller owner · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `name` | string | Có | 1–120. |
| `business_name` | string | Có | 1–200. |
| `tax_code` | string | Có | 10–14 digits, unique. |
| `description` | string | Không | Tối đa 2.000. |
| `logo_object_key` | string | Không | Chỉ object key đã presign/allowlist. |

```json
{ "data": { "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921", "step": "PROFILE", "completed": true, "onboarding_status": "IN_PROGRESS" }, "meta": { "request_id": "01912fd1-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Lỗi: `400 PROFILE_INVALID`, `403 RBAC_PERMISSION_DENIED`, `409 AUTH_TAX_CODE_EXISTS`, `409 SHOP_INVALID_STATE`.

### 3.22 `PUT /seller/onboarding/warehouse` — step Kho & vận chuyển

Quyền: Seller owner · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `warehouse_name` | string | Có | 1–120. |
| `contact_name` | string | Có | 1–120. |
| `contact_phone` | string | Có | E.164. |
| `address` | object | Có | Gồm line1, ward, district, province. |
| `carrier_preferences` | string[] | Không | Mỗi key 1–32, tối đa 10. |
| `cod_enabled` | boolean | Không | Default `true`. |

`address` dùng các field `line1` 1–255, `line2` optional 255, `ward/district/province` 1–120, `postal_code` optional 12.

```json
{ "data": { "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921", "step": "WAREHOUSE", "completed": true, "onboarding_status": "IN_PROGRESS" }, "meta": { "request_id": "01912fd2-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Lỗi: `400 PROFILE_INVALID`, `403 RBAC_PERMISSION_DENIED`, `409 SHOP_INVALID_STATE`.

### 3.23 `POST /seller/onboarding/kyc/documents/presign` — tạo signed URL

Quyền: Seller owner · Response: `201`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `document_type` | enum string | Có | Document type đã seed. |
| `file_name` | string | Có | 1–255, sanitize path. |
| `content_type` | enum | Có | `application/pdf`, `image/jpeg`, `image/png`. |
| `size_bytes` | integer | Có | 1–10 MiB (`10485760`). |
| `sha256` | hex string | Có | Đúng 64 ký tự hex. |

Response:

```json
{
  "data": {
    "document_id": "01912f4e-7a1b-7c12-9c55-8b1c34a6d921",
    "object_key": "private/kyc/01912f49/document-01912f4e.pdf",
    "upload_url": "https://storage.example/mock-upload/opaque",
    "expires_at": "2026-08-30T09:30:00Z",
    "required_headers": {
      "Content-Type": "application/pdf",
      "x-content-sha256": "hex-64-character-checksum"
    }
  },
  "meta": {"request_id": "01912f4f-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `400 KYC_DOCUMENT_INVALID`, `413 KYC_DOCUMENT_TOO_LARGE`, `409 KYC_DOCUMENT_LIMIT_REACHED`.

### 3.24 `POST /seller/onboarding/kyc/documents/complete` — complete upload

Quyền: Seller owner · Response: `201`

Request:

```json
{
  "document_id": "01912f4e-7a1b-7c12-9c55-8b1c34a6d921",
  "object_key": "private/kyc/01912f49/document-01912f4e.pdf",
  "size_bytes": 5242880,
  "content_type": "application/pdf",
  "sha256": "hex-64-character-checksum"
}
```

Service kiểm tra object metadata/checksum/content type với Storage Adapter; client không được đổi `object_key` sang case khác.

Lỗi: `400 KYC_DOCUMENT_INVALID`, `404 KYC_DOCUMENT_NOT_FOUND`, `409 KYC_DOCUMENT_ALREADY_COMPLETED`.

### 3.25 `POST /seller/onboarding/kyc/submit` — submit KYC

Quyền: Seller owner + email verified · Response: `200`

Request rỗng.

Điều kiện:

- Hồ sơ bắt buộc hoàn tất.
- Có ít nhất một document `UPLOADED/VERIFIED` hợp lệ.
- Không có upload đang `UPLOADING`.
- Shop hiện không `SUSPENDED`.

Response:

```json
{
  "data": {
    "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
    "kyc_case_id": "01912f50-7a1b-7c12-9c55-8b1c34a6d921",
    "status": "PENDING",
    "submitted_at": "2026-08-30T09:35:00Z"
  },
  "meta": {"request_id": "01912f51-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `400 KYC_DOCUMENT_INVALID`, `409 SHOP_INVALID_STATE`, `409 KYC_ALREADY_PENDING`, `403 AUTH_EMAIL_NOT_VERIFIED`.

### 3.26 `PUT /seller/onboarding/bank` — step Ngân hàng

Quyền: Seller owner · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `bank_code` | string | Có | 1–32, nằm trong bank catalog. |
| `bank_name` | string | Có | 1–120. |
| `account_name` | string | Có | 1–120, normalize uppercase theo provider nếu cần. |
| `account_number` | string | Có | 8–20 digits; encrypt trước khi lưu. |
| `confirm_account_name` | boolean | Có | Phải `true`. |

```json
{ "data": { "bank_name": "Vietcombank", "account_number_masked": "********1234", "verified": false, "step": "BANK", "completed": true, "onboarding_status": "READY_TO_SUBMIT" }, "meta": { "request_id": "01912fd3-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Response chỉ trả `bank_name`, masked account `********1234`, `verified=false/true`; không trả account number raw.

Lỗi: `400 PROFILE_INVALID`, `409 BANK_ACCOUNT_INVALID`, `403 RBAC_PERMISSION_DENIED`.

### 3.27 `GET /seller/shop` — đọc shop profile

Quyền: Seller owner/staff · Response: `200`

```json
{
  "data": {
    "id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
    "name": "Taca Home",
    "slug": "taca-home",
    "business_name": "Công ty TNHH Taca Home",
    "tax_code_masked": "03***4567",
    "description": "Đồ gia dụng chính hãng",
    "logo_url": "https://cdn.example/signed",
    "status": "ACTIVE",
    "kyc_status": "APPROVED",
    "warehouse_summary": { "warehouse_name": "Kho HCM", "district": "Quận 1", "province": "TP.HCM" },
    "bank_summary": { "bank_name": "Vietcombank", "account_number_masked": "********1234", "verified": true },
    "created_at": "2026-08-01T09:00:00Z",
    "updated_at": "2026-08-30T09:00:00Z"
  },
  "meta": { "request_id": "01912fd4-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Lỗi: `404 SHOP_NOT_FOUND`, `403 RBAC_PERMISSION_DENIED`.

### 3.28 `PUT /seller/shop` — cập nhật shop profile

Quyền: Seller owner · Response: `200`

Request cho phép `name`, `description`, `logo_object_key`; `tax_code`, owner và status không đổi tại endpoint này. `slug` đổi phải qua validation unique riêng trong service.

```json
{ "data": { "id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921", "name": "Taca Home Official", "description": "Đồ gia dụng chính hãng, bảo hành 12 tháng", "logo_url": "https://cdn.example/signed", "updated_at": "2026-08-30T09:05:00Z" }, "meta": { "request_id": "01912fd5-7a1b-7c12-9c55-8b1c34a6d921" } }
```

Lỗi: `400 PROFILE_INVALID`, `403 RBAC_PERMISSION_DENIED`, `409 SHOP_SLUG_EXISTS`, `409 SHOP_INVALID_STATE`.

### 3.29 `GET /admin/shops/kyc` — KYC queue

Quyền: `KYC_READ` · Query:

| Query | Kiểu | Default | Ràng buộc |
|---|---|---|---|
| `status` | enum | `PENDING` | `DRAFT/PENDING/NEEDS_INFO/APPROVED/REJECTED/EXPIRED/SUSPENDED`. |
| `page` | integer | 1 | `>=1`. |
| `size` | integer | 20 | `1–100`. |
| `sort` | enum | `submitted_at,asc` | `submitted_at`, `updated_at`, `expires_at`. |
| `q` | string | — | Tối đa 120; search shop name/tax code masked theo policy. |

Response item:

```json
{
  "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
  "shop_name": "Taca Home",
  "owner_user_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
  "kyc_case_id": "01912f50-7a1b-7c12-9c55-8b1c34a6d921",
  "status": "PENDING",
  "submitted_at": "2026-08-30T09:35:00Z",
  "document_count": 3,
  "age_hours": 2.5
}
```

Lỗi: `401 AUTH_TOKEN_INVALID`, `403 RBAC_PERMISSION_DENIED`, `400 AUTH_INVALID_INPUT`.

### 3.30 `GET /admin/shops/{shopId}/kyc` — KYC detail

Quyền: `KYC_READ` · Response: `200`

Response:

```json
{
  "data": {
    "shop": {
      "id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
      "name": "Taca Home",
      "business_name": "Taca Home Company",
      "tax_code_masked": "010*****123",
      "status": "ACTIVE"
    },
    "kyc_case": {
      "id": "01912f50-7a1b-7c12-9c55-8b1c34a6d921",
      "status": "PENDING",
      "submitted_at": "2026-08-30T09:35:00Z",
      "documents": [
        {
          "id": "01912f4e-7a1b-7c12-9c55-8b1c34a6d921",
          "document_type": "BUSINESS_REGISTRATION",
          "original_file_name": "business-registration.pdf",
          "content_type": "application/pdf",
          "size_bytes": 5242880,
          "status": "VERIFIED",
          "download_url": "https://storage.example/mock-download/opaque",
          "download_expires_at": "2026-08-30T09:45:00Z"
        }
      ]
    }
  },
  "meta": {"request_id": "01912f52-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

`download_url` chỉ cấp cho admin có `KYC_READ`, TTL 10 phút; không trả object key public.

Lỗi: `404 SHOP_NOT_FOUND`, `404 KYC_CASE_NOT_FOUND`, `403 RBAC_PERMISSION_DENIED`.

### 3.31 `POST /admin/shops/{shopId}/kyc/review` — quyết định KYC

Quyền: `KYC_DECIDE` + step-up 2FA khi reject/suspend · Response: `200`

Request:

```json
{
  "decision": "APPROVED",
  "reason": "Hồ sơ và tài liệu hợp lệ."
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `decision` | enum | Có | `APPROVED`, `NEEDS_INFO`, `REJECTED`. |
| `reason` | string | Có với NEEDS_INFO/REJECTED | 10–1.000 Unicode characters. APPROVED khuyến nghị có reason. |

Ràng buộc:

- Chỉ `RISK_MANAGER`/`SUPER_ADMIN` có quyền quyết định.
- Case phải `PENDING`; quyết định trùng hoặc case đã terminal trả `409 KYC_DECISION_INVALID`.
- Ghi audit với actor, reason, old/new state.
- Phát `shop.kyc.approved/needs_info/rejected` qua outbox.

Lỗi: `400 KYC_DECISION_INVALID`, `403 RBAC_PERMISSION_DENIED`, `428 RBAC_MFA_REQUIRED`, `409 SHOP_INVALID_STATE`.

### 3.32 `GET /admin/users/{userId}/roles` — đọc role/permission

Quyền: `ROLE_READ` hoặc `SUPER_ADMIN` · Response: `200`

Response:

```json
{
  "data": {
    "user_id": "01912f31-7a1b-7c12-9c55-8b1c34a6d921",
    "assignments": [
      {
        "role": "SELLER_STAFF",
        "scope_type": "SHOP",
        "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
        "permissions": ["SHOP_READ"],
        "granted_at": "2026-08-30T09:40:00Z",
        "granted_by": "01912f53-7a1b-7c12-9c55-8b1c34a6d921"
      }
    ]
  },
  "meta": {"request_id": "01912f54-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `404 AUTH_USER_NOT_FOUND`, `403 RBAC_PERMISSION_DENIED`.

### 3.33 `PATCH /admin/users/{userId}/roles` — grant/revoke role

Quyền: `ROLE_ASSIGN` + `X-MFA-Step-Up` · Response: `200`

Request:

```json
{
  "action": "GRANT",
  "role": "SELLER_STAFF",
  "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
  "reason": "Bổ sung nhân viên vận hành shop."
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `action` | enum | Có | `GRANT`, `REVOKE`. |
| `role` | enum | Có | Role baseline; không tự tạo role qua API này. |
| `shop_id` | UUID/null | Theo role | Bắt buộc với `SELLER_STAFF`; zero/system scope với admin role. |
| `reason` | string | Có | 10–1.000 characters. |

Ràng buộc:

- Không cho actor tự cấp role cho chính mình.
- Không cấp role cao hơn permission của actor.
- `SUPER_ADMIN` role assignment phải có 2FA và policy riêng.
- Mutation transaction update assignment + audit + `user.role_changed` event.

Lỗi: `400 RBAC_INVALID_ROLE`, `403 RBAC_PERMISSION_DENIED`, `428 RBAC_MFA_REQUIRED`, `409 RBAC_ASSIGNMENT_EXISTS`, `409 RBAC_ASSIGNMENT_NOT_FOUND`.

### 3.34 `PATCH /admin/users/{userId}/status` — suspend/restore

Quyền: `USER_SUSPEND` + `X-MFA-Step-Up` · Response: `200`

Request:

```json
{
  "status": "SUSPENDED",
  "reason": "Vi phạm chính sách nền tảng."
}
```

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `status` | enum | Có | `ACTIVE` hoặc `SUSPENDED`; `DELETED` không qua endpoint v1. |
| `reason` | string | Có | 10–1.000 characters. |

Khi suspend: revoke toàn bộ refresh token family, ghi audit, phát `user.status_changed` (kèm đẩy key `revoked_user:{user_id}` TTL 15 phút vào Redis chung của Gateway để chặn Access JWT ngay lập tức). Access token JWT chưa hết hạn cũng sẽ bị Gateway từ chối sau khi đồng bộ cache.

Lỗi: `404 AUTH_USER_NOT_FOUND`, `403 RBAC_PERMISSION_DENIED`, `428 RBAC_MFA_REQUIRED`, `403 AUTH_ACCOUNT_SUSPENDED`.

### 3.35 `GET /admin/audit-logs` — tra cứu audit

Quyền: `ROLE_READ`, `KYC_READ`, `USER_READ` hoặc permission tương ứng · Response: `200`

Query:

| Query | Kiểu | Ràng buộc |
|---|---|---|
| `actor_user_id` | UUID | Optional. |
| `target_type` | enum | `USER/SHOP/KYC/ROLE`. |
| `target_id` | UUID | Optional. |
| `action` | string | 1–64, allowlist. |
| `from`/`to` | ISO timestamp | Khoảng tối đa 31 ngày/request. |
| `page`,`size`,`sort` | pagination | Default page 1/size 20/sort `occurred_at,desc`. |

```json
{
  "data": [
    {
      "log_id": "01912fe0-7a1b-7c12-9c55-8b1c34a6d921",
      "actor_user_id": "01912f10-7a1b-7c12-9c55-8b1c34a6d921",
      "actor_role": "SUPER_ADMIN",
      "action": "USER_SUSPEND",
      "target_type": "USER",
      "target_id": "01912f20-7a1b-7c12-9c55-8b1c34a6d921",
      "reason": "Vi phạm chính sách nền tảng.",
      "occurred_at": "2026-08-30T09:00:00Z"
    }
  ],
  "meta": { "request_id": "01912fe1-7a1b-7c12-9c55-8b1c34a6d921", "page": 1, "size": 20, "total": 340, "total_pages": 17 }
}
```

Không trả `metadata` có secret/PII raw; field nhạy cảm phải masked.

Lỗi: `400 AUTH_INVALID_INPUT`, `403 RBAC_PERMISSION_DENIED`.

### 3.36 `GET /.well-known/jwks.json` — Public JWKS key set

Quyền: Public / Internal REST · Cache-Control: `public, max-age=600` · Response: `200`

Response:

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

- Chỉ trả public key modulus `n` và exponent `e`; tuyệt đối không chứa private exponent `d` hoặc prime numbers.
- `kid` khớp với `kid` trong header của Access Token JWT.
- Khi xoay key (key rotation), endpoint giữ cả key cũ và key mới trong thời gian overlap (tối thiểu 30 phút).

Lỗi: `500 INTERNAL_ERROR` nếu keystore không sẵn sàng.

### 3.37 `GET /users/me/favorites` — danh sách yêu thích

Quyền: Authenticated · Query: `page`, `size`, `sort=created_at,desc` · Response: `200`

```json
{
  "data": [
    { "product_id": "01912f80-7a1b-7c12-9c55-8b1c34a6d921", "created_at": "2026-08-30T09:10:00Z" }
  ],
  "meta": { "page": 1, "size": 20, "total": 12, "total_pages": 1, "request_id": "01912f81-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Chỉ trả reference `product_id`; client/BFF hydrate thẻ sản phẩm (title, giá, ảnh, còn hàng) qua Product Catalog. auth-user không có nội dung sản phẩm nên không hỗ trợ tìm kiếm theo tên trong danh sách (`Field / Search favorites` do frontend lọc client-side hoặc BFF).

Lỗi: `400 AUTH_INVALID_INPUT`, `401 AUTH_TOKEN_INVALID`.

### 3.38 `POST /users/me/favorites` — thêm yêu thích

Quyền: Authenticated · Response: `201` (hoặc `200` nếu đã có — idempotent)

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `product_id` | UUID string | Có | Chỉ kiểm định dạng; không kiểm product tồn tại. |

```json
{ "product_id": "01912f80-7a1b-7c12-9c55-8b1c34a6d921" }
```

Response `201`: `{ "data": { "product_id": "...", "created_at": "2026-08-30T09:10:00Z" }, "meta": { "request_id": "..." } }`

Lỗi: `400 AUTH_INVALID_INPUT`, `409 FAVORITE_LIMIT_REACHED` (đạt `MAX_FAVORITES_PER_USER=500`).

### 3.39 `DELETE /users/me/favorites/{productId}` — bỏ yêu thích

Quyền: Authenticated · Response: `204` (idempotent — không tồn tại vẫn `204`)

### 3.40 `GET /users/me/favorites/contains` — batch trạng thái tim

Quyền: Authenticated · Query: `product_ids` (danh sách phân tách bằng dấu phẩy, tối đa 100) · Response: `200`

```json
{
  "data": {
    "01912f80-7a1b-7c12-9c55-8b1c34a6d921": true,
    "01912f80-7a1b-7c12-9c55-8b1c34a6d922": false
  },
  "meta": { "request_id": "01912f82-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Lỗi: `400 AUTH_INVALID_INPUT` nếu quá 100 ID hoặc ID sai định dạng.

### 3.41 `POST /shops/{shopId}/follow` — theo dõi shop

Quyền: Authenticated · Response: `201` (hoặc `200` idempotent nếu đã follow)

```json
{ "data": { "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921", "followed_at": "2026-08-30T09:12:00Z" }, "meta": { "request_id": "..." } }
```

Ràng buộc: shop phải tồn tại và `status != DELETED` (kiểm bằng bảng `shops` local).

Lỗi: `404 SHOP_NOT_FOUND`, `409 SHOP_FOLLOW_LIMIT_REACHED` (đạt `MAX_FOLLOWED_SHOPS_PER_USER=1000`).

### 3.42 `DELETE /shops/{shopId}/follow` — bỏ theo dõi

Quyền: Authenticated · Response: `204` (idempotent)

### 3.43 `GET /users/me/following` — shop đang theo dõi

Quyền: Authenticated · Query: `page`, `size`, `sort=created_at,desc` · Response: `200`

```json
{
  "data": [
    { "shop_id": "01912f49-...", "name": "Taca Home", "slug": "taca-home", "logo_url": "https://cdn.example/signed", "followed_at": "2026-08-30T09:12:00Z" }
  ],
  "meta": { "page": 1, "size": 20, "total": 3, "total_pages": 1, "request_id": "..." }
}
```

`name/slug/logo_url` là snapshot từ bảng `shops`; có thể trễ so với cập nhật mới nhất của shop.

### 3.44 `GET /shops/{shopId}/followers/count` — số người theo dõi

Quyền: Public · Response: `200`

```json
{ "data": { "shop_id": "01912f49-...", "follower_count": 1284 }, "meta": { "request_id": "..." } }
```

`follower_count` eventual; không đảm bảo realtime tuyệt đối.

### 3.45 `GET /shops/{shopId}` — hồ sơ shop công khai

Quyền: Public · Response: `200`

Path segment nhận **`shop_id` (UUID) hoặc `slug`** — đây là endpoint duy nhất trong family chấp nhận cả hai, để FE vào được từ URL SEO dạng `/shop/taca-home`. Phân biệt theo hình dạng: chuỗi khớp đúng dạng UUID 36 ký tự có gạch nối → tra theo `id`; ngược lại tra theo `slug` (charset lowercase/hyphen, 3–160). Vì `slug` không bao giờ hợp lệ dưới dạng UUID nên không có trường hợp nhập nhằng.

Mọi endpoint khác trong family (`/shops/{shopId}/follow`, `/shops/{shopId}/followers/count`, `/shops/{shopId}/products` của `product-catalog`) chỉ nhận **UUID**. FE gọi endpoint này trước để lấy `id`, rồi dùng `id` cho các lời gọi tiếp theo.

```json
{
  "data": {
    "id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921",
    "name": "Taca Home",
    "slug": "taca-home",
    "logo_url": "https://cdn.example/signed",
    "description": "Đồ gia dụng chính hãng",
    "is_verified": true,
    "status": "ACTIVE",
    "follower_count": 1284,
    "created_at": "2026-08-01T09:00:00Z"
  },
  "meta": { "request_id": "01912f83-7a1b-7c12-9c55-8b1c34a6d921" }
}
```

Ràng buộc:

- `is_verified = (kyc_status == APPROVED)`; không trả `kyc_status`, `business_name`, `tax_code`, `bank_*`, `warehouse_snapshot`, `owner_user_id`.
- `rating_avg`, `product_count`, `sales_count` **không** thuộc endpoint này — do Product Catalog phục vụ; Frontend Shop hero ghép hai nguồn.
- Đáp ứng HLD mục 10.

Lỗi: `404 SHOP_NOT_FOUND` (không tồn tại hoặc `status = DELETED`).

### 3.46 `GET /health/live` — liveness (⏳ chưa có trong code)

> Code hiện chỉ expose Spring Actuator `GET /actuator/health` (permitAll trong `SecurityConfig`, `show-details: never`). Gateway healthcheck phải trỏ `/actuator/health` cho tới khi service thêm `/health/live`.

Quyền: Internal/ops · Response: `200`

```json
{ "status": "UP", "service": "auth-user", "time": "2026-08-30T09:00:00Z" }
```

Liveness process-only — không kiểm MySQL/Kafka/JWKS keystore; process còn nhận request thì `UP`. Gateway active healthcheck gọi endpoint này — ràng buộc cứng theo `api-gateway-docs/docs/lld/api-gateway.md` §5.2.

### 3.47 `GET /health/ready` — readiness (⏳ chưa có trong code)

Quyền: Internal/ops · Response: `200` hoặc `503`

Kiểm MySQL (`userdb`), Kafka và cấu hình bắt buộc (JWKS keystore, secret manager). Không trả secret/internal address.

### 3.48 `POST /auth/password/change` — đổi mật khẩu

Quyền: Authenticated · Header: `X-MFA-Step-Up` khi user bật 2FA · Response: `200`

Request: `current_password`, `new_password`, `all_sessions` (boolean, default `false`). Khi `all_sessions=true` mọi session khác bị revoke và response trả cặp token mới cho session hiện tại. Phát `user.password_changed`.

Lỗi: `401 AUTH_CURRENT_PASSWORD_INVALID`, `400 AUTH_PASSWORD_REUSE`, `400 AUTH_INVALID_INPUT`, `401 AUTH_MFA_REQUIRED`.

### 3.49 Google Sign-In — `POST /auth/google`, `GET /auth/google/authorize`, `GET /auth/google/callback`, `POST /auth/identities/google/link`

- `POST /auth/google`: body `id_token` (Google). Token được verify với JWKS Google (`AUTH_GOOGLE_JWK_SET_URI`) và `aud` thuộc `AUTH_GOOGLE_ALLOWED_CLIENT_IDS`. Trả session như `POST /auth/signin`; nếu tài khoản bật 2FA thì trả challenge và client phải gọi `POST /auth/2fa/verify` với purpose `LOGIN`.
- `GET /auth/google/authorize` → redirect Google (PKCE, state cookie 10 phút); `GET /auth/google/callback?code=&state=` trả session payload như `POST /auth/google`.
- `POST /auth/identities/google/link`: user đã đăng nhập liên kết Google identity.
- Lỗi: `401 AUTH_GOOGLE_TOKEN_INVALID`, `409 AUTH_GOOGLE_LINK_REQUIRED` (email đã có tài khoản password, cần link), `409 AUTH_GOOGLE_IDENTITY_LINKED`, `503 AUTH_GOOGLE_UNAVAILABLE`.
- Chi tiết: `docs/google-signin-contract.md` trong service repo.

### 3.50 `DELETE /seller/onboarding/kyc/documents/{documentId}` — xoá document KYC

Quyền: Seller · Response: `204`. Chỉ xoá document chưa submit. Lỗi: `409 KYC_DOCUMENT_DELETE_NOT_ALLOWED`, `404` khi không thuộc shop.

### 3.51 `GET /locations/vn/provinces`, `GET /locations/vn/provinces/{provinceCode}/wards`

Quyền: Public · Response: `200` danh sách `{code, name}`. Danh mục 2 cấp tỉnh → phường/xã (không có quận/huyện). Lỗi: `404 LOCATION_PROVINCE_NOT_FOUND`. Chi tiết: `docs/vietnam-location-api.md` trong service repo.

## 4. Mã lỗi chung

| Code | HTTP | Ý nghĩa | Thông điệp hiển thị |
|---|---:|---|---|
| `AUTH_INVALID_INPUT` | 400 | Payload/format không hợp lệ | `Thông tin gửi lên chưa đúng.` |
| `AUTH_EMAIL_EXISTS` | 409 | Email đã dùng | `Email đã được sử dụng.` |
| `AUTH_PHONE_EXISTS` | 409 | Phone đã dùng | `Số điện thoại đã được sử dụng.` |
| `AUTH_TAX_CODE_EXISTS` | 409 | Tax code đã dùng | `Mã số thuế đã được sử dụng.` |
| `AUTH_INVALID_CREDENTIALS` | 401 | Signin sai | `Email/số điện thoại hoặc mật khẩu không đúng.` |
| `AUTH_ACCOUNT_LOCKED` | 423 | Login lock 15 phút | `Tài khoản đang tạm khóa. Vui lòng thử lại sau.` |
| `AUTH_ACCOUNT_SUSPENDED` | 403 | Account suspended/deleted | `Tài khoản hiện không thể sử dụng.` |
| `AUTH_EMAIL_NOT_VERIFIED` | 403 | Chưa verify email | `Vui lòng xác thực email trước.` |
| `AUTH_PHONE_NOT_VERIFIED` ⏳ | 403 | Phone chưa verify | `Vui lòng xác thực số điện thoại trước.` |
| `AUTH_USER_NOT_FOUND` | 404 | User không tồn tại hoặc đã xoá | `Không tìm thấy tài khoản.` |
| `AUTH_TOKEN_INVALID` | 401 | Token sai | `Phiên đăng nhập không hợp lệ.` |
| `AUTH_TOKEN_EXPIRED` | 401 | Token expired | `Phiên đăng nhập đã hết hạn.` |
| `AUTH_REFRESH_REUSED` | 401 | Refresh token reuse | `Phiên đăng nhập đã bị thu hồi. Vui lòng đăng nhập lại.` |
| `AUTH_VERIFICATION_INVALID` | 400 | Verify token/OTP sai/hết hạn | `Mã xác thực không hợp lệ hoặc đã hết hạn.` |
| `AUTH_VERIFICATION_ALREADY_COMPLETE` | 409 | Đã verify rồi | `Thông tin này đã được xác thực.` |
| `AUTH_OTP_ATTEMPTS_EXCEEDED` | 429 | Quá 5 OTP attempts | `Bạn đã thử quá số lần cho phép.` |
| `AUTH_OTP_RATE_LIMITED` | 429 | Gửi OTP quá nhanh | `Vui lòng thử lại sau.` |
| `AUTH_RESEND_LIMIT_EXCEEDED` | 429 | Resend quá 3 lần/giờ | `Bạn đã gửi lại quá số lần cho phép.` |
| `AUTH_RESET_INVALID` | 400 | Reset token sai/hết hạn | `Liên kết đặt lại mật khẩu không hợp lệ.` |
| `AUTH_MFA_REQUIRED` | 401 | Cần 2FA | `Vui lòng xác thực 2FA.` |
| `AUTH_MFA_INVALID` | 401 | TOTP/recovery code sai | `Mã 2FA không đúng.` |
| `AUTH_MFA_CHALLENGE_EXPIRED` | 400 | MFA challenge hết hạn | `Phiên xác thực 2FA đã hết hạn.` |
| `AUTH_MFA_ATTEMPTS_EXCEEDED` | 429 | MFA thử quá giới hạn | `Bạn đã thử quá số lần cho phép.` |
| `AUTH_MFA_ALREADY_ENABLED` | 409 | User đã bật 2FA | `2FA đã được bật.` |
| `PROFILE_INVALID` | 400 | Profile/address field sai | `Thông tin hồ sơ chưa đúng.` |
| `ADDRESS_NOT_FOUND` | 404 | Address không tồn tại/không thuộc user | `Không tìm thấy địa chỉ.` |
| `ADDRESS_LIMIT_REACHED` | 409 | Quá 20 address sống | `Bạn đã đạt giới hạn số địa chỉ.` |
| `ADDRESS_DEFAULT_REQUIRED` | 409 | Không thể bỏ default cuối | `Cần có một địa chỉ mặc định.` |
| `SHOP_ALREADY_EXISTS` | 409 | User đã có shop | `Tài khoản đã có hồ sơ người bán.` |
| `SHOP_NOT_FOUND` | 404 | Không tìm thấy shop | `Không tìm thấy gian hàng.` |
| `SHOP_INVALID_STATE` | 409 | Action sai shop state | `Trạng thái gian hàng không cho phép thao tác.` |
| `SHOP_SLUG_EXISTS` | 409 | Slug trùng | `Đường dẫn gian hàng đã tồn tại.` |
| `FAVORITE_LIMIT_REACHED` | 409 | Đạt `MAX_FAVORITES_PER_USER` | `Bạn đã đạt giới hạn số sản phẩm yêu thích.` |
| `SHOP_FOLLOW_LIMIT_REACHED` | 409 | Đạt `MAX_FOLLOWED_SHOPS_PER_USER` | `Bạn đã đạt giới hạn số shop theo dõi.` |
| `RBAC_PERMISSION_DENIED` | 403 | Thiếu role/permission/scope | `Bạn không có quyền thực hiện thao tác này.` |
| `RBAC_MFA_REQUIRED` | 428 | Thiếu step-up 2FA | `Vui lòng xác thực lại trước khi tiếp tục.` |
| `RBAC_INVALID_ROLE` | 400 | Role/scope không hợp lệ | `Vai trò hoặc phạm vi không hợp lệ.` |
| `RBAC_ASSIGNMENT_EXISTS` | 409 | Assignment đã tồn tại | `Quyền này đã được cấp.` |
| `RBAC_ASSIGNMENT_NOT_FOUND` | 409 | Assignment không tồn tại | `Không tìm thấy quyền cần thu hồi.` |
| `KYC_DOCUMENT_INVALID` ⏳ | 400 | File metadata/checksum/type sai | `Tài liệu không hợp lệ.` |
| `KYC_DOCUMENT_TOO_LARGE` | 413 | File > 10 MiB | `Tài liệu vượt quá dung lượng cho phép.` |
| `KYC_DOCUMENT_LIMIT_REACHED` | 409 | Quá 10 file/case | `Hồ sơ đã đạt giới hạn số tài liệu.` |
| `KYC_DOCUMENT_NOT_FOUND` | 404 | Document không tồn tại | `Không tìm thấy tài liệu.` |
| `KYC_DOCUMENT_ALREADY_COMPLETED` | 409 | Complete lặp | `Tài liệu đã được hoàn tất.` |
| `KYC_ALREADY_PENDING` | 409 | Case đang chờ review | `Hồ sơ đang được xét duyệt.` |
| `KYC_DECISION_INVALID` | 400/409 | Decision/reason/state sai | `Quyết định KYC chưa hợp lệ.` |
| `KYC_CASE_NOT_FOUND` | 404 | KYC case không tồn tại | `Không tìm thấy hồ sơ KYC.` |
| `KYC_REQUIRED` ⏳ | 403 | Chưa đạt điều kiện KYC | `Vui lòng hoàn tất xác minh gian hàng.` |
| `BANK_ACCOUNT_INVALID` | 409 | Bank account không hợp lệ | `Thông tin tài khoản ngân hàng chưa hợp lệ.` |
| `RATE_LIMITED` ⏳ | 429 | Vượt request limit | `Bạn thao tác quá nhanh. Vui lòng thử lại sau.` |
| `INTERNAL_ERROR` ⏳ | 500 | Lỗi chưa phân loại | `Hệ thống đang bận. Vui lòng thử lại.` |
| `AUTH_CURRENT_PASSWORD_INVALID` | 401 | Mật khẩu hiện tại sai khi đổi mật khẩu | `Mật khẩu hiện tại không đúng.` |
| `AUTH_PASSWORD_REUSE` | 400 | Mật khẩu mới trùng mật khẩu cũ | `Mật khẩu mới phải khác mật khẩu hiện tại.` |
| `AUTH_GOOGLE_TOKEN_INVALID` | 401 | Google ID token/callback không hợp lệ | `Đăng nhập Google không hợp lệ.` |
| `AUTH_GOOGLE_LINK_REQUIRED` | 409 | Email đã có tài khoản, cần liên kết | `Email đã có tài khoản. Vui lòng đăng nhập để liên kết Google.` |
| `AUTH_GOOGLE_IDENTITY_LINKED` | 409 | Google identity đã gắn tài khoản khác | `Tài khoản Google đã được liên kết.` |
| `AUTH_GOOGLE_UNAVAILABLE` | 503 | Không verify được với Google | `Đăng nhập Google tạm thời không khả dụng.` |
| `ADDRESS_LOCATION_INVALID` | 422 | Mã tỉnh/phường sai hoặc phường không thuộc tỉnh | `Địa chỉ chưa đúng.` |
| `LOCATION_PROVINCE_NOT_FOUND` | 404 | Mã tỉnh không tồn tại | `Không tìm thấy tỉnh/thành.` |
| `VALIDATION_ERROR` | 422 | Bean Validation thất bại ở một số endpoint | `Thông tin gửi lên chưa đúng.` |
| `SELLER_PROFILE_INCOMPLETE` | 400 | Thiếu hồ sơ seller khi submit | `Hồ sơ cửa hàng chưa đủ.` |
| `KYC_DOCUMENT_*` | 400/409 | Lỗi document KYC chi tiết: `REQUEST_INVALID`, `TYPE_UNSUPPORTED`, `CONTENT_TYPE_UNSUPPORTED`, `FILENAME_INVALID`, `SIZE_INVALID`, `CHECKSUM_INVALID`, `UPLOAD_INCOMPLETE`, `VERIFICATION_FAILED`, `STATE_INVALID`, `REQUIRED` (400); `DUPLICATE`, `DELETE_NOT_ALLOWED` (409) | `Tài liệu KYC chưa hợp lệ.` |

> Mã gắn ⏳ có trong thiết kế nhưng code chưa trả. Lỗi không bắt được hiện rơi về response mặc định của Spring (không theo envelope), chưa có `INTERNAL_ERROR`.

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | V1 trả access/refresh token trong JSON body; chưa dùng HttpOnly cookie. | Ảnh hưởng CORS credentials, CSRF và Micro-Frontends (MFE) token storage. | MFE + Security |
| 2 | Email change chưa mở ở `PUT /users/me`; cần flow endpoint riêng sau v1. | Nếu Penpot yêu cầu đổi email, phải thêm new-email verification và session revoke. | Product owner |
| 3 | Exact tax code validation/provider bank verification chưa có trong HLD; API dùng baseline 10–14 digits và bank catalog mock. | Có thể cần đổi regex/async verification. | Product/Finance |
| 4 | `download_url` KYC là signed URL TTL 10 phút và chỉ admin `KYC_READ` được cấp. | Ảnh hưởng storage adapter và audit access. | Security/DevOps |
| 5 | MFA recovery codes chỉ trả một lần khi enrollment; recovery/reset UI chưa có trong Penpot. | Cần support recovery flow nếu admin mất thiết bị. | Security/Product |
| 6 | `GET /admin/audit-logs`, `/admin/users/{userId}/status` và một số role read endpoint được bổ sung từ admin screens/HLD dù chưa có endpoint cụ thể trong HLD. | Nếu v1 không có admin API này, loại khỏi implementation scope/task. | Product owner |
| 7 | Pagination dùng page/size; nếu dataset admin lớn, cần chuyển cursor pagination ở API version sau. | Ảnh hưởng query/index và UI table state. | Backend lead |
| 8 | API error envelope thống nhất `{error:{code,message,details,trace_id}}`; Gateway có thể bọc thêm request metadata. | Nếu các module MFE đã có envelope khác, cần adapter ở Gateway. | Backend leads |
| 9 | Favorites/Follow (`/users/me/favorites/**`, `/shops/{shopId}/follow`, `/users/me/following`) và public shop profile (`GET /shops/{shopId}`) được bổ sung từ Frontend Design, đặt tại `auth-user`. Favorites chỉ lưu `product_id`; frontend/BFF hydrate qua Product Catalog. | Nếu tách service `engagement` sau này, đổi base path và Gateway route. | Product owner |
| 10 | `GET /shops/{shopId}` chỉ trả identity + `is_verified` + `follower_count`; `rating_avg`/`product_count` do Product Catalog phục vụ. | Frontend phải gọi 2 nguồn cho Shop hero; hoặc dựng BFF/endpoint tổng hợp. | Product owner + Frontend |
