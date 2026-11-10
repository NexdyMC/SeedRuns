# 03. Integrasi Library Cubiomes (C-FFI di Rust)

Dokumen ini menjelaskan integrasi engine **Cubiomes** (library komputasi generasi dunia Minecraft berbasis C) ke dalam backend Rust di aplikasi **SeedRuns** untuk versi **1.16.1**.

---

## 1. Mengenal Cubiomes
[Cubiomes](https://github.com/Cubitect/cubiomes) adalah library C mandiri yang mengimplementasikan kembali algoritma penempatan biome dan struktur Minecraft secara matematis murni tanpa ketergantungan pada runtime Java. 

Keuntungan integrasi Cubiomes:
- **Kecepatan Tinggi:** Mampu mengevaluasi ribuan posisi struktur per milidetik.
- **Dukungan Penuh 1.16.1:** Generator bioma Nether (Warped Forest, Crimson Forest, Soulsand Valley, Basalt Deltas) dan struktur Bastion Remnant/Fortress didukung secara akurat.

---

## 2. Mekanisme Kompilasi via Rust `build.rs`

File source C dari Cubiomes dikompilasi langsung saat proses `cargo build` menggunakan crate `cc`:

```rust
// src-tauri/build.rs
fn main() {
    cc::Build::new()
        .file("c_src/cubiomes/finders.c")
        .file("c_src/cubiomes/generator.c")
        .file("c_src/cubiomes/layers.c")
        .file("c_src/cubiomes/biomes.c")
        .file("c_src/cubiomes/noise.c")
        .opt_level(3) // Optimasi agresif untuk performa maksimal
        .compile("cubiomes");

    tauri_build::build();
}
```

---

## 3. Rust FFI Bindings & Abstraksi

Rust membungkus pointer C Cubiomes ke dalam interface yang aman (*safe wrapper*):

```rust
// Definisi FFI C functions
extern "C" {
    fn setupGenerator(g: *mut Generator, mc_version: i32, flags: u32);
    fn applySeed(g: *mut Generator, dim: i32, seed: u64);
    fn getStructurePos(struct_type: i32, mc_version: i32, seed: u64, reg_x: i32, reg_z: i32, pos: *mut Pos) -> i32;
    fn getBastionVariant(seed: u64, x: i32, z: i32) -> i32;
}

// Enum Varian Bastion Remnant (1.16.1)
#[derive(Debug, Serialize, Deserialize)]
pub enum BastionType {
    Bridge,
    HoglinStables,
    HousingUnits,
    TreasureRoom,
    Unknown,
}

impl From<i32> for BastionType {
    fn from(variant_id: i32) -> Self {
        match variant_id {
            0 => BastionType::HousingUnits,
            1 => BastionType::HoglinStables,
            2 => BastionType::Bridge,
            3 => BastionType::TreasureRoom,
            _ => BastionType::Unknown,
        }
    }
}
```

---

## 4. Deteksi Struktur Kritis 1.16.1

### A. Dimensi Nether (Bastion & Fortress)
Di Minecraft 1.16.1, Nether dibagi menjadi kisi-kisi region struktur berukuran **$27 \times 27$ chunk** ($432 \times 432$ blok).
- Setiap region Nether dapat berisi **salah satu**: Bastion Remnant ATAU Fortress (saling bergantian berdasarkan RNG seed).
- SeedRuns mengevaluasi 4 sampai 9 region terdekat di sekitar titik $(0, 0)$ Nether:
  - Region $(0, 0)$: $X: 0..432, Z: 0..432$
  - Region $(-1, 0)$: $X: -432..-1, Z: 0..432$
  - Region $(0, -1)$: $X: 0..432, Z: -432..-1$
  - Region $(-1, -1)$: $X: -432..-1, Z: -432..-1$

### B. Dimensi Overworld (Stronghold)
- Stronghold di-generate dalam pola cincin (*concentric rings*) yang berpusat di titik $(0, 0)$ Overworld.
- Cincin pertama (*Ring 1*) berisi **3 Stronghold** pada radius antara **1.280 hingga 2.816 blok** dari origin.
- Algoritma menghitung sudut dan posisi ke-3 Stronghold cincin pertama secara instan.

---

## 5. Output Data Struktur (`AnalyzedStructures`)

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct StructureInfo {
    pub name: String,
    pub dimension: String, // "overworld" | "the_nether"
    pub x: i32,
    pub y: i32, // Default representatif: 64 di overworld, 70 di nether
    pub z: i32,
    pub distance_from_center: f64,
    pub variant_detail: Option<String>, // Misal: "Housing Units"
    pub tp_command: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct AnalyzedStructures {
    pub seed: i64,
    pub bastions: Vec<StructureInfo>,
    pub fortresses: Vec<StructureInfo>,
    pub strongholds: Vec<StructureInfo>,
}
```
