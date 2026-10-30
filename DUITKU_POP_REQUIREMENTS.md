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
| 2 | HTTP Callback | PENDING |
| 3 | JS Callback dan Return URL | PENDING |
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

## 2. Sumber tahap 1

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

## 3. Change control

- Requirement yang berstatus LOCKED adalah kontrak acuan untuk audit dan desain berikutnya, bukan izin otomatis untuk implementasi.
- Jangan mengubah schema database, RLS, grants, Edge Functions, secrets, atau deployment berdasarkan dokumen ini tanpa persetujuan ruang lingkup implementasi yang eksplisit.
- Jika halaman resmi berubah, audit ulang bagian terdampak dan catat tanggal serta perubahannya.
- Jika dokumentasi ambigu atau contoh bertentangan dengan tabel, tandai sebagai open clarification; jangan mengarang keputusan.
