# Test plan — Auth User Service

> Nguồn: `docs/api/auth-user.md` · `docs/lld/auth-user.md` · `docs/db/auth-user.md` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`

## 1. Chuẩn bị

### 1.1 Môi trường và dependency

| Mục | Nội dung |
|---|---|
| Môi trường | Local CI và staging; mọi test integration dùng MySQL 8.4/InnoDB fixture riêng. |
| Service dependency | API Gateway test hoặc direct service test; Kafka test broker; mock Notification; mock S3/MinIO; fake clock. |
| Database | Chạy toàn bộ migration từ `001` đến `013` (gồm `favorites`, `shop_follows`); seed RBAC idempotent; test teardown không dùng production database. |
| Timezone | UTC; test dùng fake clock để kiểm tra TTL 5 phút/15 phút/30 phút/24 giờ/30 ngày. |
| HTTP | JSON UTF-8; UUID canonical string; request ID cố định để assert trace. |
| Security | Tạo RSA key pair test; private key chỉ ở test secret store; JWKS fixture có rotation `kid`. |
| Test isolation | Mỗi test suite có user/shop/token riêng hoặc rollback transaction; không phụ thuộc thứ tự test. |

### 1.2 Tài khoản và dữ liệu test

| Fixture | Trạng thái/quyền | Mục đích |
|---|---|---|
| `buyer_unverified` | BUYER, email chưa verify, phone null | Signup/onboarding gate. |
| `buyer_verified` | BUYER, email/phone verified | Profile/address/OTP. |
| `seller_draft` | SELLER, shop DRAFT, KYC DRAFT | Onboarding draft. |
| `seller_pending` | SELLER, KYC PENDING | KYC pending và product publish gate. |
| `seller_approved` | SELLER, KYC APPROVED | Seller Center/action sau KYC. |
| `seller_staff` | SELLER_STAFF, scope một shop | Scope/role test. |
| `risk_admin` | RISK_MANAGER, 2FA enabled | KYC queue/review. |
| `catalog_admin` | CATALOG_ADMIN, 2FA enabled | Read catalog-related role boundary. |
| `super_admin` | SUPER_ADMIN, 2FA enabled | Role/status/MFA destructive action. |
| `support_viewer` | SUPPORT_VIEWER | Read-only/masking test. |
| `user_other` | User khác, có address/shop riêng | IDOR/ownership test. |
| `shop_with_20_addresses` | Buyer có 20 address sống | Address limit. |
| `kyc_case_with_documents` | KYC pending, PDF/JPG/PNG valid | Document/review. |

### 1.3 Dữ liệu boundary

| Dữ liệu | Giá trị biên |
|---|---|
| `full_name` | 1, 120, 121 Unicode characters. |
| `email` | 3, 254, 255 ký tự; uppercase/space; invalid format. |
| `password` | 11, 12, 72, 73 ký tự; Argon2id output. |
| `phone` | E.164 hợp lệ; thiếu `+`; quá dài; duplicate. |
| `address` | Field rỗng, đúng max, vượt max; address đầu tiên/default/cuối cùng. |
| `KYC document` | 1 byte, 10 MiB, 10 MiB + 1 byte; PDF/JPG/PNG và MIME giả. |
| `KYC reason` | 9, 10, 1.000, 1.001 Unicode characters. |
| `OTP` | Sai 5 lần, lần thứ 6, hết hạn 5 phút, resend. |
| `MFA` | Sai 5 lần, challenge hết hạn 5 phút, replay code. |

## 2. Test thủ công (QA / nghiệm thu)

| Mã | Màn hình / Luồng | Tiền điều kiện | Các bước | Kết quả mong đợi | Mức |
|---|---|---|---|---|---|
| TC-01 | Sign in overlay — signup success | Chưa có email | Mở overlay → nhập payload hợp lệ → submit | Tạo account BUYER; hiển thị trạng thái cần verify email; không hiển thị password/token raw. | Cao |
| TC-02 | Sign in overlay — validation | Không cần login | Bỏ trống field, email sai, password 11/73 ký tự | Inline error tiếng Việt; không gọi API khi client biết lỗi; focus tới field sai. | Cao |
| TC-03 | Sign in overlay — duplicate account | Email/phone đã tồn tại | Đăng ký lại | Hiển thị email/phone đã sử dụng; không tạo record thứ hai. | Cao |
| TC-04 | Sign in — buyer | Buyer verified | Nhập email/password đúng | Vào buyer context; token interceptor hoạt động; refresh khi access token hết hạn. | Cao |
| TC-05 | Sign in — MFA admin | Admin có TOTP | Đăng nhập → nhập OTP đúng/sai | Chuyển MFA challenge; OTP đúng vào Admin; OTP sai không cấp token. | Cao |
| TC-06 | Sign in — lockout | User sạch counter | Nhập sai password 5 lần trong 15 phút | Account bị lock; thông báo thử lại sau; không lộ account khác. | Cao |
| TC-07 | Email verification | Có email link còn hạn | Mở link verify một lần rồi mở lại | Lần đầu verified; lần hai báo token đã dùng/hết hạn, không mutation thêm. | Cao |
| TC-08 | Phone OTP | Buyer verified email | Đổi phone → request OTP → nhập OTP | Phone về unverified sau đổi; OTP đúng set verified; OTP sai hiển thị attempt còn lại. | Cao |
| TC-09 | Forgot/reset password | Có/không có identifier | Gửi forgot với cả hai trường hợp | Cùng message accepted; account tồn tại nhận notification, account không tồn tại không bị enumerate. | Cao |
| TC-10 | Reset password | Có reset link | Nhập password mới → dùng link lần hai | Lần đầu 204; mọi refresh session bị revoke; link lần hai bị từ chối. | Cao |
| TC-11 | Profile | Buyer đã login | Sửa full name/date/phone; thử sửa role/email | Field hợp lệ lưu; phone đổi yêu cầu verify; email/role/status không thể sửa. | Cao |
| TC-12 | Address book — default | Buyer có 0/1 address | Thêm address thường → thêm default → đổi default | Address đầu tiên tự default; chỉ có một default; UI cập nhật ngay. | Cao |
| TC-13 | Address book — limit/delete | Buyer có 20 address | Thêm address thứ 21; xóa default/cuối | Trả limit; xóa default chọn address mới; xóa cuối bị chặn khi checkout active. | Cao |
| TC-14 | Seller onboarding — draft | Buyer email verified | Register seller → đi từng bước → thoát/mở lại | Tiến độ 5 bước và blockers được giữ; draft không yêu cầu KYC approved. | Cao |
| TC-15 | KYC upload | Seller draft | Upload PDF/JPG/PNG hợp lệ và file sai type/size | Signed URL hoạt động; complete verify checksum; file sai bị reject; không lộ object key khác. | Cao |
| TC-16 | KYC state | KYC pending | Admin mở queue → xem document → NEEDS_INFO/REJECT/APPROVE | Status/reason/audit hiển thị đúng; signed URL hết hạn sau 10 phút. | Cao |
| TC-17 | KYC publish gate | Shop pending/rejected/approved | Thử publish/đổi Seller state từng case | Chỉ APPROVED được mở publish/withdraw; không có product-level approval. | Cao |
| TC-18 | Seller settings/bank | Seller owner | Lưu bank/warehouse → reload | Bank masked; không hiển thị account raw; onboarding completion cập nhật. | TB |
| TC-19 | Admin roles | Super admin + staff | Grant/revoke role, sai scope, self-escalate | Permission/role đúng; self-escalate bị chặn; có MFA step-up và audit. | Cao |
| TC-20 | Admin user status | Super/risk admin | Suspend/restore user | Refresh sessions bị revoke; access token cũ chỉ sống tối đa TTL; trạng thái hiển thị đúng. | Cao |
| TC-21 | Loading/empty/error | Mỗi màn hình | Throttle network, API 500/404/429, không có dữ liệu | Có loading skeleton, empty state, error message/code và retry action đúng Penpot. | Cao |
| TC-22 | Offline/resume | Mobile/tablet browser | Tắt mạng khi submit profile/onboarding | Hiển thị offline; không tạo duplicate mutation khi online lại; draft local không mất. | Cao |
| TC-23 | Responsive/accessibility | Desktop/mobile/tablet | Kiểm tra 3 breakpoint, keyboard, screen reader, focus | Touch target ≥44px, focus visible, label/error accessible, không overflow table/form. | TB |
| TC-24 | Data masking | Admin/support account | Mở profile/KYC/audit/bank | Không thấy password hash, refresh hash, TOTP secret, OTP raw, bank raw/KYC bytes. | Cao |
| TC-25 | Favorite Products | Buyer đã login | Thêm/bỏ tim từ card/PDP; mở `Taca Buyer / Favorite Products`; đạt giới hạn 500 | Trạng thái tim đồng bộ giữa card/PDP/list; thêm trùng không lỗi; vượt 500 báo `FAVORITE_LIMIT_REACHED`; thẻ sản phẩm hydrate từ Product Catalog. | TB |
| TC-26 | Follow shop + shop profile | Buyer login / khách | Mở `Taca Buyer / Shop` (khách xem được); bấm `+ Theo dõi`/bỏ theo dõi; xem số người theo dõi | Khách xem hồ sơ shop công khai (không thấy tax/bank); follow/unfollow idempotent; `follower_count` tăng/giảm; `is_verified` đúng theo KYC APPROVED. | TB |

Mức: `Cao` (chặn phát hành) · `TB` · `Thấp`.

## 3. Test API (integration)

### 3.1 Authentication và token

| Mã | Endpoint | Input | HTTP | Response mong đợi | Ghi chú |
|---|---|---|---:|---|---|
| IT-AUTH-01 | `POST /auth/signup` | Full payload hợp lệ | 201 | User BUYER + access/refresh + email verification accepted | DB có users/token/outbox atomic. |
| IT-AUTH-02 | `POST /auth/signup` | Body rỗng | 400 | `AUTH_INVALID_INPUT` | Không ghi DB. |
| IT-AUTH-03 | `POST /auth/signup` | `full_name` 0/121 | 400 | `AUTH_INVALID_INPUT` | Assert field details. |
| IT-AUTH-04 | `POST /auth/signup` | Email sai/255 ký tự | 400 | `AUTH_INVALID_INPUT` | Không enumerate. |
| IT-AUTH-05 | `POST /auth/signup` | Password 11/73 | 400 | `AUTH_INVALID_INPUT` | Không hash/ghi password. |
| IT-AUTH-06 | `POST /auth/signup` | Email đã tồn tại | 409 | `AUTH_EMAIL_EXISTS` | Không user duplicate. |
| IT-AUTH-07 | `POST /auth/signup` | Phone đã tồn tại | 409 | `AUTH_PHONE_EXISTS` | Null phone vẫn hợp lệ. |
| IT-AUTH-08 | `POST /auth/signup` | 11 request cùng IP trong phút | 429 | `RATE_LIMITED` + Retry-After | Counter Redis/mock. |
| IT-AUTH-09 | `POST /auth/signin` | Email/password đúng | 200 | Token pair + user summary | Access TTL 900s. |
| IT-AUTH-10 | `POST /auth/signin` | Password sai | 401 | `AUTH_INVALID_CREDENTIALS` | Không lộ user tồn tại. |
| IT-AUTH-11 | `POST /auth/signin` | Phone chưa verified | 403 | `AUTH_PHONE_NOT_VERIFIED` | Không cấp token phone login. |
| IT-AUTH-12 | `POST /auth/signin` | Account locked | 423 | `AUTH_ACCOUNT_LOCKED` | Có retry time ISO-8601, không raw policy. |
| IT-AUTH-13 | `POST /auth/signin` | Account suspended/deleted | 403 | `AUTH_ACCOUNT_SUSPENDED` | Không cấp token. |
| IT-AUTH-14 | `POST /auth/signin` | Admin có 2FA | 401 | `AUTH_MFA_REQUIRED` + challenge_id | Chưa cấp access token. |
| IT-AUTH-15 | `POST /auth/refresh` | Refresh active | 200 | Pair mới; token cũ revoked | Family giữ nguyên. |
| IT-AUTH-16 | `POST /auth/refresh` | Refresh đã expired | 401 | `AUTH_TOKEN_EXPIRED` | Không tạo token. |
| IT-AUTH-17 | `POST /auth/refresh` | Refresh token đã revoked | 401 | `AUTH_REFRESH_REUSED` | Revoke toàn bộ family. |
| IT-AUTH-18 | `POST /auth/refresh` | Token random | 401 | `AUTH_TOKEN_INVALID` | Không timing/data leak rõ ràng. |
| IT-AUTH-19 | `POST /auth/signout` | Current token/family | 204 | Không body | Refresh bị revoke. |
| IT-AUTH-20 | `POST /auth/signout` | `all_sessions=true` thiếu step-up | 428 | `RBAC_MFA_REQUIRED` | Không revoke ngoài ý muốn. |
| IT-AUTH-21 | `POST /auth/email/verify` | Token hợp lệ | 200 | email_verified=true | Phát `user.email_verified`. |
| IT-AUTH-22 | `POST /auth/email/verify` | Sai/hết hạn/đã dùng | 400 | `AUTH_VERIFICATION_INVALID` | One-time. |
| IT-AUTH-23 | `POST /auth/email/resend` | User chưa verify | 202 | accepted + expiry | Token cũ revoked. |
| IT-AUTH-24 | `POST /auth/email/resend` | Lần 4 trong 1 giờ | 429 | `AUTH_RESEND_LIMIT_EXCEEDED` | Không gửi command mới. |
| IT-AUTH-25 | `POST /auth/phone/request-otp` | Phone E.164 mới | 202 | challenge_id/masked phone | Không trả OTP raw. |
| IT-AUTH-26 | `POST /auth/phone/request-otp` | Phone duplicate | 409 | `AUTH_PHONE_EXISTS` | Không tạo challenge dùng được. |
| IT-AUTH-27 | `POST /auth/phone/verify-otp` | OTP đúng | 200 | phone_verified=true | Challenge one-time. |
| IT-AUTH-28 | `POST /auth/phone/verify-otp` | OTP sai 5 lần | 429 | `AUTH_OTP_ATTEMPTS_EXCEEDED` | Lần 6 không verify. |
| IT-AUTH-29 | `POST /auth/password/forgot` | Identifier tồn tại | 202 | Generic accepted | Có notification outbox. |
| IT-AUTH-30 | `POST /auth/password/forgot` | Identifier không tồn tại | 202 | Y hệt IT-AUTH-29 | Không enumerate. |
| IT-AUTH-31 | `POST /auth/password/reset` | Token + password hợp lệ | 204 | Không body | Revoke refresh families. |
| IT-AUTH-32 | `POST /auth/password/reset` | Token reused/expired | 400 | `AUTH_RESET_INVALID` | Không đổi password. |
| IT-AUTH-33 | `POST /auth/2fa/setup` | Admin chưa có 2FA | 201 | setup_id + otpauth_uri | Không log secret. |
| IT-AUTH-34 | `POST /auth/2fa/setup` | Admin đã enabled | 409 | `AUTH_MFA_ALREADY_ENABLED` | Không tạo credential thứ hai. |
| IT-AUTH-35 | `POST /auth/2fa/verify` | Login challenge + TOTP đúng | 200 | Token pair | Challenge verified một lần. |
| IT-AUTH-36 | `POST /auth/2fa/verify` | Sai TOTP/recovery | 401 | `AUTH_MFA_INVALID` | Không cấp token. |
| IT-AUTH-37 | `POST /auth/2fa/verify` | Challenge hết hạn | 400 | `AUTH_MFA_CHALLENGE_EXPIRED` | Không chấp nhận replay. |
| IT-AUTH-38 | `GET /.well-known/jwks.json` | Public request | 200 | Public RSA key set | Chỉ chứa n, e; kid đúng; max-age=600. |

### 3.2 Profile và address

| Mã | Endpoint | Input | HTTP | Response mong đợi | Ghi chú |
|---|---|---|---:|---|---|
| IT-PROFILE-01 | `GET /users/me` | JWT buyer | 200 | Profile không có secret | `sub` quyết định user. |
| IT-PROFILE-02 | `GET /users/me` | Không JWT/sai JWT | 401 | `AUTH_TOKEN_INVALID` | Không gọi DB business. |
| IT-PROFILE-03 | `PUT /users/me` | Full name/date valid | 200 | Profile cập nhật | Phát `user.updated`. |
| IT-PROFILE-04 | `PUT /users/me` | Full name rỗng/121 | 400 | `PROFILE_INVALID` | Không mutation. |
| IT-PROFILE-05 | `PUT /users/me` | Email/role/status/user_id | 400 | `PROFILE_INVALID` | Không privilege escalation. |
| IT-PROFILE-06 | `PUT /users/me` | Phone mới | 200 | `phone_verification_required=true` | Clear `phone_verified_at`. |
| IT-PROFILE-07 | `PUT /users/me` | Phone duplicate | 409 | `AUTH_PHONE_EXISTS` | Không đổi phone. |
| IT-PROFILE-08 | `GET /users/me/addresses` | Page/size hợp lệ | 200 | List + pagination | Chỉ address của subject. |
| IT-PROFILE-09 | `GET /users/me/addresses` | Size 0/101/sort lạ | 400 | `AUTH_INVALID_INPUT` | Không query unbounded. |
| IT-PROFILE-10 | `POST /users/me/addresses` | Address valid đầu tiên | 201 | `is_default=true` | Transaction. |
| IT-PROFILE-11 | `POST /users/me/addresses` | Field empty/over max | 400 | `PROFILE_INVALID` | Assert field detail. |
| IT-PROFILE-12 | `POST /users/me/addresses` | Address thứ 21 | 409 | `ADDRESS_LIMIT_REACHED` | Không ghi record. |
| IT-PROFILE-13 | `PUT /users/me/addresses/{id}` | Address của user khác | 404 | `ADDRESS_NOT_FOUND` | IDOR protection. |
| IT-PROFILE-14 | `PUT /users/me/addresses/{id}` | Set default | 200 | Chỉ một default | Row lock/concurrency. |
| IT-PROFILE-15 | `DELETE /users/me/addresses/{id}` | Address sống | 204 | Soft delete; gán fallback default | Không hard delete. |
| IT-PROFILE-16 | `DELETE /users/me/addresses/{id}` | Address không thuộc user | 404 | `ADDRESS_NOT_FOUND` | Không lộ resource. |
| IT-PROFILE-17 | `DELETE /users/me/addresses/{id}` | Xóa default address khi còn address khác | 204 | Address còn lại gần nhất thành default | Soft delete. |

### 3.3 Seller onboarding và KYC

| Mã | Endpoint | Input | HTTP | Response mong đợi | Ghi chú |
|---|---|---|---:|---|---|
| IT-SELLER-01 | `POST /users/register-seller` | Buyer email verified + shop valid | 201 | Shop DRAFT + onboarding PROFILE | Atomic user/shop/role. |
| IT-SELLER-02 | `POST /users/register-seller` | Email chưa verify | 403 | `AUTH_EMAIL_NOT_VERIFIED` | Không tạo shop. |
| IT-SELLER-03 | `POST /users/register-seller` | User đã có shop | 409 | `SHOP_ALREADY_EXISTS` | Owner lock. |
| IT-SELLER-04 | `POST /users/register-seller` | Tax code duplicate | 409 | `AUTH_TAX_CODE_EXISTS` | Không tạo duplicate. |
| IT-SELLER-05 | `GET /seller/onboarding` | Seller owner | 200 | 5 steps + blockers | Không trả bank raw. |
| IT-SELLER-06 | `GET /seller/onboarding` | Buyer/other shop | 403/404 | RBAC/SHOP_NOT_FOUND | Không đọc cross-shop. |
| IT-SELLER-07 | `PUT /seller/onboarding/profile` | Profile valid | 200 | Snapshot updated | `updated_at` đổi. |
| IT-SELLER-08 | `PUT /seller/onboarding/warehouse` | Address/carrier valid | 200 | Warehouse completion | JSON snapshot validated. |
| IT-SELLER-09 | `PUT /seller/onboarding/bank` | Account valid | 200 | Masked account | Ciphertext khác raw. |
| IT-SELLER-10 | `PUT /seller/onboarding/bank` | Account invalid/confirm false | 400 | `PROFILE_INVALID`/`BANK_ACCOUNT_INVALID` | Không lưu raw. |
| IT-SELLER-11 | `POST /seller/onboarding/kyc/documents/presign` | PDF/JPG/PNG <=10MiB | 201 | Signed URL TTL 10m | Private key/object. |
| IT-SELLER-12 | `POST /seller/onboarding/kyc/documents/presign` | Unsupported MIME | 400 | `KYC_DOCUMENT_INVALID` | Không presign. |
| IT-SELLER-13 | `POST /seller/onboarding/kyc/documents/presign` | 10MiB + 1 byte | 413 | `KYC_DOCUMENT_TOO_LARGE` | Không lưu metadata. |
| IT-SELLER-14 | `POST /seller/onboarding/kyc/documents/complete` | Object/checksum valid | 201 | Document UPLOADED/VERIFIED | Verify storage head. |
| IT-SELLER-15 | `POST /seller/onboarding/kyc/documents/complete` | Object key khác owner | 400/403 | `KYC_DOCUMENT_INVALID` | Không cross-case. |
| IT-SELLER-16 | `POST /seller/onboarding/kyc/documents/complete` | Checksum mismatch | 400 | `KYC_DOCUMENT_INVALID` | Không READY/UPLOADED. |
| IT-SELLER-17 | `POST /seller/onboarding/kyc/submit` | Hồ sơ đủ/doc valid | 200 | Case PENDING | Phát `shop.kyc.submitted`. |
| IT-SELLER-18 | `POST /seller/onboarding/kyc/submit` | Thiếu doc/upload pending | 400 | `KYC_DOCUMENT_INVALID` | Không tạo case pending. |
| IT-SELLER-19 | `POST /seller/onboarding/kyc/submit` | Case đang pending | 409 | `KYC_ALREADY_PENDING` | Idempotent state. |
| IT-SELLER-20 | `GET /seller/shop` | Owner/staff đúng scope | 200 | Shop masked/summary | Staff không thấy bank raw. |
| IT-SELLER-21 | `PUT /seller/shop` | Name/description/logo valid | 200 | Shop updated + event | Không sửa owner/status tùy ý. |
| IT-SELLER-22 | `PUT /seller/shop` | Tax/status/owner mutation | 400/403 | Error | Không privilege escalation. |

### 3.4 Admin KYC/RBAC/status/audit

| Mã | Endpoint | Input | HTTP | Response mong đợi | Ghi chú |
|---|---|---|---:|---|---|
| IT-ADMIN-01 | `GET /admin/shops/kyc` | RISK_MANAGER + status=PENDING | 200 | Paginated queue | Index/query bounded. |
| IT-ADMIN-02 | `GET /admin/shops/kyc` | CATALOG_ADMIN không KYC_READ | 403 | `RBAC_PERMISSION_DENIED` | Role boundary. |
| IT-ADMIN-03 | `GET /admin/shops/{id}/kyc` | KYC_READ | 200 | Metadata + signed URL | URL TTL 10m, no bytes. |
| IT-ADMIN-04 | `GET /admin/shops/{id}/kyc` | Shop không tồn tại | 404 | `SHOP_NOT_FOUND` | Không lộ tenant. |
| IT-ADMIN-05 | `POST /admin/shops/{id}/kyc/review` | APPROVED + valid case | 200 | Shop/KYC APPROVED | Outbox/audit atomic. |
| IT-ADMIN-06 | `POST /admin/shops/{id}/kyc/review` | NEEDS_INFO reason 9/10/1000/1001 | 400 | `KYC_DECISION_INVALID` | Boundary. |
| IT-ADMIN-07 | `POST /admin/shops/{id}/kyc/review` | REJECTED thiếu step-up | 428 | `RBAC_MFA_REQUIRED` | Không đổi state. |
| IT-ADMIN-08 | `POST /admin/shops/{id}/kyc/review` | Case đã terminal | 409 | `KYC_DECISION_INVALID` | Không double decision. |
| IT-ADMIN-09 | `GET /admin/users/{id}/roles` | ROLE_READ | 200 | Assignments + permissions | Scoped data. |
| IT-ADMIN-10 | `PATCH /admin/users/{id}/roles` | GRANT valid shop role | 200 | Role active + event | Unique assignment. |
| IT-ADMIN-11 | `PATCH /admin/users/{id}/roles` | Self grant/SUPER_ADMIN escalation | 403 | `RBAC_PERMISSION_DENIED` | Không tự nâng quyền. |
| IT-ADMIN-12 | `PATCH /admin/users/{id}/roles` | REVOKE missing assignment | 409 | `RBAC_ASSIGNMENT_NOT_FOUND` | Không mutation. |
| IT-ADMIN-13 | `PATCH /admin/users/{id}/status` | SUSPENDED + MFA | 200 | User suspended; refresh sessions revoked; push Redis key | Audit/event + fast revocation. |
| IT-ADMIN-14 | `PATCH /admin/users/{id}/status` | Suspend thiếu reason/MFA | 400/428 | Error | Không đổi status. |
| IT-ADMIN-15 | `GET /admin/audit-logs` | Filter 31 ngày/page size | 200 | Masked audit list | Không raw PII/secret. |
| IT-ADMIN-16 | `GET /admin/audit-logs` | Range >31 ngày/sort lạ | 400 | `AUTH_INVALID_INPUT` | Query bounded. |

### 3.5 Favorites và Follow shop

| Mã | Endpoint | Input | HTTP | Response mong đợi | Ghi chú |
|---|---|---|---:|---|---|
| IT-FAV-01 | `POST /users/me/favorites` | `product_id` UUID hợp lệ | 201 | `{product_id,created_at}` | Không gọi Product Catalog check tồn tại. |
| IT-FAV-02 | `POST /users/me/favorites` | Thêm lại product đã có | 200/201 | Idempotent, không duplicate row | Unique `(user_id,product_id)`. |
| IT-FAV-03 | `POST /users/me/favorites` | `product_id` không phải UUID | 400 | `AUTH_INVALID_INPUT` | Không ghi. |
| IT-FAV-04 | `POST /users/me/favorites` | User đã có 500 favorites | 409 | `FAVORITE_LIMIT_REACHED` | Đếm trong transaction. |
| IT-FAV-05 | `GET /users/me/favorites` | Page/size hợp lệ | 200 | List `product_id`+`created_at`, sort desc | Chỉ favorites của `sub`. |
| IT-FAV-06 | `DELETE /users/me/favorites/{productId}` | Đang có | 204 | Xóa cứng row | — |
| IT-FAV-07 | `DELETE /users/me/favorites/{productId}` | Không có | 204 | Idempotent | Không lỗi. |
| IT-FAV-08 | `GET /users/me/favorites/contains` | 3 ID (1 đã tim) | 200 | `{id: bool}` đúng | — |
| IT-FAV-09 | `GET /users/me/favorites/contains` | 101 ID | 400 | `AUTH_INVALID_INPUT` | Bounded batch. |
| IT-FOLLOW-01 | `POST /shops/{shopId}/follow` | Shop ACTIVE | 201 | `{shop_id,followed_at}` | Kiểm bằng bảng `shops` local. |
| IT-FOLLOW-02 | `POST /shops/{shopId}/follow` | Follow lại | 200/201 | Idempotent | Unique `(user_id,shop_id)`. |
| IT-FOLLOW-03 | `POST /shops/{shopId}/follow` | Shop không tồn tại/DELETED | 404 | `SHOP_NOT_FOUND` | — |
| IT-FOLLOW-04 | `POST /shops/{shopId}/follow` | User đã follow 1000 shop | 409 | `SHOP_FOLLOW_LIMIT_REACHED` | — |
| IT-FOLLOW-05 | `DELETE /shops/{shopId}/follow` | Đang follow / không follow | 204 | Idempotent | Xóa cứng. |
| IT-FOLLOW-06 | `GET /users/me/following` | Page/size | 200 | List shop_id + name/slug/logo snapshot | Chỉ của `sub`. |
| IT-FOLLOW-07 | `GET /shops/{shopId}/followers/count` | Public | 200 | `{shop_id,follower_count}` | Không cần JWT. |
| IT-SHOP-01 | `GET /shops/{shopId}` | Shop ACTIVE, KYC APPROVED | 200 | identity + `is_verified=true` + `follower_count` | Không trả `business_name/tax_code/bank_*/owner_user_id`. |
| IT-SHOP-02 | `GET /shops/{shopId}` | Shop KYC chưa APPROVED | 200 | `is_verified=false` | Vẫn public nếu `status != DELETED`. |
| IT-SHOP-03 | `GET /shops/{shopId}` | Shop `status=DELETED`/không tồn tại | 404 | `SHOP_NOT_FOUND` | — |

### 3.6 Database/concurrency/reliability

| Mã | Tình huống | Kết quả mong đợi |
|---|---|---|
| IT-DB-01 | Hai request cùng set default address | Chỉ một default sau commit; request còn lại retry/conflict an toàn. |
| IT-DB-02 | Hai request register seller cùng user | Chỉ một shop/onboarding/SELLER được tạo. |
| IT-DB-03 | Hai refresh cùng token | Một request thành công; request còn lại phát hiện reuse/đã revoke theo policy, không cấp hai family. |
| IT-DB-04 | Password reset trong lúc refresh | Reset revoke tất cả refresh family; không token mới sau reset. |
| IT-DB-05 | KYC submit/review đồng thời | State transition tuần tự; không có hai current case/decision. |
| IT-DB-06 | Kafka down sau signup/KYC | HTTP mutation vẫn commit; outbox pending retry, không duplicate domain record. |
| IT-DB-07 | Publisher restart giữa publish/ack | Consumer dedupe event_id; không tạo duplicate notification/shop event. |
| IT-DB-08 | Migration up/down trên database trống | Schema/index/check/FK tạo và rollback đúng; seed chạy idempotent. |
| IT-DB-09 | Cleanup job chạy hai instance | Kết quả idempotent; không double revoke/event/audit. |
| IT-DB-10 | MySQL deadlock retry | Transaction retry giới hạn; nếu thất bại trả lỗi an toàn, không partial mutation. |
| IT-DB-11 | Hai request `POST /users/me/favorites` cùng product | Chỉ một row; request còn lại trả idempotent success, không vi phạm unique. |
| IT-DB-12 | Hai request follow cùng shop | Chỉ một row `shop_follows`; `follower_count` không đếm trùng. |

## 4. Gợi ý unit test

| Module/hàm | Case cần phủ |
|---|---|
| `PasswordPolicy` | 11/12/72/73 ký tự; normalize không làm đổi password; Argon2id verify đúng/sai. |
| `EmailNormalizer` | Uppercase, whitespace, Unicode domain policy, invalid format, equality lookup. |
| `PhoneNormalizer` | E.164 valid/invalid, duplicate, clear verification khi đổi. |
| `LoginLockPolicy` | 4 failures không lock; failure thứ 5 trong 15m lock 15m; ngoài window reset; success reset. |
| `TokenService` | JWT claims/TTL/RS256; refresh hash; rotation; expired/revoked/reuse family. |
| `VerificationTokenPolicy` | Email TTL 24h, OTP TTL 5m, one-time, resend revoke, max attempts. |
| `PasswordResetPolicy` | TTL 30m, one-time, revoke sessions, không auto-login. |
| `AddressPolicy` | First default, one default, max 20, delete default fallback, checkout-active guard. |
| `SellerOnboardingPolicy` | 5 step completion/blockers; email gate; one shop/user; JSON snapshot validation. |
| `KycStateMachine` | DRAFT→PENDING→APPROVED/NEEDS_INFO/REJECTED; invalid transitions; reason length. |
| `KycDocumentValidator` | MIME allowlist, 10MiB boundary, checksum, object owner, private path. |
| `RbacPolicy` | Role→permission, system/shop scope, self-escalation, admin 2FA requirement. |
| `MfaService` | TOTP step 30s, challenge TTL 5m, wrong attempts, recovery code one-time. |
| `AuditMapper` | Mask token/password/OTP/bank/KYC; event actor/target/reason; append-only payload. |
| `OutboxPublisher` | 3 retries/2s backoff, DLQ, idempotent event ID, partition key. |
| `CleanupJobs` | Unlock, token/document expiry, archive, restart/idempotency. |
| `FavoritePolicy` | UUID validate, upsert idempotent, `MAX_FAVORITES_PER_USER`, contains batch ≤100, không gọi Product Catalog. |
| `ShopFollowPolicy` | Shop exists/not DELETED, upsert idempotent, `MAX_FOLLOWED_SHOPS_PER_USER`, `follower_count` = count theo index. |
| `PublicShopMapper` | Chỉ expose field công khai; `is_verified` từ `kyc_status`; ẩn `business_name/tax_code/bank_*/owner_user_id`. |

## 5. Contract, security và resilience test

### 5.1 Contract test

| Mã | Contract | Kiểm tra |
|---|---|---|
| CT-01 | Notification command | Schema version, command type, recipient masking, dedupe key, no raw token/OTP. |
| CT-02 | Storage presign | Purpose/owner/content type/size/checksum; signed URL TTL 10m; object private. |
| CT-03 | User/shop event | Envelope, aggregate key, event version, payload allowlist. |
| CT-04 | JWKS | `kid`/RS256 public key, rotation overlap, invalid key algorithm rejected. |
| CT-05 | Gateway error | HTTP/status/code/message/trace_id/details mapping; no stack trace/internal URL. |

### 5.2 Security test

| Mã | Kiểm tra | Kết quả bắt buộc |
|---|---|---|
| SEC-01 | JWT `alg=none`/HS256/unknown `kid` | Từ chối 401; không fallback. |
| SEC-02 | JWT issuer/audience/exp/nbf sai | Từ chối 401; không gọi protected business handler. |
| SEC-03 | IDOR address/shop/KYC/role | User chỉ đọc/sửa resource thuộc scope; resource khác trả 404/403 theo contract. |
| SEC-04 | Header spoofing `X-User-ID`, `X-Role`, `X-MFA-Step-Up` | Strip/reject client header; context lấy từ validated token/step-up. |
| SEC-05 | Password/token/OTP/KYC/bank log scan | Không xuất hiện raw trong application log, event, error, audit. |
| SEC-06 | Brute force | Login/OTP/MFA/resend rate limit và lockout đúng threshold. |
| SEC-07 | Reset/refresh replay | Token one-time/rotation; reuse revoke family; reset revoke sessions. |
| SEC-08 | KYC object traversal | Không download object key của case/shop khác; signed URL private/expiry. |
| SEC-09 | SQL/JSON injection | DTO/ORM binding an toàn; metadata JSON không chạy query tùy ý. |
| SEC-10 | CORS/CSRF transport | Bearer JSON flow không mở wildcard credentials; nếu chuyển cookie phải có CSRF test riêng. |

### 5.3 Resilience/performance test

| Mã | Tình huống | Kết quả mong đợi |
|---|---|---|
| RES-01 | Kafka unavailable | Signup/KYC commit không mất; outbox retry/DLQ; response không duplicate. |
| RES-02 | Storage unavailable | Presign/complete lỗi rõ; không mark document verified khi chưa verify object. |
| RES-03 | Notification unavailable | Signup/forgot accepted nếu domain commit; resend/monitor retry đúng. |
| RES-04 | MySQL transient deadlock | Retry transaction giới hạn; không partial state. |
| RES-05 | Concurrent refresh | Không cấp duplicate valid replacement cho cùng token. |
| RES-06 | Login burst | Rate limit/lockout hoạt động; DB không bị query unbounded. |
| RES-07 | KYC queue pagination | Query dùng index; page/size max 100; không full table scan bất ngờ. |
| RES-08 | Cleanup replay | Chạy nhiều instance không double event/revoke. |
| RES-09 | Load baseline | Ghi lại p50/p95/p99 và error rate cho signin, refresh, profile, KYC queue; ngưỡng release do team điền. |

## 6. Tiêu chí pass và phát hành

| Tiêu chí | Điều kiện đạt |
|---|---|
| API coverage | Mỗi endpoint trong API spec có tối thiểu 1 success, 1 validation error và 1 auth/permission/state error phù hợp. |
| Error coverage | Mọi mã lỗi trong LLD/API có ít nhất một test tạo được lỗi đó. |
| State coverage | Mọi chuyển trạng thái User, Shop/KYC, token, verification, MFA được test hợp lệ và bị chặn. |
| Security gate | SEC-01 đến SEC-10 pass; không có secret/PII raw trong log/event/audit scan. |
| DB gate | Migration clean, rollback local, unique/check/FK nội bộ và concurrency test pass. |
| Contract gate | Notification, Storage, Kafka/JWKS contract test pass theo schema version. |
| UI gate | Manual QA cover desktop/mobile/tablet, loading/empty/error/offline; không còn blocker mức Cao. |
| Reliability gate | Outbox retry/DLQ, cleanup idempotency và dependency failure được verify. |
| Regression gate | Không có defect Critical/High mở; Medium có owner và decision release. |

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | Test dùng mock Notification/S3/Kafka và test RSA/JWKS fixture; provider thật chưa tích hợp ở auth-user test suite. | Cần thêm certification test với provider thật trước production. | DevOps/Security |
| 2 | UI token transport là JSON Bearer; chưa có cookie/CSRF flow. | Nếu đổi sang HttpOnly cookie, bổ sung browser security/E2E test. | MFE/Security |
| 3 | Threshold performance để team điền theo capacity thực tế; test plan chỉ yêu cầu ghi baseline p50/p95/p99. | Không thể dùng số liệu này làm SLO nếu chưa có target. | Tech lead/DevOps |
| 4 | Seller Staff invitation UI chưa đầy đủ trong Penpot; test scope chỉ cover assignment/status cơ bản. | Nếu mở full staff lifecycle, thêm invite/accept/revoke test. | Product owner |
| 5 | Checkout-active session được mock bằng flag/service fixture vì Order Service chưa có trong test repository. | E2E address deletion cần chạy lại khi Order contract hoàn tất. | Order owner |
| 6 | Favorites/Follow bổ sung từ Frontend Design (không có trong HLD), đặt tại `auth-user`; favorites chỉ lưu `product_id`, thẻ sản phẩm hydrate qua Product Catalog trong E2E. | Nếu tách service `engagement`, di chuyển bộ test IT-FAV/IT-FOLLOW. | Product owner |
| 7 | `GET /shops/{id}` chỉ trả identity + `is_verified` + `follower_count`; rating/product count từ Product Catalog. | E2E Shop hero cần cả hai nguồn; test contract ghép ở tầng frontend/BFF. | Frontend + Product owner |
