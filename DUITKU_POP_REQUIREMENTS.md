# Duitku POP Requirements

Dokumen requirement terverifikasi untuk integrasi Duitku POP pada backend Supabase.

- **Source of truth:** [Dokumentasi resmi Duitku POP Bahasa Indonesia](https://docs.duitku.com/pop/id/)
- **Repository:** `alfazarqom/supabase-backend`
- **Status dokumen:** Bertahap; hanya bagian yang sudah diaudit dan dikunci yang berstatus `LOCKED`.
- **Tanggal pemeriksaan tahap 1:** 2026-10-31
- **Mode kerja:** Audit dan pencatatan requirement bersifat read-only terhadap sistem berjalan. Dokumen requirement ini tidak memberi izin untuk mengubah database, Edge Functions, secrets, atau deployment.

## Status audit bertahap

| Tahap | Bagian dokumentasi | Status |
|---:|---|---|
| 1 | Create Invoice dan Obyek JSON | LOCKED |
| 2 | HTTP Callback | LOCKED |
| 3 | JS Callback dan Return URL | LOCKED |
| 4 | Expiry dan status transaksi | LOCKED (open clarification) |
| 5 | Payment Method dan fitur khusus | LOCKED (availability merchant perlu konfirmasi) |
| 6 | Errors dan rekonsiliasi | LOCKED (inquiry belum terverifikasi) |

---

## 1. Create Invoice — LOCKED

### 1.1 Endpoint dan transport

| Ketentuan | Requirement |
|---|---|
| HTTP method | `POST` |
| Content-Type | `application/json` |
| Production endpoint | `https://api-prod.duitku.com/api/merchant/createInvoice` |
| Sandbox endpoint | `https://api-sandbox.duitku.com/api/merchant/createInvoice` |

Environment harus dipilih melalui konfigurasi server-side yang eksplisit. Jangan mencampur Sandbox credentials dengan Production endpoint atau sebaliknya.

### 1.2 Request headers dan autentikasi

Ketiga header autentikasi berikut wajib dikirim bersama Content-Type:

- `Content-Type: application/json`
- `x-duitku-timestamp`
- `x-duitku-signature`
- `x-duitku-merchantcode`

Timestamp menggunakan UNIX milliseconds.

Formula signature Create Invoice khusus POP:

```text
stringToSign = merchantCode + timestamp
signature = HMAC_SHA256(stringToSign, apiKey)
```

Ketentuan:
- Signature dihitung di backend menggunakan API Key (Merchant Key).
- API Key dan merchant credential tidak boleh diekspos ke browser, frontend bundle, repository, atau log.
- Metode signature lama SHA256 biasa dinyatakan obsolete oleh dokumentasi; jangan menggunakannya.
- Jangan mengganti formula ini dengan formula dari dokumentasi Duitku API umum atau produk Duitku lain.

### 1.3 Request parameters

| Parameter | Tipe/batas | Wajib |
|---|---|---|
| `paymentAmount` | Integer | Ya |
| `merchantOrderId` | String, maksimum 50 karakter | Ya |
| `productDetails` | String, maksimum 255 karakter | Ya |
| `email` | String, maksimum 255 karakter | Ya |
| `callbackUrl` | String, maksimum 255 karakter | Ya |
| `returnUrl` | String, maksimum 255 karakter | Ya |
| `additionalParam` | String, maksimum 255 karakter | Tidak |
| `merchantUserInfo` | String, maksimum 255 karakter | Tidak |
| `customerVaName` | String, maksimum 20 karakter | Tidak |
| `phoneNumber` | String, maksimum 50 karakter | Tidak |
| `itemDetails` | Object array | Tidak |
| `customerDetail` | Object | Tidak |
| `expiryPeriod` | Integer, satuan menit | Tidak |
| `paymentMethod` | String | Tidak |
| `creditCardDetail` | Object | Tidak |

Validasi tipe, panjang string, dan kewajiban parameter harus dilakukan sebelum request ke Duitku.

### 1.4 Item Details

Dokumentasi mendefinisikan setiap item dengan:

| Field | Tipe | Wajib |
|---|---|---|
| `name` | String, maksimum 50 karakter | Ya |
| `quantity` | Integer | Ya |
| `price` | Integer, tanpa desimal | Ya |

Dokumentasi menyatakan total seluruh nilai `price` pada item details harus sama dengan `paymentAmount`. Namun contoh request pada halaman yang sama menunjukkan `paymentAmount = 40000`, dua harga item 10000 dan 30000, serta quantity masing-masing 1 dan 3. Contoh tersebut tidak menjelaskan secara konsisten apakah pemeriksaan total harus memperhitungkan quantity.

**Open clarification:** Jangan mengasumsikan formula subtotal `price × quantity` atau mengubah nilai item untuk menyesuaikan total tanpa klarifikasi Duitku atau pengujian Sandbox yang membuktikan perilakunya. Jika `itemDetails` digunakan, catat hasil verifikasi kontrak tersebut terlebih dahulu.

### 1.5 Customer Detail dan Address

`customerDetail` bersifat opsional. Field yang didokumentasikan:

- `firstName`
- `lastName`
- `email`
- `phoneNumber`
- `billingAddress`
- `shippingAddress`
- `merchantCustomerId`

Address mendokumentasikan field:

- `firstName`
- `lastName`
- `address`
- `city`
- `postalCode`
- `phone`
- `countryCode`

Field-field nested tersebut dinyatakan opsional pada tabel dokumentasi. `merchantCustomerId` menjadi wajib untuk skenario tokenisasi sesuai ketentuan khusus di bawah.

### 1.6 Tokenisasi kartu

Untuk transaksi tokenisasi (VC atau MG), dokumentasi menyatakan parameter berikut menjadi wajib:

- `email`
- `customerDetail.email`
- `customerDetail.merchantCustomerId`
- `creditCardDetail.saveCardToken`

Jangan mengaktifkan atau mengimplementasikan tokenisasi sebagai bagian dari alur standar jika tidak termasuk kebutuhan dan persetujuan produk.

### 1.7 Create Invoice response

Response mendokumentasikan field:

| Field | Fungsi |
|---|---|
| `merchantCode` | Merchant code |
| `reference` | Referensi Duitku; wajib disimpan untuk pencatatan/pelacakan |
| `paymentUrl` | URL halaman pembayaran Duitku |
| `statusCode` | Kode hasil request; contoh sukses `00` |
| `statusMessage` | Pesan hasil request; contoh `SUCCESS` |

Backend harus memeriksa response provider dan status bisnisnya. HTTP success saja tidak cukup untuk menyatakan Create Invoice berhasil. Jangan membuat atau memalsukan `reference` maupun `paymentUrl`.

### 1.8 Merchant order ID dan pengulangan request

Ketentuan resmi yang dicatat:

1. Setiap request untuk transaksi baru harus menggunakan `merchantOrderId` baru.
2. Duitku menyatakan response yang sama dapat diberikan selama lima menit untuk request dengan order ID yang sama.
3. Jika pesanan berubah, dokumentasi meminta menunggu sekitar lima menit sebelum request ulang.
4. Request bersamaan dengan order ID yang sama dapat menghasilkan error `The transaction is still in progress` atau `Amount is different please try again later`.

Implikasi requirement backend:
- Gunakan merchant order ID yang stabil dan unik untuk satu transaksi bisnis.
- Cegah request invoice paralel untuk merchant order ID yang sama.
- Jangan mengubah nominal invoice untuk order ID yang sama lalu langsung mengulang request.
- Perilaku retry harus mempertimbangkan jendela lima menit yang didokumentasikan dan status lokal/provider.
- Perubahan pesanan yang menghasilkan transaksi baru harus mengikuti aturan identitas transaksi baru; jangan menggunakan ulang order ID secara sembarang.

### 1.9 Additional parameter

Jika `additionalParam` digunakan, nilai/parameter yang dikirim harus URL-encoded sebagaimana dinyatakan dokumentasi.

### 1.10 Catatan inkonsistensi dokumentasi yang perlu diklarifikasi

1. Bagian Endpoint menuliskan path `/api/merchant/createInvoice`, sedangkan contoh kode PHP menuliskan `/api/merchant/createinvoice`. Requirement menggunakan bentuk pada bagian Endpoint sebagai referensi utama; perbedaan kapitalisasi dicatat dan perlu dikonfirmasi bila menimbulkan masalah pada request aktual.
2. Aturan kesetaraan `itemDetails.price` dengan `paymentAmount` tidak sepenuhnya jelas terhadap quantity pada contoh yang disediakan. Jangan menebak perhitungan final.

---



## 2. HTTP Callback — LOCKED

### 2.1 Callback endpoint requirements

| Ketentuan | Requirement |
|---|---|
| HTTP method | `POST` |
| Content-Type | `application/x-www-form-urlencoded` |
| Callback URL | Harus dapat diakses secara publik oleh server Duitku |
| Port | 80 atau 443 |
| Response | HTTP 200 OK setelah callback berhasil ditangani |

Callback URL harus diarahkan ke backend server-side, bukan halaman frontend/browser.

### 2.2 Callback signature

Formula callback POP:

```text
stringToSign = merchantCode + amount + merchantOrderId
signature = HMAC_SHA256(stringToSign, apiKey)
```

Ketentuan:
- Formula callback berbeda dari formula Create Invoice.
- Gunakan API Key/Merchant Key yang sesuai dengan environment merchant.
- Signature wajib diverifikasi sebelum status transaksi diproses.
- Metode signature MD5 lama dinyatakan obsolete oleh dokumentasi.
- Jangan menggunakan formula signature dari dokumentasi Duitku API umum atau produk lain.

### 2.3 Parameter callback

Parameter inti yang digunakan untuk validasi dan pencocokan transaksi:

| Parameter | Fungsi | Ketentuan backend |
|---|---|---|
| `merchantCode` | Kode merchant | Cocokkan dengan merchant yang dikonfigurasi |
| `amount` | Nominal pembayaran | Cocokkan dengan nominal transaksi tersimpan |
| `merchantOrderId` | ID transaksi merchant | Harus menunjuk ke transaksi yang dikenal |
| `resultCode` | Hasil pembayaran | `00` = Success; `01` = Failed |
| `reference` | Referensi Duitku | Cocokkan dengan reference invoice yang tersimpan |
| `signature` | Bukti integritas/autentikasi callback | Verifikasi dengan formula resmi |

Parameter lain yang didokumentasikan dapat mencakup:
- `productDetail`
- `additionalParam`
- `paymentCode`
- `merchantUserId`
- `publisherOrderId`
- `spUserHash`
- `settlementDate`
- `issuerCode`
- `customerName`
- `bankAppCode`
- `bankOrderId`
- `bankRespCode`
- `bankRespMsg`
- `cardName`
- `cardType`
- `maskedNumber`
- `tokenId`
- `transactionState`
- `transactionStateStatus`
- `merchantCustomerId`
- `expiryDate`

Parameter tambahan bersifat relevan sesuai metode pembayaran dan fitur yang digunakan. Jangan mewajibkan semua parameter opsional untuk setiap callback.

### 2.4 Callback processing requirements

Backend wajib:
1. Menerima callback dengan parser yang sesuai untuk `application/x-www-form-urlencoded`.
2. Menolak parameter inti yang hilang atau format yang tidak valid.
3. Memverifikasi callback signature menggunakan API Key server-side.
4. Memastikan `merchantCode` sesuai konfigurasi merchant.
5. Mencari transaksi berdasarkan `merchantOrderId` yang dikenal.
6. Memastikan `amount` sama dengan nominal transaksi yang tersimpan.
7. Memastikan `reference` cocok dengan reference invoice yang sebelumnya disimpan.
8. Memvalidasi `resultCode` dan legalitas transisi status berdasarkan state transaksi yang ada.
9. Memproses callback secara idempotent agar callback berulang tidak menggandakan perubahan status, stok, atau fulfillment.
10. Menyimpan hasil pemrosesan secara aman agar callback yang gagal ditangani dapat ditelusuri dan dipulihkan.

Signature yang valid saja tidak cukup untuk mengubah sembarang transaksi. Pemetaan order, amount, reference, environment, dan transisi status tetap harus diperiksa.

### 2.5 HTTP response, retry, dan recovery

Dokumentasi menyatakan:
- Callback endpoint perlu mengembalikan HTTP 200 OK.
- Duitku mengirim ulang callback jika server merchant belum memberikan HTTP 200.
- Pengiriman ulang dilakukan sampai maksimum lima kali.
- Setelah upaya maksimum, Duitku mengirim pemberitahuan callback gagal melalui email.
- Merchant dapat mengirim ulang callback melalui fitur resend pada menu Report di dashboard Duitku.

Requirement backend:
- Jangan mengirim HTTP 200 setelah callback gagal diproses dan event dibuang.
- Pastikan pemrosesan lokal tahan terhadap callback duplikat karena provider dapat melakukan retry.
- Jika database/RPC gagal, pertahankan kemampuan retry/recovery dan jangan menandai event sebagai berhasil diproses.
- Jangan mengasumsikan interval retry karena intervalnya tidak ditetapkan dalam bagian dokumentasi yang diperiksa.

### 2.6 IP allowlist

Dokumentasi menyediakan daftar outgoing IP terpisah untuk Production dan Sandbox, serta menyebut port 80/443 dan URL publik.

IP allowlist dapat dipertimbangkan sebagai defense-in-depth jika sesuai dengan infrastruktur. IP allowlist tidak menggantikan verifikasi HMAC signature dan pencocokan transaksi. Daftar IP harus diperiksa ulang pada dokumentasi resmi saat konfigurasi jaringan dilakukan; jangan mengandalkan daftar yang disalin permanen tanpa verifikasi terbaru.

### 2.7 Inkonsistensi/hal yang perlu diperhatikan

1. Tabel parameter callback menulis `productDetail` (singular), sedangkan contoh cURL pada halaman menggunakan `productDetails` (plural). Jangan menjadikan field ini sebagai syarat inti callback sebelum format aktual dikonfirmasi.
2. Contoh kode menggunakan variabel `$apiKey`, sedangkan deskripsi formula menyebut `merchantKey`. Requirement menganggap keduanya merujuk pada credential API Key/Merchant Key milik proyek Duitku; nama secret internal harus konsisten dan tidak mengubah formula.
3. Formula signature tidak memasukkan `reference` ke string yang ditandatangani. Karena itu backend tetap harus mencocokkan reference terhadap data invoice tersimpan secara independen.
4. Dokumentasi menyebut maksimum lima kali pengiriman ulang, tetapi tidak menetapkan interval retry pada bagian yang diperiksa.\n5. Terdapat konflik result code HTTP Callback: tabel Parameter menyebut 01 = Failed, sedangkan Errors → API Response menyebut 02 = Failed. Kode gagal belum boleh dipetakan final sebelum klarifikasi.

---



## 3. JS Callback dan Return URL — LOCKED

### 3.1 Dua metode menampilkan halaman pembayaran

Dokumentasi POP menyediakan dua metode frontend:

**A. Duitku JS / Popup**
- Sertakan library sesuai environment:
  - Production: `https://app-prod.duitku.com/lib/js/duitku.js`
  - Sandbox: `https://app-sandbox.duitku.com/lib/js/duitku.js`
- Mulai pembayaran dengan `checkout.process(duitkuReference, options)`.
- `duitkuReference` berasal dari hasil Create Invoice backend.
- Event opsional yang didokumentasikan: `successEvent`, `pendingEvent`, `errorEvent`, dan `closeEvent`.
- Opsi `defaultLanguage` mendukung `id` dan `en`.
- Opsi `currency` hanya estimasi konversi; pembayaran tetap dilakukan dalam IDR dan rate dapat berubah.

**B. Window Redirection**
- Alternatif terhadap Duitku JS.
- Gunakan `paymentUrl` yang diterima dari response Create Invoice.
- Browser diarahkan ke halaman pembayaran Duitku.
- Dokumentasi menyebut opsi query `lang=id` atau `lang=en`.
- Opsi `currency=USD` atau `currency=EUR` hanya estimasi; pembayaran tetap IDR.

Pemilihan metode frontend harus menjadi keputusan integrasi yang eksplisit. Jangan menganggap kedua metode harus digunakan sekaligus atau mengganti metode yang sudah dipilih tanpa persetujuan.

### 3.2 JS callback — bukan bukti pembayaran

Objek JS callback yang didokumentasikan memuat:

| Field | Tipe |
|---|---|
| `resultCode` | String |
| `merchantOrderId` | String |
| `reference` | String |

Event UI yang didokumentasikan:
- `successEvent`: callback UI untuk hasil sukses (contoh result code 00).
- `pendingEvent`: callback UI untuk status pending (contoh result code 01).
- `errorEvent`: callback UI untuk error.
- `closeEvent`: callback UI ketika pelanggan menutup popup tanpa menyelesaikan pembayaran (contoh result code 02).

**Ketentuan keamanan yang eksplisit dari Duitku:**
- Response JS callback hanya boleh digunakan untuk notifikasi/informasi kepada pelanggan.
- JS callback tidak diperkenankan mengubah status pembayaran pada sistem merchant.
- Perubahan status transaksi harus menggunakan HTTP Callback.

Backend dan frontend tidak boleh menganggap event `successEvent`, `pendingEvent`, `errorEvent`, atau `closeEvent` sebagai otorisasi untuk mengubah status pembayaran, stok, atau fulfillment.

### 3.3 Return URL / Redirect

Setelah transaksi dibuat, Duitku mengarahkan pelanggan ke `returnUrl` setelah pembayaran atau ketika pelanggan menggunakan tombol kembali.

Dokumentasi memberikan contoh parameter GET:
- `merchantOrderId`
- `resultCode`
- `reference`

Ketentuan:
- Return URL merupakan landing page untuk navigasi dan penyampaian informasi kepada pelanggan.
- Dokumentasi secara eksplisit melarang penggunaan `resultCode` redirect untuk memperbarui status pembayaran.
- URL dan query parameter dapat diubah secara manual oleh pelanggan.
- Halaman return harus meminta status terkini dari backend berdasarkan order yang sah; status otoritatif berasal dari pemrosesan HTTP Callback, bukan query string browser.

### 3.4 Pemisahan status dan konteks resultCode

Jangan menyamakan result code antar mekanisme:
- JS callback: event UI success/pending/close menggunakan contoh kode 00/01/02.
- Redirect: tabel error mendefinisikan 00 = Success, 01 = Process, 02 = Canceled/Failed.
- HTTP callback: dokumentasi memiliki perbedaan definisi kode gagal antara tabel parameter callback dan tabel API response; dicatat sebagai open clarification pada bagian HTTP Callback.

Status dari JS callback atau redirect tidak boleh dipakai untuk menyelesaikan perbedaan tersebut dan tidak boleh menjadi dasar mutasi pembayaran.

---



## 4. Expiry dan Status Transaksi — LOCKED (open clarification)

### 4.1 expiryPeriod pada dokumentasi POP

Dokumentasi POP menyatakan:
- `expiryPeriod` merupakan parameter opsional pada Create Invoice.
- Tipe parameter: integer.
- Satuan: menit.
- Contoh nilai yang ditampilkan: 5, 10, atau 60 menit.

Batas penting:
- Bagian POP yang diperiksa tidak menetapkan nilai default secara rinci.
- Bagian POP yang diperiksa tidak menetapkan nilai maksimum yang berlaku umum.
- Jangan menganggap nilai 1.440 menit (24 jam) didukung untuk semua metode pembayaran.
- Jangan mengambil tabel default/maksimum expiry dari dokumentasi Duitku API umum sebagai kontrak POP.

### 4.2 Pemisahan expiry provider dan deadline bisnis

Aturan bisnis aplikasi Asri menetapkan tenggat pembayaran 24 jam. Aturan bisnis ini harus dipisahkan dari expiry invoice yang diterima dan diterapkan provider.

Requirement:
1. Simpan deadline bisnis order secara independen dari parameter expiry provider.
2. Jangan menyatakan invoice provider kedaluwarsa hanya karena deadline lokal tercapai, kecuali perilaku provider untuk metode pembayaran tersebut sudah dikonfirmasi.
3. Sebelum mengirim `expiryPeriod = 1440`, konfirmasi dukungan untuk metode pembayaran yang digunakan melalui dokumentasi POP yang lebih spesifik, merchant support, atau pengujian Sandbox yang disetujui.
4. Jangan mengasumsikan default atau maksimum per payment method dari dokumentasi produk Duitku lain.
5. Jika provider expiry tidak dapat disamakan dengan deadline bisnis, definisikan penanganan lokal secara eksplisit tanpa mengklaim bahwa invoice provider sudah expired.

**Open clarification:** Dukungan expiry 24 jam untuk metode pembayaran yang akan dipakai Asri belum terverifikasi dari bagian dokumentasi POP yang diperiksa.

### 4.3 Konteks resultCode yang didokumentasikan

Kode hasil tidak boleh dipetakan tanpa memperhatikan konteks asalnya.

| Konteks | Kode | Arti pada dokumentasi |
|---|---:|---|
| HTTP Callback — tabel Parameter | `00` | Success |
| HTTP Callback — tabel Parameter | `01` | Failed |
| HTTP Callback — bagian Errors / API Response | `00` | Success |
| HTTP Callback — bagian Errors / API Response | `02` | Failed |
| JS Callback event | `00` | Success event |
| JS Callback event | `01` | Pending event |
| JS Callback event | `02` | Close event |
| Return Redirect | `00` | Success |
| Return Redirect | `01` | Process / belum terbayar |
| Return Redirect | `02` | Canceled / Failed |

### 4.4 Konflik status callback — wajib diklarifikasi

Dokumentasi POP pada halaman yang sama tidak konsisten:
- Tabel parameter HTTP Callback menyatakan `01 = Failed`.
- Bagian Errors → API Response → Callback menyatakan `02 = Failed`.

Requirement:
- `00` dicatat sebagai kode sukses yang konsisten pada kedua bagian callback.
- Kode gagal callback belum boleh dipetakan final ke `01` atau `02` sebelum klarifikasi resmi atau pengujian Sandbox yang dapat dipertanggungjawabkan.
- Jangan memakai tabel JS Callback atau Redirect untuk menyelesaikan konflik kode HTTP Callback.
- Implementasi harus menolak asumsi bahwa kode selain `00` otomatis memiliki arti gagal yang sama di semua konteks.
- Simpan raw result code dan konteks callback secara aman untuk audit; jangan mengubahnya menjadi status bisnis yang tidak didukung bukti.

### 4.5 Status inquiry / reconciliation

Pada bagian Integrasi Backend POP yang diperiksa, dokumentasi menjelaskan Create Invoice dan HTTP Callback; endpoint inquiry/status-check tidak ditemukan pada bagian tersebut.

Requirement:
- Jangan mengarang endpoint inquiry, request signature, atau response status.
- Jika rekonsiliasi status tambahan diperlukan, verifikasi dulu apakah ada mekanisme resmi yang berlaku untuk produk POP dan merchant terkait.
- Sampai mekanisme tersebut terverifikasi, rekonsiliasi harus diperlakukan sebagai kebutuhan desain yang belum memiliki kontrak provider terkonfirmasi.

---



## 5. Payment Method dan Fitur Khusus — LOCKED (availability merchant perlu konfirmasi)

### 5.1 Cara memilih metode pembayaran

Dokumentasi POP menyatakan parameter `paymentMethod` pada Create Invoice bersifat opsional. Jika diisi dengan kode metode pembayaran tertentu, pelanggan akan diarahkan langsung ke metode tersebut tanpa memilih metode lain pada halaman pembayaran.

Requirement:
- Jika `paymentMethod` tidak dikirim, jangan mengasumsikan perilaku channel tertentu di luar konfigurasi merchant.
- Jika `paymentMethod` dikirim, gunakan hanya kode yang tercantum pada dokumentasi POP dan telah dikonfirmasi aktif untuk merchant.
- Daftar channel pada dokumentasi bukan bukti bahwa seluruh channel aktif pada akun merchant Asri.
- Jangan hardcode seluruh channel sebagai tersedia di UI sebelum status aktivasi merchant diverifikasi.

### 5.2 Daftar payment method dan kode yang tercantum

Daftar berikut disalin sebagai referensi kode dari halaman Payment Method POP yang diperiksa. Aktivasi, biaya, limit, dan ketersediaan aktual tetap harus dikonfirmasi pada merchant portal/akun Duitku.

| Kategori | Kode | Metode |
|---|---|---|
| Credit Card | `VC` | Visa / MasterCard / JCB |
| Virtual Account | `BC` | BCA Virtual Account |
| Virtual Account | `M2` | Mandiri Virtual Account |
| Virtual Account | `VA` | Maybank Virtual Account |
| Virtual Account | `I1` | BNI Virtual Account |
| Virtual Account | `B1` | CIMB Niaga Virtual Account |
| Virtual Account | `BT` | Permata Bank Virtual Account |
| Virtual Account | `A1` | ATM Bersama |
| Virtual Account | `AG` | Bank Artha Graha |
| Virtual Account | `NC` | Bank Neo Commerce / BNC |
| Virtual Account | `BR` | BRIVA |
| Virtual Account | `S1` | Bank Sahabat Sampoerna |
| Virtual Account | `DM` | Danamon Virtual Account |
| Virtual Account | `BV` | BSI Virtual Account |
| Retail | `FT` | Pegadaian / ALFA / Pos |
| Retail | `IR` | Indomaret |
| E-Wallet | `OV` | OVO (support void) |
| E-Wallet | `SA` | ShopeePay Apps (support void) |
| E-Wallet | `LF` | LinkAja Apps (fixed fee) |
| E-Wallet | `LA` | LinkAja Apps (percentage fee) |
| E-Wallet | `DA` | DANA |
| E-Wallet | `SL` | ShopeePay Account Link |
| E-Wallet | `OL` | OVO Account Link |
| QRIS | `SP` | ShopeePay |
| QRIS | `NQ` | Nobu |
| QRIS | `GQ` | Gudang Voucher |
| QRIS | `SQ` | Nusapay |
| Credit / Paylater | `DN` | Indodana Paylater |
| Credit / Paylater | `AT` | ATOME |
| E-Banking | `JP` | Jenius Pay |
| E-Commerce | `T1` | Tokopedia Card Payment |
| E-Commerce | `T2` | Tokopedia E-Wallet |
| E-Commerce | `T3` | Tokopedia Others |

### 5.3 Parameter wajib berdasarkan kategori

Dokumentasi memberikan persyaratan tambahan berikut:

| Kategori | Parameter tambahan yang diwajibkan |
|---|---|
| Credit / Paylater (`DN`, `AT`) | `customerDetail` dan `itemDetails` |
| E-Commerce (`T1`, `T2`, `T3`) | `customerVaName` |

Ketentuan:
- Validasi parameter khusus kategori sebelum mengirim Create Invoice.
- Jangan menganggap field opsional secara umum tetap opsional ketika metode pembayaran mengaktifkan aturan khusus.
- Untuk Credit / Paylater, detail customer dan item harus dibentuk dari data backend yang tervalidasi.
- Untuk E-Commerce, `customerVaName` harus tersedia dan memenuhi batas panjang parameter Create Invoice.

### 5.4 Payment method dengan fitur khusus

Dari deskripsi daftar POP:
- `OV` OVO: support void.
- `SA` ShopeePay Apps: support void.
- `LF` LinkAja Apps: fixed fee.
- `LA` LinkAja Apps: percentage fee.
- `SL` ShopeePay Account Link.
- `OL` OVO Account Link.

Requirement:
- Jangan menganggap fitur void, fee, account link, atau settlement behavior telah diimplementasikan hanya karena channel terdaftar.
- Proses void/refund, perhitungan fee, dan status khusus memerlukan audit dokumentasi terkait serta konfirmasi fitur merchant sebelum implementasi.
- Jangan mengasumsikan kode payment method yang serupa memiliki perilaku identik.

### 5.5 QRIS issuer list

Dokumentasi POP menyediakan daftar issuer QRIS terpisah, dengan kode issuer dan nama issuer. Callback dapat mengirimkan `issuerCode`; field ini merupakan informasi issuer, bukan pengganti payment method code atau bukti pembayaran.

Requirement:
- Simpan `issuerCode` jika diterima dan diperlukan untuk pelacakan/analitik.
- Jangan hardcode daftar issuer sebagai daftar metode pembayaran.
- Jika UI atau bisnis membutuhkan nama issuer, cocokkan kode dengan daftar issuer resmi yang diperiksa saat implementasi.
- Daftar issuer dapat berubah; periksa ulang dokumentasi sebelum memperbarui katalog.
- `customerName` pada callback hanya berlaku untuk Nobu QRIS (`NQ`) dan Nusapay QRIS (`SQ`) menurut dokumentasi.

### 5.6 Uji coba Sandbox per metode

Dokumentasi menyediakan bagian uji coba terpisah untuk:
- Kartu kredit 3-D Secure.
- Virtual Account demo.
- ShopeePay staging app untuk E-Money dan Shopee QRIS.
- Gudang Voucher QRIS melalui demo VA.
- Nusapay QRIS staging app.
- Indodana dan ATOME Paylater.
- Jenius Pay.
- Tokopedia E-Commerce.

Requirement:
- Gunakan hanya fasilitas dan kredensial uji Sandbox yang dipublikasikan Duitku untuk metode terkait.
- Jangan menjalankan transaksi riil untuk membuktikan channel.
- Jangan menyalin kredensial uji ke source code, dokumentasi publik proyek, atau log.
- Pengujian suatu metode tidak membuktikan semua metode aktif atau berfungsi pada merchant.
- Catat hasil aktual per channel, environment, tanggal, dan bukti hasil.

### 5.7 Batasan dan hal yang belum diverifikasi

1. Ketersediaan setiap channel pada merchant Asri belum diverifikasi.
2. Tarif, biaya tetap/persentase, limit transaksi, settlement time, dan eligibility merchant tidak dirinci lengkap pada tabel Payment Method POP; jangan mengarang nilai.
3. Detail operasional void untuk channel yang menyatakan support void belum diaudit pada tahap ini.
4. Daftar QRIS issuer merupakan daftar terpisah dan dapat berkembang; tidak otomatis menyatakan semua issuer/channel aktif bagi merchant.
5. Kode channel yang tercantum adalah katalog dokumentasi saat diperiksa, bukan kontrak bahwa channel pasti tersedia pada production account.

---



## 6. Errors dan Rekonsiliasi — LOCKED (inquiry belum terverifikasi)

### 6.1 HTTP response errors

Dokumentasi POP mencantumkan respons HTTP berikut:

| HTTP code | Error | Arti menurut dokumentasi | Requirement backend |
|---:|---|---|---|
| 400 | Bad Request | Kesalahan saat mengirim request API | Periksa payload, tipe, parameter wajib, dan format |
| 400 | Amount is different please try again later | Request ulang untuk order ID sama dengan nominal berbeda | Jangan membuat retry paralel atau mengganti nominal pada order ID yang sama |
| 400 | SaveCardToken is not available | Akun belum mendukung fitur tokenisasi | Jangan retry seolah-olah masalah jaringan; fitur perlu aktivasi/konfirmasi merchant |
| 401 | Unauthorized | Akses ditolak | Periksa merchant code, API Key, signature, timestamp, dan environment tanpa membocorkan secret |
| 404 | Not Found | API/halaman tidak ditemukan | Verifikasi endpoint, method, dan environment dari dokumentasi |
| 409 | The transaction is still in progress | Request dengan order ID sama masih diproses | Perlakukan sebagai transaksi/invoice yang sedang berjalan; jangan membuat order ID baru secara otomatis |
| 500 | Internal Server Error | Kesalahan proses pada sisi API | Tangani sebagai kegagalan provider sementara/unknown; jangan menyatakan invoice berhasil |

Backend harus memisahkan:
- HTTP transport status.
- API/business response body.
- Status transaksi pembayaran yang hanya ditetapkan dari mekanisme konfirmasi yang sah.

HTTP 200 saja tidak membuktikan invoice sukses. Untuk Create Invoice, periksa response body termasuk `statusCode`, `statusMessage`, `reference`, dan `paymentUrl` sesuai kontrak tahap 1.

### 6.2 API response result codes

Bagian Errors → API Response pada dokumentasi POP menyebut:

**Callback**
| Kode | Nama | Arti |
|---|---|---|
| `00` | Success | Transaksi sukses terbayarkan |
| `02` | Failed | Transaksi gagal terbayarkan |

**Redirect**
| Kode | Nama | Arti |
|---|---|---|
| `00` | Success | Transaksi terbayar |
| `01` | Process | Transaksi belum terbayar |
| `02` | Canceled/Failed | Transaksi dibatalkan atau tidak terbayar |

Konflik callback dengan tabel Parameter:
- Tabel Parameter Callback sebelumnya menyebut `01 = Failed`.
- Bagian Errors → API Response → Callback menyebut `02 = Failed`.
- Konflik ini tetap open clarification. Requirement implementasi tidak boleh menetapkan `01` atau `02` sebagai kode gagal final sebelum konfirmasi Duitku atau pengujian Sandbox yang dapat diverifikasi.
- Kode Redirect tidak boleh digunakan untuk menyelesaikan konflik callback karena redirect tidak otoritatif untuk mutasi status.

### 6.3 Rekonsiliasi dan status inquiry

Dalam dokumentasi POP yang diperiksa pada bagian backend, Create Invoice dan Callback tersedia sebagai mekanisme utama yang dijelaskan. Bagian yang diperiksa tidak memberikan kontrak endpoint inquiry/status-check yang dapat digunakan untuk rekonsiliasi.

Requirement:
1. Jangan mengarang URL inquiry, HTTP method, parameter, signature, atau response.
2. Jangan menganggap fitur inquiry dari produk/API Duitku lain otomatis berlaku pada POP.
3. Jika perlu rekonsiliasi, gunakan bukti yang tersedia melalui callback resend pada dashboard Duitku dan pencocokan catatan transaksi merchant, atau minta konfirmasi mekanisme resmi yang berlaku untuk POP.
4. Status lokal tidak boleh diubah hanya karena timeout, error HTTP, browser redirect, atau ketiadaan callback.
5. Tandai status transaksi yang tidak dapat dipastikan sebagai perlu pemeriksaan/reconciliation, bukan sukses.
6. Simpan jejak transaksi dan callback yang cukup untuk investigasi tanpa menyimpan secret atau data sensitif yang tidak diperlukan.

### 6.4 Callback resend dan pemulihan

Dokumentasi Callback menyatakan:
- Duitku mengirim ulang callback bila belum menerima HTTP 200.
- Pengiriman ulang dilakukan hingga maksimum lima kali.
- Jika maksimum tercapai, merchant menerima pemberitahuan melalui email.
- Merchant dapat menggunakan fitur resend callback pada menu Report dashboard Duitku.

Requirement backend:
- Handler callback harus idempotent agar resend tidak menggandakan perubahan.
- Simpan outcome pemrosesan callback secara aman.
- Jika pemrosesan lokal gagal, jangan mengakui callback sebagai selesai tanpa mekanisme durable recovery yang telah dirancang.
- Prosedur operator untuk callback resend harus mencocokkan order, amount, reference, dan signature sebelum status diproses ulang.

### 6.5 Sandbox test resources yang didokumentasikan

Dokumentasi POP menyediakan resource uji coba per kategori:
- Kartu kredit 3-D Secure dengan kartu dummy.
- Demo transaksi Virtual Account.
- Shopee staging app untuk ShopeePay.
- Demo transaksi Gudang Voucher.
- Nusapay staging app dan data uji.
- Data uji Indodana dan ATOME.
- Portal uji Jenius Pay.
- Demo transaksi Tokopedia.

Requirement:
- Gunakan hanya pada Sandbox dan sesuai instruksi dokumentasi yang berlaku.
- Jangan gunakan kredensial atau nomor uji sebagai data produksi.
- Jangan menganggap daftar data uji menjamin merchant telah mengaktifkan channel terkait.
- Pengujian nyata harus mencatat environment, channel, tanggal, hasil, dan bukti.
- Tidak ada transaksi Sandbox/Production yang dijalankan dalam audit dokumentasi ini.

### 6.6 Status akhir bagian errors/reconciliation

| Area | Status |
|---|---|
| Daftar HTTP error | Tercatat |
| Pemisahan HTTP error dan business result | Dikunci |
| Callback result code conflict | Open clarification |
| Redirect result code | Tercatat, non-authoritative |
| Callback resend maksimum lima kali | Tercatat |
| Dashboard callback resend | Tercatat |
| POP inquiry/status endpoint | Belum terverifikasi |
| Sandbox transaction test | Belum dijalankan |

---

## 7. Sumber tahap 1

Dibaca langsung dari [Duitku POP Bahasa Indonesia](https://docs.duitku.com/pop/id/), bagian:
- Integrasi Backend atau Server → Create Invoice
- Endpoint
- Request Headers
- Request Parameter
- Response Parameter
- Obyek JSON → Item Details
- Obyek JSON → Customer Detail
- Obyek JSON → Address
- Obyek JSON → Credit Card Detail

Referensi halaman: https://docs.duitku.com/pop/id/

## 8. Change control

- Requirement yang berstatus LOCKED adalah kontrak acuan untuk audit dan desain berikutnya, bukan izin otomatis untuk implementasi.
- Jangan mengubah schema database, RLS, grants, Edge Functions, secrets, atau deployment berdasarkan dokumen ini tanpa persetujuan ruang lingkup implementasi yang eksplisit.
- Jika halaman resmi berubah, audit ulang bagian terdampak dan catat tanggal serta perubahannya.
- Jika dokumentasi ambigu atau contoh bertentangan dengan tabel, tandai sebagai open clarification; jangan mengarang keputusan.
