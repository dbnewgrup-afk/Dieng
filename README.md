# Scope Audit: Booking-Villa vs dieng scope.docx

Dokumen ini membandingkan kondisi repo saat ini dengan scope pada file:
`C:\Users\IQBAL\Documents\dieng scope.docx`

Audit dilakukan pada 2026-04-09 dengan cara:
- membaca isi scope Word
- mengecek struktur folder repo
- mengecek file inti API, web public, web admin, dan schema database

## Ringkasan Singkat

Kesimpulan cepat: repo ini belum sepenuhnya sesuai dengan scope Word.

Yang sudah terlihat ada:
- monorepo dengan pemisahan `web-public`, `web-admin`, dan `api`
- frontend Next.js untuk public site dan beberapa dashboard internal
- backend API berbasis Express + Prisma + PostgreSQL
- auth dasar (`login`, `refresh`, `me`)
- produk/order/payment dasar
- webhook payment Midtrans
- audit log dasar
- i18n dasar di web public

Yang paling berbeda dari scope:
- scope minta `Laravel API`, repo ini memakai `Express + TypeScript + Prisma`
- scope minta `Xendit`, repo ini memakai `Midtrans`
- scope minta marketplace `multi-vendor seller`, repo saat ini masih dominan model `product/order` umum
- scope minta `review`, `availability`, `anti double booking`, dan disbursement payout final; area ini masih belum lengkap
- banyak halaman admin/partner/super-admin masih memakai `MOCK_*` data atau `localStorage`

Secara implementasi bisnis, repo ini baru cocok sebagai fondasi awal dan belum match penuh dengan scope produksi marketplace booking pada dokumen Word.

## Update Progress 2026-04-09

Update terbaru setelah audit dan dokumen roadmap/schema:

- schema Prisma additive untuk domain baru sudah dibuat di `Booking`, `Listing`, `SellerProfile`, `ListingAvailability`, `Wallet`, `WalletTransaction`, `Withdraw`, dan `Review`
- Prisma Client sudah di-regenerate mengikuti schema baru
- baseline compile API sekarang sudah hijau lagi dengan `npm run build` di `Booking-Villa/apps/api`
- route compatibility lama `/api/products` sekarang sudah diarahkan ke model `Listing`
- route compatibility lama `/api/orders` sekarang sudah diarahkan ke model `Booking`
- flow `mark-cash` sekarang sudah menulis ke `Payment` provider `MANUAL` dan update status `Booking`
- webhook `/api/webhooks/midtrans` sekarang sudah sinkron ke `Booking` dan `Payment` model baru
- helper auth/JWT/RBAC sudah diselaraskan dengan role baru termasuk `SELLER`
- endpoint partner `balance`, `payouts`, `requests`, dan `reports/partner/me` sekarang sudah baca data nyata dari `Wallet`, `Withdraw`, `Booking`, dan `Listing`
- admin finance `/api/payouts/batches` sekarang sudah ter-mount dan tidak lagi hanya `Not Implemented`
- batch payout admin sekarang memakai model `Withdraw` + `WalletTransaction.referenceCode` sebagai batch code sintetis tanpa perlu schema baru
- flow create payout batch admin sekarang membuat item withdraw nyata per seller
- flow approve payout batch admin sekarang mengubah `Withdraw` dari `PENDING` ke `APPROVED`
- settlement booking ke wallet seller sekarang sudah mulai hidup secara idempotent melalui ledger `BOOKING_IN`
- flow `mark-cash` dan webhook `/api/webhooks/midtrans` sekarang sudah mendorong dana booking ke `balancePending`
- endpoint `POST /api/orders/:code/complete` sekarang sudah menandai booking `COMPLETED` sekaligus release escrow ke `balanceAvailable` lewat ledger `ESCROW_RELEASE`
- endpoint `GET /api/orders` untuk dashboard admin sekarang sudah hidup
- UI admin order table sudah mulai disambungkan ke action `Complete`, tapi build `web-admin` belum tervalidasi penuh karena `next build` sempat ngehang
- endpoint `GET /api/partner/bookings` sekarang sudah hidup untuk baca booking seller dari domain `Booking`
- helper fetch partner di `web-admin` sekarang sudah pakai `NEXT_PUBLIC_API_BASE_URL` + bearer token, jadi tidak lagi mengandalkan path relatif yang rawan fallback kosong
- tab report partner sekarang sudah membaca booking nyata dari backend untuk agregasi per listing/hari/minggu/bulan
- baseline compile API tetap hijau lagi setelah penambahan route partner dan payout batch
- baseline compile API tetap hijau setelah penambahan helper settlement booking + complete order endpoint

Arti update ini:
- kita sudah lewat fase "schema doang"
- backend mulai punya compatibility layer nyata dari domain lama `Product/Order` ke domain baru `Listing/Booking`
- dashboard partner sudah mulai bisa hidup dari data backend nyata untuk area saldo, request withdraw, payout history, dan summary
- dashboard partner report sekarang juga mulai hidup dari booking backend nyata, walau request produk masih mock
- finance domain sudah mulai punya flow escrow nyata dari `paid -> pending -> available`
- admin payout sudah mulai punya backend nyata, walau masih tahap approval internal dan belum sampai disbursement provider
- tapi booking availability lock, refund reversal, payout disbursement final, dan test end-to-end masih belum selesai

### Lanjutan Paling Masuk Akal Setelah Update Ini

- sambungkan approval payout ke disbursement provider final (`Xendit` bila mengikuti scope)
- sambungkan UI partner ke flow completion / settlement yang baru
- tambahkan reversal ledger untuk kasus refund/cancel setelah dana sempat masuk escrow
- tambahkan worker/cron untuk booking expiry dan finance settlement
- kurangi fallback mock di `web-admin` yang masih belum memakai data backend nyata
- tambahkan integration test untuk partner withdraw flow dan admin payout batch flow
- rapikan `web-admin` build/lint karena `next build` sempat ngehang lama dan `npm run lint` gagal karena belum ada `eslint.config.*`

## Struktur Folder Saat Ini

Root repo ini tipis. Aplikasi utamanya ada di folder `Booking-Villa/`.

Struktur penting:

```text
villa/
|- README.md
|- package-lock.json
`- Booking-Villa/
   |- apps/
   |  |- api/
   |  |- web-admin/
   |  `- web-public/
   |- infra/
   |  |- cdn/
   |  |- k8s/
   |  `- terraform/
   `- packages/
      |- client/
      |- contracts/
      |- ui-kit/
      `- utils/
```

Makna struktur:
- `apps/api`: backend Express + Prisma
- `apps/web-public`: website user/public
- `apps/web-admin`: dashboard admin, kasir, partner, affiliate, super admin
- `infra`: arah deployment/infrastruktur sudah mulai disiapkan
- `packages`: shared package untuk monorepo

## Perbandingan Scope vs Kondisi Repo

### 1. Arsitektur

| Scope Word | Kondisi Repo | Status |
| --- | --- | --- |
| Frontend User (Next.js) | Ada `apps/web-public` | Sesuai |
| Seller Panel (Next.js) | Ada UI `partner` dan `affiliate`, tapi belum benar-benar seller-core di backend | Parsial |
| Admin Panel (Next.js) | Ada `apps/web-admin` | Parsial |
| Laravel API | Backend pakai Express + TypeScript + Prisma | Tidak sesuai |
| PostgreSQL | Ada via Prisma | Sesuai |
| Redis (queue, cache, lock) | Tidak ditemukan implementasi Redis aktif | Belum ada |
| Xendit payment/disbursement | Implementasi payment memakai Midtrans | Tidak sesuai |

Catatan:
- Perbedaan stack backend bukan otomatis salah, tapi secara spesifikasi dokumen Word memang belum match.

### 2. Role System

Scope Word:
- guest
- user
- seller
- admin

Kondisi repo:
- `SUPER_ADMIN`
- `ADMIN`
- `KASIR`
- `USER`

Status:
- Parsial

Gap utama:
- muncul role operasional tambahan (`SUPER_ADMIN`, `KASIR`) yang tidak ada di scope Word
- UI partner/affiliate sudah ada, tetapi model otorisasi backend belum menunjukkan seller marketplace yang utuh

### 3. Module System

| Modul Scope | Kondisi Repo | Status |
| --- | --- | --- |
| Auth | Ada login/refresh/me | Parsial |
| Listing multi-category | Ada `Product`, belum `Listing` multi-vendor penuh | Parsial |
| Booking date-based | Ada order + tanggal di FE, tapi belum booking engine yang utuh | Parsial |
| Payment | Ada Midtrans webhook/verify | Parsial |
| Wallet | Sudah ada wallet seller + saldo available/pending | Parsial |
| Escrow | Sudah ada flow `paid -> pending -> available`, reversal belum lengkap | Parsial |
| Withdraw | Sudah ada request/approval batch dasar, disbursement final belum ada | Parsial |
| Review | Schema sudah ada, API dan FE belum final | Parsial |
| Admin moderation/report/finance | Ada sebagian UI, banyak masih mock/placeholder | Parsial |

## ERD Scope vs Schema Prisma Saat Ini

### Entity yang ada di schema saat ini

Schema Prisma sekarang sudah menunjukkan entitas inti berikut:
- `User`
- `SellerProfile`
- `Listing`
- `ListingImage`
- `ListingAvailability`
- `Booking`
- `Payment`
- `Wallet`
- `WalletTransaction`
- `Withdraw`
- `Review`
- `Audit`

### Entity yang diminta di scope tapi belum ada sebagai model inti

- tidak ada gap besar di model inti utama
- gap terbesar sekarang ada di kelengkapan flow dan wiring endpoint/UI, bukan di ketiadaan tabel inti

### Gap field penting

#### User
Scope minta field seperti:
- `name`
- `phone`
- `avatar`
- `status`
- `email_verified_at`

Schema saat ini hanya terlihat aman di area dasar:
- `id`
- `email`
- `password`
- `role`
- timestamp

#### Listing/Product
Scope minta konsep listing dengan:
- `user_id`
- `type`
- `title`
- `slug`
- `description`
- `location`
- `latitude`
- `longitude`
- `price_base`
- `max_guest`
- `status`

Repo saat ini memakai `Product` generik:
- `id`
- `slug`
- `name`
- `price`
- `active`
- `metadata`

Artinya struktur data produk/listing belum mengikuti ERD production di Word.

## API Endpoint Check

### Endpoint yang sudah terlihat ada

- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `GET /api/auth/me`
- `GET /api/products`
- `GET /api/products/:idOrSlug`
- `POST /api/products`
- `PATCH /api/products/:idOrSlug`
- `DELETE /api/products/:idOrSlug`
- `GET /api/orders`
- `POST /api/orders`
- `GET /api/orders/:code`
- `POST /api/orders/:code/pay`
- `GET /api/orders/:code/verify`
- `POST /api/orders/:code/mark-cash`
- `POST /api/orders/:code/complete`
- `GET /api/partner/balance`
- `GET /api/partner/payouts`
- `GET /api/partner/requests`
- `POST /api/partner/requests`
- `GET /api/reports/partner/me`
- `GET /api/payouts/batches`
- `POST /api/payouts/batches`
- `GET /api/payouts/batches/:id`
- `POST /api/payouts/batches/:id/approve`
- webhook Midtrans

### Endpoint scope yang belum sesuai atau belum ada

Auth scope:
- `POST /api/register` -> belum ada
- `POST /api/logout` -> belum ada

Listing scope:
- family endpoint `listings` -> repo masih `products`
- images endpoint listing -> belum terlihat sebagai implementasi nyata
- availability endpoint listing -> belum ada di backend utama

Payment scope:
- scope minta `Xendit` create payment + webhook
- repo pakai Midtrans

Wallet scope:
- belum ada endpoint wallet generik seperti `GET /api/wallet`
- saldo wallet saat ini diekspos lewat endpoint partner `GET /api/partner/balance`
- transaksi wallet detail generik belum ada

Withdraw scope:
- flow withdraw saat ini hidup lewat endpoint partner `POST /api/partner/requests` dan `GET /api/partner/requests`
- batch approval admin hidup lewat `/api/payouts/batches`
- `POST /api/withdraw/{id}/cancel` -> belum ada

Admin scope:
- dashboard/users/ban/listing approve-reject/withdraw approve-reject -> belum utuh

Review scope:
- `POST /api/reviews` -> belum ada
- `GET /api/listings/{id}/reviews` -> belum ada

### Endpoint placeholder yang terdeteksi

Masih ada route yang memang belum selesai:
- `/api/affiliates` -> `Not Implemented`
- `/api/reports` -> sebagian placeholder
- invite routes -> `Not Implemented`

## Critical Logic Check

### Anti Double Booking
Scope mewajibkan:
- DB transaction
- row lock availability

Kondisi repo:
- belum terlihat model `listing_availability`
- belum terlihat lock row ketersediaan
- belum terlihat conflict check booking per tanggal yang kuat

Status:
- Belum ada

### Webhook Idempotent
Scope mewajibkan:
- cek `external_id`
- jangan double update

Kondisi repo:
- webhook aktif `/api/webhooks/midtrans` sudah update `Booking` + `Payment`
- ledger escrow pending sekarang ikut ditulis saat payment sukses
- masih ada beberapa file webhook/controller lama yang perlu dirapikan supaya source of truth tidak bercabang

Status:
- Parsial

### Escrow System
Scope mewajibkan:
- uang masuk ke pending balance dulu
- baru pindah ke available balance setelah booking selesai

Kondisi repo:
- `wallet`, `balancePending`, `balanceAvailable`, dan `WalletTransaction` sudah ada
- booking yang masuk status bayar sekarang sudah menulis ledger `BOOKING_IN` dan menaikkan pending balance seller
- endpoint complete booking sekarang sudah menulis `ESCROW_RELEASE` dan memindahkan saldo ke available
- reversal untuk refund/cancel setelah dana masuk escrow belum lengkap

Status:
- Parsial

### Booking Expired
Scope mewajibkan:
- auto cancel jika belum bayar

Kondisi repo:
- ada enum status `EXPIRED`
- belum terlihat worker/job/queue/scheduler yang menjalankan expiry otomatis

Status:
- Belum lengkap

## UI Scope Check

### User UI
Scope minta:
- homepage
- search/filter
- detail
- checkout
- dashboard

Kondisi repo:
- homepage ada
- catalog/search ada
- detail product ada
- cart/checkout ada
- dashboard user murni belum terlihat matang

Status:
- Parsial

### Seller UI
Scope minta:
- dashboard
- listing
- calendar
- withdraw

Kondisi repo:
- ada panel `partner` dan `affiliate`
- tetapi masih banyak mock data di area selain saldo/payout/request/report summary
- calendar seller berbasis availability belum terlihat utuh
- withdraw partner sekarang sudah mulai punya backend nyata untuk request, balance, dan riwayat

Status:
- Parsial cenderung belum selesai

### Admin UI
Scope minta:
- analytics
- finance
- moderation

Kondisi repo:
- halaman admin/super-admin ada
- tetapi banyak komponen memakai `MOCK_*`, fallback data, atau `localStorage`

Status:
- Parsial

## Advanced Scope Check

| Fitur Advanced | Kondisi Repo | Status |
| --- | --- | --- |
| dynamic pricing | baru terlihat helper pricing sederhana, belum engine weekend/season | Parsial |
| coupon/voucher | belum ditemukan modul nyata | Belum ada |
| refund system | status ada sebagian, flow bisnis lengkap belum jelas | Parsial |
| rating weight system | belum ada | Belum ada |
| multi currency | belum ada | Belum ada |
| multi language | ada i18n dasar di web public | Parsial |
| SEO SSR | Next.js public site + robots/sitemap ada | Parsial |
| queue system | belum ada Redis/queue worker nyata | Belum ada |
| audit log | ada model `Audit` dan repo audit dasar | Parsial |

## Temuan Teknis Penting

1. Ada ketidakkonsistenan implementasi payment/webhook.
   Beberapa file memakai Midtrans dengan schema sekarang, tetapi ada controller webhook yang terlihat mengacu ke field yang tidak ada di schema aktif.

2. `web-admin` belum sepenuhnya source of truth backend.
   Banyak halaman masih hidup dari mock atau `localStorage`, jadi UI terlihat lengkap tetapi belum berarti fitur backend benar-benar selesai.

3. Model data masih terlalu generik untuk scope marketplace booking.
   `Product` dan `Order` cocok untuk fondasi, tetapi belum cukup untuk kebutuhan listing multi-vendor, availability, escrow, withdraw, dan review.

4. Belum terlihat automated test yang berarti.
   Saya tidak menemukan file test/spec yang jelas di repo ini saat audit cepat.

## Status Akhir per Area

### Sudah cukup dekat dengan scope
- pemisahan public/admin/api
- penggunaan PostgreSQL
- auth dasar
- order/payment dasar
- public site untuk katalog dan checkout awal

### Sudah ada, tapi masih parsial
- admin dashboard
- partner/affiliate dashboard
- report/audit
- multi-language
- SEO/public marketing pages
- payment verification flow

### Belum sesuai scope atau belum ada
- Laravel API
- Xendit integration
- Redis/queue/lock
- seller role yang utuh
- listing availability engine
- anti double booking
- wallet
- escrow
- withdraw end-to-end
- review/rating backend
- ERD production sesuai dokumen
- admin moderation & finance flow yang benar-benar production-ready

## Prioritas Pengerjaan Jika Ingin Menyamakan dengan Scope Word

### Priority 1: foundation yang wajib dulu
- putuskan final stack: tetap Express atau migrasi ke Laravel
- putuskan provider payment: tetap Midtrans atau ganti ke Xendit sesuai scope
- redesign schema supaya sesuai ERD marketplace booking
- ubah `Product` menjadi `Listing` atau tambahkan bounded model yang setara

### Priority 2: core marketplace logic
- implement availability per tanggal
- implement anti double booking dengan transaction + locking
- implement booking lifecycle yang jelas
- implement seller ownership pada listing

### Priority 3: finance system
- implement wallet
- implement escrow pending vs available
- implement transaction ledger
- implement withdraw request + approval + disbursement

### Priority 4: quality and production hardening
- hilangkan mock/localStorage untuk area admin kritikal
- rapikan webhook idempotent
- tambahkan queue/background jobs
- tambahkan test untuk booking, payment, webhook, payout

## Kesimpulan Final

Kalau pertanyaannya: `apakah web ini sudah sesuai dengan scope di Word?`

Jawabannya: belum.

Kalau diringkas:
- struktur proyeknya sudah mengarah ke sana
- sebagian UI dan flow dasar sudah ada
- tetapi inti bisnis marketplace booking multi-vendor pada dokumen Word masih belum lengkap
- gap terbesar ada di data model, payment provider, wallet/escrow/withdraw, availability booking, dan production readiness admin/seller flow

Jika mau, langkah berikutnya yang paling masuk akal adalah saya bantu bikin dokumen lanjutan berisi:
- checklist per file/module yang harus diubah
- urutan implementasi paling aman
- mapping schema lama ke schema target Word

## Checklist Production Ready Launch

Legenda:
- `[x]` sudah ada dan relatif usable
- `[-]` sudah ada tapi belum production ready
- `[ ]` belum ada / belum memenuhi

### A. Product Direction & Scope Lock

- [ ] finalisasi apakah stack harus tetap `Express + Prisma` atau wajib pindah ke `Laravel`
- [ ] finalisasi provider payment: tetap `Midtrans` atau ganti `Xendit` sesuai scope Word
- [ ] finalisasi model bisnis: `partner/affiliate` sekarang akan disatukan jadi `seller` atau tetap dipisah
- [ ] finalisasi ERD target production sebelum lanjut banyak coding

### B. Public Web / User App

- [x] homepage public tersedia
- [x] catalog / listing page dasar tersedia
- [x] detail page produk tersedia
- [x] cart / checkout awal tersedia
- [x] SEO dasar ada (`robots.txt`, `sitemap.xml`, SSR Next.js)
- [-] search/filter sudah ada tapi belum tervalidasi end-to-end dengan backend production
- [-] i18n dasar sudah ada tapi belum full multilingual production
- [ ] dashboard user untuk histori booking, status payment, invoice, profile
- [ ] review submission dan review list nyata
- [ ] halaman error, empty state, loading state tervalidasi untuk semua flow kritikal

### C. Seller / Partner App

- [-] halaman partner ada
- [-] halaman affiliate ada
- [-] UI withdraw/request sudah ada dan backend dasar untuk balance/request/history sudah hidup
- [ ] role `seller` yang benar-benar hidup di backend
- [ ] CRUD listing seller yang benar-benar tersimpan ke database production
- [ ] calendar / availability management per listing
- [-] dashboard seller berbasis data nyata mulai ada untuk saldo, payout, request, dan summary
- [-] laporan seller berbasis transaksi/booking nyata mulai ada di `reports/partner/me`, tapi belum lengkap

### D. Admin App

- [x] admin app tersedia
- [x] super-admin / kasir / partner panel secara UI tersedia
- [-] analytics/report panel ada, tapi banyak fallback/mocks
- [-] moderation panel ada secara struktur UI
- [ ] semua panel admin harus lepas dari `MOCK_*`
- [ ] semua panel admin harus lepas dari `localStorage` sebagai source utama
- [ ] moderation listing approve/reject production-ready
- [ ] user management dan ban/suspend flow production-ready
- [-] finance dashboard production-ready
- [-] payout approval flow production-ready

### E. Backend API Foundation

- [x] backend API tersedia
- [x] auth login/refresh/me tersedia
- [x] products routes dasar tersedia
- [x] orders routes dasar tersedia
- [x] payment verification dasar tersedia
- [x] audit model dasar tersedia
- [x] compatibility layer awal `products -> listings` sudah mulai hidup
- [x] compatibility layer awal `orders -> bookings` sudah mulai hidup
- [x] build TypeScript API kembali hijau setelah schema baru
- [x] partner finance/report endpoints dasar sudah hidup
- [x] admin payout batch endpoints dasar sudah hidup
- [x] endpoint dasar complete booking + release escrow sudah hidup
- [x] endpoint list order admin untuk dashboard sudah hidup
- [-] auth belum lengkap karena belum ada register/logout sesuai scope
- [-] reports ada tapi sebagian placeholder, walau `reports/partner/me` sudah hidup
- [x] wallet API dasar
- [-] withdraw API sudah hidup untuk request/approval dasar, tapi belum final
- [ ] reviews API
- [ ] listing availability API
- [ ] admin dashboard API final
- [-] seller dashboard API final
- [ ] konsistensi semua controller/route/schema yang sekarang masih bercampur versi lama vs baru

### F. Database & Data Model

- [x] PostgreSQL via Prisma sudah ada
- [x] migrasi Prisma sudah ada
- [-] model `User`, `Product`, `Order`, `Payment`, `Audit` sudah ada
- [x] model `Listing` yang sesuai scope sudah ditambahkan
- [x] model `ListingImage` sudah ditambahkan
- [x] model `ListingAvailability` sudah ditambahkan
- [x] model `Wallet` sudah ditambahkan
- [x] model `Transaction Ledger` sudah ditambahkan sebagai `WalletTransaction`
- [x] model `Withdraw` sudah ditambahkan
- [x] model `Review` sudah ditambahkan
- [x] relasi seller -> listing
- [-] relasi booking -> escrow -> withdraw
- [-] field user production seperti `name`, `phone`, `avatar`, `status`, `email_verified_at` sudah mulai ada di schema, tapi belum seluruh flow lama disesuaikan

### G. Booking Engine

- [-] order creation dasar ada
- [-] compatibility create/read order ke `Booking` sudah mulai ada
- [-] date input ada di frontend
- [x] booking entity yang benar-benar sesuai scope (`start_date`, `end_date`, `qty`, `price_per_day`, dst) sudah ada di schema
- [ ] availability per tanggal
- [ ] validasi bentrok booking
- [ ] anti double booking pakai DB transaction + locking
- [ ] booking expiry otomatis untuk unpaid booking
- [-] status lifecycle booking yang lengkap: pending -> paid -> completed / cancelled / expired / refunded
- [x] endpoint dasar complete booking admin sudah ada
- [ ] invoice/booking history yang konsisten untuk user dan admin

### H. Payment

- [x] payment integration dasar ada
- [x] webhook dasar ada
- [-] webhook sudah mulai diarahkan ke model `Booking` + `Payment` baru
- [-] idempotency sudah sebagian disentuh tapi belum rapi penuh
- [-] verify payment manual ada
- [ ] provider harus sesuai keputusan final scope
- [ ] create invoice/payment intent production-ready
- [ ] webhook signature verification yang final dan konsisten
- [ ] retry-safe/idempotent update di seluruh payment flow
- [ ] expiry handling otomatis
- [ ] refund handling final
- [-] settlement flow yang sinkron ke ledger/wallet

### I. Wallet / Escrow / Withdraw

- [x] wallet balance per seller
- [x] pending balance vs available balance
- [x] escrow release saat booking selesai
- [-] transaction ledger dasar sudah hidup, tapi belum lengkap
- [x] withdraw request
- [x] withdraw approval admin dasar
- [ ] disbursement ke payment provider
- [ ] cancel/reject withdraw
- [ ] audit trail finance lengkap

### J. Review & Reputation

- [ ] rating per listing
- [ ] comment review
- [ ] endpoint create review
- [ ] endpoint list review
- [ ] validasi hanya user yang selesai booking bisa review
- [ ] moderation review bila dibutuhkan
- [ ] rating aggregate / weighted score

### K. Infra, Queue, and Operations

- [x] folder `infra` sudah ada
- [-] ada arah deploy (`terraform`, `k8s`, `cdn`) tapi belum tervalidasi operasionalnya
- [ ] Redis untuk cache/queue/lock sesuai scope
- [ ] job worker untuk expiry booking
- [ ] job worker untuk email / notif / webhook retry
- [ ] secrets management production
- [ ] environment separation dev/staging/prod yang rapi
- [ ] backup & restore database plan
- [ ] monitoring + alerting
- [ ] rate limiting final per endpoint kritikal

### L. Security & Compliance

- [x] auth middleware dasar ada
- [x] RBAC dasar ada
- [-] role `SELLER` sudah masuk schema/helper auth, tapi access control bisnis belum lengkap
- [-] rate limit dasar ada
- [ ] password reset / email verification
- [ ] secure session/token rotation policy
- [ ] audit log action admin yang konsisten
- [ ] webhook security hardening
- [ ] input validation menyeluruh untuk semua endpoint
- [ ] access control test untuk role guest/user/seller/admin
- [ ] secret leakage check dan cleanup env sample

### M. Quality Assurance

- [ ] unit test backend
- [ ] integration test API
- [ ] webhook test
- [ ] booking conflict test
- [ ] payout/withdraw finance test
- [ ] e2e test web-public
- [ ] e2e test web-admin
- [ ] staging smoke test sebelum launch
- [ ] load/performance test minimal untuk booking + payment callback

### N. Launch Checklist

- [ ] semua `MOCK_*`, placeholder, dan `Not Implemented` dibersihkan dari flow produksi
- [ ] semua flow kritikal memakai database/API nyata, bukan `localStorage`
- [ ] schema database sudah final dan dimigrasikan dengan aman
- [ ] payment provider final sudah sandbox-tested dan production-tested
- [ ] booking flow end-to-end lulus test
- [ ] wallet/escrow/withdraw lulus test
- [ ] admin moderation dan finance flow lulus UAT
- [ ] observability aktif sebelum launch
- [ ] SOP incident, refund, dan manual fallback tersedia
- [ ] deploy staging stabil
- [ ] go-live checklist dan rollback plan tersedia

## Quick Verdict Menuju Launch

### Sudah bisa dianggap siap fondasi
- public web dasar
- admin shell/dashboard dasar
- API dasar
- PostgreSQL + Prisma
- auth dasar
- order/payment dasar

### Belum siap production launch
- data model marketplace final
- seller system final
- booking engine final
- wallet / escrow / withdraw
- review system
- Redis / queue / expiry jobs
- penghapusan mock/localStorage dari area kritikal
- automated tests
- operational hardening

### Estimasi status saat ini
- Foundation/UI exploration: `cukup jauh jalan`
- Business core marketplace: `belum selesai`
- Finance & payout readiness: `belum siap`
- Production launch readiness: `belum siap launch`
