# Pedoman Sistem Perkebunan (FARMease Gardening System Guide)

Dokumen ini merupakan pedoman teknis arsitektur, instalasi, pengoperasian, dan pengujian untuk ekosistem sistem **FARMease** khususnya modul **Perkebunan (Kebun)** yang terintegrasi dengan modul Peternakan (Ternak) dan Single Sign-On (SSO).

---

## 1. Arsitektur & Komponen Sistem

Modul Perkebunan dirancang menggunakan pendekatan layanan terpisah (multi-service) berbasis **Go (native backend)** dan **Vue 3 (Vite + TSX frontend)**. Layanan ini berkomunikasi dengan SSO untuk otentikasi serta modul Peternakan secara asinkron menggunakan RabbitMQ.

```
                  ┌─────────────────┐
                  │   SSO Portal    │ (Vue 3, Port 3000)
                  └────────┬────────┘
                           │ Authenticate
                           ▼
                  ┌─────────────────┐
                  │   SSO Backend   │ (Go, Port 8080)
                  └────────┬────────┘
                           │
       ┌───────────────────┼───────────────────┐
       ▼ Session Tokens    ▼ Session Tokens    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Kebun Portal │   │  SSO Database │   │  Ternak Portal│ (Vue 3, Port 3001 & 3002)
└───────┬───────┘   └───────────────┘   └───────┬───────┘
        │                                       │
        ▼ HTTP REST                             ▼ HTTP REST
┌───────────────┐                       ┌───────────────┐
│ Kebun Backend │                       │Ternak Backend │ (Go, Port 8081 & 8082)
└───────┬───────┘                       └───────┬───────┘
        │                                       │
        │           ┌─────────────────┐         │
        └──────────►│   RabbitMQ      │◄────────┘
     Publish Event  │(Message Broker) │ Consume Event
                    └─────────────────┘
```

### Detail Komponen:
1. **Perkebunan / Kebun Portal & Backend (Port 3002 & 8082)**: Portal utama bagi Operator Kebun dan Admin untuk mencatat pengelolaan lahan perkebunan, tanaman/pohon, rekam aktivitas harian (penyiraman, pembersihan, pemupukan/pengobatan, pemangkasan), serta panen buah.
2. **SSO Portal & Backend (Port 3000 & 8080)**: Mengelola sesi login pengguna, peran (Owner, Admin, Operator), dan memvalidasi akses ke modul Perkebunan.
3. **RabbitMQ (Port 5672 & 15672)**: Bertanggung jawab mengirimkan event integrasi saat aktivitas pemangkasan menghasilkan sisa panen (misal: daun/rumput potong) untuk secara otomatis ditambahkan ke stok pakan peternakan.
4. **PostgreSQL (Port 5435)**: Menyimpan skema database perkebunan di database `farmease_kebun`.
5. **Redis (Port 6381)**: Menyimpan sesi otentikasi.

---

## 2. Struktur Proyek & Repositori (Project & Repository Structure)

Untuk membantu pengembang baru memahami organisasi berkas dan arsitektur kode modul Perkebunan, berikut adalah penjelasan detail mengenai struktur direktori repositori, backend, dan frontend.

### 2.1. Peta Direktori Workspace (Workspace Directory Map)
Repositori ini dirancang sebagai monorepo/multi-service workspace yang membagi modul peternakan, perkebunan, dan Single Sign-On (SSO):
- [Farmease-BE](../Farmease-BE): Backend untuk modul Peternakan (Ternak) [Go].
- [Farmease](../Farmease): Frontend/Portal untuk modul Peternakan (Ternak) [Vue 3].
- [sso-be](../sso-be): Backend untuk layanan SSO (Otentikasi & Sesi) [Go].
- [Farmease_sso](../Farmease_sso): Frontend/Portal untuk layanan SSO [Vue 3].
- [kebun-be](../kebun-be): Backend untuk modul Perkebunan (Kebun) [Go Workspace].
- [Farmease_kebun](../Farmease_kebun): Frontend/Portal untuk modul Perkebunan (Kebun) [Vue 3 + TSX].
- [docs](../docs): Berkas dokumentasi perancangan, diagram kelas, dan pedoman sistem.

---

### 2.2. Arsitektur Go Backend (`kebun-be`)
Backend modul perkebunan diorganisasikan menggunakan **Go Workspaces** (`go.work`) untuk memisahkan kode domain dengan kerangka kerja (framework) dan pustaka (libraries).

#### Visualisasi Struktur Folder Backend:
```text
kebun-be/
├── framework/            # Inisialisasi Fiber, Middleware, Logging, bunnymq
├── libraries/            # Utilitas reusable (publisher & consumer RabbitMQ)
├── go.work               # Go workspace configuration
└── kebun/                # Aplikasi utama Perkebunan
    ├── cmd/              # CLI entrypoints (serve, seed)
    ├── config/           # Konfigurasi aplikasi
    ├── migrations/       # SQL migrations (26+ migrasi)
    ├── seeders/          # Seeder data
    └── module/           # Domain modules (Clean Architecture)
        └── lahan/        # Contoh Modul Lahan
            ├── domain/   # Interface & Entitas Data Lahan
            ├── usecase/  # Logika Bisnis Lahan
            ├── repository/# Implementasi akses DB Lahan
            ├── delivery/ # HTTP Handler & Router Lahan (Fiber)
            └── module.go # FX Dependency Injection Wiring
```

#### Struktur Utama `kebun-be`:
1. `go.work`: Berkas konfigurasi workspace yang menyatukan sub-modul `./framework`, `./libraries`, dan `./kebun`.
2. [framework](../kebun-be/framework): Berisi implementasi boilerplate pemrosesan aplikasi tingkat tinggi seperti inisialisasi server Fiber, middleware, logging, konfigurasi Consul, koneksi database, dan RabbitMQ (`bunnymq`).
3. [libraries](../kebun-be/libraries): Pustaka utilitas pembantu yang reusable, seperti publisher dan consumer RabbitMQ.
4. [kebun](../kebun-be/kebun): Aplikasi utama perkebunan.
   - `cmd/`: Titik masuk eksekusi program (entrypoint CLI) seperti `serve` untuk menjalankan API server dan `seed` untuk data pengujian.
   - `config/`: Berkas konfigurasi spesifik perkebunan.
   - `migrations/`: Berkas SQL migrasi skema database PostgreSQL.
   - `seeders/`: Injeksi data awal (seeder) untuk pengujian.
   - `module/`: Implementasi logika domain perkebunan yang menggunakan **Clean Architecture** yang didekorasi dengan Dependency Injection **Uber Fx** (`go.uber.org/fx`).

#### Clean Architecture Layer di `kebun/module/<domain_name>`:
Setiap modul di bawah folder `module/` (seperti `lahan`, `pohon`, `pemangkasan`, `panen`, dll) mengikuti struktur lapisan berikut:
- **`domain/`**: Berisi definisi kontrak berupa interface repositori & usecase serta struktur data (entities/models) inti domain tersebut. Lapisan ini tidak memiliki dependensi ke pustaka eksternal (framework agnostic).
- **`usecase/`**: Berisi implementasi dari interface usecase (logika bisnis utama). Lapisan ini memproses aturan bisnis, memanggil repositori, dan memicu event eksternal (seperti memublikasikan event sisa panen ke RabbitMQ).
- **`repository/`**: Berisi implementasi dari interface repositori (akses data ke PostgreSQL menggunakan SQL native atau helper DB).
- **`delivery/`**: Lapisan luar yang berhadapan dengan klien. Biasanya berupa sub-direktori `http/` yang berisi HTTP handler dan pendaftaran router menggunakan framework Fiber.
- **`module.go`**: Berkas integrasi Uber Fx. Berkas ini menggunakan `fx.Module` untuk mendaftarkan konstruktor repository, usecase, handler, serta router group ke dalam siklus hidup (lifecycle) aplikasi agar dapat diinjeksi secara otomatis saat server dinyalakan.

---

### 2.3. Arsitektur Vue 3 Frontend (`Farmease_kebun`)
Frontend perkebunan dibangun menggunakan Vue 3, Vite, dan TypeScript. Seluruh halaman utama dan fungsionalitas UI diimplementasikan menggunakan syntax **TSX (TypeScript XML)**, bukan template SFC `.vue` standar.

#### Visualisasi Struktur Folder Frontend:
```text
Farmease_kebun/
├── src/
│   ├── modules/          # Folder modular berbasis peran/fitur
│   │   ├── admin/        # Menu & Dasbor Admin
│   │   ├── kebun/        # Menu Operator Kebun
│   │   │   ├── assets/   # Stylesheet khusus (.css)
│   │   │   ├── components/# Komponen UI khusus kebun
│   │   │   ├── layouts/  # Tata letak / navbar kebun
│   │   │   ├── views/    # Halaman utama kebun (.tsx)
│   │   │   ├── router.ts # Sub-rute kebun
│   │   │   └── index.ts  # Ekspor modul kebun
│   │   └── pemilik/      # Menu Pemilik Kebun
│   ├── shared/           # Asset dan kode yang digunakan bersama
│   │   ├── api/          # HTTP Clients & Services (auth, perkebunan, peternakan)
│   │   ├── ui/           # Komponen UI reusable (buttons, modals)
│   │   └── composables/  # Vue Composables
│   ├── store/            # State management (Pinia)
│   ├── router/           # Routing utama & SSO Auth Guard (index.ts)
│   ├── App.tsx           # Komponen root aplikasi
│   └── main.ts           # Titik masuk aplikasi
├── vite.config.ts        # Konfigurasi Vite
└── package.json          # Dependensi frontend
```

#### Struktur Utama `Farmease_kebun/src`:
1. [modules](../Farmease_kebun/src/modules): Mengikuti pendekatan modular berbasis fitur/peran pengguna:
   - `admin/`: Modul dasbor dan pengelolaan level tinggi untuk Administrator.
   - `kebun/`: Modul utama pencatatan aktivitas, pohon, dan dasbor lahan perkebunan.
   - `pemilik/`: Modul ringkasan laporan dan performa lahan untuk Pemilik.
2. [shared](../Farmease_kebun/src/shared): Utility bersama yang digunakan lintas modul:
   - `api/`: Berisi API Client (`client.ts`), otentikasi (`auth.ts`), API perkebunan (`perkebunan.ts`), dan API integrasi peternakan (`peternakan.ts`).
   - `ui/`: Komponen UI reusable (button, input, modal, dll).
   - `composables/`: Vue composables fungsi utilitas reaktif.
3. [store](../Farmease_kebun/src/store): State management global (Pinia) untuk mengelola otentikasi sesi dan navigasi.
4. [router](../Farmease_kebun/src/router): Router utama Vue Router yang menggabungkan rute-rute dari masing-masing modul (`adminRoutes`, `kebunRoutes`, `pemilikKebunRoutes`).

#### Detail Organisasi di dalam Modul (`src/modules/kebun/`):
- `views/`: Komponen halaman utama (contoh: `PerkebunanPage.tsx`, `DasborLahanPage.tsx`).
- `components/`: UI sub-komponen (contoh: `KebunGenericFormFields.tsx` untuk field formulir dinamis).
- `layouts/`: Kerangka tata letak halaman (contoh: `KebunLayout.tsx` yang membungkus sidebar navigasi).
- `router.ts`: Mendefinisikan sub-rute modul kebun (seperti `/kebun`, `/kebun/dasbor-lahan`, `/kebun/daftar`, dll).
- `assets/css/`: File stylesheet Vanilla CSS (contoh: `PerkebunanDetailPages.css`).
- `index.ts`: Pintu ekspor modul ke luar.

---

### 2.4. Mekanisme Integrasi & Sesi
1. **SSO Authentication Guard**:
   Sistem otentikasi terintegrasi secara Single Sign-On (SSO). Apabila pengguna membuka modul Perkebunan tanpa token sesi di LocalStorage, guard di [router/index.ts](../Farmease_kebun/src/router/index.ts) akan secara otomatis mengalihkan pengguna ke SSO Portal (`http://localhost:3000/?service=kebun`).
   Setelah login berhasil, SSO akan mengalihkan kembali pengguna ke modul Perkebunan dengan membawa parameter sesi (`token`, `role`, `username`, `code`) di URL Query. Parameter tersebut ditangkap, disimpan ke LocalStorage via `authApi.setAuth`, dan dibersihkan dari bilah URL untuk menjaga keamanan sesi.
2. **Event-Driven Integration (RabbitMQ)**:
   Saat aktivitas pemangkasan (`pemangkasan`) dicatat di backend perkebunan, usecase pemangkasan akan mengevaluasi hasil pangkasan. Jika menghasilkan sisa panen (daun/rumput potong) yang dapat dijadikan pakan ternak, backend perkebunan akan secara otomatis memublikasikan event sisa panen ke RabbitMQ menggunakan client `publisher`.
   Consumer RabbitMQ di modul Peternakan (`Farmease-BE`) akan menangkap event ini secara asinkron dan menambahkan stok pakan ternak secara real-time.

---

## 3. Persiapan Lingkungan (Prerequisites)

Perangkat lunak yang wajib terinstal di perangkat Anda:
- **Docker Desktop** (untuk menjalankan seluruh layanan Database, Redis, RabbitMQ, Backend, dan Frontend di dalam kontainer).
- **Go Compiler** (versi 1.20+ - jika ingin menjalankan di luar Docker/native).
- **Node.js** (versi 20+) & **npm** (jika ingin menjalankan di luar Docker/native).

---

## 4. Konfigurasi Database & Infrastruktur

Seluruh kontainer diatur secara independen menggunakan Docker Compose di dalam sub-direktori masing-masing modul. Database PostgreSQL lokal akan secara otomatis membuat skema database utama saat pertama kali menyala:
- `farmease_kebun` (dikelola oleh kontainer database Perkebunan)

---

## 5. Cara Menjalankan Sistem secara Lokal (Menggunakan Docker Compose)

Modul Perkebunan dan SSO dapat dijalankan baik secara **mandiri** (per modul) maupun secara **bersamaan** dari direktori root proyek menggunakan fitur Docker Compose Shared Network.

### 5.1. Membuat Shared Network Docker (Satu Kali)
Sebelum menjalankan layanan (baik mandiri maupun gabungan), pastikan shared network sudah terbuat di terminal Anda:
```bash
docker network create farmease_shared_network
```

### 5.2. Opsi A: Menjalankan Seluruh Modul Sekaligus (Sangat Direkomendasikan)
Untuk menjalankan seluruh sistem (SSO, Peternakan, dan Perkebunan) sekaligus tanpa harus berpindah direktori, jalankan perintah berikut langsung dari **direktori root proyek**:
```bash
docker compose up -d
```
*Docker Compose di root akan memicu proses `include` untuk membaca file konfigurasi mandiri masing-masing modul dan meluncurkannya bersama-sama.*

### 5.3. Opsi B: Menjalankan Layanan secara Mandiri
Apabila Anda ingin menjalankan modul secara terisolasi (misalnya hanya SSO dan Perkebunan):

1. **Jalankan modul SSO**:
   ```bash
   cd sso-be
   docker compose up -d
   ```
2. **Jalankan modul Perkebunan (Kebun)**:
   Buka terminal baru, masuk ke direktori Perkebunan backend dan jalankan Docker Compose:
   ```bash
   cd kebun-be
   docker compose up -d
   ```

### 5.4. Detail Port Layanan yang Terbuka
- **Portal SSO (Frontend)**: `http://localhost:3000`
- **Portal Perkebunan (Kebun)**: `http://localhost:3002`
- **SSO Backend API**: `http://localhost:8080`
- **Perkebunan Backend API**: `http://localhost:8082`

### 5.5. Menghentikan Layanan
*   Jika dijalankan sekaligus dari root:
    ```bash
    docker compose stop
    ```
*   Jika dijalankan mandiri: Jalankan `docker compose stop` dari dalam direktori modul bersangkutan.

---

## 6. Manajemen Migrasi & Reset Data (Database Migration)

Apabila skema database berubah atau Anda ingin membersihkan database Perkebunan dari data lama untuk memuat data pengujian baru:

### 6.1. Reset dan Jalankan Ulang Database & Seeder
Masuk ke direktori `kebun-be` di terminal Anda dan jalankan perintah:
```bash
# 1. Hentikan seluruh kontainer dan hapus volume database perkebunan
docker compose down -v

# 2. Jalankan kembali kontainer dan jalankan migrasi segar + seeder otomatis
docker compose up -d

# 3. Jalankan penyuntikan seeder secara manual jika diperlukan ulang
docker compose run --rm kebun_seeder
```
*(Lakukan langkah serupa di dalam direktori `sso-be` untuk mereset modul SSO dengan menjalankan seeder `sso_seeder`).*

---

## 7. Pengujian Sistem Perkebunan (Testing Guide)

Pengujian unit backend dan frontend dapat dijalankan langsung menggunakan perintah standar masing-masing teknologi (tanpa skrip wrapper tambahan):

### 7.1. Pengujian Unit Go Backend Perkebunan
1. **Pengujian Unit Seluruh Modul Perkebunan**:
   ```bash
   cd kebun-be/kebun
   go test ./...
   ```
2. **Pengujian Modul Spesifik (misal: lahan)**:
   ```bash
   cd kebun-be/kebun
   go test -v ./module/lahan/usecase
   ```

### 7.2. Pengujian Frontend Vue 3 (di direktori `Farmease_kebun`)
```bash
cd Farmease_kebun
# Menjalankan Uji Unit Frontend
npm run test:unit

# Menjalankan Uji E2E (Playwright)
npm run test:e2e
```

---

## 8. Diagram Alur & Desain Terkait Perkebunan
Untuk memahami struktur logis perangkat lunak perkebunan, silakan merujuk ke berkas dokumentasi di direktori `docs`:
- [Class Diagram Perkebunan](garden_class_diagram.md) - Menjelaskan pemodelan entitas (Lahan, Pohon, Aktivitas, Penyiraman, Pemangkasan, Panen, dll) secara Object-Oriented.
- [Sequence Diagram Modular Perkebunan](garden_module_sequence_diagrams.md) - Alur terperinci interaksi sistem pada setiap Use Case Perkebunan (Melihat Dasbor, Registrasi Pohon, Aktivitas Lahan, Pemangkasan).
- [Integration Guide Perkebunan & Peternakan](../Farmease_kebun/INTEGRATION_GUIDE.md) - Petunjuk integrasi modul perkebunan ke peternakan secara RESTful.

