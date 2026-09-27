# POS KASIR by FAKHRUL ZAEIN — POS + ERP Mini F&B + CMS Company Profile

Bantuan: 0857-0178-4952

Sistem kasir, dashboard manajemen, dan website company profile untuk bisnis F&B.
**100% native**: tanpa framework frontend (React/Vue/Tailwind) dan tanpa framework backend (Express/Nest).
Dependensi npm hanya `mysql2` (driver database) dan `nodemailer` (kirim backup via email).

---

## 1. Instalasi untuk klien baru (±10 menit)

**Syarat:** Node.js 18+ dan MySQL 5.7+/8.x (atau MariaDB 10.4+).

```bash
# 1. Salin folder proyek ke komputer/server klien, lalu:
npm install

# 2. Buat file konfigurasi
cp .env.example .env        # Windows: copy .env.example .env
#    Edit .env → isi DB_USER, DB_PASSWORD, DB_NAME, ADMIN_PASSWORD, SITE_URL

# 3. Buat database, tabel, akun admin, dan website awal
npm run setup

# 4. (opsional) Isi data contoh untuk demo ke calon klien
npm run seed:demo           # login kasir demo: kasir1 / kasir123

# 5. Jalankan
npm start
```

| Halaman | URL |
|---|---|
| Layar kasir | `http://localhost:3000/kasir` |
| Dashboard admin | `http://localhost:3000/admin` |
| Website publik | `http://localhost:3000/` |

Perintah lain: `npm run dev` (auto-restart saat kode diubah), `npm run backup` (backup manual), `npm run build:site` (bangun ulang website),
`npm run seed:history` (riwayat penjualan 14 hari palsu untuk demo — **jangan** di database klien).

### Pasang online di VPS (disarankan untuk klien)
Satu perintah di VPS Ubuntu 22.04/24.04 — memasang Node.js, MySQL, Nginx, HTTPS Let's Encrypt, firewall, dan systemd:
```bash
sudo bash deploy/install.sh --name kedaiku --domain kasir.kedaiku.com --email pemilik@gmail.com
sudo bash deploy/update.sh --name kedaiku      # update ke versi baru (backup otomatis + rollback)
sudo bash deploy/list.sh                       # status semua klien di VPS
```
Satu VPS bisa menampung banyak klien (jalankan install dengan `--name`/`--domain` berbeda).
Panduan lengkap (bagian pemilik + bagian teknisi): **`docs/panduan-deploy-vps.html`**.

### Panduan untuk pengguna
**`docs/panduan-pengguna.html`** — satu file HTML utuh (bisa dibuka offline, dikirim via WhatsApp/email) untuk kasir & pemilik.
Versi yang sama bisa dibuka dari tombol **Panduan** di layar kasir dan dashboard admin (`/panduan`, wajib login).

---

## 2. Struktur folder

```text
native-js-pos/
├── backend/
│   ├── server.js            Entry point: HTTP server + static server + guard halaman
│   ├── routes.js            Router Map: 'METHOD /path' → controller (tanpa if-else)
│   ├── middleware.js        withAuth / withAdminAuth (HOF), sesi cookie+DB, anti-CSRF
│   ├── database.js          Pool MySQL + helper query/transaction (prepared statement)
│   ├── cron.js              setInterval: backup harian, artikel terjadwal, bersih sesi
│   ├── ssg.js               Static Site Generator website publik
│   ├── backup.js            Dump database .sql.gz native + email + retensi
│   ├── config.js            Pembaca .env native (tanpa dotenv)
│   ├── lib/                 http, static, security (scrypt), validate, settings, shift
│   └── controllers/
│       ├── authController.js     Login, logout, ganti password, rate-limit
│       ├── posController.js      Kasir: checkout, pajak, promo, shift, petty cash
│       ├── adminController.js    CRUD produk, kategori, promo, pelanggan, user, pengaturan, upload
│       ├── reportController.js   Dashboard, laporan, CSV, transaksi & void, arus kas, shift
│       ├── stockController.js    Bahan baku & stock opname
│       ├── cmsController.js      Artikel blog (memicu SSG)
│       ├── backupController.js   Daftar/jalankan/unduh backup
│       └── sseController.js      Server-Sent Events real-time
├── frontend/
│   ├── login/               Halaman login
│   ├── panduan/             Panduan pengguna (dibangun dari docs/src, wajib login)
│   ├── kasir/               Layar POS (index.html + pos.js)
│   ├── admin/               Dashboard admin (admin.js = router hash, pages/*.js = tiap menu)
│   ├── public/              HASIL SSG — jangan diedit manual (ditimpa otomatis)
│   └── assets/
│       ├── style.css        CSS master aplikasi
│       ├── site.css         CSS website publik
│       ├── app.js           Utilitas frontend (api, esc, rupiah, modal <dialog>)
│       ├── calc.js          Rumus pajak/promo — DIPAKAI BERSAMA kasir & server
│       ├── receipt.js       Render & cetak struk thermal
│       └── uploads/         Gambar yang diunggah admin
├── database/schema.sql      Skema lengkap (relasi, index, default setting)
├── scripts/                 setup, seed-demo, seed-history, backup-now, build-site
├── deploy/                  install.sh, update.sh, list.sh + template Nginx & systemd
├── docs/                    panduan-pengguna.html, panduan-deploy-vps.html (+ src/ untuk membangun ulang)
└── backups/                 File backup .sql.gz
```

---

## 3. Arsitektur & keputusan teknis

### Backend
- **Router Map** — semua rute ada di satu object di `routes.js`. Rute statis dicari O(1), rute ber-parameter (`/api/admin/products/:id`) dikompilasi sekali jadi RegExp saat start.
- **Middleware HOF** — `withAuth(handler)` dan `withAdminAuth(handler)` membungkus controller.
- **Sesi** — token acak 32 byte di cookie `HttpOnly; SameSite=Strict`. Database hanya menyimpan **hash SHA-256** token (`sessions.cookie_hash`), jadi token tidak bisa dipakai walau database bocor.
- **Anti-CSRF** — semua request API yang mengubah data wajib membawa header `X-Requested-With: fetch` (ditambahkan otomatis oleh `app.js`).
- **Brute force** — login dikunci 15 menit setelah 5 kali salah per IP+username.
- **SQL Injection** — semua query memakai prepared statement (`pool.execute`). LIMIT/OFFSET hanya dari angka yang sudah divalidasi.
- **XSS** — semua data di-escape sebelum masuk `innerHTML`, ditambah Content-Security-Policy `script-src 'self'` (script inline & `onclick=` di HTML tidak akan jalan).
- **Upload** — gambar dikirim base64 JSON; jenis file diverifikasi dari *magic bytes*, bukan nama file. Maks 3 MB.

### Logika bisnis F&B
| Fitur | Implementasi |
|---|---|
| Pajak Opsi 1 (Eksklusif) | `Total = (Subtotal − Diskon) + Pajak` |
| Pajak Opsi 2 (Inklusif) | `Total = Harga − Diskon`, pajak = `Total × tarif / (100 + tarif)` |
| Satu sumber rumus | `frontend/assets/calc.js` dipakai kasir (tampilan) **dan** server (checkout). Harga & total dari browser tidak dipercaya. |
| Harga terkunci | `transaction_details.price_at_transaction` + `cost_at_transaction` + `product_name` disalin saat transaksi |
| Stok barang jadi/kemasan | `products.track_stock = 1` → berkurang otomatis, baris dikunci `FOR UPDATE` agar tidak oversell |
| Bahan baku | Manual: Stok Masuk + **Stock Opname** (selisih fisik tercatat sebagai nilai susut Rp) |
| Shift & laci | Buka shift (modal awal) → penjualan tunai, petty cash, void tunai tercatat di `cash_flows` → tutup shift: uang seharusnya vs fisik, selisih tersimpan |
| Petty cash | Dari layar kasir, ditolak bila melebihi uang di laci |
| Void | Admin saja; stok dikembalikan, arus kas dibalik, transaksi tetap tersimpan (audit) |
| Anti transaksi ganda | `client_ref` (UUID) unik per transaksi; klik "Bayar" dua kali tidak membuat 2 transaksi |
| Hapus data berhistori | Produk/promo/bahan yang sudah dipakai → dinonaktifkan, bukan dihapus |

### Real-time (SSE)
`GET /api/stream` ditahan terbuka. Event: `transaction`, `void`, `petty_cash`, `shift`, `low_stock`.
Dashboard admin langsung menambah transaksi ke feed dan memperbarui angka omzet tanpa refresh.

### Website publik (SSG)
Setiap kali artikel, produk, kategori, atau pengaturan website disimpan, `ssg.js` membangun ulang
`index.html`, `articles/*.html`, `sitemap.xml`, `robots.txt`, dan `404.html` sebagai **file fisik**.
Pengunjung tidak memicu query database sama sekali. SEO: title, meta description, canonical, Open Graph, JSON-LD (Restaurant & Article).
Artikel dengan jadwal terbit di masa depan diterbitkan otomatis oleh cron.

### Backup
Setiap hari pada jam `BACKUP_TIME`, cron membuat `backups/backup-YYYYMMDD-HHMMSS.sql.gz` memakai snapshot konsisten
(aman saat kasir sedang transaksi), mengirim ke email bila SMTP diisi, dan menghapus file yang lebih tua dari `BACKUP_RETENTION_DAYS`.

**Restore:**
```bash
gunzip backup-20260926-233000.sql.gz
mysql -u root -p native_pos < backup-20260926-233000.sql
```

---

## 4. Penyesuaian dari dokumen spesifikasi awal

| Spesifikasi | Implementasi | Alasan |
|---|---|---|
| `bcrypt` untuk hash password | `crypto.scrypt` bawaan Node.js | Setara keamanannya, tanpa kompilasi native yang sering gagal di komputer kasir Windows |
| 3 controller | 8 controller | `adminController` akan terlalu besar; dipecah per domain agar mudah dirawat tim |
| `transactions.date` | `transactions.created_at` | Konsisten dengan semua tabel lain |
| Backup dikirim ke "Cloud" | Email (nodemailer) + folder lokal | Tidak butuh SDK cloud; file email mudah disimpan ke Drive klien |
| — | Tabel tambahan: `settings`, `categories`, `shifts`, `stock_movements`, `raw_material_movements`, `stock_opnames`, `stock_opname_items`, `backup_logs` | Dibutuhkan untuk shift kasir, riwayat stok, dan opname yang bisa diaudit |
| `innerHTML +=` | `innerHTML =` per bagian + event delegation | `+=` membuat ulang seluruh DOM dan menghapus listener; hasilnya sama-sama native |

---

## 5. Kustomisasi per klien

- **Nama usaha, alamat, struk, pajak, metode bayar** → Admin › Pengaturan (tanpa ubah kode)
- **Warna tema aplikasi** → variabel `--primary` di `frontend/assets/style.css`
- **Warna website** → variabel `--c-primary` di `frontend/assets/site.css`
- **Template website** → fungsi `layout()` dan `buildHome()` di `backend/ssg.js`
- **Kategori petty cash** → `PETTY_CATEGORIES` di `posController.js`

## 6. Akun & peran
- **admin** — semua menu + layar kasir
- **kasir** — hanya layar kasir (transaksi, shift, petty cash, cetak ulang struk miliknya)
