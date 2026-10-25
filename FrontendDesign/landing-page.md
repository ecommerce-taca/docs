# Frontend Design — Landing page & public entry routing

> Cập nhật: `2026-10-25` · Trạng thái: **Đề xuất (chờ review)**
> Liên quan: `HighLevelDesign/HLD.md` (khối Micro-Frontend: `mfe-shell`, `mfe-catalog`, `mfe-buyer`, `mfe-seller`), `FrontendDesign/Taca.penpot.zip` (`Taca Buyer / Home`, `Overlay / Sign in`, `Overlay / Seller onboarding`), `auth-user-docs/docs/lld/auth-user.md`, `api-gateway-docs/docs/lld/api-gateway.md`.

## 1. Kết luận

**Có — Taca cần landing page public**, nhưng tách theo đúng đối tượng:

| Đối tượng | Trang vào đầu tiên | Lý do |
|---|---|---|
| Buyer | `/` = **storefront Home** (`Taca Buyer / Home` đã có trong Penpot: Hero, Flash sale, Featured categories, Official shops, campaign) | Marketplace kiểu Tiki: buyer xem/tìm sản phẩm **không cần đăng nhập** (HLD §2 "Buyer browsing (no login required)"). Đặt một trang giới thiệu chắn trước cửa hàng chỉ thêm 1 bước và giảm conversion. Home chính là trang marketing cho buyer. |
| Seller tiềm năng | `/sell` = **landing "Bán hàng cùng Taca"** (chưa có trong Penpot) | Seller cần được thuyết phục (phí, quy trình KYC, công cụ Seller Center) trước khi đăng ký. Hiện chỉ có `Overlay / Seller onboarding` — yêu cầu đã đăng nhập, không có trang nào giải thích *vì sao* mở shop. |
| Mọi người / bot | `/about`, `/help/**`, `/policies/**` (điều khoản, bảo mật, đổi trả) | Nội dung tĩnh, cần SEO và link được từ footer/email. |

Gọi chung nhóm `/sell`, `/about`, `/help/**`, `/policies/**` là **marketing pages**.

## 2. Đặt ở đâu

### 2.1 Các lựa chọn

| Tiêu chí | A. Route trong `mfe-shell` | B. Site tĩnh riêng `web-landing` (SSG) trên CDN, cùng domain | C. SSR server (Next.js) cho toàn bộ public |
|---|---|---|---|
| SEO | Kém: SPA client-render, crawler nhận HTML rỗng cho tới khi shell + remote tải xong | Tốt: HTML đầy đủ, meta/OG/JSON-LD có sẵn | Tốt |
| Hiệu năng (LCP) | Phải tải React + Module Federation runtime + shell trước khi thấy nội dung | Nhanh nhất: HTML + CSS từ CDN, JS gần như bằng 0 | Tốt nhưng phụ thuộc server, có TTFB |
| Vận hành | Không thêm gì | Thêm 1 pipeline build tĩnh + 1 rule CDN | Thêm 1 runtime Node phải scale/giám sát |
| Ai sửa nội dung | Dev phải deploy shell (ảnh hưởng toàn app) | Deploy độc lập, không đụng app | Deploy độc lập |
| Rủi ro với MFE | Đẩy code marketing vào host chung, tăng bundle cho mọi user | Tách biệt hoàn toàn | Phải chia sẻ design-system qua 2 cách render |

### 2.2 Chọn **B**

- Marketing pages build thành HTML tĩnh (đề xuất Astro hoặc Next.js `output: export`), dùng chung **design tokens** của `design-system` (cùng package đã đánh dấu `singleton` trong Module Federation) để giao diện đồng nhất với app, nhưng **không** tải React/MFE runtime trừ khi một island thực sự cần.
- Deploy lên S3/MinIO + CDN, **cùng domain** với app (không dùng subdomain) để: chia sẻ SEO authority, không phải mở thêm origin trong CORS allowlist của Gateway, link nội bộ không bị coi là cross-site.
- `mfe-shell` **không** chứa code marketing; chỉ chứa header/footer có link tới marketing pages.

### 2.3 Còn lại: SEO của storefront

`/`, `/c/**` (category), `/p/**` (product), `/shop/**` vẫn do `mfe-shell` + `mfe-catalog` render phía client — cũng là những trang SEO quan trọng nhất của marketplace. Ngoài phạm vi tài liệu này, nhưng ghi lại để không bị quên:

- v1: `index.html` của shell có `<title>`, meta description, OG tag mặc định và skeleton hero tĩnh để LCP không phải chờ remote.
- v2: prerender/SSR cho category và product detail (quyết định riêng, cần Product Catalog/Search cung cấp dữ liệu cho build hoặc edge).

## 3. Routing

### 3.1 Phân luồng tại CDN

```text
Browser ──► CDN (taca.vn)
             ├─ /sell, /sell/**        ─► web-landing (HTML tĩnh)
             ├─ /about, /help/**,
             │  /policies/**           ─► web-landing (HTML tĩnh)
             ├─ /api/v1/**, /ws/**     ─► API Gateway (Kong)
             ├─ /assets/**, remoteEntry─► static bucket của từng MFE
             └─ mọi path khác          ─► mfe-shell index.html (SPA fallback)
                                            ├─ /, /c/**, /p/**, /search, /shop/** → mfe-catalog
                                            ├─ /cart, /checkout, /account/**      → mfe-buyer
                                            └─ /seller/**                          → mfe-seller
```

Danh sách prefix của `web-landing` là **allowlist cố định** trong cấu hình CDN; thêm trang marketing mới dưới các prefix trên không cần đổi CDN.

### 3.2 Landing không giữ auth

Toàn bộ trạng thái đăng nhập (access token 15 phút, refresh token 30 ngày theo `auth-user` LLD) chỉ do `mfe-shell` quản lý. Marketing pages là HTML tĩnh, **không** đọc token và **không** gọi API trong v1. Vì vậy mọi CTA trỏ về một route của shell, và shell quyết định theo trạng thái thật:

| CTA trên landing | Link | Shell xử lý |
|---|---|---|
| "Đăng nhập" | `/signin?next=<path>` | Mở `Overlay / Sign in`; thành công → `next` (mặc định `/`) |
| "Đăng ký" | `/signup?next=<path>` | Mở overlay đăng ký; thành công → `next` |
| "Mua sắm ngay" | `/` | Storefront Home |
| "Bắt đầu bán hàng" (CTA chính của `/sell`) | `/seller/start` | Chưa đăng nhập → `/signup?next=/seller/start` · Buyer chưa có shop → `Overlay / Seller onboarding` · Đã có shop → `/seller` (Seller Center Dashboard) |

`/signin`, `/signup`, `/seller/start` là route mới của `mfe-shell` (deep-link được, không chỉ là overlay mở bằng nút).

### 3.3 Quy tắc bảo mật cho `next`

- Chỉ chấp nhận đường dẫn tương đối bắt đầu bằng một `/` (không `//`, không `\`, không scheme) — chặn open redirect.
- Path không hợp lệ → bỏ qua, về `/`.
- Giữ nguyên query `utm_*` khi chuyển từ landing sang shell để đo conversion theo campaign.

## 4. Nội dung `/sell` — "Bán hàng cùng Taca"

Thứ tự section, trên xuống:

1. **Header tối giản**: logo `taca`, link "Mua sắm" (`/`), "Đăng nhập", CTA "Bắt đầu bán hàng". Không dùng `Header / Search row` của buyer — tránh kéo người đọc đi chỗ khác.
2. **Hero**: tiêu đề giá trị chính (tiếp cận buyer toàn quốc), 1 câu phụ, CTA chính + CTA phụ "Xem biểu phí". Ảnh Seller Center Dashboard thật (`Taca Seller / Dashboard`).
3. **Số liệu tin cậy**: số shop, số buyer, số đơn/tháng. Chỉ hiển thị khi có số thật; build-time inject, không gọi API runtime.
4. **Vì sao bán trên Taca**: 3–4 thẻ — tiếp cận buyer toàn quốc, thanh toán VNPAY QR và COD (HLD), vận chuyển tích hợp (Shipment), voucher shop (Seller Vouchers).
5. **Quy trình 5 bước**: khớp `Overlay / Seller onboarding` — Hồ sơ → KYC → Kho/vận chuyển → Ngân hàng → Sản phẩm đầu tiên. Ghi rõ: tạo shop và vào Seller Center được ngay; KYC chỉ cần để **publish** sản phẩm và **rút tiền** (chính sách trong `auth-user` LLD).
6. **Biểu phí**: commission, thuế, chu kỳ đối soát và rút tiền (Payment Wallet). Bảng rõ ràng, không "liên hệ để biết".
7. **Công cụ Seller Center**: Products, Orders, Finance, Vouchers, Messaging — ảnh chụp từ các màn `Taca Seller / *` đã có.
8. **FAQ**: giấy tờ KYC cần gì, bao lâu được duyệt, phí, rút tiền mấy ngày, có cần đăng ký kinh doanh không. Đánh dấu `FAQPage` JSON-LD.
9. **CTA cuối trang**: lặp lại "Bắt đầu bán hàng".
10. **Footer chung** (xem §5).

## 5. Thay đổi trên storefront & shell

- **Footer** chung của shell và landing: About, Help, Policies, "Bán hàng cùng Taca" (`/sell`), tải app (Android/iOS theo NFR), mạng xã hội.
- **Header buyer**: thêm link "Kênh người bán" → `/sell` cho user chưa có shop, → `/seller` cho seller.
- **Home** (`Taca Buyer / Home`) giữ nguyên vai trò trang chủ; có thể thêm 1 banner "Mở shop trên Taca" trỏ `/sell`.

## 6. Yêu cầu phi chức năng

| Mục | Mục tiêu |
|---|---|
| Hiệu năng | LCP ≤ 2.5s (4G, mobile), JS ≤ 30 KB gzip cho `/sell`, ảnh WebP/AVIF có `srcset` |
| SEO | `<title>`/meta description riêng từng trang, canonical, OG/Twitter card, `sitemap.xml` (gộp marketing + catalog), `robots.txt` chặn `/cart`, `/checkout`, `/account/**`, `/seller/**` |
| Responsive | Bắt buộc theo NFR "Must have responsive for web version"; thiết kế mobile trước |
| Ngôn ngữ | Tiếng Việt v1; cấu trúc path để sau thêm `/en/**` không phá link |
| Analytics | Sự kiện `landing_view`, `cta_click{cta, page}`, `seller_start{auth_state}`; giữ `utm_*` qua `next` |
| Accessibility | WCAG 2.1 AA: tương phản, focus, alt text |

## 7. Việc cần làm

| # | Việc | Owner |
|---|---|---|
| 1 | Thiết kế `Taca Marketing / Sell` (desktop + mobile) và footer chung trong Penpot | Design |
| 2 | Tạo repo/pipeline `web-landing` (SSG) dùng design tokens chung | Frontend |
| 3 | Thêm route `/signin`, `/signup`, `/seller/start` + validate `next` trong `mfe-shell` | Frontend |
| 4 | Rule CDN cho prefix marketing + SPA fallback | Platform |
| 5 | Nội dung biểu phí và FAQ (số thật) | Product owner |

## 8. Câu hỏi mở

| # | Câu hỏi | Đề xuất mặc định |
|---|---|---|
| 1 | Có cần landing riêng cho buyer (ví dụ trang giới thiệu app mobile) không? | Không trong v1; dùng banner trên Home |
| 2 | SSG framework: Astro hay Next.js export? | Astro (mặc định 0 JS, phù hợp trang nội dung) |
| 3 | Thời điểm SSR/prerender cho category/product | Quyết định riêng sau khi có số liệu traffic organic |
