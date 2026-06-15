# FINAL PROJECT TEKNOLOGI KOMPUTASI AWAN 2026

## A. Cakupan Capaian Pembelajaran Mata Kuliah (CPMK)

1. Mampu memahami dan menerapkan berbagai servis pada layanan awan.
2. Mampu merancang, mengimplementasikan, dan mengelola aplikasi terdistribusi berbasis komputasi awan.

---

## B. Constraint Final Project

1. FP ini dikerjakan secara berkelompok.
2. Lingkungan cloud yang digunakan (pilih salah satu):

   - `Google Cloud Platform (Credit 300$)`

     - Boleh memanfaatkan semua fitur yang ada
     - Harga sesuai dengan yang disediakan provider
   - `Digital Ocean (Credit 200$)`

     - Boleh memanfaatkan semua fitur yang ada
     - Harga sesuai dengan yang disediakan provider
   - `Microsoft Azure (Credit 100$)`

     - Boleh memanfaatkan semua fitur yang ada
     - Harga sesuai dengan pricing calculator Azure
   - `Local Virtual Machine (VirtualBox/Vagrant) sebagai alternatif simulasi cloud`

     - Apabila membuat lebih dari 1 VM, maka VM harus dibuat **minimal** dari 2 komputer/host yang berbeda
     - Hanya boleh membuat VM dengan spesifikasi sebagai berikut:
       | No | Tipe | CPU   | Memory | Harga/bulan |
       | -- | ---- | ----- | ------ | ----------- |
       | 1  | vm1  | 1vCPU | 512 MB | 4 US$       |
       | 2  | vm2  | 1vCPU | 1 GB   | 6 US$       |
       | 3  | vm3  | 1vCPU | 2 GB   | 12 US$      |
       | 4  | vm4  | 2vCPU | 2 GB   | 18 US$      |
       | 5  | vm5  | 2vCPU | 4 GB   | 24 US$      |
       | 6  | vm6  | 4vCPU | 8 GB   | 48 US$      |

---

## C. Permasalahan

Anda adalah seorang lulusan Teknologi Informasi yang bekerja sebagai Cloud Engineer di sebuah perusahaan rintisan (startup) bidang e-commerce. Perusahaan tersebut sedang mengembangkan platform jual-beli online dan membutuhkan backend **Order Processing Service** — layanan inti yang menangani pembuatan pesanan, pengecekan status, dan riwayat transaksi.

Sebagai Cloud Engineer, Anda diminta untuk **mendeploy, mengonfigurasi, dan mengoptimalkan** layanan tersebut di atas infrastruktur cloud agar dapat menangani lonjakan traffic (flash sale, promo, dsb.) dengan andal dan efisien.

### Spesifikasi Aplikasi

Backend disediakan dalam bentuk REST API berbasis **Python (Flask)** dengan database **MongoDB**. Source code tersedia di folder `Resources/BE/app.py`.

#### Endpoints

**1. Create Order**

- **Endpoint:** `POST /order`
- **Deskripsi:** Membuat pesanan baru. Sistem akan menyimpan data pesanan beserta timestamp dan status awal `"pending"`.
- **Request Body:**
  ```json
  {
    "product": "Nama Produk",
    "quantity": 2,
    "price": 150000
  }
  ```
- **Response (201 Created):**
  ```json
  {
    "order_id": "<uuid>",
    "product": "Nama Produk",
    "quantity": 2,
    "price": 150000,
    "total": 300000,
    "status": "pending",
    "created_at": "2025-06-15T10:00:00Z"
  }
  ```

**2. Get Order Status**

- **Endpoint:** `GET /order/<order_id>`
- **Deskripsi:** Mengambil status dan detail sebuah pesanan berdasarkan `order_id`.
- **Response (200 OK):**
  ```json
  {
    "order_id": "<uuid>",
    "product": "Nama Produk",
    "quantity": 2,
    "price": 150000,
    "total": 300000,
    "status": "pending",
    "created_at": "2025-06-15T10:00:00Z"
  }
  ```
- **Response (404 Not Found):**
  ```json
  { "error": "Order not found" }
  ```

**3. Get Order History**

- **Endpoint:** `GET /orders`
- **Deskripsi:** Mengambil seluruh riwayat pesanan, diurutkan dari yang paling baru.
- **Response (200 OK):**
  ```json
  [
    {
      "order_id": "<uuid>",
      "product": "Nama Produk",
      "quantity": 2,
      "price": 150000,
      "total": 300000,
      "status": "pending",
      "created_at": "2025-06-15T10:00:00Z"
    },
    { "...": "..." }
  ]
  ```

**4. Update Order Status**

- **Endpoint:** `PUT /order/<order_id>`
- **Deskripsi:** Mengubah status pesanan (misalnya dari `"pending"` menjadi `"processing"` atau `"completed"`).
- **Request Body:**
  ```json
  { "status": "completed" }
  ```
- **Response (200 OK):**
  ```json
  {
    "order_id": "<uuid>",
    "status": "completed"
  }
  ```

---

Selain backend, disediakan pula **Frontend** sederhana (`Resources/FE/index.html` dan `styles.css`) yang memungkinkan pengguna membuat pesanan, melihat status, dan menelusuri riwayat transaksi melalui antarmuka berbasis web.

---

Dengan **budget maksimal 1.3 juta rupiah per bulan (≈ 75 US$)**, rancang dan implementasikan arsitektur cloud terbaik yang mampu menerima request tertinggi dengan stabil

## D. Output Final Project dan Penilaian

### 1. Rancangan Arsitektur Cloud (20%)

- Buat diagram arsitektur cloud menggunakan [draw.io](https://app.diagrams.net/) yang menggambarkan komponen-komponen berikut:
  - VM/instance untuk aplikasi backend
  - Load balancer (jika digunakan)
  - Database server (MongoDB)
  - Frontend server atau CDN (opsional)
  - tambahan komponen lain jika diperlukan
- Sertakan **tabel spesifikasi** setiap VM beserta harga per bulan dan total biaya keseluruhan.
- Jelaskan alasan pemilihan konfigurasi tersebut ditinjau dari sisi performa dan efisiensi biaya.

### 2. Implementasi dan Pengujian Aplikasi (20%)

- Deploy seluruh komponen (backend, database, frontend) sesuai rancangan arsitektur.
- Pastikan **semua endpoint dapat diakses dan berfungsi dengan benar**.
- Dokumentasikan hasil pengujian setiap endpoint menggunakan **Postman** atau tools sejenis, disertai screenshot respons.
- Tampilkan screenshot antarmuka frontend yang sudah berjalan.

### 3. Load Testing dengan Locust (35%)

Jalankan Locust menggunakan file `Resources/Test/locustfile.py` dengan ketentuan berikut:

- Locust **harus dijalankan dari komputer/host yang berbeda** dari server aplikasi.
- Hapus isi database yang **di insert di setiap skenario** pengujian agar tidak terjadi akumulasi data. (tidak diperkenankan hapus isi database awal)
- Lakukan pengujian dengan **5 skenario** berikut:| No | Skenario                           | Parameter                                                     | Durasi   |
  | -- | ---------------------------------- | ------------------------------------------------------------- | -------- |
  | 1  | Maksimum RPS (0% failure)          | Naikkan user secara bertahap                                  | 60 detik |
  | 2  | Peak Concurrency – Spawn Rate 50  | Tingkatkan user hingga failure muncul, catat nilai sebelumnya | 60 detik |
  | 3  | Peak Concurrency – Spawn Rate 100 | Sama seperti di atas                                          | 60 detik |
  | 4  | Peak Concurrency – Spawn Rate 200 | Sama seperti di atas                                          | 60 detik |
  | 5  | Peak Concurrency – Spawn Rate 500 | Sama seperti di atas                                          | 60 detik |
- Untuk **Skenario 1**: Catat **rata-rata RPS** tertinggi dengan tingkat kegagalan 0%.
- Untuk **Skenario 2–5**: Catat jumlah **concurrent user** tertinggi yang masih dapat dilayani dengan failure 0%.
- Sertakan screenshot hasil Locust (grafik RPS, response time, failure rate) dan screenshot resource utilization (CPU, memory) server selama pengujian.

### 4. Dokumentasi Laporan di GitHub (25%)

Buat laporan dalam format **Markdown** yang dipublish di repository GitHub kelompok, dengan struktur sebagai berikut:

1. **Introduction** — Jelaskan latar belakang dan permasalahan (dapat mereferensi ke soal ini)
2. **Arsitektur Cloud** — Gambar diagram arsitektur dan tabel harga/spesifikasi VM
3. **Implementasi** — Langkah-langkah konfigurasi secara detail (instalasi Flask, MongoDB, konfigurasi Nginx/load balancer, dll.) disertai screenshot
4. **Hasil Pengujian Endpoint** — Screenshot Postman untuk setiap endpoint dan tampilan antarmuka frontend
5. **Hasil Load Testing** — Screenshot dan analisis hasil Locust untuk kelima skenario
6. **Kesimpulan dan Saran** — Analisis mendalam terhadap hasil FP dan rekomendasi untuk deployment nyata di masa depan

---

## E. Tips and Tricks

1. **Mulai dari konfigurasi terkecil** — Deploy dengan 1 VM terlebih dahulu, ukur baseline performa, baru lakukan scale-out.
2. **Optimalkan sebelum scale** — Sebelum menambah VM, pastikan konfigurasi aplikasi (worker Gunicorn, connection pool MongoDB) sudah optimal untuk VM yang ada.
3. **Eksplorasi load balancing** — Coba berbagai strategi: round-robin, least connection, atau weighted. Bandingkan hasilnya.
4. **Monitor resource secara real-time** — Gunakan `htop`, `vmstat`, atau monitoring bawaan cloud provider selama load testing berlangsung.
5. **Pisahkan database dari app server** — Menempatkan MongoDB di VM terpisah biasanya meningkatkan performa secara signifikan.
6. **Bersihkan database sebelum setiap skenario Locust** — Data yang menumpuk di collection `orders` akan memperlambat query `GET /orders`.
7. **Eksplorasi indexing MongoDB** — Tambahkan index pada field `created_at` atau `order_id` untuk mempercepat query history.
8. **Berpikir out of the box** — Manfaatkan semua yang telah dipelajari di kelas dan praktikum: provisioning otomatis, Ansible, Docker, dan sebagainya.

---

## F. Teknis Pengumpulan dan Penilaian

### Pengumpulan

1. Kumpulkan link repository GitHub kelompok melalui spreadsheet jadwal demo & pengumpulan final di: [Spreadsheet Jadwal Demo &amp; Pengumpulan FP](https://docs.google.com/spreadsheets/d/1_Bw4N_qTTn6DVUIoRETbTm5fuZdBaEYqyptDmtFxkgI/edit?usp=sharing)
2. Repository harus bersifat **public** dan berisi:
   - Source code backend (`app.py`) dan frontend
   - Locustfile yang digunakan
   - `README.md` berisi laporan lengkap
3. Batas waktu pengumpulan: **[minggu 17**. Penilaian diambil dari commit terakhir sebelum deadline.

### Rubrik Penilaian

| Komponen                          | Bobot | Detail                                                        |
| --------------------------------- | ----- | ------------------------------------------------------------- |
| Rancangan Arsitektur              | 20%   | Diagram arsitektur (10 poin) + rancangan harga (10 poin)      |
| Implementasi & Pengujian Endpoint | 20%   | Teknis implementasi (10 poin) + pengujian endpoint (10 poin)  |
| Load Testing Locust               | 35%   | Maksimum RPS (30 poin) + peak concurrency 4 skenario (5 poin) |
| Dokumentasi Laporan               | 25%   | Kelengkapan & kualitas tiap bagian laporan                    |

> **Catatan Penilaian RPS:**
> Nilai dihitung berdasarkan rata-rata RPS tertinggi dengan 0% failure.
> Contoh: **aggregat RPS** = 120 → Nilai = (120/200) × 30 = **18 poin**

---

## Catatan Khusus

- Nilai dianggap sama untuk seluruh anggota tim, kecuali ada laporan dari anggota tim bahwa ada yang tidak berkontribusi.
- Segala bentuk kecurangan (plagiarisme laporan, manipulasi hasil screenshot, dsb.) akan berdampak pada pengurangan nilai.
- **JANGAN LUPA DESTROY SEMUA RESOURCES SETELAH FP BERAKHIR.**

---

## Lampiran: Struktur Repository yang Disarankan

```
fp-tka-[nama-kelompok]/
├── README.md               ← Laporan utama
├── Resources/
│   ├── BE/
│   │   └── app.py          ← Backend Flask
│   ├── FE/
│   │   ├── index.html
│   │   └── styles.css
│   └── Test/
│       └── locustfile.py   ← Script load testing
└── result/
    ├── locust_rps.png
    ├── locust_concurrency_*.png
    └── cpu_usage_*.png
```
