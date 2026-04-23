# 🧬 HTAP Database — Hybrid Transactional/Analytical Processing

> **HTAP (Hybrid Transactional/Analytical Processing)** adalah arsitektur database yang menggabungkan kemampuan **OLTP (transaksional)** dan **OLAP (analitik)** dalam satu sistem. Dengan HTAP, data yang baru saja masuk bisa langsung dianalisis tanpa perlu proses ETL terpisah.

---

## 📋 Table of Contents

- [Konsep Dasar: OLTP vs OLAP](#konsep-dasar-oltp-vs-olap)
- [Apa Itu HTAP?](#apa-itu-htap)
- [Mengapa HTAP Dibutuhkan?](#mengapa-htap-dibutuhkan)
- [Arsitektur HTAP](#arsitektur-htap)
- [Kelebihan HTAP](#kelebihan-htap)
- [Use Case HTAP](#use-case-htap)
- [Tools & Database HTAP](#tools--database-htap)
  - [1. TiDB (via TiUP)](#1-tidb-via-tiup)
  - [2. SingleStore (MemSQL)](#2-singlestore-memsql)
  - [3. SAP HANA](#3-sap-hana)
  - [4. Google AlloyDB](#4-google-alloydb)
  - [5. MySQL HeatWave](#5-mysql-heatwave)
  - [6. PolarDB (Alibaba Cloud)](#6-polardb-alibaba-cloud)
  - [7. CockroachDB](#7-cockroachdb)
  - [8. Apache Doris (Pendekatan HTAP)](#8-apache-doris-pendekatan-htap)
- [Perbandingan Tools HTAP](#perbandingan-tools-htap)
- [Kapan Harus Menggunakan HTAP?](#kapan-harus-menggunakan-htap)
- [Referensi](#referensi)

---

## Konsep Dasar: OLTP vs OLAP

Sebelum memahami HTAP, perlu dipahami dua tipe pemrosesan data yang menjadi fondasi:

### OLTP (Online Transactional Processing)

Database untuk **operasional sehari-hari** seperti MySQL atau PostgreSQL. Tugasnya melayani **transaksi cepat** meliputi `INSERT`, `UPDATE`, `DELETE`. Data disimpan dalam format **row (baris)**.

```
Contoh OLTP:
├── Pengguna mendaftar akun baru          → INSERT
├── Pengguna mengubah alamat              → UPDATE  
├── Admin menghapus data duplikat         → DELETE
└── Kasir mencatat transaksi penjualan    → INSERT
```

### OLAP (Online Analytical Processing)

Database untuk **analitik** yang tugasnya membuat **report, trend, atau dashboard**. Data disimpan dalam format **columnar (kolom)** agar query agregasi lebih cepat.

```
Contoh OLAP:
├── Berapa total penjualan bulan ini?     → SUM, GROUP BY
├── Produk apa yang paling laris?         → ORDER BY, LIMIT
├── Tren pendapatan 12 bulan terakhir?    → Time-series analysis
└── Dashboard performa harian             → Aggregation queries
```

### Perbandingan OLTP vs OLAP

| Aspek               | OLTP                              | OLAP                                |
|----------------------|-----------------------------------|--------------------------------------|
| **Tujuan**           | Operasional / Transaksi           | Analitik / Reporting                |
| **Query**            | Simple (INSERT/UPDATE/DELETE)     | Complex (JOIN, GROUP BY, AGG)       |
| **Data Format**      | Row-based (per baris)             | Columnar (per kolom)                |
| **Response Time**    | Milidetik                         | Detik sampai menit                  |
| **Volume Query**     | Banyak query kecil                | Sedikit query besar                 |
| **Contoh Database**  | MySQL, PostgreSQL, MariaDB        | ClickHouse, Apache Doris, BigQuery  |

---

## Apa Itu HTAP?

**HTAP (Hybrid Transactional/Analytical Processing)** adalah satu sistem database yang memiliki fungsi **OLTP dan OLAP sekaligus di tempat yang sama**.

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│           Arsitektur Tradisional vs HTAP                             │
│                                                                      │
│   ❌ TRADISIONAL (Terpisah)              ✅ HTAP (Satu Sistem)      │
│                                                                      │
│   ┌──────────┐    ETL     ┌──────────┐   ┌──────────────────────┐   │
│   │   OLTP   │───────────▶│   OLAP   │   │       HTAP DB        │   │
│   │  (MySQL) │  (berjam-  │(BigQuery)│   │                      │   │
│   │          │   jam)     │          │   │  OLTP ◄──► OLAP      │   │
│   └──────────┘            └──────────┘   │  (Row)    (Columnar) │   │
│                                          │                      │   │
│   • 2 sistem terpisah                    │  • 1 sistem terpadu  │   │
│   • Butuh pipeline ETL                   │  • Tanpa ETL         │   │
│   • Data delay berjam-jam                │  • Real-time         │   │
│   • Maintenance 2x lipat                 │  • Maintenance mudah │   │
│                                          └──────────────────────┘   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

> 💡 **Analogi Sederhana**: Bayangkan OLTP seperti **kasir toko** yang mencatat setiap transaksi. OLAP seperti **akuntan** yang menganalisis laporan keuangan. **HTAP** adalah sistem yang bisa jadi kasir sekaligus akuntan secara bersamaan, tanpa harus menyalin buku catatan dulu.

---

## Mengapa HTAP Dibutuhkan?

Pada arsitektur tradisional, perusahaan harus memiliki:

1. **Database OLTP** (MySQL/PostgreSQL) → untuk menjalankan aplikasi
2. **Data Pipeline / ETL** (Airflow, Flink, dsb.) → untuk memindahkan data  
3. **Data Warehouse OLAP** (BigQuery, Doris, dsb.) → untuk menganalisis data

Masalah yang sering terjadi:

| Masalah                              | Penjelasan                                                                  |
|--------------------------------------|-----------------------------------------------------------------------------|
| ⏱️ **Data Delay**                   | Proses ETL memakan waktu berjam-jam, data analitik selalu *ketinggalan*     |
| 🔧 **Maintenance Kompleks**          | Harus maintain 2+ sistem database + pipeline ETL                          |
| 💰 **Biaya Tinggi**                 | Lebih banyak server, lebih banyak lisensi, lebih banyak biaya operasional   |
| 🔄 **Inkonsistensi Data**            | Data di OLTP dan OLAP bisa berbeda karena delay sinkronisasi               |

**HTAP menyelesaikan semua masalah di atas** dengan menyatukan OLTP dan OLAP dalam satu sistem.

---

## Arsitektur HTAP

Secara umum, database HTAP menggunakan salah satu dari pendekatan arsitektur berikut:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                     Pendekatan Arsitektur HTAP                            │
│                                                                           │
│   1️⃣ SEPARATED ENGINE                    2️⃣ UNIFIED ENGINE              │
│   (Mesin Terpisah)                        (Mesin Gabungan)               │
│                                                                           │
│   ┌──────────┐  ┌──────────────┐          ┌──────────────────────┐       │
│   │  TiKV    │  │   TiFlash    │          │    SingleStore       │       │
│   │ (Row)    │◄─│  (Columnar)  │          │                      │       │
│   │  OLTP    │  │    OLAP      │          │  Rowstore + Colstore │       │
│   └──────────┘  └──────────────┘          │  dalam 1 engine      │       │
│   Replikasi otomatis via Raft             └──────────────────────┘       │
│                                                                           │
│   Contoh: TiDB, ByteHTAP                 Contoh: SingleStore, SAP HANA  │
│                                                                           │
│                                                                           │
│   3️⃣ IN-MEMORY ACCELERATION              4️⃣ ADD-ON / EXTENSION         │
│   (Akselerasi di Memori)                  (Ekstensi di DB yang ada)      │
│                                                                           │
│   ┌──────────────────────┐                ┌──────────────────────┐       │
│   │   SAP HANA           │                │  MySQL + HeatWave    │       │
│   │                      │                │  (Plugin analitik)   │       │
│   │  Semua data di RAM   │                │                      │       │
│   │  Row + Columnar      │                │  PostgreSQL + AlloyDB│       │
│   └──────────────────────┘                │  (Managed service)   │       │
│                                           └──────────────────────┘       │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Kelebihan HTAP

| Kelebihan                                    | Penjelasan                                                                                                 |
|----------------------------------------------|------------------------------------------------------------------------------------------------------------|
| 🚀 **Analitik Real-Time**                   | Data yang baru masuk sedetik lalu bisa langsung dianalisis, tanpa menunggu proses ETL berjam-jam.           |
| 🏗️ **Arsitektur Lebih Sederhana**           | Tidak perlu membangun dan memaintain pipeline ETL terpisah, cukup 1 sistem untuk semua kebutuhan.           |
| 🔄 **Single Source of Truth**                | Data tidak perlu disalin ke banyak tempat, mengurangi inkonsistensi dan redundansi data.                   |
| 📈 **Flexible Scale-Out**                    | Sebagian besar database HTAP bersifat distributed, bisa scale dengan menambah node tanpa downtime.         |
| 🛡️ **High Availability**                   | Data direplikasi di banyak node, jika satu server mati sistem tetap berjalan normal.                       |
| 💰 **Cost Efficient**                        | Mengurangi jumlah sistem yang perlu dikelola, lisensi, dan biaya operasional.                              |
| ⚡ **Keputusan Bisnis Lebih Cepat**          | Tim bisnis bisa mendapatkan insight secara real-time, bukan menunggu laporan harian/mingguan.               |

---

## Use Case HTAP

| Industri               | Use Case                                           | Penjelasan                                                                                                |
|------------------------|-----------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| 🏦 **Finansial**       | Fraud Detection                                     | Menganalisis transaksi secara real-time untuk mendeteksi dan memblokir aktivitas mencurigakan.              |
| 🛒 **E-Commerce**      | Personalisasi Produk                                | Merekomendasikan produk berdasarkan aktivitas browsing dan pembelian terkini pelanggan.                     |
| 🚚 **Logistik**        | Optimisasi Supply Chain                             | Mengoptimalkan rute pengiriman dan inventaris berdasarkan data operasional real-time.                       |
| 📡 **IoT**             | Smart Devices / Smart Grid                          | Memproses data sensor berkecepatan tinggi untuk memicu aksi otomatis tanpa delay.                           |
| 🤖 **AI / ML**         | Real-Time Model Training                            | Model machine learning bisa belajar dari data terbaru yang masuk tanpa menunggu batch processing.           |
| 📊 **Reporting**       | Dashboard Operasional                               | Membuat dashboard yang selalu up-to-date tanpa butuh pipeline refresh data.                                |

---

## Tools & Database HTAP

---

### 1. TiDB (via TiUP)

```
┌──────────────────────────────────────────────────────────────┐
│                       TiDB (HTAP)                             │
│                                                               │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│  │  TiDB    │   │  TiKV    │   │ TiFlash  │   │    PD    │  │
│  │ Server   │   │ (Row)    │──▶│ (Column) │   │ (Router) │  │
│  │ (SQL     │   │  OLTP    │   │  OLAP    │   │ Manager  │  │
│  │ Parser)  │   │          │   │          │   │          │  │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘  │
└──────────────────────────────────────────────────────────────┘
```

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Open-source, Distributed SQL                                                                  |
| **Pendekatan HTAP**| Separated Engine — TiKV (row) + TiFlash (columnar)                                            |
| **Kompatibel**     | MySQL (drop-in replacement)                                                                   |
| **Lisensi**        | Apache 2.0 (Open Source)                                                                      |
| **Install Tool**   | **TiUP** — package manager untuk seluruh ekosistem TiDB                                       |
| **Storage Engine** | TiKV (row-based, OLTP) + TiFlash (columnar, OLAP)                                            |
| **Replikasi**      | Otomatis via algoritma konsensus **Raft**, data di TiKV disalin real-time ke TiFlash          |
| **Scaling**        | Horizontal scale-out dengan menambah node                                                     |
| **Fitur Unik**     | Smart Selection (otomatis pilih TiKV/TiFlash), MVCC (AS OF TIMESTAMP)                         |
| **Website**        | [pingcap.com/tidb](https://www.pingcap.com/)                                                  |

**Arsitektur TiDB** memisahkan storage menjadi 2 engine:
- **TiKV** → menyimpan data per baris untuk transaksi cepat (INSERT/UPDATE/DELETE)
- **TiFlash** → menyimpan data per kolom untuk analitik cepat (SUM, GROUP BY, JOIN besar)
- Data dari TiKV **otomatis direplikasi** ke TiFlash secara real-time menggunakan Raft

> 📖 **Dokumentasi instalasi TiDB tersedia di**: [10 instalasi & setup TiDB menggunakan TiUP.md](./10%20instalasi%20%26%20setup%20TiDB%20menggunakan%20TiUP.md)

---

### 2. SingleStore (MemSQL)

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Proprietary, Distributed SQL                                                                  |
| **Pendekatan HTAP**| Unified Engine — rowstore + columnstore dalam satu engine                                      |
| **Kompatibel**     | MySQL (wire protocol compatible)                                                               |
| **Lisensi**        | Proprietary (berbayar, ada free tier)                                                         |
| **Storage Engine** | Universal Storage (in-memory rowstore + on-disk columnstore)                                   |
| **Keunggulan**     | Performa sangat cepat karena in-memory, cocok untuk high-concurrency                          |
| **Kekurangan**     | Vendor lock-in, tidak open-source                                                             |
| **Website**        | [singlestore.com](https://www.singlestore.com/)                                               |

**Cara kerja**: SingleStore menggunakan **Universal Storage** yang menggabungkan rowstore dan columnstore dalam satu tabel. Data baru masuk ke rowstore (in-memory) dan secara otomatis di-flush ke columnstore (on-disk) untuk analitik.

```
┌────────────────────────────────────┐
│          SingleStore               │
│                                    │
│  ┌─────────────────────────────┐   │
│  │       Universal Storage     │   │
│  │  ┌──────────┬────────────┐  │   │
│  │  │ Rowstore │ Columnstore│  │   │
│  │  │ (Memory) │ (Disk)     │  │   │
│  │  │  OLTP    │   OLAP     │  │   │
│  │  └──────────┴────────────┘  │   │
│  └─────────────────────────────┘   │
└────────────────────────────────────┘
```

---

### 3. SAP HANA

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Proprietary, In-Memory Database                                                               |
| **Pendekatan HTAP**| In-Memory — semua data disimpan di RAM                                                        |
| **Kompatibel**     | SQL Standard                                                                                  |
| **Lisensi**        | Proprietary (Enterprise, sangat mahal)                                                        |
| **Storage Engine** | In-memory row + column storage                                                                 |
| **Keunggulan**     | Pioneer HTAP, performa sangat tinggi, integrasi penuh dengan ekosistem SAP                     |
| **Kekurangan**     | Sangat mahal, vendor lock-in ke SAP, butuh hardware khusus                                     |
| **Website**        | [sap.com/hana](https://www.sap.com/products/technology-platform/hana.html)                     |

**Keterangan**: SAP HANA adalah **pelopor HTAP** sejak 2010. Seluruh data disimpan di **RAM** sehingga sangat cepat untuk transaksi maupun analitik. Namun biayanya sangat tinggi karena membutuhkan server dengan RAM yang sangat besar.

---

### 4. Google AlloyDB

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Managed Service (Google Cloud)                                                                |
| **Pendekatan HTAP**| Add-on — PostgreSQL + columnar engine + vector search                                         |
| **Kompatibel**     | PostgreSQL (fully compatible)                                                                  |
| **Lisensi**        | Proprietary (Managed Service GCP)                                                             |
| **Keunggulan**     | PostgreSQL-compatible, built-in ML/AI support, fully managed                                   |
| **Kekurangan**     | Terikat Google Cloud, tidak bisa self-hosted                                                   |
| **Website**        | [cloud.google.com/alloydb](https://cloud.google.com/alloydb)                                   |

**Keterangan**: AlloyDB cocok untuk tim yang sudah menggunakan **Google Cloud** dan ingin PostgreSQL dengan kemampuan analitik dan AI tanpa berpindah platform.

---

### 5. MySQL HeatWave

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Managed Service (Oracle Cloud)                                                                |
| **Pendekatan HTAP**| Add-on — MySQL + in-memory analytics engine                                                   |
| **Kompatibel**     | MySQL (native)                                                                                |
| **Lisensi**        | Proprietary (Managed Service OCI)                                                             |
| **Keunggulan**     | Native MySQL, tidak perlu migrasi, analitik cepat via HeatWave engine                          |
| **Kekurangan**     | Terikat Oracle Cloud, biaya per penggunaan                                                     |
| **Website**        | [mysql.com/heatwave](https://www.mysql.com/cloud/)                                             |

**Keterangan**: MySQL HeatWave menambahkan **engine analitik in-memory** di atas MySQL biasa. Query analitik bisa **400x lebih cepat** dibanding MySQL standar tanpa perlu ubah kode aplikasi.

---

### 6. PolarDB (Alibaba Cloud)

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Managed Service (Alibaba Cloud)                                                               |
| **Pendekatan HTAP**| In-Memory Columnar Index (IMCI) di read-only node                                             |
| **Kompatibel**     | MySQL / PostgreSQL                                                                            |
| **Lisensi**        | Proprietary (Managed Service)                                                                 |
| **Keunggulan**     | Dual-format storage, IMCI untuk analitik real-time, integrasi Alibaba Cloud                    |
| **Kekurangan**     | Terikat Alibaba Cloud ecosystem                                                               |
| **Website**        | [alibabacloud.com/polardb](https://www.alibabacloud.com/product/polardb)                       |

---

### 7. CockroachDB

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Source-available, Distributed SQL                                                             |
| **Pendekatan HTAP**| **Bukan HTAP murni** — fokus utama OLTP                                                       |
| **Kompatibel**     | PostgreSQL                                                                                    |
| **Lisensi**        | BSL (Business Source License)                                                                 |
| **Keunggulan**     | Global scale, strong consistency, multi-region, ACID compliance                                |
| **Kekurangan**     | Tidak punya columnar engine, butuh sistem lain untuk OLAP                                      |
| **Website**        | [cockroachlabs.com](https://www.cockroachlabs.com/)                                            |

> ⚠️ **Catatan**: CockroachDB **bukan HTAP database** dalam arti sebenarnya. CockroachDB fokus pada **OLTP terdistribusi** dengan konsistensi kuat. Untuk analitik, data perlu di-stream ke sistem OLAP terpisah.

---

### 8. Apache Doris (Pendekatan HTAP)

| Aspek              | Detail                                                                                        |
|--------------------|-----------------------------------------------------------------------------------------------|
| **Tipe**           | Open-source, MPP Analytical Database                                                          |
| **Pendekatan HTAP**| **Bukan HTAP murni** — fokus OLAP, tapi bisa dipasangkan sebagai bagian arsitektur HTAP        |
| **Kompatibel**     | MySQL (protocol)                                                                              |
| **Lisensi**        | Apache 2.0 (Open Source)                                                                      |
| **Keunggulan**     | Performa analitik sangat tinggi, real-time ingestion, mudah dioperasikan                       |
| **Website**        | [doris.apache.org](https://doris.apache.org/)                                                  |

> 💡 Apache Doris sering digunakan dalam **arsitektur HTAP** sebagai **layer OLAP** yang dipasangkan dengan database OLTP (MySQL/MariaDB). Data di-sync dari OLTP ke Doris menggunakan tools seperti **Apache Flink CDC** atau **TiDB DM**.

> 📖 **Dokumentasi instalasi Doris tersedia di**: [1 instalasi setup & penggunaan apache doris.md](./1%20instalasi%20setup%20%26%20penggunaan%20apache%20doris.md)

---

## Perbandingan Tools HTAP

### Tabel Perbandingan Lengkap

| Fitur                   | TiDB            | SingleStore      | SAP HANA          | AlloyDB         | MySQL HeatWave   | PolarDB         |
|-------------------------|-----------------|------------------|--------------------|-----------------|-------------------|-----------------|
| **HTAP Murni**          | ✅ Ya           | ✅ Ya            | ✅ Ya              | ✅ Ya           | ✅ Ya             | ✅ Ya           |
| **Open Source**         | ✅ Apache 2.0   | ❌ Proprietary   | ❌ Proprietary     | ❌ Managed      | ❌ Managed        | ❌ Managed      |
| **MySQL Compatible**    | ✅              | ✅               | ❌                 | ❌              | ✅                | ✅              |
| **PostgreSQL Compatible**| ❌             | ❌               | ❌                 | ✅              | ❌                | ✅              |
| **Self-Hosted**         | ✅              | ✅               | ✅                 | ❌              | ❌                | ❌              |
| **Cloud Managed**       | ✅ TiDB Cloud   | ✅               | ✅                 | ✅ GCP only     | ✅ OCI only       | ✅ Alibaba only |
| **Scale-Out**           | ✅ Horizontal   | ✅ Horizontal    | ⚠️ Vertical       | ✅ Automatic    | ✅ Automatic      | ✅ Automatic    |
| **Biaya**               | 💚 Gratis       | 💛 Free Tier     | 🔴 Sangat Mahal   | 💛 Pay-per-use  | 💛 Pay-per-use    | 💛 Pay-per-use  |
| **Install Tool**        | TiUP            | CLI/Docker       | SAP Installer     | GCP Console     | OCI Console       | Ali Console     |

### Kapan Menggunakan Yang Mana?

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Flowchart Pemilihan Database HTAP                   │
│                                                                     │
│             Butuh HTAP?                                             │
│                 │                                                   │
│          ┌──────┴──────┐                                            │
│          ▼             ▼                                            │
│        Ya            Tidak                                          │
│          │             │                                            │
│    Open Source?    Butuh OLTP atau OLAP?                             │
│     │        │         │            │                               │
│     ▼        ▼         ▼            ▼                               │
│    Ya      Tidak     OLTP         OLAP                              │
│     │        │         │            │                               │
│     ▼        ▼         ▼            ▼                               │
│  ┌──────┐ Cloud     CockroachDB  Apache Doris                      │
│  │ TiDB │ mana?     PostgreSQL   ClickHouse                        │
│  └──────┘    │       MySQL       BigQuery                           │
│         ┌────┼────┐                                                 │
│         ▼    ▼    ▼                                                 │
│        GCP  OCI  Ali                                                │
│         │    │    │                                                  │
│         ▼    ▼    ▼                                                  │
│      AlloyDB HeatWave PolarDB                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Kapan Harus Menggunakan HTAP?

### ✅ Gunakan HTAP jika:

- Butuh **analitik real-time** pada data operasional (tanpa delay)
- Ingin **menghilangkan pipeline ETL** yang kompleks
- Membutuhkan **satu sumber kebenaran (single source of truth)** untuk transaksi dan analitik
- Skala data yang besar dan butuh **horizontal scaling**
- Tim tidak ingin me-maintain banyak sistem database

### ❌ Tidak perlu HTAP jika:

- Kebutuhan hanya **murni OLTP** (contoh: backend aplikasi sederhana) → gunakan MySQL / PostgreSQL
- Kebutuhan hanya **murni OLAP** (contoh: data warehouse) → gunakan Apache Doris / ClickHouse / BigQuery
- Data analitik **boleh delay** beberapa jam (batch processing sudah cukup)
- **Skala kecil** — overhead HTAP tidak sebanding dengan manfaatnya

---

## Referensi

- [PingCAP — What is HTAP?](https://www.pingcap.com/blog/how-we-build-an-htap-database-that-simplifies-your-data-platform/)
- [TiDB Official Documentation](https://docs.pingcap.com/tidb/stable)
- [SingleStore Documentation](https://docs.singlestore.com/)
- [SAP HANA Overview](https://www.sap.com/products/technology-platform/hana.html)
- [Google AlloyDB Documentation](https://cloud.google.com/alloydb/docs)
- [MySQL HeatWave Documentation](https://dev.mysql.com/doc/heatwave/en/)
- [PolarDB Documentation](https://www.alibabacloud.com/help/en/polardb/)
- [CockroachDB Documentation](https://www.cockroachlabs.com/docs/)
- [Apache Doris Documentation](https://doris.apache.org/docs/)
- [HTAP Database — Wikipedia](https://en.wikipedia.org/wiki/Hybrid_transactional/analytical_processing)

---
