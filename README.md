# SeedRuns (Minecraft 1.16.1 Seed & Structure Analyzer)

## [Definisi]
**SeedRuns** adalah aplikasi GUI desktop berkinerja tinggi berbasis **Tauri** yang dirancang untuk mendeteksi, membaca, menganalisis file penyimpanan dunia Minecraft (`level.dat`), serta memprediksi posisi struktur penting (khususnya di dimensi Nether dan Overworld). Aplikasi ini menggabungkan kecepatan pemrosesan native dari bahasa **Rust** dan algoritma pencarian struktur berkecepatan tinggi berbasis **Cubiomes (C)** dengan antarmuka web modern yang ringan dan responsif.

---

## [Tujuan]
Proyek ini dibuat untuk kebutuhan personal/latihan (*training practice*) bagi **speedrunner pemula** pada Minecraft versi **1.16.1 (Any% RSG / SSG)** dengan tujuan:
1. Mempermudah pengecekan seed dan struktur penting secara instan dari world yang baru dibuat tanpa harus membuka game secara penuh atau menggunakan tool web berat.
2. Membantu latihan navigasi speedrun dengan menyediakan koordinat akurat dan varian struktur (tipe Bastion, Fortress, Stronghold).
3. Menghemat waktu latihan dengan menyediakan fitur pencarian seed berbasis radius Nether (*two-phase filtering* berkecepatan tinggi) serta pembuatan world baru secara otomatis langsung ke folder save Minecraft.

---

## [Fitur yang Ingin Ditambahkan]

### 1. Manajemen & Pemindaian Save World
- **Manual Reload Button**: Tombol penyegaran manual untuk membaca ulang folder saves guna menghemat penggunaan memori (RAM) dan CPU tanpa file-watching terus-menerus.
- **Auto-Sort by Last Played**: Mengurutkan dunia dari yang paling baru dimainkan/dibuat di urutan paling atas berdasarkan timestamp `LastPlayed`.
- **Custom Directory Selector**: Dukungan kustomisasi path save folder (default: `%APPDATA%/.minecraft/saves`, dengan opsi instance Prism Launcher / MultiMC).
- **World Thumbnail Preview**: Menampilkan `icon.png` dari save folder untuk mempermudah identifikasi dunia.

### 2. Analisis Seed & Struktur
- **Ekstraksi Seed Cepat**: Membaca seed numerik langsung dari file NBT `level.dat` menggunakan parser biner terkompresi.
- **Deteksi Struktur Kritis 1.16.1**:
  - **Bastion Remnant**: Menghitung koordinat Nether beserta jenis variannya (*Bridge*, *Housing*, *Hoglin Stables*, *Treasure*).
  - **Nether Fortress**: Menghitung lokasi benteng terdekat dari titik spawn Nether.
  - **Stronghold**: Menghitung lokasi portal End terdekat di Overworld.
- **Dual Copy Button**:
  - Tombol 1: Salin koordinat mentah (`X Y Z`) untuk kalkulasi/catatan.
  - Tombol 2: Salin perintah teleport in-game (`/execute in minecraft:the_nether run tp @s X Y Z` atau `/tp X Y Z`).

### 3. Generator & Filter Seed Berkecepatan Tinggi
- **Radius-Based Filter**: Mencari seed otomatis dengan batasan jarak maksimum (radius) ke Bastion dan Fortress dari titik tengah Nether $(0, 0)$.
- **Zero-RAM Leaks Looping**: Looping 48-bit *structure seed* langsung di register CPU tanpa memuat chunk/blok ke memori. Seed yang tidak masuk radius langsung dibuang (*discard*).
- **Multi-threaded Background Worker**: Menggunakan `rayon` untuk memindai ribuan/jutaan seed per detik di multi-core CPU tanpa membuat GUI macet (*freeze*).

### 4. Visualisasi & Pembuat World (World Creator)
- **2D Map Preview**: Menampilkan visualisasi peta 2D ringkas menggunakan binding library `cubiomes`.
- **One-Click Create World**: Membuat folder save baru lengkap dengan file `level.dat` standar 1.16.1 yang siap dimainkan langsung di game.

---

## [Cara Menggunakannya]
1. Jalankan file eksekusi aplikasi (`SeedRuns.exe`).
2. Aplikasi akan memindai folder `.minecraft/saves` dan menampilkan daftar dunia yang tersedia.
3. Klik tombol **Reload** jika baru saja membuat world baru di dalam game.
4. Klik salah satu kartu world pada daftar:
   - Informasi detail seed akan muncul di panel utama.
   - Lokasi Bastion (beserta tipe varian), Fortress, dan Stronghold akan dikalkulasi otomatis.
5. Gunakan tombol **Copy Coordinates** atau **Copy In-Game Command** sesuai kebutuhan latihan.
6. Untuk mencari seed baru idaman:
   - Masuk ke menu **Seed Finder**.
   - Masukkan batas radius maksimal dari $(0, 0)$ Nether.
   - Tekan **Find Seed**. Aplikasi akan melakukan looping cepat di background hingga menemukan seed yang cocok.
   - Periksa pratinjau 2D map, lalu klik **Create World**.
   - Buka Minecraft 1.16.1 dan world baru sudah siap dimainkan di menu singleplayer.

---

## [Bahan-Bahan Beserta Alat-Alat yang Digunakan]

| Kategori | Alat / Bahan | Fungsi |
|---|---|---|
| **Framework GUI** | Tauri v2 | Framework desktop ultra-ringan dengan backend Rust & frontend WebView |
| **Backend Core** | Rust (Edition 2021) | Bahasa sistem berkecepatan tinggi, hemat memori, dan *memory-safe* |
| **Pustaka C** | Cubiomes Library | Algoritma pembalik dan pencari struktur biome Minecraft berbasis C |
| **Frontend UI** | React / Svelte + TypeScript + Tailwind CSS | Antarmuka pengguna interaktif, modern, dan responsif |
| **Kompresi & NBT Parser**| `simdnbt` / `fastnbt` + `flate2` | Membaca format biner NBT `level.dat` dengan akselerasi SIMD |
| **Paralelisasi Thread** | `rayon` | Multi-threading untuk scanning jutaan seed di seluruh thread CPU |
| **Komunikasi IPC** | Tauri Commands & Events | Pengiriman data progress dan status asinkron dari Rust ke Frontend |

---

## [Setup Install Project Start]

### 1. Prasyarat Sistem (Prerequisites)
Pastikan alat-alat berikut sudah terinstal di komputer Windows Anda:
- [Node.js](https://nodejs.org/) (versi LTS 18+ atau 20+) & pnpm / npm.
- [Rust & Cargo](https://rustup.rs/) (versi stable terbaru, toolchain MSVC).
- C++ Build Tools (Visual Studio C++ Build Tools) untuk kompilasi modul C `cubiomes`.

### 2. Struktur Library & Dependency

#### Backend (`src-tauri/Cargo.toml`)
```toml
[package]
name = "seedruns"
version = "0.1.0"
edition = "2021"

[dependencies]
tauri = { version = "2", features = [] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
simdnbt = "0.5" # Parser NBT kecepatan tinggi
flate2 = "1.0"   # Dekompresi Gzip
rayon = "1.8"    # Multi-threading paralel
cc = "1.0"       # Untuk mengompilasi kode C Cubiomes saat build

[build-dependencies]
tauri-build = { version = "2", features = [] }
cc = "1.0"
```

#### Frontend (`package.json`)
```bash
# Inisialisasi frontend dengan Tailwind CSS & Lucide Icons
npm install lucide-react clsx tailwind-merge
```

### 3. Menjalankan Mode Pengembangan (Development)
```bash
# Clone atau buka direktori proyek
cd "F:/project website/seedruns"

# Install dependensi frontend
npm install

# Jalankan aplikasi dalam mode dev (Hot Reload Frontend + Kompilasi Backend)
npm run tauri dev
```

### 4. Build Eksekusi Siap Pakai (Production)
```bash
npm run tauri build
```
File `.exe` portabel dan ringan akan dihasilkan di folder `src-tauri/target/release/bundle/`.

---

## [Informasi Terkait Aplikasi & Versi Game]

- **Nama Aplikasi:** SeedRuns
- **Versi Aplikasi:** `v0.1.0-alpha` (Early Prototype)
- **Target Versi Minecraft:** **Java Edition 1.16.1**
  - *DataVersion ID:* `2567`
  - *Generator:* Standard Biomes 1.16.1 (Nether Update: Bastion Remnant, Crimson Forest, Warped Forest, Soul Sand Valley, Basalt Deltas).
- **Target OS:** Windows 10 / Windows 11 (64-bit).
- **Lisensi:** Private Project (Speedrunning Practice).
