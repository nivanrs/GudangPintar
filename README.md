# GudangPintar: Inventory Control Analysis

Analisis data inventori untuk **GudangPintar**, perusahaan jasa pergudangan dan logistik e-commerce.
Dibuat sebagai bagian dari project-based learning Windata — seri D04.

## Konteks Bisnis

**Peran:** Inventory Analyst
**Klien:** Operations Manager Pak Yusuf
**Masalah:** Kerugian Rp 500 juta/bulan akibat:
- Stok habis (stockout) di beberapa SKU → kehilangan penjualan
- Stok berlebihan (overstock) di SKU lain → modal tertahan

**Misi:** Temukan root cause, buat laporan analisis beserta rekomendasi actionable.

## Pertanyaan Analisis

| # | Pertanyaan | Metrik |
|---|-----------|--------|
| 1 | SKU & kategori mana yang paling sering stockout? | Stock Out Rate (%) |
| 2 | Supplier mana yang tidak reliable? | Avg & Std Dev Lead Time |
| 3 | Kategori mana yang overstock & berapa modal tertahan? | Rasio stock vs ROP, Modal Rp |

## Dataset

**File:** `dataset/data.csv`
**Periode:** Januari 2025 – Maret 2026 (15 bulan)
**Volume:** 312 baris, 20 SKU, 5 kategori, 10 supplier

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `SKU` | TEXT | Kode produk unik (SKU-001 s/d SKU-020) |
| `ProductName` | TEXT | Nama produk |
| `Category` | TEXT | Electronics / Fashion / Home & Living / Beauty / Food & Beverage |
| `CurrentStock` | FLOAT | Jumlah stok saat snapshot (unit) |
| `ReorderPoint` | INT | Batas minimum stok — jika di bawah ini harus reorder |
| `DailySalesAvg` | FLOAT | Rata-rata penjualan harian (unit/hari) |
| `LeadTime` | FLOAT | Waktu tunggu pengiriman dari supplier (hari) |
| `SupplierID` | TEXT | ID supplier (10 supplier, 2 per kategori) |
| `LastRestockDate` | DATE | Tanggal terakhir restock |
| `UnitPrice` | FLOAT | Harga jual per unit (Rupiah) |
| `SnapshotDate` | DATE | Tanggal pencatatan (selalu tanggal 1 tiap bulan) |

> **Data Quality:** 10 nilai NULL tersebar di 6 baris → di-impute dengan median per SKU di notebook.

## Struktur File

```
GudangPintar/
├── analysis.ipynb                   # Notebook analisis utama
├── README.md                        # Dokumentasi ini
├── dataset/
│   ├── data.csv                     # Dataset mentah (CSV)
│   └── data.xlsx                    # Dataset mentah (Excel)
├── Inventory Analysis Report.pdf    # Laporan final
├── Inventory Analysis Report.pptx  # Presentasi untuk Pak Yusuf
└── notes-gudangpintar-playlist.md   # Catatan dari playlist YouTube
```

## Cara Menjalankan

### Prasyarat

```bash
pip install jupysql duckdb-engine pandas matplotlib seaborn
```

### Buka Notebook

```bash
jupyter lab analysis.ipynb
# atau
jupyter notebook analysis.ipynb
```

Jalankan sel dari atas ke bawah: `Shift+Enter` tiap sel, atau **Kernel → Restart & Run All**.

> **Penting:** Jalankan notebook dari direktori root proyek (bukan dari dalam folder `dataset/`) agar path `dataset/data.csv` terbaca dengan benar.

## Alur Analisis

```
1. Load Data          → Baca CSV langsung ke DuckDB (tanpa pandas)
2. Eksplorasi Awal    → Cek NULL, distribusi per kategori & bulan
3. Data Cleaning      → Impute 10 NULL dengan median per SKU
4. Analisis Stockout  → Stock out rate per SKU, kategori, tren bulanan
5. Analisis Supplier  → Lead time avg & std dev, reliability map
6. Analisis Overstock → Rasio stok vs reorder point, modal tertahan (Rp)
7. Day of Stock       → Bandingkan DoS vs Lead Time → root cause stockout
8. Ringkasan          → Temuan utama & 4 rekomendasi untuk Pak Yusuf
```

## Temuan Utama

> Detail lengkap ada di notebook — bagian **8. Ringkasan & Rekomendasi**.

- **Stockout:** Beberapa SKU mengalami stockout di lebih dari separuh bulan yang dicatat
- **Supplier Bermasalah:** Supplier dengan std dev lead time > 3 hari membuat perencanaan stok tidak bisa diandalkan
- **Overstock:** Beberapa kategori menahan modal jutaan rupiah di stok yang melebihi 2× reorder point
- **Root Cause:** Banyak SKU memiliki Day of Stock ≤ Lead Time → stok habis sebelum restock tiba

## Tools

| Tool | Versi | Fungsi |
|------|-------|--------|
| JupySQL | 0.11.1 | SQL magic cells (`%%sql`) di Jupyter |
| DuckDB | 1.1.3 | In-process SQL engine — baca CSV langsung tanpa ETL |
| pandas | 2.0.3 | Konversi hasil query ke DataFrame untuk chart |
| matplotlib | 3.8.0 | Visualisasi bar chart & line chart |
| seaborn | 0.12.2 | Palet warna & scatter plot |

---

*Project: D04 — Windata Project-Based Learning | [YouTube Playlist](https://www.youtube.com/playlist?list=PLcIzUe1DEgTL5RAqGDxswBwEuFFU1mMbb)*
