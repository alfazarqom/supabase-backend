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
| 4 | Expiry dan status transaksi | PENDING |
| 5 | Payment Method dan fitur khusus | PENDING |
| 6 | Errors dan rekonsiliasi | PENDING |

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
4. Dokumentasi menyebut maksimum lima kali pengiriman ulang, tetapi tidak menetapkan interval retry pada bagian yang diperiksa.

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

## 4. Sumber tahap 1

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

## 5. Change control

- Requirement yang berstatus LOCKED adalah kontrak acuan untuk audit dan desain berikutnya, bukan izin otomatis untuk implementasi.
- Jangan mengubah schema database, RLS, grants, Edge Functions, secrets, atau deployment berdasarkan dokumen ini tanpa persetujuan ruang lingkup implementasi yang eksplisit.
- Jika halaman resmi berubah, audit ulang bagian terdampak dan catat tanggal serta perubahannya.
- Jika dokumentasi ambigu atau contoh bertentangan dengan tabel, tandai sebagai open clarification; jangan mengarang keputusan.
