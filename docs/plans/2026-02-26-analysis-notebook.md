# GudangPintar Analysis Notebook Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create `analysis.ipynb` (SQL-first jupysql + DuckDB analysis of GudangPintar inventory data) and `README.md` in Indonesian.

**Architecture:** Single Jupyter notebook with jupysql `%%sql` magic cells pointing at DuckDB in-memory database. CSV loaded as a DuckDB table once at setup, then all cleaning/analysis done via SQL. Matplotlib/seaborn handle charts using `%sqlplot` or by converting query results to pandas with `<<`. README documents context, schema, analysis questions, and how to run.

**Tech Stack:** jupysql 0.11.1, duckdb 1.1.3 (via duckdb_engine), pandas 2.0.3, matplotlib 3.8.0, seaborn 0.12.2, nbformat (to create .ipynb programmatically)

---

## Context

**Dataset:** `dataset/data.csv` — 312 rows, 20 SKUs, 15 months (Jan 2025–Mar 2026)

**Columns:**
- `SKU` — kode produk unik
- `ProductName` — nama produk
- `Category` — Electronics / Fashion / Home & Living / Beauty / Food & Beverage
- `CurrentStock` — stok saat snapshot (3 nulls)
- `ReorderPoint` — batas minimum sebelum reorder
- `DailySalesAvg` — rata-rata penjualan harian (1 null)
- `LeadTime` — hari tunggu dari supplier (4 nulls)
- `SupplierID` — 10 supplier (2 per kategori)
- `LastRestockDate` — tanggal restock terakhir
- `UnitPrice` — harga per unit (2 nulls)
- `SnapshotDate` — tanggal snapshot (tanggal 1 tiap bulan)

**3 Analysis Questions:**
1. SKU & kategori mana yang paling sering stockout?
2. Supplier mana yang tidak reliable (lead time tinggi/tidak konsisten)?
3. Kategori mana yang overstock dan berapa modal yang tertahan?

**Known nulls to handle:**
- `CurrentStock`: 3 nulls (SKU-018 Apr-25, SKU-011 Jan-25, SKU-016 Aug-25) → impute dengan median per SKU
- `DailySalesAvg`: 1 null (SKU-014 Jan-25) → impute dengan median per SKU
- `LeadTime`: 4 nulls (SKU-005 May-25, SKU-012 Feb-25, SKU-019 Nov-25, SKU-020 Mar-26) → impute dengan median per SKU
- `UnitPrice`: 2 nulls (SKU-019 Feb-25, SKU-010 Jun-25) → impute dengan median per SKU

---

## Task 1: Scaffold the Notebook File

**Files:**
- Create: `analysis.ipynb`

**Step 1: Create a minimal valid notebook shell**

Run this Python script in the terminal (from the project root):

```python
import nbformat

nb = nbformat.v4.new_notebook()
nb['metadata'] = {
    'kernelspec': {
        'display_name': 'Python 3',
        'language': 'python',
        'name': 'python3'
    },
    'language_info': {
        'name': 'python',
        'version': '3.11.0'
    }
}
nb['cells'] = []

with open('analysis.ipynb', 'w') as f:
    nbformat.write(nb, f)

print("Created analysis.ipynb")
```

Run: `cd "/Users/nivanrs/Library/Mobile Documents/com~apple~CloudDocs/code/ngabuburit_data/GudangPintar" && python3 -c "<above script>"`

**Step 2: Verify file exists**

Run: `ls -la analysis.ipynb`
Expected: file present, ~200 bytes

---

## Task 2: Write Cell 1 — Title & Setup Imports

**Files:**
- Modify: `analysis.ipynb` (add cells)

**Step 1: Add title markdown cell**

Cell type: `markdown`

Content:
```markdown
# 🏭 GudangPintar: Inventory Control Analysis
**Dataset:** 20 SKU × 15 bulan (Januari 2025 – Maret 2026) | 312 baris
**Tools:** JupySQL + DuckDB + Matplotlib
**Tujuan:** Menjawab 3 pertanyaan analisis untuk Operations Manager Pak Yusuf

---
> **Skenario:** GudangPintar mengalami kerugian **Rp 500 juta/bulan** akibat stockout berulang sekaligus overstock di kategori lain. Kita bertugas membuat laporan analisis inventory untuk menemukan root cause dan rekomendasi.

## Pertanyaan Analisis
1. 🔴 SKU & kategori mana yang paling sering **stockout**?
2. ⚠️ Supplier mana yang **tidak reliable** (lead time tinggi & tidak konsisten)?
3. 💰 Kategori mana yang **overstock** dan berapa modal yang tertahan?
```

**Step 2: Add imports code cell**

Cell type: `code`

Content:
```python
# ── Setup & Imports ──────────────────────────────────────────────────────────
import warnings
warnings.filterwarnings('ignore')

import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import seaborn as sns
import pandas as pd

# Gaya visualisasi
plt.rcParams.update({
    'figure.figsize': (10, 5),
    'axes.titlesize': 13,
    'axes.titleweight': 'bold',
    'axes.spines.top': False,
    'axes.spines.right': False,
})
sns.set_palette('Set2')

print("✅ Library berhasil dimuat")
```

**Step 3: Add jupysql setup code cell**

Cell type: `code`

Content:
```python
# ── JupySQL + DuckDB Setup ────────────────────────────────────────────────────
%load_ext sql

# Hubungkan ke DuckDB in-memory
%sql duckdb:///:memory:

# Konfigurasi tampilan hasil query
%config SqlMagic.autopandas = False
%config SqlMagic.feedback = 0
%config SqlMagic.displaycon = False

print("✅ JupySQL + DuckDB terhubung")
```

---

## Task 3: Write Cell — Load Data & Create Table

**Step 1: Add markdown header cell**

Cell type: `markdown`

Content:
```markdown
---
## 1. Load Data

Kita load file CSV langsung ke DuckDB menggunakan `read_csv_auto()`. DuckDB bisa membaca CSV tanpa perlu load ke memori pandas terlebih dahulu — inilah keunggulan SQL-first workflow.
```

**Step 2: Add load data code cell**

Cell type: `code`

Content:
```python
%%sql
-- Buat tabel dari CSV langsung (DuckDB auto-detect types)
CREATE OR REPLACE TABLE inventory_raw AS
SELECT * FROM read_csv_auto('dataset/data.csv',
    nullstr = ['', 'nan', 'NULL', 'null'],
    dateformat = '%Y-%m-%d'
);

-- Konfirmasi
SELECT COUNT(*) AS total_baris FROM inventory_raw;
```

**Step 3: Add schema inspection cell**

Cell type: `code`

Content:
```python
%%sql
-- Cek struktur tabel (nama kolom & tipe data)
DESCRIBE inventory_raw;
```

**Step 4: Add sample data cell**

Cell type: `code`

Content:
```python
%%sql
-- Lihat 5 baris pertama
SELECT * FROM inventory_raw LIMIT 5;
```

---

## Task 4: Write Cell — Eksplorasi Data Awal

**Step 1: Add markdown header cell**

Cell type: `markdown`

Content:
```markdown
---
## 2. Eksplorasi Data Awal

Sebelum analisis, kita perlu memahami:
- Berapa baris & kolom?
- Distribusi data per kategori dan per bulan?
- Ada nilai kosong (NULL) di kolom mana saja?
```

**Step 2: Add null count cell**

Cell type: `code`

Content:
```python
%%sql
-- Hitung NULL per kolom
SELECT
    COUNT(*) - COUNT(CurrentStock)  AS null_CurrentStock,
    COUNT(*) - COUNT(DailySalesAvg) AS null_DailySalesAvg,
    COUNT(*) - COUNT(LeadTime)      AS null_LeadTime,
    COUNT(*) - COUNT(UnitPrice)     AS null_UnitPrice,
    COUNT(*) - COUNT(LastRestockDate) AS null_LastRestockDate
FROM inventory_raw;
```

**Step 3: Add data distribution cell**

Cell type: `code`

Content:
```python
%%sql
-- Distribusi baris per kategori
SELECT
    Category,
    COUNT(*) AS jumlah_baris,
    COUNT(DISTINCT SKU) AS jumlah_sku,
    COUNT(DISTINCT SnapshotDate) AS jumlah_bulan
FROM inventory_raw
GROUP BY Category
ORDER BY jumlah_baris DESC;
```

**Step 4: Add monthly coverage cell**

Cell type: `code`

Content:
```python
%%sql
-- Daftar bulan yang ada
SELECT
    strftime(SnapshotDate, '%Y-%m') AS bulan,
    COUNT(*) AS jumlah_record
FROM inventory_raw
GROUP BY bulan
ORDER BY bulan;
```

---

## Task 5: Write Cell — Data Cleaning

**Step 1: Add markdown header cell**

Cell type: `markdown`

Content:
```markdown
---
## 3. Data Cleaning

Kita temukan **10 nilai NULL** di 4 kolom. Strategi pembersihan:

| Kolom | Jumlah NULL | Strategi |
|-------|-------------|----------|
| `CurrentStock` | 3 | Impute dengan **median per SKU** |
| `DailySalesAvg` | 1 | Impute dengan **median per SKU** |
| `LeadTime` | 4 | Impute dengan **median per SKU** |
| `UnitPrice` | 2 | Impute dengan **median per SKU** |

**Mengapa median?** Median lebih robust terhadap outlier dibanding mean. Untuk produk yang sama (SKU sama), nilai historis bulan lain adalah estimasi terbaik.
```

**Step 2: Add cleaning SQL cell**

Cell type: `code`

Content:
```python
%%sql
-- Buat tabel bersih dengan NULL ter-impute menggunakan median per SKU
CREATE OR REPLACE TABLE inventory_clean AS
WITH median_per_sku AS (
    SELECT
        SKU,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY CurrentStock)  AS med_stock,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY DailySalesAvg) AS med_sales,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY LeadTime)      AS med_lead,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY UnitPrice)     AS med_price
    FROM inventory_raw
    GROUP BY SKU
)
SELECT
    r.SKU,
    r.ProductName,
    r.Category,
    COALESCE(r.CurrentStock,  m.med_stock)  AS CurrentStock,
    r.ReorderPoint,
    COALESCE(r.DailySalesAvg, m.med_sales)  AS DailySalesAvg,
    COALESCE(r.LeadTime,      m.med_lead)   AS LeadTime,
    r.SupplierID,
    r.LastRestockDate,
    COALESCE(r.UnitPrice,     m.med_price)  AS UnitPrice,
    r.SnapshotDate
FROM inventory_raw r
JOIN median_per_sku m USING (SKU);

-- Verifikasi: tidak boleh ada NULL tersisa
SELECT
    COUNT(*) - COUNT(CurrentStock)  AS null_CurrentStock,
    COUNT(*) - COUNT(DailySalesAvg) AS null_DailySalesAvg,
    COUNT(*) - COUNT(LeadTime)      AS null_LeadTime,
    COUNT(*) - COUNT(UnitPrice)     AS null_UnitPrice
FROM inventory_clean;
```

**Step 3: Add verification cell**

Cell type: `code`

Content:
```python
%%sql
-- Konfirmasi total baris tetap sama setelah cleaning
SELECT
    (SELECT COUNT(*) FROM inventory_raw)   AS baris_raw,
    (SELECT COUNT(*) FROM inventory_clean) AS baris_clean;
```

---

## Task 6: Write Cells — Analisis Stockout

**Step 1: Add markdown header cell**

Cell type: `markdown`

Content:
```markdown
---
## 4. Analisis Stockout

**Definisi stockout:** `CurrentStock < ReorderPoint`
Artinya stok sudah di bawah batas aman dan seharusnya sudah di-reorder.

**Stock Out Rate** = jumlah bulan stockout ÷ total bulan yang dicatat × 100%
```

**Step 2: Add stockout per SKU SQL cell**

Cell type: `code`

Content:
```python
%%sql
-- Stock out rate per SKU (diurutkan dari tertinggi)
SELECT
    SKU,
    ProductName,
    Category,
    COUNT(*) AS total_bulan,
    SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) AS bulan_stockout,
    ROUND(
        100.0 * SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) / COUNT(*),
        1
    ) AS stock_out_rate_pct
FROM inventory_clean
GROUP BY SKU, ProductName, Category
ORDER BY stock_out_rate_pct DESC
LIMIT 10;
```

**Step 3: Add stockout chart code cell**

Cell type: `code`

Content:
```python
# Ambil hasil query ke pandas untuk visualisasi
stockout_sku = %sql SELECT SKU, ProductName, Category, \
    ROUND(100.0 * SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) / COUNT(*), 1) AS rate \
    FROM inventory_clean GROUP BY SKU, ProductName, Category ORDER BY rate DESC LIMIT 10
df_stockout = stockout_sku.DataFrame()

fig, ax = plt.subplots(figsize=(11, 5))
colors = ['#e74c3c' if r >= 50 else '#e67e22' if r >= 25 else '#2ecc71'
          for r in df_stockout['rate']]
bars = ax.barh(df_stockout['ProductName'], df_stockout['rate'], color=colors)

# Tambah label nilai
for bar, val in zip(bars, df_stockout['rate']):
    ax.text(bar.get_width() + 0.5, bar.get_y() + bar.get_height()/2,
            f'{val:.0f}%', va='center', fontsize=10)

ax.set_xlabel('Stock Out Rate (%)')
ax.set_title('Top 10 SKU — Stock Out Rate (% bulan stockout dari total 15 bulan)')
ax.axvline(50, color='red', linestyle='--', alpha=0.5, label='Batas kritis 50%')
ax.legend()
ax.invert_yaxis()
plt.tight_layout()
plt.show()
```

**Step 4: Add stockout per category cell**

Cell type: `code`

Content:
```python
%%sql
-- Stock out rate per kategori
SELECT
    Category,
    COUNT(*) AS total_records,
    SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) AS total_stockout,
    ROUND(
        100.0 * SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) / COUNT(*),
        1
    ) AS stock_out_rate_pct,
    COUNT(DISTINCT SKU) AS jumlah_sku
FROM inventory_clean
GROUP BY Category
ORDER BY stock_out_rate_pct DESC;
```

**Step 5: Add monthly trend cell**

Cell type: `code`

Content:
```python
%%sql
-- Tren stockout per bulan (total SKU yang stockout tiap bulan)
SELECT
    strftime(SnapshotDate, '%Y-%m') AS bulan,
    COUNT(*) AS total_sku_dicatat,
    SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) AS sku_stockout,
    ROUND(
        100.0 * SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) / COUNT(*),
        1
    ) AS pct_stockout
FROM inventory_clean
GROUP BY bulan
ORDER BY bulan;
```

**Step 6: Add trend chart cell**

Cell type: `code`

Content:
```python
trend = %sql SELECT strftime(SnapshotDate, '%Y-%m') AS bulan, \
    SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) AS sku_stockout \
    FROM inventory_clean GROUP BY bulan ORDER BY bulan
df_trend = trend.DataFrame()

fig, ax = plt.subplots(figsize=(12, 4))
ax.plot(df_trend['bulan'], df_trend['sku_stockout'], marker='o', linewidth=2.5,
        color='#e74c3c', markerfacecolor='white', markeredgewidth=2)
ax.fill_between(df_trend['bulan'], df_trend['sku_stockout'], alpha=0.1, color='#e74c3c')
ax.set_xlabel('Bulan')
ax.set_ylabel('Jumlah SKU Stockout')
ax.set_title('Tren Stockout per Bulan (Jan 2025 – Mar 2026)')
ax.tick_params(axis='x', rotation=45)
for i, (b, v) in enumerate(zip(df_trend['bulan'], df_trend['sku_stockout'])):
    ax.annotate(str(v), (b, v), textcoords='offset points', xytext=(0, 8),
                ha='center', fontsize=9)
plt.tight_layout()
plt.show()
```

---

## Task 7: Write Cells — Analisis Supplier Reliability

**Step 1: Add markdown header cell**

Cell type: `markdown`

Content:
```markdown
---
## 5. Analisis Supplier Reliability

Supplier yang tidak reliable = lead time panjang DAN/ATAU tidak konsisten.

Metrik yang digunakan:
- **Avg Lead Time** — rata-rata waktu tunggu (hari)
- **Std Dev Lead Time** — seberapa bervariasi lead time-nya; semakin tinggi = semakin tidak bisa diprediksi
- **Max Lead Time** — worst-case scenario pengiriman

**Aturan praktis:** Supplier dengan std dev > 3 hari dianggap tidak konsisten (sulit diplan).
```

**Step 2: Add supplier stats SQL cell**

Cell type: `code`

Content:
```python
%%sql
-- Statistik lead time per supplier
SELECT
    SupplierID,
    COUNT(DISTINCT SKU) AS jumlah_sku,
    COUNT(*) AS total_records,
    ROUND(AVG(LeadTime), 1)    AS avg_lead_time,
    ROUND(STDDEV(LeadTime), 1) AS stddev_lead_time,
    MIN(LeadTime)               AS min_lead_time,
    MAX(LeadTime)               AS max_lead_time,
    CASE
        WHEN STDDEV(LeadTime) > 3 AND AVG(LeadTime) > 7 THEN '🔴 Tidak Reliable'
        WHEN STDDEV(LeadTime) > 3 OR AVG(LeadTime) > 7  THEN '🟡 Perlu Perhatian'
        ELSE '🟢 OK'
    END AS status
FROM inventory_clean
GROUP BY SupplierID
ORDER BY stddev_lead_time DESC;
```

**Step 3: Add supplier chart cell**

Cell type: `code`

Content:
```python
sup = %sql SELECT SupplierID, ROUND(AVG(LeadTime),1) AS avg_lt, ROUND(STDDEV(LeadTime),1) AS std_lt \
    FROM inventory_clean GROUP BY SupplierID ORDER BY std_lt DESC
df_sup = sup.DataFrame()

fig, ax = plt.subplots(figsize=(10, 5))
scatter = ax.scatter(df_sup['avg_lt'], df_sup['std_lt'],
                     s=200, c=df_sup['std_lt'], cmap='RdYlGn_r',
                     edgecolors='black', linewidths=0.8, zorder=3)

for _, row in df_sup.iterrows():
    ax.annotate(row['SupplierID'], (row['avg_lt'], row['std_lt']),
                textcoords='offset points', xytext=(6, 4), fontsize=9)

# Garis batas
ax.axhline(3, color='red', linestyle='--', alpha=0.6, label='Batas std dev (3 hari)')
ax.axvline(7, color='orange', linestyle='--', alpha=0.6, label='Batas avg lead time (7 hari)')
ax.set_xlabel('Rata-rata Lead Time (hari)', fontsize=11)
ax.set_ylabel('Std Dev Lead Time (hari)', fontsize=11)
ax.set_title('Supplier Reliability Map\n(pojok kanan atas = paling tidak reliable)', fontsize=12)
ax.legend(fontsize=9)

# Anotasi kuadran
ax.text(ax.get_xlim()[1]*0.7, 3.5, '⚠️ Tidak Konsisten', color='red', fontsize=8, alpha=0.7)
plt.tight_layout()
plt.show()
```

**Step 4: Add supplier per SKU detail cell**

Cell type: `code`

Content:
```python
%%sql
-- Detail SKU per supplier bermasalah
SELECT
    SupplierID,
    SKU,
    ProductName,
    ROUND(AVG(LeadTime), 1)    AS avg_lead_time,
    ROUND(STDDEV(LeadTime), 1) AS stddev_lead_time,
    MIN(LeadTime)               AS min_lt,
    MAX(LeadTime)               AS max_lt
FROM inventory_clean
WHERE SupplierID IN (
    SELECT SupplierID FROM inventory_clean
    GROUP BY SupplierID
    HAVING STDDEV(LeadTime) > 3 OR AVG(LeadTime) > 7
)
GROUP BY SupplierID, SKU, ProductName
ORDER BY SupplierID, stddev_lead_time DESC;
```

---

## Task 8: Write Cells — Analisis Overstock

**Step 1: Add markdown header cell**

Cell type: `markdown`

Content:
```markdown
---
## 6. Analisis Overstock & Modal Tertahan

**Definisi overstock:** `CurrentStock > 2 × ReorderPoint`
Artinya stok lebih dari 2 kali lipat batas aman — ada kelebihan yang mengikat modal.

**Modal Tertahan** = `(CurrentStock - ReorderPoint) × UnitPrice`
Ini adalah nilai rupiah dari stok yang "kelebihan" di atas reorder point.
```

**Step 2: Add overstock per category SQL cell**

Cell type: `code`

Content:
```python
%%sql
-- Overstock per kategori (rata-rata snapshot terbaru = Maret 2026)
SELECT
    Category,
    COUNT(DISTINCT SKU) AS jumlah_sku,
    ROUND(AVG(CurrentStock), 1)   AS avg_stock,
    ROUND(AVG(ReorderPoint), 1)   AS avg_reorder_point,
    ROUND(AVG(CurrentStock::FLOAT / NULLIF(ReorderPoint, 0)), 2) AS rasio_stock_vs_rop,
    SUM(CASE WHEN CurrentStock > 2 * ReorderPoint THEN 1 ELSE 0 END) AS jumlah_overstock_records
FROM inventory_clean
WHERE YEAR(SnapshotDate) = 2026 AND MONTH(SnapshotDate) = 3  -- snapshot terbaru
GROUP BY Category
ORDER BY rasio_stock_vs_rop DESC;
```

**Step 3: Add modal tertahan SQL cell**

Cell type: `code`

Content:
```python
%%sql
-- Modal tertahan per SKU (snapshot terbaru, hanya yang overstock)
SELECT
    SKU,
    ProductName,
    Category,
    CurrentStock,
    ReorderPoint,
    UnitPrice,
    (CurrentStock - ReorderPoint) AS kelebihan_stok,
    ROUND((CurrentStock - ReorderPoint) * UnitPrice) AS modal_tertahan_rp
FROM inventory_clean
WHERE SnapshotDate = (SELECT MAX(SnapshotDate) FROM inventory_clean)
  AND CurrentStock > ReorderPoint
ORDER BY modal_tertahan_rp DESC;
```

**Step 4: Add modal chart cell**

Cell type: `code`

Content:
```python
modal = %sql SELECT SKU, ProductName, Category, \
    ROUND((CurrentStock - ReorderPoint) * UnitPrice) AS modal_tertahan \
    FROM inventory_clean \
    WHERE SnapshotDate = (SELECT MAX(SnapshotDate) FROM inventory_clean) \
      AND CurrentStock > ReorderPoint \
    ORDER BY modal_tertahan DESC LIMIT 10
df_modal = modal.DataFrame()

fig, ax = plt.subplots(figsize=(11, 5))
palette = sns.color_palette('Set2', len(df_modal['Category'].unique()))
cat_colors = {c: palette[i] for i, c in enumerate(df_modal['Category'].unique())}
colors = [cat_colors[c] for c in df_modal['Category']]

bars = ax.barh(df_modal['ProductName'], df_modal['modal_tertahan'] / 1e6, color=colors)
for bar, val in zip(bars, df_modal['modal_tertahan']):
    ax.text(bar.get_width() + 0.3, bar.get_y() + bar.get_height()/2,
            f'Rp {val/1e6:.1f} jt', va='center', fontsize=9)

ax.set_xlabel('Modal Tertahan (juta Rp)')
ax.set_title('Top 10 SKU — Modal Tertahan Akibat Overstock (Snapshot Terakhir)')
ax.invert_yaxis()

# Legend kategori
from matplotlib.patches import Patch
legend_elements = [Patch(facecolor=cat_colors[c], label=c) for c in cat_colors]
ax.legend(handles=legend_elements, fontsize=9, loc='lower right')
plt.tight_layout()
plt.show()
```

---

## Task 9: Write Cells — Day of Stock vs Lead Time

**Step 1: Add markdown header cell**

Cell type: `markdown`

Content:
```markdown
---
## 7. Day of Stock vs Lead Time

**Day of Stock (DoS)** = `CurrentStock / DailySalesAvg`
Artinya: dengan stok saat ini dan rata-rata penjualan harian, stok akan habis dalam berapa hari?

**Masalah terjadi ketika:** `Day of Stock ≤ Lead Time`
Ini berarti stok akan habis **sebelum** pesanan baru datang dari supplier → pasti stockout!

Ini adalah root cause paling langsung dari stockout berulang.
```

**Step 2: Add DoS SQL cell**

Cell type: `code`

Content:
```python
%%sql
-- Day of Stock vs Lead Time per SKU (rata-rata seluruh periode)
SELECT
    SKU,
    ProductName,
    SupplierID,
    ROUND(AVG(CurrentStock / NULLIF(DailySalesAvg, 0)), 1) AS avg_day_of_stock,
    ROUND(AVG(LeadTime), 1)                                 AS avg_lead_time,
    ROUND(AVG(CurrentStock / NULLIF(DailySalesAvg, 0)) - AVG(LeadTime), 1) AS buffer_hari,
    CASE
        WHEN AVG(CurrentStock / NULLIF(DailySalesAvg, 0)) <= AVG(LeadTime)
            THEN '🔴 Kritis — Stok habis sebelum restock'
        WHEN AVG(CurrentStock / NULLIF(DailySalesAvg, 0)) <= AVG(LeadTime) * 1.5
            THEN '🟡 Rentan — Buffer tipis'
        ELSE '🟢 Aman'
    END AS status
FROM inventory_clean
GROUP BY SKU, ProductName, SupplierID
ORDER BY buffer_hari ASC;
```

**Step 3: Add DoS chart cell**

Cell type: `code`

Content:
```python
dos = %sql SELECT SKU, ProductName, \
    ROUND(AVG(CurrentStock / NULLIF(DailySalesAvg, 0)), 1) AS avg_dos, \
    ROUND(AVG(LeadTime), 1) AS avg_lt \
    FROM inventory_clean GROUP BY SKU, ProductName ORDER BY avg_dos ASC
df_dos = dos.DataFrame()

fig, ax = plt.subplots(figsize=(12, 6))
x = range(len(df_dos))
width = 0.35

bars1 = ax.bar([i - width/2 for i in x], df_dos['avg_dos'], width,
               label='Day of Stock (hari stok tersedia)', color='#3498db', alpha=0.85)
bars2 = ax.bar([i + width/2 for i in x], df_dos['avg_lt'], width,
               label='Lead Time (hari tunggu supplier)', color='#e74c3c', alpha=0.85)

ax.set_xticks(list(x))
ax.set_xticklabels(df_dos['SKU'], rotation=45, ha='right', fontsize=9)
ax.set_ylabel('Hari')
ax.set_title('Day of Stock vs Lead Time per SKU\n(Jika batang merah > batang biru → risiko stockout tinggi)')
ax.legend()

# Tandai SKU berisiko
for i, (dos_val, lt_val) in enumerate(zip(df_dos['avg_dos'], df_dos['avg_lt'])):
    if dos_val <= lt_val * 1.2:
        ax.annotate('⚠️', (i, max(dos_val, lt_val) + 0.3), ha='center', fontsize=11)

plt.tight_layout()
plt.show()
```

---

## Task 10: Write Cell — Ringkasan & Rekomendasi

**Step 1: Add markdown cell**

Cell type: `markdown`

Content:
```markdown
---
## 8. Ringkasan & Rekomendasi untuk Pak Yusuf

### Temuan Utama

| # | Masalah | Detail | Dampak |
|---|---------|--------|--------|
| 1 | **Stockout Kritis** | SKU dengan stock out rate ≥ 50% menunjukkan stok habis lebih dari separuh waktu | Kehilangan penjualan langsung |
| 2 | **Supplier Tidak Reliable** | Supplier dengan std dev lead time > 3 hari membuat planning stok hampir mustahil | Stockout tak terduga |
| 3 | **Overstock** | Beberapa kategori menahan modal besar di stok yang terlalu banyak | Modal tidak produktif |
| 4 | **Day of Stock < Lead Time** | Beberapa SKU memiliki buffer stok yang lebih kecil dari waktu tunggu supplier | Root cause utama stockout |

### Rekomendasi

1. **Naikkan Reorder Point** untuk SKU dengan DoS ≤ Lead Time
   Formula baru: `Reorder Point = (DailySalesAvg × LeadTime) + Safety Stock`
   Safety Stock minimal = `DailySalesAvg × Std Dev Lead Time × 1.65` (service level 95%)

2. **Evaluasi/Ganti Supplier Bermasalah**
   Minta SLA tertulis dari supplier dengan std dev > 3 hari, atau cari alternatif supplier

3. **Kurangi Overstock** di kategori dengan rasio stock > 2× reorder point
   Pertimbangkan flash sale atau transfer stok antar gudang

4. **Pasang Alert Otomatis**
   Set notifikasi ketika `CurrentStock < ReorderPoint` di sistem WMS

---
*Analisis dilakukan dengan JupySQL + DuckDB | Data: 20 SKU × 15 bulan (Jan 2025–Mar 2026)*
```

**Step 2: Add final summary query cell**

Cell type: `code`

Content:
```python
%%sql
-- ── Ringkasan Eksekutif ───────────────────────────────────────────────────────
SELECT
    'Total Records'        AS metrik, CAST(COUNT(*) AS VARCHAR) AS nilai FROM inventory_clean
UNION ALL SELECT 'Total SKU', CAST(COUNT(DISTINCT SKU) AS VARCHAR) FROM inventory_clean
UNION ALL SELECT 'Total Bulan', CAST(COUNT(DISTINCT SnapshotDate) AS VARCHAR) FROM inventory_clean
UNION ALL SELECT 'SKU Pernah Stockout', CAST(COUNT(DISTINCT SKU) AS VARCHAR)
    FROM inventory_clean WHERE CurrentStock < ReorderPoint
UNION ALL SELECT 'Total Kejadian Stockout',
    CAST(SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) AS VARCHAR)
    FROM inventory_clean
UNION ALL SELECT 'Stockout Rate Keseluruhan',
    CONCAT(CAST(ROUND(100.0 * SUM(CASE WHEN CurrentStock < ReorderPoint THEN 1 ELSE 0 END) / COUNT(*), 1) AS VARCHAR), '%')
    FROM inventory_clean;
```

---

## Task 11: Assemble and Write the Notebook

**Files:**
- Modify: `analysis.ipynb` (write all cells)

**Step 1: Run the notebook builder script**

Create a Python script at `.firecrawl/scratchpad/build_notebook.py` that assembles all cells from Tasks 2–10 into the notebook using `nbformat`, then writes `analysis.ipynb`.

The script structure:
```python
import nbformat

nb = nbformat.v4.new_notebook()
nb['metadata'] = {
    'kernelspec': {'display_name': 'Python 3', 'language': 'python', 'name': 'python3'},
    'language_info': {'name': 'python', 'version': '3.11.0'}
}

def md(source): return nbformat.v4.new_markdown_cell(source)
def code(source): return nbformat.v4.new_code_cell(source)

nb['cells'] = [
    # All cells from tasks 2-10 assembled here in order
    ...
]

with open('analysis.ipynb', 'w') as f:
    nbformat.write(nb, f)
print("✅ analysis.ipynb berhasil dibuat")
```

**Step 2: Run notebook with papermill to verify it executes cleanly**

```bash
pip install papermill -q
papermill analysis.ipynb analysis_executed.ipynb -k python3 2>&1 | tail -20
```

Expected: all cells execute without errors, output file created.

**Step 3: Check for execution errors**

If any cell fails, fix the SQL or Python in that cell and re-run.

**Step 4: Remove executed copy**

```bash
rm analysis_executed.ipynb
```

---

## Task 12: Create README.md

**Files:**
- Create: `README.md`

**Content:**

```markdown
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
├── analysis.ipynb              # ← Notebook analisis utama (ini)
├── README.md                   # ← Dokumentasi ini
├── dataset/
│   ├── data.csv                # Dataset mentah
│   └── data.xlsx               # Dataset dalam format Excel
├── Inventory Analysis Report.pdf   # Laporan final (output)
├── Inventory Analysis Report.pptx  # Presentasi untuk Pak Yusuf
└── notes-gudangpintar-playlist.md  # Catatan dari playlist YouTube
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

Jalankan sel dari atas ke bawah (`Shift+Enter` tiap sel, atau `Kernel → Restart & Run All`).

## Temuan Utama

> Detail lengkap ada di notebook bagian **8. Ringkasan & Rekomendasi**.

- **Stockout:** Beberapa SKU mengalami stockout di > 50% bulan yang dicatat
- **Supplier Bermasalah:** Supplier dengan std dev lead time > 3 hari membuat perencanaan stok tidak bisa diandalkan
- **Overstock:** Kategori tertentu menahan modal jutaan rupiah di stok yang melebihi 2× reorder point
- **Root Cause:** Banyak SKU memiliki Day of Stock ≤ Lead Time → stok habis sebelum restock tiba

## Tools

| Tool | Versi | Fungsi |
|------|-------|--------|
| JupySQL | 0.11.1 | SQL magic cells di Jupyter |
| DuckDB | 1.1.3 | In-process SQL engine (baca CSV langsung) |
| pandas | 2.0.3 | Konversi hasil query ke DataFrame untuk chart |
| matplotlib | 3.8.0 | Visualisasi |
| seaborn | 0.12.2 | Palet warna & style chart |

---

*Project: D04 — Windata Project-Based Learning | [YouTube Playlist](https://www.youtube.com/playlist?list=PLcIzUe1DEgTL5RAqGDxswBwEuFFU1mMbb)*
```

---

## Execution Notes

- Working directory for all commands: `/Users/nivanrs/Library/Mobile Documents/com~apple~CloudDocs/code/ngabuburit_data/GudangPintar`
- Kernel to use for notebook: `python3` (miniconda3, Python 3.11)
- DuckDB reads CSV with relative path `dataset/data.csv` — notebook must be run from project root
- `PERCENTILE_CONT` is standard SQL supported by DuckDB 1.1.3
- `strftime` format in DuckDB uses `%Y-%m-%d` syntax
- For `%sql` line magic returning results, use `result = %sql <query>` then `result.DataFrame()`
