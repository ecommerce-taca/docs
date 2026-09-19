# High-Level Design — Marketplace Platform (Tiki-style)

> **Bản dịch nguyên trạng (verbatim transcription)** từ `docs/HighLevelDesign/EcommercePlatform-v4.excalidraw` sang Markdown, để tiện đọc/diff/tìm kiếm.
> Không biên tập nội dung, không lọc trùng lặp, không sửa mâu thuẫn nội tại của bản vẽ gốc (ví dụ: file gốc có 2 lớp mô tả DB schema khác nhau — "7.1 Per-Service Database Schemas (Deep Dive)" và "9. Database Schema per Service (Final 11-service design)" — cả hai đều được giữ nguyên bên dưới dù có thể lệch nhau).
> Canvas gốc là sơ đồ 2D (nhiều cột cạnh nhau: sequence diagram bên phải song song với outline yêu cầu/API bên trái, rồi tới khối kiến trúc/DB deep-dive, rồi khối DB schema cuối). Markdown là tuyến tính nên tài liệu này nhóm nội dung thành 3 phần theo layout thật của canvas (xem ghi chú đầu mỗi phần) và sắp xếp trong mỗi phần theo thứ tự trên-xuống, trái-qua-phải — **thứ tự đọc có thể không phản ánh đúng 100% quan hệ trực quan** (mũi tên, vị trí cạnh nhau) của bản vẽ gốc. Muốn xem chính xác bố cục/mũi tên, mở file `.excalidraw` bằng https://excalidraw.com.
> **Tài liệu chốt/nguồn đúng của hệ thống là `docs/<service>-docs/docs/{api,db,lld,test}/*.md`** (44 file, 11 service) — HLD (cả bản excalidraw lẫn bản dịch `.md` này) là bản phác thảo ban đầu, một số nội dung đã lỗi thời so với các doc đó (xem `reports/no-task-docs-consistency/2026-09-19-hld-excalidraw-consistency.md` và `plans/DOCS-CONSISTENCY-01-cdc-hld.md`).

---

## Phần A — Sequence Diagram: Marketplace Platform (Tiki-style) — End-to-End

> Cột bên phải của canvas (Buyer/User · System/Platform · Payment Gateway · Shop/Seller làm 4 "swimlane"), nằm song song về mặt bố cục với outline yêu cầu/API ở Phần B, không phải phần tiếp nối tuần tự của Phần B.

**Buyer/User**

**System/Platform**

**Payment Gateway**

**Shop/Seller**

### 1. Seller onboarding & product management

**Sign up + business info**

**Shop created, confirmation**

**Sign in**

**Auth token**

**Create product (CRUD, publish/unpublish)**

**Confirmation**

### 2. Buyer browsing (no login required)

**Search / filter products by keyword, category, price**

**Product list**

**Get product detail / view shop**

**Product info + images**

### 3. Buyer auth & cart management

**Sign up**

**Account created**

**Sign in**

**Auth token**

**Add to cart (item, qty) [must be signed in]**

**Cart updated**

**Update quantity / remove item**

**Cart updated**

### 4. Checkout & payment

**Select item(s) in cart to order**

**Order summary (selected items)**

**Apply voucher code**

**Cost breakdown (subtotal, discount, shipping, total)**

**Confirm order (address, shipping, payment method)**

**Create payment request (if VNPAY selected)**

**Return VNPAY QR code**

**Display VNPAY QR code to user**

**Scan QR & pay via VNPAY app**

**Submit payment proof / confirm payment**

**Payment noted (pending confirmation)**

**Payment webhook callback (auto-confirm)**

**Payment auto-confirmed**

**New order notification**

**Order accepted**

**Order confirmation + invoice**

**Email notification: order success + invoice**

### 5. Order tracking, shipping & cancellation

**Get order status**

**Order status**

Create shipment, get
tracking code

**Shipment created, tracking code**

**Shipping status updates (picked up / in transit / delivered)**

**Cancel order (must be before shipping)**

**Cancel order notice**

**Order canceled**

**Cancellation confirmed**

### 6. Review & rating (after delivery)

**Submit rating / review**

**Review saved**

**View comments / ratings**

**Reviews list**

### 7. Seller: orders, revenue & voucher management

**Get own orders / invoices**

**Orders list**

**View revenue (sales, tax, platform commission)**

**Revenue report**

**Manage voucher (CRUD)**

**Voucher saved**

**Withdraw funds request**

**Process withdrawal**

**Withdrawal result**

**Withdrawal confirmation**

**Buyer/User**

**System/Platform**

**Payment Gateway**

**Shop/Seller**

---

## Phần B — Outline: Yêu cầu chức năng, Entity, GUI, API Design, Micro-Frontend (mục 1–5, 10)

**Design Marketplace Platform - Tiki Style -V1**

- Online marketplace where users/buyers purchase from multiple sellers
- Sellers manage own shop / products

Marketplace Platform (Tiki-style) - End-to-End Sequence Diagram

### 1. Functional Requirements — 2 actors: Users/Buyers, Sellers/Merchants/Shop

**Users / Buyers**

**Shop / Merchants / Seller**

• Must sign in, provide detailed business information
• Manage own products and stock [CRUD, publish, unpublish]
• Manage own revenue [include taxes, platform income/commission]
• View comments / ratings as part of buyers
• Withdraw funds from platform
• Cancel order 
• Manage own orders and invoices
• CRUD and manage shop vouchers

• Manage his/her own orders and invoices
• Sign in, sign up, sign out / update information
• Search and filter products by keyword / category / price
• View product details
• Add products to cart, update quantity [must be signed in]
• Checkout: address, shipping, payment method [must be signed in]
• Track order status and cancel before shipping
• Rate / review product after delivery
• Choose and apply voucher of system platform or shop
• Don't need to sign in for search, view product, view shop
• Must sign in to place an order
• Can register to become a seller
. User can payment via VNPAY QR Generate or cash on delivery 

System / platform



• Notify to user email when order success with invoice


### 2. Non-Function Requirement

3. Identify Core Entity


### 4. GUI : Based on Tiki

. user
. product
. cart
. order
. comment
. voucher
. payment
. checkout
. invoice
. notification
. shop
. shipment

. For Buyers : Support many OS and Devices :  Web + Android + Desktop App + IOS
        Optimize user experience + simple user interface
. For Seller/Shop : Web
. Must have responsive for web version
. Maximum automatic

5. API Designing


#### -- Seller: Shop & Product --

#### -- Buyer: Auth & Profile --

```
26.GET/PUT /api/v1/seller/shop -> business profile
27.POST    /api/v1/seller/products {fields} -> productId
28.GET     /api/v1/seller/products -> list own products 
29.PUT     /api/v1/seller/products/{id} -> update  
30.PATCH   /api/v1/seller/products/{id}/status
           {publish|unpublish} 
31.PATCH   /api/v1/seller/products/{id}/stock {qty} 
32.DELETE  /api/v1/seller/products/{id} 
```

```
1. POST /api/v1/auth/signup {name,email,password + user info} -> token
2. POST /api/v1/auth/signin {email,password} -> token
3. POST /api/v1/auth/signout -> 204
4. GET  /api/v1/users/me -> profile
   PUT  /api/v1/users/me {fields} -> updated user
5. POST /api/v1/users/register-seller {shop info} -> shopId
```

#### -- Buyer: Address --

#### -- Seller: Orders --

```
6. GET  /api/v1/users/me/addresses -> list
   POST /api/v1/users/me/addresses {addr} -> addressId
   PUT/DELETE /api/v1/users/me/addresses/{id}
```

```
33.GET   /api/v1/seller/orders -> list orders for shop 
34.GET   /api/v1/seller/orders/{orderId} -> detail  
35.PATCH /api/v1/seller/orders/{orderId}/fulfill
         {status,tracking}  
36.POST  /api/v1/seller/orders/{orderId}/cancel {reason}
         -> result + notify to email for user
37.GET   /api/v1/seller/orders/{orderId}/invoice 
```

#### -- Buyer: Product & Search (public) --

```
7. GET /api/v1/products/search?q=&category=&price= -> List<Product>
8. GET /api/v1/products/{productId} -> details
9. GET /api/v1/categories -> tree 
10.GET /api/v1/shops/{shopId} -> public shop profile 
```

#### -- Seller: Revenue & Wallet --

#### -- Buyer: Cart --

```
38.GET  /api/v1/seller/revenue?range= -> revenue report
        incl. tax/commission breakdown 
39.GET  /api/v1/seller/wallet -> balance
40.POST /api/v1/seller/wallet/withdraw {amount} -> payoutId
```

```
11.GET    /api/v1/cart -> current cart
12.POST   /api/v1/cart/items {productId,qty} -> cartId
13.PUT    /api/v1/cart/items/{itemId} {qty} -> update 
14.DELETE /api/v1/cart/items/{itemId} -> 204  
```

#### -- Seller: Voucher (full CRUD) --

#### -- Buyer: Checkout, Voucher, Payment --

```
41.POST   /api/v1/seller/vouchers {code,discount} -> voucherId
42.GET    /api/v1/seller/vouchers -> list  
43.PUT    /api/v1/seller/vouchers/{id} -> update  
44.DELETE /api/v1/seller/vouchers/{id} 
```

```
15.GET  /api/v1/vouchers/validate?code=&cartId= -> discount info
15b. GET /api/v1/checkout/shipping-fee?fromShopId=&toAddressId=&items[]=
     -> shippingFee
16.POST /api/v1/checkout {cartId,addressId,paymentMethod,
        voucherCode?} -> orderId(s), split per shop
17.POST /api/v1/payments {orderId,paymentDetails} -> confirmation
18.POST /api/v1/payments/webhook (internal) -> reconcile
```

#### -- Buyer: Orders & Review --

```
-- Buyer/Seller: Shipping --
45. GET  /api/v1/orders/{orderId}/shipment -> tracking
status
46. GET  /api/v1/seller/orders/{orderId}/shipment ->
tracking status
47. POST /api/v1/webhooks/shipping/{carrier} -> ack
(internal, no auth JWT)
```

```
19.GET   /api/v1/orders -> list own orders  
20.GET   /api/v1/orders/{orderId} -> detail 
21.GET   /api/v1/orders/{orderId}/status -> status
22.PATCH /api/v1/orders/{orderId}/cancel -> status
23.GET   /api/v1/orders/{orderId}/invoice -> invoice
24.POST  /api/v1/products/{id}/reviews {rating,comment} -> reviewId
        (only allowed if order status = delivered)
25.GET   /api/v1/products/{id}/reviews -> paginated list
```

---

## Phần C — Kiến trúc & Database (mục 6–9): High-Level Design, Deep Dive, Database Schema per Service

**7.1  Per-Service Database Schemas (Deep Dive)**

one DB per service - no cross-service FKs - keys only (full columns in schema doc)

**Comment & Rating Service  →  ratingdb  (MongoDB)**

```
reviews {
  _id, productId, shopId, orderId,
  buyerUserId, rating(1-5), comment, images[],
  isVerifiedPurchase, sellerReply, status }
  uniq(orderId, productId, buyerUserId)
rating_aggregates {
  _id = productId, avg, count,
  distribution{1..5} }   -> Product/Shop cache
```

**Rating DB**

Comment
    and 
Rating  Service

6. High-Level Design


7. DeepDive


**MongoDb**

Comment
    and 
Rating  Service

**User Service  →  userdb  (MySQL)**

```
users(id PK, email UNIQUE, password_hash,
  full_name, phone, role, status,
  email_verified, created_at)
addresses(id PK, user_id, recipient, line1,
  ward, district, province, is_default)
shops(id PK, owner_user_id UNIQUE, name,
  slug UNIQUE, business_name, tax_code,
  logo_url, status, rating_avg*)
refresh_tokens(id PK, user_id, token_hash,
  expires_at, revoked_at)   -> sign-out
```

**Web App Service**

**User Service**

**UserDB**

**MySQL**

**User Service**

**CDN**

**Message Service  →  messagedb  (MongoDB)**

```
conversations {
  _id, buyerUserId, shopId, lastMessageAt,
  lastMsgPreview, unread{buyer, seller} }
messages {
  _id, conversationId, senderType, senderId,
  body, attachments[], readAt, createdAt }
  idx(conversationId, createdAt)
```

**Message Service**

```
MessageDB
(MongoDB)
```

**Product Service  →  productdb  (MongoDB)**

**Product Service**

products {
  _id, shopId, title, slug UNIQUE, description,
  categoryId, categoryPath[], brand, images[],
  attributes{}, variants[{sku, price,
  compareAtPrice}], basePrice, status,
  ratingAvg*, salesCount }
categories {
  _id, name, slug, parentId, ancestors[], level }
  * stock is owned by Inventory, not here

**Clients**

**Clients**

S3
MinIO

S3
MinIO

**Product Service**

**ProductDB**

**MongoDB**

API GATEWAY
 . Authentication
 . Authorization
 . Rate Limit
 . Routing


API GATEWAY
 . Authentication
 . Authorization
 . Rate Limit
 . Routing


**CDC**

**Search Service  →  products index  (Elasticsearch)**

```
title       text (analyzed)   full-text
shopId      keyword           filter
categoryPath keyword[]        facet+filter
brand       keyword           facet
price       long              range / sort
ratingAvg   float             sort
status      keyword           only PUBLISHED
attributes  flattened         facets
suggest     completion        autocomplete
Fed by CDC from Product (product.* -> Kafka)
```

**Clients**

**Clients**

**Search Service**

**Elastic Search**

**Search Service**

**CDC**

**Order Service  →  orderdb  (MySQL) + Cart in Redis**

**Clients**

```
carts(id, user_id UNIQUE, status)
cart_items(id, cart_id, product_id, sku,
  shop_id, qty, unit_price_snap)
orders(id PK, order_number UNIQUE, buyer_user_id,
  shop_id, status, subtotal, discount,
  shipping_fee, tax, grand_total, payment_method,
  address_snapshot JSON, placed_at)
order_items(id, order_id, sku, title_snap,
  unit_price, qty, line_total)
invoices(id, order_id UNIQUE, invoice_no, pdf_url)
vouchers(id, code, scope, shop_id, type, value,
  min_order, usage_limit, used_count)
voucher_redemptions(id, voucher_id, order_id,
  user_id)   uniq(voucher, user)
outbox(id, aggregate, event_type, payload,
  published_at)        <- reliable events
idempotency_keys(key PK, user_id, response_snap,
  expires_at)          <- safe retries
```

**Clients**

**OrderDB**

**order services**

**(Cart+Checkout+Invoice+Voucher)**

**Inventory Service**

**Order Service**

**Inventory DB**

**Clients**

**Clients**

**OrderDB**

**Inventory Service  →  inventorydb  (MySQL)**

```
inventory_items(id, sku UNIQUE, shop_id,
  product_id, qty_available, qty_reserved,
  version)             <- optimistic lock (no oversell)
stock_reservations(id, sku, order_id, qty,
  status[RESERVED|COMMITTED|RELEASED|EXPIRED],
  expires_at)
stock_movements(id, sku, delta, reason, ref_type,
  ref_id, created_at)  <- audit ledger
```

**Shipping Service**

**Payment Service**

**Check inventory before checkout**

**Shipment Service  →  shipmentdb  (MySQL)**

```
shipments(id, order_id, shop_id, buyer_user_id,
  carrier, tracking_code UNIQUE, status,
  from_addr_snap, to_addr_snap, fee)
shipment_events(id, shipment_id, status,
  description, occurred_at, source)  tracking timeline
carrier_webhook_log(id, carrier,
  external_event_id UNIQUE, shipment_id)
```

**Implementation**

**Web App Service**

Shipment
Services

**INVENTORY CONSUMER**

**Payment Service  →  paymentdb  (MySQL)**

**ORDER CONSUMER**

```
payments(id, buyer_user_id, method[VNPAY|COD],
  amount, status, provider_txn_ref,
  idempotency_key UNIQUE)
payment_allocations(id, payment_id, order_id,
  shop_id, gross, commission, tax, seller_net)
  <- marketplace split + commission
payment_events(id, payment_id,
  provider_event_id UNIQUE, raw_payload)
  <- webhook idempotency
wallets(id, shop_id UNIQUE, avail, pending)
ledger_entries(id, wallet_id, type[CR|DR], amount,
  balance_after, ref)  <- double-entry
payouts(id, shop_id, amount, status,
  bank_acct_snap, idempotency_key)
refunds(id, payment_id, order_id, amount)
```

Notification 
    Service

**PaymentDB**

**Payment Gateway**

**Shipment DB**

Shipment
Consumer

```
mfe-shell
(host container)
auth, routing, layout
```

**KAFKA**

**Browser**

**Notification Service  →  notificationdb  (MySQL) -> EMAIL**

```
notifications(id, user_id, channel[EMAIL|IN_APP],
  template, payload,
  status[QUEUED|PROCESSING|SENT|FAILED|SKIPPED|EXPIRED],
  ref_type, ref_id, sent_at)
Consumes Kafka:
  order.confirmed -> order + invoice email
  (KHONG dung order.paid cho email nay - COD toi sau khi giao)
  notification.commands.v1 -> AUTH_VERIFICATION_REQUESTED,
  PASSWORD_RESET_REQUESTED, PHONE_OTP_REQUESTED,
  MESSAGE_RECEIVED, REVIEW_REQUESTED (review prompt)
```

**Shipment DB**

Notification 
    Service

**EMAIL**

**PaymentDB**

**Payment Service**

```
mfe-catalog
(browse, search, product
detail, ratings)
```

```
mfe-buyer
(cart, checkout, orders,
account, messages)
```

```
mfe-seller
(shop dashboard: products,
orders, revenue, shipment)
```

```
calls (via Gateway):
- Product Service
- Search Service
- Rating & Comment
Service
```

```
calls (via Gateway):
- Order Service
- User Service
- Payment Service
- Message Service
```

```
calls (via Gateway):
- User Service (Shop)
- Product Service
- Order Service
- Inventory Service
- Shipment Service
```

**Payment Gateway**

```
Module Federation wiring:
mfe-shell webpack.config remotes: { catalog, buyer, seller } -> each remote's remoteEntry.js
Each remote exposes top-level route components; shell lazy-loads by route
Shared deps (react, react-dom, design-system) marked singleton:true in ModuleFederationPlugin
```
