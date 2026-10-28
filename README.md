📊 Superstore Sales Performance & Customer RFM Segmentation Analysis
> \*\*Live Interactive Dashboard\*\*: https://zedfoxcode666.github.io/Superstore-RFM-Analysis/
> 📥 **Download PBIX File**: [Klik di sini untuk mengunduh file Power BI](./Superstore_Sales_RFM_Analytics.pbix)

---

## 📌 Ringkasan Eksekutif & Permasalahan Bisnis

Retailer global mencatatkan akumulasi transaksi penjualan sebesar **$2.30M** dan laba bersih **$286.4K** dari 793 pelanggan. Namun, manajemen menghadapi dua tantangan utama:
1. **Fluktuasi Pertumbuhan & Defisit Produk**: Pertumbuhan laba tertahan akibat adanya kategori produk tertentu yang mengalami margin negatif.
2. **Ketiadaan Segmentasi Pelanggan**: Promosi dan alokasi diskon dilakukan secara seragam tanpa membedakan loyalitas atau riwayat belanja, memicu risiko churn pelanggan bernilai tinggi (*high-value churn*).

**Tujuan Proyek**:
Membangun solusi Business Intelligence terintegrasi menggunakan arsitektur **Star Schema**, kalkulasi **Time Intelligence DAX**, serta pemodelan perilaku **RFM (Recency, Frequency, Monetary)** untuk menghasilkan insight retensi dan evaluasi margin yang aplikatif.

---

## 🏗️ Pemodelan Data (Star Schema Architecture)

Model data dirancang dengan skema bintang relasional *1-to-Many* (`1:*`) searah (*single direction*) guna menjamin efisiensi komputasi VertiPaq engine dan keakuratan filter konteks:

```text
+-----------------------------------+              +----------------------------------+
|           dim_customers           |              |           Dim_Calendar           |
+-----------------------------------+              +----------------------------------+
| Customer ID (PK)                  | 1          1 | Date (PK)                        |
| Customer Name                     |              | Year                             |
| Segment                           |              | Month / Quarter                  |
| Recency days                      |              +-----------------+----------------+
| Frequency Orders                  |                                |
| Monetary Sales                    |                                | 1
| R / F / M Scores (1-5)            |                                |
| Customer Segment                  |                                |
+-----------------+-----------------+                                |
                  | 1                                                |
                  |                                                  |
                  | *                                                | *
+-----------------+--------------------------------------------------+----------------+
|                                    fact_orders                                      |
+-------------------------------------------------------------------------------------+
| Row ID (PK)                                                                         |
| Order ID                                                                            |
| Order Date (FK -> Dim_Calendar[Date])                                               |
| Customer ID (FK -> dim_customers[Customer ID])                                      |
| Sales                                                                               |
| Profit                                                                              |
| Discount                                                                            |
| Quantity                                                                            |
+-------------------------------------------------------------------------------------+
```
---
📐 Rekayasa DAX & Metrik Kunci
1. Ukuran Finansial Dasar (Core Measures)
```dax
-- Total Omset
Total Sales = SUM(fact_orders[Sales])

-- Total Laba Bersih
Total Profit = SUM(fact_orders[Profit])

-- Rasio Margin Keuntungan
Profit Margin Pct = DIVIDE([Total Profit], [Total Sales], 0)

-- Basis Pelanggan Unik
Total Customers = DISTINCTCOUNT(dim_customers[Customer ID])
```
2. Time Intelligence (Analisis Pertumbuhan Tahunan)
```dax
-- Penjualan Periode Tahun Lalu (SPLY)
Sales Last Year = 
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR(Dim_Calendar[Date])
)

-- Persentase Pertumbuhan YoY
Sales YoY Growth % = 
VAR CurrentSales = [Total Sales]
VAR PriorSales = [Sales Last Year]
RETURN
DIVIDE(CurrentSales - PriorSales, PriorSales, 0)
```
---
🎯 Metodologi Segmentasi RFM (Recency, Frequency, Monetary)
Segmentasi diimplementasikan pada tabel `dim\_customers` menggunakan 3 pilar:
A. Pilar Perhitungan Mentah
Recency Days: Menghitung jarak hari antara order terakhir pelanggan dengan tanggal referensi akhir dataset.
```dax
 Recency days = 
VAR MaxDatasetDate = MAXX(ALL(fact_orders), fact_orders[Order Date])
VAR LastCustomerOrder = MAXX(RELATEDTABLE(fact_orders), fact_orders[Order Date])
RETURN
DATEDIFF(LastCustomerOrder, MaxDatasetDate, DAY)
  ```
Frequency Orders: Menghitung jumlah transaksi unik (`Order ID`).
```dax
Frequency Orders = 
COUNTROWS(
    DISTINCT(
        SELECTCOLUMNS(
            RELATEDTABLE(fact_orders),
            "OrderID", fact_orders[Order ID]
        )
    )
)
  ```
Monetary Sales: Akumulasi total nominal belanjaan pelanggan.
```dax
  Monetary Sales = 
SUMX(
    RELATEDTABLE(fact_orders),
    fact_orders[Sales]
)
  ```
B. Matriks Penilaian Skor (Skala 1 - 5)
R Score: <= 30 hari -> 5; <= 90 -> 4; <= 180 -> 3; <= 365 -> 2; > 365 -> 1.
F Score: >= 9 orders -> 5; >= 7 -> 4; >= 5 -> 3; >= 3 -> 2; <= 2 -> 1.
M Score: >= $4,000 -> 5; >= $2,500 -> 4; >= $1,500 -> 3; >= $750 -> 2; < $750 -> 1.
C. Logika Klasifikasi Segmen Pelanggan
```dax
Customer Segment = 
SWITCH(
    TRUE(),
    -- Champions: Baru transaksi, frekuensi tinggi, nilai transaksi besar
    dim_customers[R Score] >= 4 && dim_customers[F Score] >= 4 && dim_customers[M Score] >= 4,
    "Champions",

    -- At Risk: Dulu royal & sering beli, tapi tidak aktif > 6-12 bulan
    dim_customers[R Score] <= 2 && dim_customers[F Score] >= 3 && dim_customers[M Score] >= 3,
    "At Risk",

    -- Loyalist: Frekuensi transaksi rutin dan stabil
    dim_customers[R Score] >= 3 && dim_customers[F Score] >= 3,
    "Loyalist",

    -- Potential Loyalist: Pelanggan baru berpotensi dengan nilai order awal solid
    dim_customers[R Score] >= 4 && dim_customers[F Score] <= 2,
    "Potential Loyalist",

    -- Lost: Jarang transaksi dan sudah tidak aktif dalam durasi panjang
    dim_customers[R Score] <= 2 && dim_customers[F Score] <= 2,
    "Lost",

    "Needs Attention"
)
```
---
