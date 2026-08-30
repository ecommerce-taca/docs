# Database — Auth User Service

> Nguồn: `docs/lld/auth-user.md` · `EcommercePlatform-v4(6).excalidraw` · `New File 1.penpot.zip` · Cập nhật: `2026-08-30`
> Engine: MySQL 8.4 · Database/schema: `userdb` · Mỗi service một database riêng

## 1. Quy ước chung

| Mục | Quy định |
|---|---|
| Đặt tên bảng | `snake_case`, số nhiều: `users`, `refresh_tokens`. |
| Khoá chính | UUIDv7; API trả canonical string 36 ký tự; MySQL lưu `BINARY(16)` để giảm kích thước index. |
| ID nghiệp vụ | Không dùng auto-increment để tránh lộ volume và collision khi nhiều instance. |
| Cột thời gian | `DATETIME(6)`, lưu UTC; API trả ISO-8601 UTC. |
| Audit cơ bản | Bảng mutable có `created_at`, `updated_at`; mutation nhạy cảm ghi `audit_logs`. |
| Xoá mềm | `deleted_at DATETIME(6) NULL`; record còn sống có `deleted_at IS NULL`. User/shop không hard delete trong v1. |
| Enum | `VARCHAR(32)` + `CHECK` constraint; giá trị viết HOA, liệt kê ở mục 5. |
| Boolean | `TINYINT(1)` với `CHECK (value IN (0,1))`. |
| Tiền tệ | Auth-user không sở hữu tiền; nếu có phí/balance phát sinh thì không lưu ở database này. |
| Text/Unicode | `utf8mb4`, collation `utf8mb4_0900_ai_ci` cho field hiển thị; normalized email/phone dùng binary collation để unique chính xác. |
| JSON | Chỉ dùng cho snapshot/provider metadata không cần join/filter thường xuyên; field query-critical được tách cột. |
| PII/secret | Password chỉ lưu Argon2id hash; TOTP secret, bank account đầy đủ và sensitive metadata phải encrypt application-level trước khi ghi. |
| Cross-service reference | `user_id`, `shop_id`, `product_id` bên ngoài chỉ lưu dạng ID/reference; không tạo FK sang database service khác. |
| Transaction | Mutation cần atomicity dùng transaction; outbox ghi cùng transaction domain. MySQL chạy InnoDB. |
| Charset lưu token | Token raw/OTP raw không lưu; chỉ lưu SHA-256 hex hoặc binary digest. |

### 1.1 Chuẩn kiểu dùng trong tài liệu

| Ký hiệu | Ý nghĩa |
|---|---|
| `UUID` | `BINARY(16)` UUIDv7. |
| `VARCHAR(n)` | Tối đa `n` ký tự Unicode, trừ khi ghi rõ byte. |
| `VARBINARY(n)` | Dữ liệu bytes đã mã hóa hoặc hash. |
| `JSON` | JSON hợp lệ, schema chi tiết do application validator kiểm tra. |
| `TIMESTAMP` | Tài liệu dùng tên gọi ngắn cho `DATETIME(6)` UTC. |

### 1.2 Quy tắc bảo mật dữ liệu

- `password_hash` dùng Argon2id; không lưu password raw, token raw, OTP raw hoặc TOTP secret plaintext.
- Email/phone cần normalized value để lookup và unique; raw display value có thể giữ ở cột riêng nếu UI cần giữ format.
- `bank_account_ciphertext`, `totp_secret_ciphertext` và recovery code hash không được đưa vào API response hoặc log.
- KYC document chỉ lưu object key/checksum/metadata; file bytes nằm ở private S3/MinIO.
- `audit_logs` append-only về mặt application; chỉ migration/retention job được archive theo policy 365 ngày.

## 2. Quan hệ

```text
users ──1:n──► addresses
  │
  ├──1:n──► refresh_tokens / verification_tokens / password_reset_tokens
  ├──n:m──► roles ──n:m──► permissions     (qua user_roles / role_permissions)
  ├──1:n──► shops ──1:1──► seller_onboarding
  │             │
  │             └──1:n──► kyc_cases ──1:n──► kyc_documents
  │
  ├──1:1──► two_factor_credentials ──1:n──► two_factor_recovery_codes
  └──1:n──► mfa_challenges / login_attempts / audit_logs

shops ──n:m──► users                         (qua shop_staff)
mọi domain event ──1:n──► outbox_events
```

| Quan hệ | Kiểu | Khoá nội bộ | Khi cha bị xoá |
|---|---|---|---|
| `users → addresses` | 1:n | `addresses.user_id → users.id` | Không hard delete user; address soft delete. FK `RESTRICT`. |
| `users → refresh_tokens` | 1:n | `refresh_tokens.user_id → users.id` | Revoke trước; cleanup job mới xóa token hết hạn. FK `RESTRICT`. |
| `users → shops` | 1:n về mặt lịch sử, tối đa 1 onboarding/active shop ở v1 | `shops.owner_user_id → users.id` | Không hard delete; shop chuyển status. FK `RESTRICT`. |
| `shops → seller_onboarding` | 1:1 | `seller_onboarding.shop_id → shops.id` | Xóa soft cùng shop nếu migration cleanup. FK `CASCADE` chỉ cho hard-delete test fixture. |
| `shops → kyc_cases` | 1:n | `kyc_cases.shop_id → shops.id` | Giữ lịch sử KYC; FK `RESTRICT`. |
| `kyc_cases → kyc_documents` | 1:n | `kyc_documents.kyc_case_id → kyc_cases.id` | Xóa metadata khi test cleanup; production archive. FK `CASCADE` cho fixture. |
| `users ↔ roles` | n:m | `user_roles.user_id/role_id` | Revoke assignment, giữ audit. FK `RESTRICT`. |
| `roles ↔ permissions` | n:m | `role_permissions.role_id/permission_id` | Seed migration quản lý; không xóa role đang dùng. FK `RESTRICT`. |
| `shops ↔ users` | n:m | `shop_staff.shop_id/user_id` | Staff chuyển `REVOKED`, không xóa lịch sử. FK `RESTRICT`. |
| Service khác → user/shop | Reference ID | Không có FK cross-service | Consumer tự xử lý event mất/unknown ID. |

## 3. Chi tiết bảng

### 3.1 `users` — tài khoản buyer, seller và admin

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | UUIDv7. |
| `email` | `VARCHAR(254)` | N | — | Display email; email mới phải verification flow | Email hiện tại. |
| `email_normalized` | `VARCHAR(254)` | N | — | UNIQUE, binary collation | Lowercase + trim. |
| `email_verified_at` | `DATETIME(6)` | Y | `NULL` | — | Null khi chưa verify. |
| `phone` | `VARCHAR(16)` | Y | `NULL` | E.164 format | Số hiển thị/đã normalize. |
| `phone_normalized` | `VARCHAR(16)` | Y | `NULL` | UNIQUE khi non-null | Dùng login/lookup. |
| `phone_verified_at` | `DATETIME(6)` | Y | `NULL` | — | Đổi phone phải clear. |
| `password_hash` | `VARCHAR(255)` | N | — | Argon2id encoded string | Không lưu raw password. |
| `full_name` | `VARCHAR(120)` | N | — | trim, length 1–120 | Tên hiển thị. |
| `date_of_birth` | `DATE` | Y | `NULL` | Không nhận future date | Ngày sinh theo profile UI. |
| `status` | `VARCHAR(16)` | N | `'ACTIVE'` | `ACTIVE/LOCKED/SUSPENDED/DELETED` | Account lifecycle. |
| `failed_login_count` | `SMALLINT UNSIGNED` | N | `0` | `>=0` | Counter trong login window. |
| `failed_login_window_started_at` | `DATETIME(6)` | Y | `NULL` | — | Mốc bắt đầu window 15 phút. |
| `locked_until` | `DATETIME(6)` | Y | `NULL` | — | Hết thời điểm thì có thể mở khóa tự động. |
| `password_changed_at` | `DATETIME(6)` | Y | `NULL` | — | Dùng security audit/session policy. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | Update on mutation | UTC. |
| `deleted_at` | `DATETIME(6)` | Y | `NULL` | Soft delete marker | Set cùng status `DELETED`. |

Khoá & ràng buộc:

| Loại | Cột | Ghi chú |
|---|---|---|
| PK | `id` | UUIDv7. |
| UNIQUE | `email_normalized` | Không phân biệt hoa thường do application normalization + binary collation. |
| UNIQUE | `phone_normalized` | MySQL cho phép nhiều `NULL`; chỉ phone non-null phải duy nhất. |
| CHECK | `status` | Chỉ nhận enum ở mục 5.1. |
| CHECK | `failed_login_count` | Không âm. |

### 3.2 `addresses` — sổ địa chỉ của user

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | UUIDv7. |
| `user_id` | `BINARY(16)` | N | — | FK `users.id` | Owner. |
| `recipient` | `VARCHAR(120)` | N | — | Length 1–120 | Người nhận. |
| `phone` | `VARCHAR(16)` | N | — | E.164 | Phone giao hàng. |
| `line1` | `VARCHAR(255)` | N | — | Length 1–255 | Số nhà/đường. |
| `line2` | `VARCHAR(255)` | Y | `NULL` | — | Căn hộ/tòa nhà. |
| `ward` | `VARCHAR(120)` | N | — | — | Phường/xã. |
| `district` | `VARCHAR(120)` | N | — | — | Quận/huyện. |
| `province` | `VARCHAR(120)` | N | — | — | Tỉnh/thành. |
| `postal_code` | `VARCHAR(12)` | Y | `NULL` | — | Optional. |
| `is_default` | `TINYINT(1)` | N | `0` | `0/1` | Tối đa một default/user ở application transaction. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | Update on mutation | UTC. |
| `deleted_at` | `DATETIME(6)` | Y | `NULL` | Soft delete | Không tính vào limit 20. |

Ràng buộc nghiệp vụ:

- Tối đa 20 record sống/user.
- Khi tạo address đầu tiên, set `is_default=1`.
- Set default phải clear default cũ trong cùng transaction.
- Không xóa address cuối nếu user có checkout session active; Order lưu snapshot riêng.

### 3.3 `shops` — hồ sơ shop và snapshot KYC gate

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | `shop_id`. |
| `owner_user_id` | `BINARY(16)` | N | — | FK `users.id` | Chủ shop. |
| `name` | `VARCHAR(120)` | N | — | — | Tên hiển thị shop. |
| `slug` | `VARCHAR(160)` | N | — | UNIQUE | URL shop, lowercase/hyphen. |
| `business_name` | `VARCHAR(200)` | N | — | — | Tên pháp lý/doanh nghiệp. |
| `tax_code` | `VARCHAR(20)` | Y | `NULL` | UNIQUE non-null | Mã số thuế đã normalize. |
| `description` | `VARCHAR(2000)` | Y | `NULL` | — | Mô tả Seller Center. |
| `logo_object_key` | `VARCHAR(512)` | Y | `NULL` | Private object only | Không lưu bytes. |
| `status` | `VARCHAR(16)` | N | `'DRAFT'` | Shop enum | Shop lifecycle. |
| `kyc_status` | `VARCHAR(16)` | N | `'DRAFT'` | KYC enum | Projection/current gate. |
| `warehouse_snapshot` | `JSON` | Y | `NULL` | JSON valid | Address/contact/carrier snapshot. |
| `bank_name` | `VARCHAR(120)` | Y | `NULL` | — | Bank display. |
| `bank_account_name` | `VARCHAR(120)` | Y | `NULL` | — | Tên chủ tài khoản. |
| `bank_account_last4` | `CHAR(4)` | Y | `NULL` | Digits only | Hiển thị masked. |
| `bank_account_ciphertext` | `VARBINARY(2048)` | Y | `NULL` | Application encrypted | Số tài khoản đầy đủ. |
| `bank_key_version` | `VARCHAR(32)` | Y | `NULL` | — | KMS key version. |
| `bank_verified_at` | `DATETIME(6)` | Y | `NULL` | — | Xác nhận bank metadata. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | Update on mutation | UTC. |
| `deleted_at` | `DATETIME(6)` | Y | `NULL` | Soft delete | Không hard delete production. |

Ràng buộc:

- V1: một user chỉ có tối đa một shop đang onboarding/active; application lock theo `owner_user_id`.
- Publish product/withdraw không dựa vào `status` riêng lẻ; phải có `status=ACTIVE` và `kyc_status=APPROVED` theo service policy.
- Không lưu KYC document bytes, password, bank account raw hoặc signed URL vào bảng.

### 3.4 `seller_onboarding` — tiến độ 5 bước trong Penpot

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | UUIDv7. |
| `shop_id` | `BINARY(16)` | N | — | UNIQUE + FK `shops.id` | 1 record/shop. |
| `current_step` | `VARCHAR(24)` | N | `'PROFILE'` | `PROFILE/KYC/WAREHOUSE/BANK/FIRST_PRODUCT/COMPLETED` | Step hiện tại. |
| `profile_completed` | `TINYINT(1)` | N | `0` | `0/1` | Step 1. |
| `kyc_completed` | `TINYINT(1)` | N | `0` | `0/1` | Step 2. |
| `warehouse_completed` | `TINYINT(1)` | N | `0` | `0/1` | Step 3. |
| `bank_completed` | `TINYINT(1)` | N | `0` | `0/1` | Step 4. |
| `first_product_completed` | `TINYINT(1)` | N | `0` | `0/1` | Step 5; product service reference only. |
| `blockers_json` | `JSON` | N | `(JSON_ARRAY())` | JSON valid | Blocker codes hiển thị Seller Center. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

`first_product_completed` chỉ là projection/flag nhận từ Product Service event; không tạo FK sang product database.

### 3.5 `refresh_tokens` — refresh token rotation/revoke

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | Session/token record. |
| `user_id` | `BINARY(16)` | N | — | FK `users.id` | Owner. |
| `token_hash` | `CHAR(64)` | N | — | UNIQUE | SHA-256 hex của refresh token. |
| `family_id` | `BINARY(16)` | N | — | Index | Rotation family. |
| `issued_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |
| `expires_at` | `DATETIME(6)` | N | — | `> issued_at` | TTL 30 ngày. |
| `revoked_at` | `DATETIME(6)` | Y | `NULL` | — | Revoke/rotation. |
| `revoke_reason` | `VARCHAR(32)` | Y | `NULL` | Enum-like | `ROTATED/SIGNOUT/RESET/REUSE/SUSPEND`. |
| `replaced_by_token_id` | `BINARY(16)` | Y | `NULL` | Internal reference | Token kế tiếp cùng family. Không FK bắt buộc để tránh cycle. |
| `last_seen_at` | `DATETIME(6)` | Y | `NULL` | — | Security telemetry. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

### 3.6 `verification_tokens` — email verification và channel token

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | UUIDv7. |
| `user_id` | `BINARY(16)` | N | — | FK `users.id` | Owner. |
| `channel` | `VARCHAR(8)` | N | — | `EMAIL/PHONE` | Kênh gửi. |
| `purpose` | `VARCHAR(32)` | N | — | `EMAIL_VERIFY/PHONE_VERIFY` | Mục đích. |
| `token_hash` | `CHAR(64)` | N | — | Index | SHA-256 token/OTP challenge secret. |
| `recipient_masked` | `VARCHAR(254)` | N | — | Không raw OTP | Email/phone masked để support. |
| `expires_at` | `DATETIME(6)` | N | — | — | Email 24h, phone 5m. |
| `used_at` | `DATETIME(6)` | Y | `NULL` | — | One-time use. |
| `revoked_at` | `DATETIME(6)` | Y | `NULL` | Resend revoke | Token cũ không dùng lại. |
| `attempt_count` | `TINYINT UNSIGNED` | N | `0` | `0–5` | OTP attempts. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

### 3.7 `password_reset_tokens` — reset password one-time token

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | UUIDv7. |
| `user_id` | `BINARY(16)` | N | — | FK `users.id` | Owner. |
| `token_hash` | `CHAR(64)` | N | — | UNIQUE | SHA-256 token. |
| `expires_at` | `DATETIME(6)` | N | — | TTL 30m | — |
| `used_at` | `DATETIME(6)` | Y | `NULL` | One-time | — |
| `revoked_at` | `DATETIME(6)` | Y | `NULL` | New reset revokes old | — |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

### 3.8 `roles` — role catalog

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | Seed UUID ổn định. |
| `role_key` | `VARCHAR(32)` | N | — | UNIQUE | `BUYER`, `SELLER`, `SELLER_STAFF`, admin roles. |
| `scope_type` | `VARCHAR(16)` | N | `'USER'` | `USER/SHOP/SYSTEM` | Scope assignment. |
| `description` | `VARCHAR(255)` | N | — | — | Internal description. |
| `is_system` | `TINYINT(1)` | N | `1` | `0/1` | Không xóa system role. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

### 3.9 `permissions` — permission catalog

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | Seed UUID ổn định. |
| `permission_key` | `VARCHAR(64)` | N | — | UNIQUE | Ví dụ `KYC_DECIDE`, `USER_SUSPEND`. |
| `scope_type` | `VARCHAR(16)` | N | `'SYSTEM'` | `USER/SHOP/SYSTEM` | Scope yêu cầu. |
| `description` | `VARCHAR(255)` | N | — | — | Internal description. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

### 3.10 `role_permissions` — mapping role/permission

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `role_id` | `BINARY(16)` | N | — | FK `roles.id` | — |
| `permission_id` | `BINARY(16)` | N | — | FK `permissions.id` | — |
| `granted_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |
| `granted_by` | `BINARY(16)` | Y | `NULL` | User reference nội bộ | Null với seed. |

PK/constraint: composite PK (`role_id`, `permission_id`); không duplicate mapping.

### 3.11 `user_roles` — assignment hiện tại của user

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | Assignment ID. |
| `user_id` | `BINARY(16)` | N | — | FK `users.id` | User. |
| `role_id` | `BINARY(16)` | N | — | FK `roles.id` | Role. |
| `shop_id` | `BINARY(16)` | Y | `NULL` | `NULL` = system scope; otherwise FK `shops.id` | Scope. |
| `granted_by` | `BINARY(16)` | Y | `NULL` | User reference | Actor grant. |
| `granted_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |
| `revoked_at` | `DATETIME(6)` | Y | `NULL` | — | Revoke assignment; row có thể re-activate. |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |

UNIQUE (`user_id`, `role_id`, `shop_id`) hoặc unique active assignment; lịch sử grant/revoke nằm ở `audit_logs`.

### 3.12 `shop_staff` — thành viên Seller Staff

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | — |
| `shop_id` | `BINARY(16)` | N | — | FK `shops.id` | — |
| `user_id` | `BINARY(16)` | N | — | FK `users.id` | — |
| `staff_role` | `VARCHAR(32)` | N | — | Permission profile | V1 có thể `STAFF`. |
| `status` | `VARCHAR(16)` | N | `'INVITED'` | `INVITED/ACTIVE/REVOKED` | — |
| `invited_by` | `BINARY(16)` | N | — | FK `users.id` | — |
| `invited_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |
| `joined_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `revoked_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |

UNIQUE (`shop_id`, `user_id`); status revoke giữ row.

### 3.13 `kyc_cases` — case review của shop

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | KYC case ID. |
| `shop_id` | `BINARY(16)` | N | — | FK `shops.id` | — |
| `status` | `VARCHAR(16)` | N | `'DRAFT'` | KYC enum | — |
| `submitted_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `reviewed_by` | `BINARY(16)` | Y | `NULL` | User reference | Admin actor. |
| `reviewed_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `decision_reason` | `VARCHAR(1000)` | Y | `NULL` | 10–1000 với NEEDS_INFO/REJECTED | — |
| `expires_at` | `DATETIME(6)` | Y | `NULL` | — | KYC/document expiry. |
| `source_version` | `INT UNSIGNED` | N | `1` | Monotonic per shop | Event/projection version. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |

Index/app invariant: chỉ một case `PENDING/NEEDS_INFO/APPROVED` hiện hành cho một shop; transaction phải lock shop row trước khi submit/review.

### 3.14 `kyc_documents` — metadata object storage

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | — |
| `kyc_case_id` | `BINARY(16)` | N | — | FK `kyc_cases.id` | — |
| `document_type` | `VARCHAR(32)` | N | — | Seeded enum | ID/business registration/etc. |
| `object_key` | `VARCHAR(512)` | N | — | Private prefix | Không nhận arbitrary foreign key từ client. |
| `original_file_name` | `VARCHAR(255)` | N | — | Sanitize path | Tên hiển thị. |
| `content_type` | `VARCHAR(64)` | N | — | `application/pdf`, `image/jpeg`, `image/png` | Allowlist. |
| `size_bytes` | `INT UNSIGNED` | N | — | `1–10485760` | Max 10 MiB. |
| `sha256` | `CHAR(64)` | N | — | Hex checksum | Verify complete upload. |
| `status` | `VARCHAR(16)` | N | `'UPLOADING'` | Document enum | — |
| `uploaded_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `verified_at` | `DATETIME(6)` | Y | `NULL` | — | Metadata verification. |
| `deleted_at` | `DATETIME(6)` | Y | `NULL` | Soft delete | — |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |

UNIQUE (`kyc_case_id`, `document_type`, `sha256`) cho document sống; không lưu file bytes.

### 3.15 `two_factor_credentials` — TOTP credential

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | — |
| `user_id` | `BINARY(16)` | N | — | UNIQUE + FK `users.id` | Một credential active/user. |
| `secret_ciphertext` | `VARBINARY(1024)` | N | — | Application encrypted | TOTP secret. |
| `key_version` | `VARCHAR(32)` | N | — | KMS key version | — |
| `status` | `VARCHAR(16)` | N | `'ENROLLING'` | `DISABLED/ENROLLING/ENABLED/RESET_REQUIRED` | — |
| `enabled_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `disabled_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |
| `updated_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |

### 3.16 `two_factor_recovery_codes` — hash code recovery

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | — |
| `credential_id` | `BINARY(16)` | N | — | FK `two_factor_credentials.id` | — |
| `code_hash` | `CHAR(64)` | N | — | UNIQUE | SHA-256 recovery code. |
| `used_at` | `DATETIME(6)` | Y | `NULL` | One-time | — |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |

### 3.17 `mfa_challenges` — admin login/step-up challenge

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK | Challenge ID trả client. |
| `user_id` | `BINARY(16)` | N | — | FK `users.id` | — |
| `purpose` | `VARCHAR(16)` | N | — | `LOGIN/STEP_UP/ENROLL` | — |
| `code_hash` | `CHAR(64)` | Y | `NULL` | Không lưu raw code | Với recovery/OTP adapter nếu cần. |
| `expires_at` | `DATETIME(6)` | N | — | TTL 5m | — |
| `attempt_count` | `TINYINT UNSIGNED` | N | `0` | `0–5` | — |
| `verified_at` | `DATETIME(6)` | Y | `NULL` | One-time | — |
| `revoked_at` | `DATETIME(6)` | Y | `NULL` | — | — |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | — |

### 3.18 `login_attempts` — security/rate-limit evidence

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BIGINT UNSIGNED` | N | AUTO_INCREMENT | PK nội bộ | Bảng append lớn. Không expose ID. |
| `user_id` | `BINARY(16)` | Y | `NULL` | FK `users.id` khi resolve được | Null với unknown identifier. |
| `identifier_hash` | `CHAR(64)` | N | — | Index | Hash email/phone input. |
| `succeeded` | `TINYINT(1)` | N | `0` | `0/1` | — |
| `failure_reason` | `VARCHAR(32)` | Y | `NULL` | Không lưu password | — |
| `ip_hash` | `CHAR(64)` | N | — | Không lưu raw IP nếu không cần | Security aggregation. |
| `user_agent_hash` | `CHAR(64)` | Y | `NULL` | — | — |
| `occurred_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

### 3.19 `audit_logs` — immutable security/business audit

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BIGINT UNSIGNED` | N | AUTO_INCREMENT | PK nội bộ | Sequence audit. |
| `event_id` | `BINARY(16)` | N | — | UNIQUE | UUIDv7. |
| `actor_user_id` | `BINARY(16)` | Y | `NULL` | User reference | Null với system job. |
| `action` | `VARCHAR(64)` | N | — | Allowlist application | Ví dụ `AUTH_SIGNIN`, `KYC_APPROVE`. |
| `target_type` | `VARCHAR(32)` | N | — | — | `USER/SHOP/KYC/ROLE`. |
| `target_id` | `BINARY(16)` | Y | `NULL` | Reference ID | — |
| `reason` | `VARCHAR(1000)` | Y | `NULL` | — | Required action-specific. |
| `metadata` | `JSON` | N | `(JSON_OBJECT())` | Không chứa secret | Changed fields/masked values. |
| `ip_hash` | `CHAR(64)` | Y | `NULL` | — | Hash. |
| `occurred_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

Không update/delete audit log qua application API.

### 3.20 `outbox_events` — event phát hành đáng tin cậy

| Cột | Kiểu | Null | Default | Ràng buộc | Mô tả |
|---|---|---:|---|---|---|
| `id` | `BINARY(16)` | N | — | PK/event ID | UUIDv7. |
| `aggregate_type` | `VARCHAR(32)` | N | — | `USER/SHOP/KYC` | — |
| `aggregate_id` | `BINARY(16)` | N | — | Reference ID | — |
| `event_type` | `VARCHAR(64)` | N | — | Allowlist | — |
| `schema_version` | `SMALLINT UNSIGNED` | N | `1` | — | — |
| `partition_key` | `VARCHAR(128)` | N | — | User/shop ID string | Kafka key. |
| `payload` | `JSON` | N | — | No secret/PII raw | Event payload. |
| `attempt_count` | `TINYINT UNSIGNED` | N | `0` | `0–3` | — |
| `next_retry_at` | `DATETIME(6)` | Y | `NULL` | — | Retry scheduler. |
| `published_at` | `DATETIME(6)` | Y | `NULL` | — | Null = pending. |
| `failed_at` | `DATETIME(6)` | Y | `NULL` | — | DLQ marker. |
| `last_error_code` | `VARCHAR(64)` | Y | `NULL` | Sanitized | Không raw stack trace. |
| `created_at` | `DATETIME(6)` | N | `CURRENT_TIMESTAMP(6)` | — | UTC. |

## 4. Index

| Tên index | Bảng | Cột | Loại | Truy vấn/màn hình phục vụ |
|---|---|---|---|---|
| `uk_users_email_normalized` | `users` | `email_normalized` | UNIQUE | Sign up/sign in bằng email. |
| `uk_users_phone_normalized` | `users` | `phone_normalized` | UNIQUE | Phone login/phone uniqueness; null được phép. |
| `ix_users_status_created` | `users` | `status, created_at` | B-tree | Admin user list/status operation. |
| `ix_users_locked_until` | `users` | `status, locked_until` | B-tree | Unlock scheduled job. |
| `uk_addresses_user_id_id` | `addresses` | `user_id, id` | UNIQUE/lookup | Ownership lookup/pagination. |
| `ix_addresses_user_live_default` | `addresses` | `user_id, deleted_at, is_default` | B-tree | Address book/default address. |
| `ix_addresses_user_updated` | `addresses` | `user_id, updated_at` | B-tree | Sort address list. |
| `uk_shops_slug` | `shops` | `slug` | UNIQUE | Shop URL/read. |
| `uk_shops_tax_code` | `shops` | `tax_code` | UNIQUE | Seller registration conflict. |
| `ix_shops_owner_status` | `shops` | `owner_user_id, status, deleted_at` | B-tree | Register seller/owner lookup. |
| `ix_shops_kyc_queue` | `shops` | `kyc_status, status, updated_at` | B-tree | Admin KYC queue. |
| `uk_seller_onboarding_shop` | `seller_onboarding` | `shop_id` | UNIQUE | Current onboarding. |
| `ix_onboarding_step_updated` | `seller_onboarding` | `current_step, updated_at` | B-tree | Seller ops queue/analytics. |
| `uk_refresh_token_hash` | `refresh_tokens` | `token_hash` | UNIQUE | Refresh lookup. |
| `ix_refresh_user_active` | `refresh_tokens` | `user_id, revoked_at, expires_at` | B-tree | Signout/revoke active sessions. |
| `ix_refresh_family` | `refresh_tokens` | `family_id, revoked_at` | B-tree | Reuse detection/revoke family. |
| `ix_refresh_expiry` | `refresh_tokens` | `expires_at` | B-tree | Cleanup job. |
| `ix_verification_token_hash` | `verification_tokens` | `token_hash, used_at, revoked_at` | B-tree | Verify token lookup. |
| `ix_verification_user_purpose` | `verification_tokens` | `user_id, purpose, channel, created_at` | B-tree | Resend/revoke/rate limit. |
| `ix_verification_expiry` | `verification_tokens` | `expires_at` | B-tree | Cleanup job. |
| `uk_password_reset_token_hash` | `password_reset_tokens` | `token_hash` | UNIQUE | Reset lookup. |
| `ix_password_reset_user_active` | `password_reset_tokens` | `user_id, used_at, revoked_at, expires_at` | B-tree | Revoke active reset links. |
| `uk_roles_key` | `roles` | `role_key` | UNIQUE | RBAC resolution. |
| `uk_permissions_key` | `permissions` | `permission_key` | UNIQUE | Permission resolution. |
| `uk_role_permission` | `role_permissions` | `role_id, permission_id` | PK | Role matrix. |
| `uk_user_role_scope` | `user_roles` | `user_id, role_id, shop_id` | UNIQUE | Không duplicate assignment. |
| `ix_user_roles_user_active` | `user_roles` | `user_id, revoked_at` | B-tree | JWT role resolution/admin role screen. |
| `ix_user_roles_shop_active` | `user_roles` | `shop_id, revoked_at` | B-tree | Seller scope. |
| `uk_shop_staff_member` | `shop_staff` | `shop_id, user_id` | UNIQUE | Staff membership. |
| `ix_shop_staff_user_status` | `shop_staff` | `user_id, status` | B-tree | Staff lookup. |
| `ix_kyc_cases_shop_status` | `kyc_cases` | `shop_id, status, updated_at` | B-tree | Current KYC/detail. |
| `ix_kyc_cases_queue` | `kyc_cases` | `status, submitted_at` | B-tree | Admin queue sort. |
| `ix_kyc_cases_expiry` | `kyc_cases` | `expires_at, status` | B-tree | Expiry job. |
| `uk_kyc_doc_checksum` | `kyc_documents` | `kyc_case_id, document_type, sha256` | UNIQUE | Không duplicate file cùng type. |
| `ix_kyc_docs_case_status` | `kyc_documents` | `kyc_case_id, status` | B-tree | KYC detail/readiness. |
| `uk_two_factor_user` | `two_factor_credentials` | `user_id` | UNIQUE | Một credential/user. |
| `ix_recovery_credential_used` | `two_factor_recovery_codes` | `credential_id, used_at` | B-tree | Recovery code check/rotation. |
| `ix_mfa_challenge_user_expiry` | `mfa_challenges` | `user_id, purpose, expires_at, verified_at` | B-tree | Admin 2FA challenge. |
| `ix_login_identifier_time` | `login_attempts` | `identifier_hash, occurred_at` | B-tree | 5 failures/15m. |
| `ix_login_user_time` | `login_attempts` | `user_id, occurred_at` | B-tree | Account security audit. |
| `uk_audit_event_id` | `audit_logs` | `event_id` | UNIQUE | Dedupe audit. |
| `ix_audit_target_time` | `audit_logs` | `target_type, target_id, occurred_at` | B-tree | Admin audit detail. |
| `ix_audit_actor_time` | `audit_logs` | `actor_user_id, occurred_at` | B-tree | Admin audit query. |
| `ix_outbox_pending` | `outbox_events` | `published_at, failed_at, next_retry_at, created_at` | B-tree | Publisher polling. |
| `ix_outbox_aggregate` | `outbox_events` | `aggregate_type, aggregate_id, created_at` | B-tree | Replay/order per aggregate. |

Không tạo index trên raw JSON `warehouse_snapshot`, `metadata` hoặc encrypted ciphertext nếu không có truy vấn thật.

## 5. Enum & quy tắc dữ liệu

### 5.1 Enum lưu trong database

| Enum | Giá trị |
|---|---|
| `UserStatus` | `ACTIVE`, `LOCKED`, `SUSPENDED`, `DELETED` |
| `ShopStatus` | `DRAFT`, `ACTIVE`, `SUSPENDED`, `DELETED` |
| `KycStatus` | `DRAFT`, `PENDING`, `NEEDS_INFO`, `APPROVED`, `REJECTED`, `EXPIRED`, `SUSPENDED` |
| `OnboardingStep` | `PROFILE`, `KYC`, `WAREHOUSE`, `BANK`, `FIRST_PRODUCT`, `COMPLETED` |
| `VerificationChannel` | `EMAIL`, `PHONE` |
| `VerificationPurpose` | `EMAIL_VERIFY`, `PHONE_VERIFY` |
| `TokenRevokeReason` | `ROTATED`, `SIGNOUT`, `RESET`, `REUSE`, `SUSPEND`, `EXPIRED` |
| `ShopStaffStatus` | `INVITED`, `ACTIVE`, `REVOKED` |
| `KycDocumentStatus` | `UPLOADING`, `UPLOADED`, `VERIFIED`, `REJECTED`, `EXPIRED`, `DELETED` |
| `TwoFactorStatus` | `DISABLED`, `ENROLLING`, `ENABLED`, `RESET_REQUIRED` |
| `MfaPurpose` | `LOGIN`, `STEP_UP`, `ENROLL` |

### 5.2 Quy tắc nghiệp vụ không diễn tả hoàn toàn bằng schema

| Quy tắc | Kiểm ở đâu |
|---|---|
| Email required, normalized và unique | DTO validator + transaction trước insert. |
| Phone optional, E.164 và unique non-null | DTO validator + DB unique. |
| Password 12–72 ký tự, Argon2id | Application service trước hash. |
| 5 login failures trong 15 phút khóa 15 phút | Transaction/row lock trên `users` + `login_attempts`. |
| Access token 15m, refresh token 30d, rotation bắt buộc | Token service + `refresh_tokens`. |
| Refresh token reuse revoke toàn bộ family | Transaction lock theo `family_id` + audit. |
| Tối đa 20 address sống/user, một default | Address service transaction. |
| Một user tối đa một onboarding/active shop v1 | Lock owner user + `shops.owner_user_id/status`. |
| KYC decision NEEDS_INFO/REJECTED cần reason 10–1000 | Application validator + audit. |
| Publish/withdraw gate KYC APPROVED | Product/Payment service; auth-user phát event/status. |
| Role scope system dùng NULL, shop role dùng shop ID | RBAC service + DB check `shop_id` FK khi khác NULL. |
| Admin mutation nhạy cảm phải có 2FA step-up 5 phút | Auth guard + `mfa_challenges`/TOTP. |
| KYC file tối đa 10 MiB, PDF/JPG/PNG | Presign/complete adapter + document constraints. |
| Audit không chứa secret/OTP/password/bank raw | Application redaction + code review/test. |
| Outbox payload không chứa password/token/KYC bytes | Event mapper + contract test. |

### 5.3 Cross-service reference policy

| Field | Nguồn ngoài | Cách lưu | Không được làm |
|---|---|---|---|
| `product_id` trong onboarding flag/event metadata | Product Catalog | `BINARY(16)` hoặc JSON ID | Không FK, không query product DB. |
| `order_id`/`payment_id` trong audit metadata | Order/Payment | JSON masked reference nếu cần | Không join/query runtime. |
| `shop_id` ở Product/Payment/Order | Auth User | ID + event projection ở service đó | Không tạo FK từ database khác. |

## 6. Migration & seed

### 6.1 Thứ tự migration

| Thứ tự | Nội dung | Phụ thuộc |
|---:|---|---|
| 001 | Tạo database `userdb`, charset/collation và migration metadata | — |
| 002 | Tạo `users` | — |
| 003 | Tạo `roles`, `permissions`, `role_permissions` | — |
| 004 | Tạo `addresses` | `users` |
| 005 | Tạo `shops`, `seller_onboarding` | `users` |
| 006 | Tạo `user_roles`, `shop_staff` | `users`, `roles`, `shops` |
| 007 | Tạo `refresh_tokens`, `verification_tokens`, `password_reset_tokens` | `users` |
| 008 | Tạo `kyc_cases`, `kyc_documents` | `shops` |
| 009 | Tạo `two_factor_credentials`, `two_factor_recovery_codes`, `mfa_challenges` | `users` |
| 010 | Tạo `login_attempts`, `audit_logs`, `outbox_events` | `users` |
| 011 | Thêm CHECK constraints, indexes và scheduled cleanup metadata | Tất cả bảng liên quan |
| 012 | Seed RBAC baseline và permission matrix | `roles`, `permissions` |

Mỗi migration phải có `up` và `down` cho local/test. Production rollback destructive phải có migration kế tiếp hoặc backup plan; không drop bảng trực tiếp khi có dữ liệu thật.

### 6.2 Seed bắt buộc

| Nhóm | Dữ liệu |
|---|---|
| User roles | `BUYER`, `SELLER`, `SELLER_STAFF`. |
| Admin roles | `SUPER_ADMIN`, `RISK_MANAGER`, `CATALOG_ADMIN`, `FINANCE_OPS`, `SUPPORT_VIEWER`. |
| KYC permissions | `KYC_READ`, `KYC_DECIDE`, `KYC_REQUEST_INFO`. |
| User permissions | `USER_READ`, `USER_SUSPEND`, `ROLE_READ`, `ROLE_ASSIGN`. |
| Seller permissions | `SHOP_READ`, `SHOP_UPDATE`, `SELLER_STAFF_MANAGE`. |
| Seed admin account | Không seed password cố định; bootstrap job tạo invite/reset flow và bắt buộc bật 2FA. |
| Feature/config | Không seed secret, private key, S3 credential, Kafka credential hoặc provider API key. |

### 6.3 Cleanup/retention job

| Job | Lịch | Xử lý |
|---|---|---|
| `unlock_accounts` | Mỗi 5 phút | Reset `LOCKED → ACTIVE` khi `locked_until` đã qua, trừ account bị suspend. |
| `expire_verification_tokens` | Mỗi giờ | Mark expired qua query; không log token. |
| `expire_refresh_tokens` | Hàng ngày | Xóa/revoke token quá hạn theo retention đã duyệt. |
| `expire_kyc_documents` | `00:05 UTC` mỗi ngày | Mark document/case expired, phát event idempotent. |
| `outbox_reaper` | Mỗi phút | Retry pending theo `next_retry_at`; sau 3 lần đưa DLQ. |
| `audit_archive` | Hàng ngày | Archive theo retention 365 ngày, không xóa audit đang cần compliance. |
| `login_attempts_archive` | Hàng ngày | Partition/archive theo policy security, giữ aggregate cần thiết. |

## 7. Giả định & câu hỏi mở

| # | Nội dung | Ảnh hưởng nếu sai | Cần ai xác nhận |
|---|---|---|---|
| 1 | MySQL 8.4 và InnoDB được dùng cho `userdb`; deployment phải hỗ trợ transaction/row lock. | Ảnh hưởng UUID binary, migration và atomic default-address/KYC state. | Tech lead/DevOps |
| 2 | UUIDv7 lưu `BINARY(16)`, API serialize canonical UUID string. | Nếu team muốn `CHAR(36)`, toàn bộ index/DTO/migration phải đổi. | Backend lead |
| 3 | `tax_code`, `slug`, email và phone unique theo phạm vi được mô tả; soft-deleted email/phone vẫn giữ unique trong v1. | Ảnh hưởng account recreation và seller resubmission. | Product owner |
| 4 | Bank account đầy đủ và TOTP secret mã hóa application-level bằng KMS; key provider/rotation chưa chọn. | Ảnh hưởng deployment secret, migration và recovery. | Security/DevOps |
| 5 | `warehouse_snapshot`/`blockers_json` dùng JSON vì không phải query-critical; fields filter thường xuyên phải tách cột sau khi có metrics. | Nếu cần report theo field JSON, schema/index phải bổ sung. | Backend lead |
| 6 | `kyc_cases` chỉ enforce một case hiện hành bằng transaction/application lock, chưa dùng partial unique index. | Concurrent submit/review cần integration test và row lock đúng. | Backend lead |
| 7 | `mfa_challenges` lưu MySQL thay vì Redis vì HLD chưa chốt Redis cho auth-user. | Có thể tăng DB write; đổi sang Redis sẽ cần TTL/HA contract. | Architecture owner |
| 8 | Audit retention baseline 365 ngày; compliance retention dài hơn cần partition/archive policy riêng. | Ảnh hưởng storage cost và legal hold. | Security/Compliance |
| 9 | KYC object storage dùng private S3/MinIO, max 10 MiB/file và signed URL 10 phút; virus scan chưa có provider. | Ảnh hưởng trạng thái `SCANNING` và publish readiness. | Security/DevOps |
