# 06. Spesifikasi Antarmuka & Alur Pengguna (UI/UX)

Dokumen ini mendefinisikan desain tampilan, tata letak komponen, interaksi pengguna, dan status feedback visual pada aplikasi desktop **SeedRuns**.

---

## 1. Prinsip Desain
- **Kecepatan & Responsivitas:** Antarmuka tidak boleh membeku (*non-blocking*) saat scanning file atau kalkulasi seed.
- **Minimalis & Terfokus:** Didesain khusus untuk speedrunner—informasi esensial (Seed, Bastion, Fortress, Stronghold) langsung terlihat dalam satu layar tanpa scroll berlebih.
- **Dark Mode Native:** Menggunakan palet gelap modern (nuansa abu-abu gelap dengan aksen emas/ungu Nether).

---

## 2. Tata Letak Antarmuka (Layout Wireframe)

```text
+-----------------------------------------------------------------------------------+
|  [Logo] SeedRuns (1.16.1)       [Path: %APPDATA%/.minecraft/saves] [Browse] [Reload] |
+------------------------------------+----------------------------------------------+
| DAFTAR SAVED WORLDS                | DETAIL DUNIA & STRUKTUR TERPILIH            |
| Search: [........................] |                                              |
|                                    | Nama Dunia: Speedrun Attempt #42             |
| +--------------------------------+ | Seed: -7481928471928374182 [Copy Seed]      |
| | [Img] World #42 (1.16.1)       | | Versi: 1.16.1 (Release)                      |
| |       Last: 2 menit lalu       | +----------------------------------------------+
| +--------------------------------+ | STRUKTUR NETHER                              |
| | [Img] World #41 (1.16.1)       | |                                              |
| |       Last: 15 menit lalu      | | 1. BASTION REMNANT [Housing Units]           |
| +--------------------------------+ |    Posisi: X: 64, Z: -128 (Jarak: 143 blok)  |
| | [Img] Practice Seed (1.16.1)   | |    [Copy Coords]  [Copy /tp Command]         |
| |       Last: Kemarin            | |                                              |
| +--------------------------------+ | 2. NETHER FORTRESS                           |
|                                    |    Posisi: X: -96, Z: 80 (Jarak: 124 blok)   |
| [ + Buat World Baru via Filter ]   |    [Copy Coords]  [Copy /tp Command]         |
|                                    +----------------------------------------------+
|                                    | STRUKTUR OVERWORLD                           |
|                                    | 3. STRONGHOLD TERDEKAT                       |
|                                    |    Posisi: X: 1420, Z: -680                  |
|                                    |    [Copy Coords]  [Copy /tp Command]         |
+------------------------------------+----------------------------------------------+
| STATUS BAR: Ready | Ditemukan 18 Dunia | Waktu Refresh: 19:45:10                  |
+-----------------------------------------------------------------------------------+
```

---

## 3. Komponen Utama & Interaksi

### A. Header Panel
1. **Directory Selector & Browse:**
   - Menampilkan path yang sedang dipindai.
   - Tombol **Browse** membuka dialog native Windows (`tauri-plugin-dialog`) untuk memilih folder save lain (misal folder instance Prism/MultiMC).
2. **Tombol Reload (Manual):**
   - Ikon panah melingkar.
   - Saat diklik: Animasi putar aktif, teks status berubah menjadi *"Memindai saves..."*, tombol disable sementara selama 300ms untuk mencegah spam-klik.

### B. World List Sidebar (Kiri)
- Menampilkan kartu-kartu world yang diurutkan dari `LastPlayed` terbaru.
- Menampilkan thumbnail `icon.png` (64x64). Jika tidak ada, gunakan ikon placeholder *Grass Block*.
- Label penanda versi: Warna hijau jika `1.16.1`, warna kuning/abu jika versi lain.

### C. Inspection Panel (Kanan)
- **Kartu Bastion Remnant:**
  - Menampilkan badge varian: `[Bridge]` (Kuning), `[Housing]` (Biru), `[Stables]` (Cokelat), `[Treasure]` (Emas).
  - Jarak Euclidean dari $(0, 0)$ Nether: $d = \sqrt{x^2 + z^2}$.
- **Tombol Aksi Salin (Dual Copy Button):**
  - **Tombol "Copy Coords":** Menyalin string format `64 70 -128`.
  - **Tombol "Copy /tp Command":** Menyalin string format `/execute in minecraft:the_nether run tp @s 64 70 -128`.
  - Memberikan feedback visual tooltip *"Tersalin!"* selama 1.5 detik.

### D. Modal Dialog: Seed Finder & Creator (Poin 5 & 6)
Ketika tombol **"+ Buat World Baru via Filter"** ditekan:
1. Muncul modal overlay.
2. Form Input:
   - *Maksimal Radius Bastion & Fortress:* Slider/Number input (Default: `150` blok).
   - *Pilihan Varian Bastion Idaman:* Checkbox (Semua / Housing / Treasure / dll.).
3. Tombol **"Cari Seed"**:
   - Menampilkan counter realtime: *"Memindai 450.000 seed/detik..."*
   - Progress bar / animasi radar.
4. Ketika ditemukan:
   - Preview mini-map 2D (menampilkan titik spawn, titik Bastion, titik Fortress).
   - Form input *Nama World* (Default: `SeedRuns_<timestamp>`).
   - Tombol **"Create World"**: Menulis save langsung ke disk, menutup modal, dan otomatis memilih world baru tersebut di daftar.
