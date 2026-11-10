# 05. Spesifikasi Pembuat World (World Creator)

Dokumen ini menjelaskan alur teknis penulisan folder penyimpanan dunia Minecraft baru beserta file `level.dat` langsung dari aplikasi **SeedRuns** ke folder `.minecraft/saves`.

---

## 1. Konsep Injeksi World Tanpa Java Runtime

Biasanya, sebuah world Minecraft dibuat melalui menu dalam game Minecraft. Namun, Minecraft Java Edition membaca kelayakan sebuah world **hanya dari integritas file `level.dat`** di dalam folder `saves/<NamaWorld>/`.

SeedRuns memanfaatkan hal ini:
1. Pengguna memilih atau menemukan seed idaman dari modul Seed Finder.
2. Pengguna menentukan nama dunia (misal: `Speedrun Practice #1`).
3. SeedRuns langsung membuat struktur folder di disk dan menulis file biner `level.dat` berekstensi NBT Gzip dengan tag metadata Minecraft 1.16.1.
4. Ketika pengguna membuka Minecraft (atau kembali ke menu Singleplayer), game akan mendeteksi world tersebut seolah-olah sudah dibuat dari dalam game. Saat world dibuka ("Play Selected World"), game Java akan men-generate chunk-chunk di sekitar spawn secara otomatis berdasarkan seed yang sudah diinjeksi.

---

## 2. Struktur Folder & File yang Dibuat

```text
.minecraft/saves/SeedRuns_<Timestamp>/
├── level.dat           # File NBT utama terkompresi Gzip (WAJIB)
├── level.dat_old       # Salinan backup identik (OPSIONAL tapi direkomendasikan)
└── icon.png            # Icon khusus penanda bahwa world dibuat via SeedRuns
```

---

## 3. Skema Data Tag `level.dat` Baru (Minecraft 1.16.1)

Struktur NBT biner minimal yang valid agar Minecraft 1.16.1 dapat memuat world tanpa crash atau dialog warning konversi versi:

```text
ROOT (Compound)
└── Data (Compound)
    ├── allowCommands: Byte (1)             -> Mengizinkan cheat /tp untuk latihan
    ├── clearWeatherTime: Int (0)
    ├── DataVersion: Int (2567)             -> DataVersion 1.16.1
    ├── DayTime: Long (0)                   -> Pagi hari (Awal mula speedrun)
    ├── Difficulty: Byte (3)                -> Hard (Standar Any% RSG)
    ├── DifficultyLocked: Byte (0)
    ├── GameType: Int (0)                   -> Survival Mode
    ├── Hardcore: Byte (0)
    ├── initialized: Byte (0)               -> Menandakan game perlu men-generate spawn saat pertama dibuka
    ├── LastPlayed: Long (<CurrentTimestampMillis>)
    ├── LevelName: String ("SeedRuns Practice")
    ├── MapFeatures: Byte (1)               -> Generate Structures: ON
    ├── raining: Byte (0)
    ├── rainTime: Int (0)
    ├── RandomSeed: Long (<SelectedSeed>)   -> SEED HASIL FILTER
    ├── SizeOnDisk: Long (0)
    ├── SpawnX: Int (0)
    ├── SpawnY: Int (64)
    ├── SpawnZ: Int (0)
    ├── thundering: Byte (0)
    ├── thunderTime: Int (0)
    ├── Time: Long (0)
    ├── Version: Compound
    │   ├── Id: Int (2567)
    │   ├── Name: String ("1.16.1")
    │   ├── Series: String ("main")
    │   └── Snapshot: Byte (0)
    └── WorldGenSettings: Compound          -> Pengaturan generator bioma 1.16.1
        ├── bonus_chest: Byte (0)
        ├── generate_features: Byte (1)
        ├── seed: Long (<SelectedSeed>)
        └── dimensions: Compound
            ├── minecraft:overworld: Compound ...
            ├── minecraft:the_nether: Compound ...
            └── minecraft:the_end: Compound ...
```

---

## 4. Implementasi NBT Serialization di Rust

Backend Rust menggunakan crate `simdnbt` atau `fastnbt` untuk meng-encode compound tree ke byte array, lalu membungkusnya dengan `flate2::write::GzEncoder`:

```rust
use std::fs::File;
use std::path::Path;
use flate2::write::GzEncoder;
use flate2::Compression;
use std::io::Write;

pub fn create_world_save(
    saves_path: &Path,
    folder_name: &str,
    display_name: &str,
    seed: i64,
) -> Result<(), Box<dyn std::error::Error>> {
    let world_dir = saves_path.join(folder_name);
    std::fs::create_dir_all(&world_dir)?;

    let level_dat_path = world_dir.join("level.dat");
    let file = File::create(level_dat_path)?;
    let mut gz = GzEncoder::new(file, Compression::default());

    // Serialize NBT data
    let nbt_bytes = build_1_16_1_nbt_payload(display_name, seed)?;
    gz.write_all(&nbt_bytes)?;
    gz.finish()?;

    // Copy icon branding SeedRuns
    let icon_path = world_dir.join("icon.png");
    std::fs::write(icon_path, include_bytes!("../assets/seedruns_icon.png"))?;

    Ok(())
}
```

---

## 5. Validasi Keamanan
1. **Pencegahan Overwrite:** Nama folder target otomatis diberi sufiks tanggal/jam (misal: `SeedRuns_20261109_194500`) untuk mencegah penumpukan atau penghapusan save world yang sudah ada.
2. **Karakter Terlarang:** Karakter nama folder divalidasi agar aman dari aturan sistem operasi Windows (`\ / : * ? " < > |`).
