# API — Auth User Service

> Nguồn: `docs/lld/auth-user.md` · `docs/db/auth-user.md` · `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Base path: `/api/v1` · External access: qua `api-gateway` · Internal service owner: `auth-user`

## 1. Quy ước chung

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

### 1.3 Danh sách endpoint

| # | Method | Path | Mục đích | Quyền | Màn hình/consumer |
|---:|---|---|---|---|---|
| 1 | `POST` | `/auth/signup` | Tạo buyer account | Public | Sign in overlay → đăng ký |
| 2 | `POST` | `/auth/signin` | Đăng nhập email/phone | Public | Sign in overlay |
| 3 | `POST` | `/auth/refresh` | Đổi refresh token | Public với refresh token | Client auth interceptor |
| 4 | `POST` | `/auth/signout` | Revoke session/token family | Authenticated | Account/sign out |
| 5 | `POST` | `/auth/email/verify` | Verify email | Public với token | Email link |
| 6 | `POST` | `/auth/email/resend` | Gửi lại email verification | Authenticated hoặc recovery token | Sign in/onboarding |
| 7 | `POST` | `/auth/phone/request-otp` | Gửi phone OTP | Authenticated | Profile/onboarding |
| 8 | `POST` | `/auth/phone/verify-otp` | Verify phone OTP | Authenticated | Profile/onboarding |
| 9 | `POST` | `/auth/password/forgot` | Khởi tạo reset password | Public | Forgot password |
| 10 | `POST` | `/auth/password/reset` | Đặt password mới | Public với reset token | Reset password |
| 11 | `POST` | `/auth/2fa/setup` | Tạo TOTP enrollment | Admin authenticated | Admin security settings |
| 12 | `POST` | `/auth/2fa/verify` | Verify login/2FA enrollment | Challenge/authenticated | Admin login/step-up |
| 13 | `GET` | `/users/me` | Đọc profile | Authenticated | Buyer Account/Profile |
| 14 | `PUT` | `/users/me` | Cập nhật profile | Authenticated | Buyer Account/Profile |
| 15 | `GET` | `/users/me/addresses` | Danh sách địa chỉ | Authenticated | Address book/checkout |
| 16 | `POST` | `/users/me/addresses` | Thêm địa chỉ | Authenticated | Address book |
| 17 | `PUT` | `/users/me/addresses/{addressId}` | Sửa địa chỉ | Authenticated | Address book |
| 18 | `DELETE` | `/users/me/addresses/{addressId}` | Soft delete địa chỉ | Authenticated | Address book |
| 19 | `POST` | `/users/register-seller` | Tạo shop/onboarding | Authenticated + email verified | Seller onboarding |
| 20 | `GET` | `/seller/onboarding` | Đọc tiến độ onboarding | Seller | Seller onboarding |
| 21 | `PUT` | `/seller/onboarding/profile` | Lưu hồ sơ shop | Seller | Step 1 — Hồ sơ |
| 22 | `PUT` | `/seller/onboarding/warehouse` | Lưu kho/vận chuyển | Seller | Step 3 — Kho & vận chuyển |
| 23 | `POST` | `/seller/onboarding/kyc/documents/presign` | Lấy signed upload URL | Seller | Step 2 — KYC |
| 24 | `POST` | `/seller/onboarding/kyc/documents/complete` | Xác nhận upload document | Seller | Step 2 — KYC |
| 25 | `POST` | `/seller/onboarding/kyc/submit` | Submit KYC case | Seller | Step 2 — KYC |
| 26 | `PUT` | `/seller/onboarding/bank` | Lưu thông tin ngân hàng | Seller | Step 4 — Ngân hàng |
| 27 | `GET` | `/seller/shop` | Đọc shop profile | Seller/staff | Seller Settings |
| 28 | `PUT` | `/seller/shop` | Cập nhật shop profile | Seller | Seller Settings |
| 29 | `GET` | `/admin/shops/kyc` | KYC queue | `KYC_READ` | Admin Shops/KYC |
| 30 | `GET` | `/admin/shops/{shopId}/kyc` | KYC detail/document metadata | `KYC_READ` | KYC review overlay |
| 31 | `POST` | `/admin/shops/{shopId}/kyc/review` | Approve/request info/reject | `KYC_DECIDE` | KYC review overlay |
| 32 | `GET` | `/admin/users/{userId}/roles` | Đọc role/permission | `ROLE_READ` | Admin Users/Roles |
| 33 | `PATCH` | `/admin/users/{userId}/roles` | Grant/revoke role | `ROLE_ASSIGN` + 2FA | Admin role editor |
| 34 | `PATCH` | `/admin/users/{userId}/status` | Suspend/restore user | `USER_SUSPEND` + 2FA | Admin Users |
| 35 | `GET` | `/admin/audit-logs` | Tra cứu audit | Role được cấp | Admin audit/support |
| 36 | `GET` | `/.well-known/jwks.json` | Public JWKS key set cho Gateway và internal services | Public / Internal REST | API Gateway, internal clients |
| 37 | `GET` | `/users/me/favorites` | Danh sách sản phẩm yêu thích (reference) | Authenticated | Favorite Products |
| 38 | `POST` | `/users/me/favorites` | Thêm sản phẩm vào yêu thích | Authenticated | Product card/PDP |
| 39 | `DELETE` | `/users/me/favorites/{productId}` | Bỏ yêu thích | Authenticated | Favorite Products/PDP |
| 40 | `GET` | `/users/me/favorites/contains` | Batch kiểm tra trạng thái tim | Authenticated | Product list/PDP |
| 41 | `POST` | `/shops/{shopId}/follow` | Theo dõi shop | Authenticated | Shop hero/PDP shop card |
| 42 | `DELETE` | `/shops/{shopId}/follow` | Bỏ theo dõi shop | Authenticated | Shop hero |
| 43 | `GET` | `/users/me/following` | Danh sách shop đang theo dõi | Authenticated | Account |
| 44 | `GET` | `/shops/{shopId}/followers/count` | Số người theo dõi shop | Public | Shop hero |
| 45 | `GET` | `/shops/{shopId}` | Hồ sơ shop công khai (HLD #10) | Public | `Taca Buyer / Shop` |

## 2. Chi tiết endpoint

### 2.1 `POST /auth/signup` — tạo buyer account

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

### 2.2 `POST /auth/signin` — đăng nhập

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
| 403 | `AUTH_ACCOUNT_SUSPENDED` | Account suspended/deleted. |
| 423 | `AUTH_ACCOUNT_LOCKED` | 5 lần fail trong 15 phút, lock còn hiệu lực. |
| 429 | `RATE_LIMITED` | Vượt rate limit. |

### 2.3 `POST /auth/refresh` — refresh token rotation

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

### 2.4 `POST /auth/signout` — revoke session

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

### 2.5 `POST /auth/email/verify` — verify email

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

### 2.6 `POST /auth/email/resend` — gửi lại email verification

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

### 2.7 `POST /auth/phone/request-otp` — yêu cầu phone OTP

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

### 2.8 `POST /auth/phone/verify-otp` — verify phone

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

### 2.9 `POST /auth/password/forgot` — khởi tạo reset

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

### 2.10 `POST /auth/password/reset` — đặt password mới

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

### 2.11 `POST /auth/2fa/setup` — tạo TOTP enrollment

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

### 2.12 `POST /auth/2fa/verify` — login hoặc enrollment

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

### 2.13 `GET /users/me` — đọc profile

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

### 2.14 `PUT /users/me` — cập nhật profile

Quyền: Authenticated · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `full_name` | string | Có | 1–120 Unicode characters. |
| `phone` | string/null | Không | E.164; đổi phone clear verification. |
| `date_of_birth` | date/null | Không | Không ở tương lai. |

`email`, `role`, `status`, `user_id` không được gửi hoặc sẽ bị từ chối.

Response trả profile mới và `phone_verification_required=true` nếu phone vừa đổi.

Lỗi: `400 PROFILE_INVALID`, `409 AUTH_PHONE_EXISTS`, `401 AUTH_TOKEN_INVALID`.

### 2.15 `GET /users/me/addresses` — list address

Quyền: Authenticated · Query: `page`, `size`, `sort=created_at,desc` · Response: `200`

Response item:

```json
{
  "id": "01912f4b-7a1b-7c12-9c55-8b1c34a6d921",
  "recipient": "Nguyễn Minh Anh",
  "phone": "+84901234567",
  "line1": "12 Nguyễn Huệ",
  "line2": "Tòa A, căn 501",
  "ward": "Bến Nghé",
  "district": "Quận 1",
  "province": "Hồ Chí Minh",
  "postal_code": "700000",
  "is_default": true,
  "created_at": "2026-08-30T09:10:00Z",
  "updated_at": "2026-08-30T09:10:00Z"
}
```

Lỗi: `400 AUTH_INVALID_INPUT` nếu query sai, `401 AUTH_TOKEN_INVALID`.

### 2.16 `POST /users/me/addresses` — thêm address

Quyền: Authenticated · Response: `201`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc | Ví dụ |
|---|---|---:|---|---|
| `recipient` | string | Có | 1–120 | `"Nguyễn Minh Anh"` |
| `phone` | string | Có | E.164 | `"+84901234567"` |
| `line1` | string | Có | 1–255 | `"12 Nguyễn Huệ"` |
| `line2` | string | Không | Tối đa 255 | `"Tòa A"` |
| `ward` | string | Có | 1–120 | `"Bến Nghé"` |
| `district` | string | Có | 1–120 | `"Quận 1"` |
| `province` | string | Có | 1–120 | `"Hồ Chí Minh"` |
| `postal_code` | string | Không | Tối đa 12 | `"700000"` |
| `is_default` | boolean | Không | Default `false`; address đầu tiên tự default | `true` |

Lỗi: `400 PROFILE_INVALID`, `409 ADDRESS_LIMIT_REACHED`, `409 ADDRESS_DEFAULT_REQUIRED`.

### 2.17 `PUT /users/me/addresses/{addressId}` — sửa address

Quyền: Authenticated · Response: `200`

Request: cùng field với POST; `addressId` UUID string.

Ràng buộc:

- Chỉ update address có `user_id = JWT.sub` và `deleted_at IS NULL`.
- Đổi `is_default=true` clear default cũ trong cùng transaction.
- Không tin `user_id` trong body.

Lỗi: `400 PROFILE_INVALID`, `404 ADDRESS_NOT_FOUND`, `409 ADDRESS_DEFAULT_REQUIRED`.

### 2.18 `DELETE /users/me/addresses/{addressId}` — soft delete

Quyền: Authenticated · Response: `204`

Ràng buộc:

- Không xóa cứng; chỉ set `deleted_at`.
- Nếu xóa default, tự động chọn address sống gần nhất theo `updated_at DESC` làm default mới.
- Order Service lưu `address_snapshot` riêng độc lập; soft-delete address tại Auth User không ảnh hưởng đơn hàng đã tạo hoặc phiên checkout đã clone snapshot.
- Address không thuộc user trả `404 ADDRESS_NOT_FOUND`, không trả 403.

Lỗi: `404 ADDRESS_NOT_FOUND`, `409 ADDRESS_DEFAULT_REQUIRED`.

### 2.19 `POST /users/register-seller` — tạo shop/onboarding

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
      "blockers": ["EMAIL_VERIFICATION_REQUIRED"]
    }
  },
  "meta": {"request_id": "01912f4c-7a1b-7c12-9c55-8b1c34a6d921"}
}
```

Lỗi: `403 AUTH_EMAIL_NOT_VERIFIED`, `409 SHOP_ALREADY_EXISTS`, `409 AUTH_TAX_CODE_EXISTS`, `400 PROFILE_INVALID`.

### 2.20 `GET /seller/onboarding` — tiến độ onboarding

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

### 2.21 `PUT /seller/onboarding/profile` — step Hồ sơ

Quyền: Seller owner · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `name` | string | Có | 1–120. |
| `business_name` | string | Có | 1–200. |
| `tax_code` | string | Có | 10–14 digits, unique. |
| `description` | string | Không | Tối đa 2.000. |
| `logo_object_key` | string | Không | Chỉ object key đã presign/allowlist. |

Lỗi: `400 PROFILE_INVALID`, `403 RBAC_PERMISSION_DENIED`, `409 AUTH_TAX_CODE_EXISTS`, `409 SHOP_INVALID_STATE`.

### 2.22 `PUT /seller/onboarding/warehouse` — step Kho & vận chuyển

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

Lỗi: `400 PROFILE_INVALID`, `403 RBAC_PERMISSION_DENIED`, `409 SHOP_INVALID_STATE`.

### 2.23 `POST /seller/onboarding/kyc/documents/presign` — tạo signed URL

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

### 2.24 `POST /seller/onboarding/kyc/documents/complete` — complete upload

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

### 2.25 `POST /seller/onboarding/kyc/submit` — submit KYC

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

### 2.26 `PUT /seller/onboarding/bank` — step Ngân hàng

Quyền: Seller owner · Response: `200`

Request:

| Field | Kiểu | Bắt buộc | Ràng buộc |
|---|---|---:|---|
| `bank_code` | string | Có | 1–32, nằm trong bank catalog. |
| `bank_name` | string | Có | 1–120. |
| `account_name` | string | Có | 1–120, normalize uppercase theo provider nếu cần. |
| `account_number` | string | Có | 8–20 digits; encrypt trước khi lưu. |
| `confirm_account_name` | boolean | Có | Phải `true`. |

Response chỉ trả `bank_name`, masked account `********1234`, `verified=false/true`; không trả account number raw.

Lỗi: `400 PROFILE_INVALID`, `409 BANK_ACCOUNT_INVALID`, `403 RBAC_PERMISSION_DENIED`.

### 2.27 `GET /seller/shop` — đọc shop profile

Quyền: Seller owner/staff · Response: `200`

Response gồm `id,name,slug,business_name,tax_code_masked,description,logo_url,status,kyc_status,warehouse_summary,bank_summary,created_at,updated_at`.

Lỗi: `404 SHOP_NOT_FOUND`, `403 RBAC_PERMISSION_DENIED`.

### 2.28 `PUT /seller/shop` — cập nhật shop profile

Quyền: Seller owner · Response: `200`

Request cho phép `name`, `description`, `logo_object_key`; `tax_code`, owner và status không đổi tại endpoint này. `slug` đổi phải qua validation unique riêng trong service.

Lỗi: `400 PROFILE_INVALID`, `403 RBAC_PERMISSION_DENIED`, `409 SHOP_SLUG_EXISTS`, `409 SHOP_INVALID_STATE`.

### 2.29 `GET /admin/shops/kyc` — KYC queue

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

### 2.30 `GET /admin/shops/{shopId}/kyc` — KYC detail

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

### 2.31 `POST /admin/shops/{shopId}/kyc/review` — quyết định KYC

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

### 2.32 `GET /admin/users/{userId}/roles` — đọc role/permission

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

### 2.33 `PATCH /admin/users/{userId}/roles` — grant/revoke role

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

### 2.34 `PATCH /admin/users/{userId}/status` — suspend/restore

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

Khi suspend: revoke toàn bộ refresh token family, ghi audit, phát `user.status_changed` (kèm đẩy `revoked_user_id` TTL 15m vào Redis chung của Gateway để chặn Access JWT ngay lập tức). Access token JWT chưa hết hạn cũng sẽ bị Gateway từ chối sau khi đồng bộ cache.

Lỗi: `404 AUTH_USER_NOT_FOUND`, `403 RBAC_PERMISSION_DENIED`, `428 RBAC_MFA_REQUIRED`, `409 AUTH_ACCOUNT_SUSPENDED`.

### 2.35 `GET /admin/audit-logs` — tra cứu audit

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

Không trả `metadata` có secret/PII raw; field nhạy cảm phải masked.

Lỗi: `400 AUTH_INVALID_INPUT`, `403 RBAC_PERMISSION_DENIED`.

### 2.36 `GET /.well-known/jwks.json` — Public JWKS key set

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

### 2.37 `GET /users/me/favorites` — danh sách yêu thích

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

### 2.38 `POST /users/me/favorites` — thêm yêu thích

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

### 2.39 `DELETE /users/me/favorites/{productId}` — bỏ yêu thích

Quyền: Authenticated · Response: `204` (idempotent — không tồn tại vẫn `204`)

### 2.40 `GET /users/me/favorites/contains` — batch trạng thái tim

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

### 2.41 `POST /shops/{shopId}/follow` — theo dõi shop

Quyền: Authenticated · Response: `201` (hoặc `200` idempotent nếu đã follow)

```json
{ "data": { "shop_id": "01912f49-7a1b-7c12-9c55-8b1c34a6d921", "followed_at": "2026-08-30T09:12:00Z" }, "meta": { "request_id": "..." } }
```

Ràng buộc: shop phải tồn tại và `status != DELETED` (kiểm bằng bảng `shops` local).

Lỗi: `404 SHOP_NOT_FOUND`, `409 SHOP_FOLLOW_LIMIT_REACHED` (đạt `MAX_FOLLOWED_SHOPS_PER_USER=1000`).

### 2.42 `DELETE /shops/{shopId}/follow` — bỏ theo dõi

Quyền: Authenticated · Response: `204` (idempotent)

### 2.43 `GET /users/me/following` — shop đang theo dõi

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

### 2.44 `GET /shops/{shopId}/followers/count` — số người theo dõi

Quyền: Public · Response: `200`

```json
{ "data": { "shop_id": "01912f49-...", "follower_count": 1284 }, "meta": { "request_id": "..." } }
```

`follower_count` eventual; không đảm bảo realtime tuyệt đối.

### 2.45 `GET /shops/{shopId}` — hồ sơ shop công khai

Quyền: Public · Response: `200`

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

## 3. Bảng mã lỗi dùng chung

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
| `AUTH_PHONE_NOT_VERIFIED` | 403 | Phone chưa verify | `Vui lòng xác thực số điện thoại trước.` |
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
| `KYC_DOCUMENT_INVALID` | 400 | File metadata/checksum/type sai | `Tài liệu không hợp lệ.` |
| `KYC_DOCUMENT_TOO_LARGE` | 413 | File > 10 MiB | `Tài liệu vượt quá dung lượng cho phép.` |
| `KYC_DOCUMENT_LIMIT_REACHED` | 409 | Quá 10 file/case | `Hồ sơ đã đạt giới hạn số tài liệu.` |
| `KYC_DOCUMENT_NOT_FOUND` | 404 | Document không tồn tại | `Không tìm thấy tài liệu.` |
| `KYC_DOCUMENT_ALREADY_COMPLETED` | 409 | Complete lặp | `Tài liệu đã được hoàn tất.` |
| `KYC_ALREADY_PENDING` | 409 | Case đang chờ review | `Hồ sơ đang được xét duyệt.` |
| `KYC_DECISION_INVALID` | 400/409 | Decision/reason/state sai | `Quyết định KYC chưa hợp lệ.` |
| `BANK_ACCOUNT_INVALID` | 409 | Bank account không hợp lệ | `Thông tin tài khoản ngân hàng chưa hợp lệ.` |
| `RATE_LIMITED` | 429 | Vượt request limit | `Bạn thao tác quá nhanh. Vui lòng thử lại sau.` |
| `INTERNAL_ERROR` | 500 | Lỗi chưa phân loại | `Hệ thống đang bận. Vui lòng thử lại.` |

## 4. Mock contract tích hợp

### 4.1 Notification command

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
    "reset_token": "opaque-token-not-logged",
    "expires_at": "2026-08-30T09:30:00Z"
  }
}
```

### 4.2 Object storage presign

Mock request/response được mô tả ở endpoint KYC presign. Adapter phải kiểm tra `owner_id`, `purpose`, `content_type`, `size_bytes`, `sha256`; không cho client dùng object key của case khác.

### 4.3 Event downstream

Các event `user.created`, `user.updated`, `user.email_verified`, `user.status_changed`, `user.role_changed`, `shop.created`, `shop.kyc.*` dùng envelope có `event_id`, `schema_version`, `occurred_at`, `aggregate_type`, `aggregate_id`, `payload`. Consumer dedupe theo `event_id`.

## 5. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | V1 trả access/refresh token trong JSON body; chưa dùng HttpOnly cookie. | Ảnh hưởng CORS credentials, CSRF và Micro-Frontends (MFE) token storage. | MFE + Security |
| 2 | Email change chưa mở ở `PUT /users/me`; cần flow endpoint riêng sau v1. | Nếu Penpot yêu cầu đổi email, phải thêm new-email verification và session revoke. | Product owner |
| 3 | Exact tax code validation/provider bank verification chưa có trong HLD; API dùng baseline 10–14 digits và bank catalog mock. | Có thể cần đổi regex/async verification. | Product/Finance |
| 4 | `download_url` KYC là signed URL TTL 10 phút và chỉ admin `KYC_READ` được cấp. | Ảnh hưởng storage adapter và audit access. | Security/DevOps |
| 5 | MFA recovery codes chỉ trả một lần khi enrollment; recovery/reset UI chưa có trong Penpot. | Cần support recovery flow nếu admin mất thiết bị. | Security/Product |
| 6 | `GET /admin/audit-logs`, `/admin/users/{id}/status` và một số role read endpoint được bổ sung từ admin screens/HLD dù chưa có endpoint cụ thể trong HLD. | Nếu v1 không có admin API này, loại khỏi implementation scope/task. | Product owner |
| 7 | Pagination dùng page/size; nếu dataset admin lớn, cần chuyển cursor pagination ở API version sau. | Ảnh hưởng query/index và UI table state. | Backend lead |
| 8 | API error envelope thống nhất `{error:{code,message,details,trace_id}}`; Gateway có thể bọc thêm request metadata. | Nếu các module MFE đã có envelope khác, cần adapter ở Gateway. | Backend leads |
| 9 | Favorites/Follow (`/users/me/favorites/**`, `/shops/{id}/follow`, `/users/me/following`) và public shop profile (`GET /shops/{id}`) được bổ sung từ Frontend Design, đặt tại `auth-user`. Favorites chỉ lưu `product_id`; frontend/BFF hydrate qua Product Catalog. | Nếu tách service `engagement` sau này, đổi base path và Gateway route. | Product owner |
| 10 | `GET /shops/{id}` chỉ trả identity + `is_verified` + `follower_count`; `rating_avg`/`product_count` do Product Catalog phục vụ. | Frontend phải gọi 2 nguồn cho Shop hero; hoặc dựng BFF/endpoint tổng hợp. | Product owner + Frontend |
