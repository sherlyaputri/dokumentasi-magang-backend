# 🗄️ Cara Instalasi & Setup TiDB menggunakan TiUP

> **TiDB** adalah database **HTAP (Hybrid Transactional/Analytical Processing)** open-source, distributed SQL, dan **kompatibel penuh dengan MySQL**. TiDB mampu menangani beban kerja OLTP dan OLAP secara bersamaan dalam satu sistem database.

---

## 📋 Table of Contents

- [Konsep Dasar: OLTP, OLAP & HTAP](#konsep-dasar-oltp-olap--htap)
- [Arsitektur TiDB](#arsitektur-tidb)
- [Kelebihan TiDB](#kelebihan-tidb)
- [Konsep Region dalam TiDB](#konsep-region-dalam-tidb)
- [Fitur Unggulan TiDB](#fitur-unggulan-tidb)
- [Prerequisites](#prerequisites)
- [Instalasi TiDB (Playground Mode)](#instalasi-tidb-playground-mode)
  - [1. Install Dependencies](#1-install-dependencies)
  - [2. Install TiUP](#2-install-tiup)
  - [3. Jalankan TiDB Playground](#3-jalankan-tidb-playground)
  - [4. Akses TiDB](#4-akses-tidb)
- [Deploy TiDB Cluster (Production)](#deploy-tidb-cluster-production)
  - [1. Generate Template Topology](#1-generate-template-topology)
  - [2. Pre-check Cluster](#2-pre-check-cluster)
  - [3. Deploy Cluster](#3-deploy-cluster)
  - [4. Start Cluster](#4-start-cluster)
  - [5. Cek Status Cluster](#5-cek-status-cluster)
  - [6. Akses Cluster](#6-akses-cluster)
- [Migrasi Data menggunakan TiDB Data Migration (DM)](#migrasi-data-menggunakan-tidb-data-migration-dm)
  - [Fase Migrasi](#fase-migrasi)
  - [Menjalankan DM (Playground)](#menjalankan-dm-playground)
  - [Menjalankan DM (Develop Cluster)](#menjalankan-dm-develop-cluster)
  - [Penambahan Tabel untuk Migrasi](#penambahan-tabel-untuk-migrasi)
- [Troubleshooting](#troubleshooting)
- [Referensi](#referensi)

---

## Konsep Dasar: OLTP, OLAP & HTAP

Sebelum memahami TiDB, penting untuk mengenal tiga konsep database berikut:

| Konsep   | Singkatan dari                              | Fungsi                                                             | Format Penyimpanan | Contoh Database          |
|----------|---------------------------------------------|--------------------------------------------------------------------|---------------------|--------------------------|
| **OLTP** | Online Transactional Processing             | Database untuk operasional sehari-hari (INSERT, UPDATE, DELETE)     | Row-based           | MySQL, PostgreSQL        |
| **OLAP** | Online Analytical Processing                | Database untuk analitik, report, trend, dan dashboard              | Column-based        | ClickHouse, Apache Doris |
| **HTAP** | Hybrid Transactional/Analytical Processing  | Satu sistem yang memiliki fungsi **OLTP dan OLAP sekaligus**       | Row + Column        | **TiDB**                 |

> 💡 **Mengapa HTAP penting?** Dengan HTAP, kamu tidak perlu membangun pipeline ETL terpisah untuk memindahkan data dari database transaksional ke data warehouse. Semua bisa dilakukan di satu tempat.

---

## Arsitektur TiDB

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            TiDB Architecture (HTAP)                             │
│                                                                                 │
│                          ┌──────────────────────┐                               │
│                          │     Application /     │                               │
│                          │     MySQL Client      │                               │
│                          └──────────┬───────────┘                               │
│                                     │ SQL Query                                 │
│                                     ▼                                           │
│                          ┌──────────────────────┐                               │
│                          │    TiDB Server (FE)   │                               │
│                          │   SQL Parser & Query  │                               │
│                          │   Optimizer/Executor  │                               │
│                          │     (Stateless)       │                               │
│                          └──────────┬───────────┘                               │
│                                     │                                           │
│                    ┌────────────────┼────────────────┐                          │
│                    ▼                ▼                ▼                          │
│         ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐               │
│         │    TiKV       │  │  TiFlash      │  │   PD (Placement  │               │
│         │ (Row-based)   │  │ (Columnar)    │  │     Driver)      │               │
│         │   ── OLTP ──  │  │   ── OLAP ──  │  │  ── Manager ──   │               │
│         │               │  │               │  │                  │               │
│         │ • Data per    │  │ • Data per    │  │ • Distribusi     │               │
│         │   baris       │◀─│   kolom       │  │   data           │               │
│         │ • Transaksi   │  │ • Analitik    │  │ • Load balancing │               │
│         │   cepat       │  │   real-time   │  │ • Scheduling     │               │
│         │ • INSERT /    │──▶│ • Replikasi   │  │   region         │               │
│         │   UPDATE /    │  │   otomatis    │  │                  │               │
│         │   DELETE      │  │   via Raft    │  │                  │               │
│         └──────────────┘  └──────────────┘  └──────────────────┘               │
│                                                                                 │
│              Data di TiKV disalin otomatis & real-time ke TiFlash               │
│                    menggunakan algoritma konsensus (Raft)                        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Penjelasan Komponen:

| Komponen         | Fungsi                                                                                                                          |
|------------------|---------------------------------------------------------------------------------------------------------------------------------|
| **TiDB Server**  | Menerima SQL query dari aplikasi, menerjemahkan, dan mencari jalur paling optimal untuk mengeksekusi query. Bersifat **stateless** (tidak menyimpan data). |
| **TiKV**         | Ruang penyimpanan untuk tugas **OLTP**. Data disimpan per baris (**row-based storage**).                                        |
| **TiFlash**      | Ruang penyimpanan untuk tugas **OLAP**. Data disimpan per kolom (**columnar storage**). Data dari TiKV disalin otomatis dan real-time ke TiFlash menggunakan **algoritma konsensus Raft**. |
| **PD**           | **Placement Driver** — manajer/router yang mengatur distribusi data, ketersediaan server, dan load balancing di dalam cluster.  |

---

## Kelebihan TiDB

| Kelebihan                      | Penjelasan                                                                                                                    |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 🚀 **Analitik Real-Time**     | Tidak perlu menunggu proses ETL berjam-jam. Data yang baru masuk sedetik lalu bisa **langsung dianalisis**.                    |
| 📈 **Flexible Scale-Out**     | TiDB bisa mendistribusikan ke puluhan server jika kapasitas penuh dengan **menambah node (server) baru tanpa downtime**.       |
| 🛡️ **High Availability**     | Data direplikasi di banyak node. Jika ada satu server mati, **sistem tetap berjalan normal**.                                 |
| 🐬 **MySQL Compatible**       | Kompatibel penuh dengan MySQL, sehingga migrasi dari MySQL sangat mudah tanpa ubah kode aplikasi.                             |

---

## Konsep Region dalam TiDB

TiDB memecah tabel dalam potongan-potongan kecil yang disebut **Region**. Setiap region memiliki ukuran **~96 MB**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Konsep Region di TiDB                            │
│                                                                     │
│   Tabel Besar                                                       │
│   ┌──────┬──────┬──────┬──────┬──────┬──────┐                      │
│   │Reg 1 │Reg 2 │Reg 3 │Reg 4 │Reg 5 │Reg 6 │  ← Dipecah ~96MB   │
│   └──┬───┴──┬───┴──┬───┴──┬───┴──┬───┴──┬───┘                      │
│      │      │      │      │      │      │                           │
│      ▼      ▼      ▼      ▼      ▼      ▼      PD mengatur         │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐       distribusi          │
│   │ TiKV #1  │ │ TiKV #2  │ │ TiKV #3  │       otomatis            │
│   │ Reg 1    │ │ Reg 2    │ │ Reg 3    │                           │
│   │ Reg 4(R) │ │ Reg 1(R) │ │ Reg 2(R) │  (R) = Replika           │
│   │ Reg 5(R) │ │ Reg 6(R) │ │ Reg 4(R) │                           │
│   └──────────┘ └──────────┘ └──────────┘                           │
│                                                                     │
│   ✅ Setiap region memiliki 3 replika di server TiKV berbeda        │
│   ✅ Jika 1 server mati, masih ada 2 salinan di server lain        │
│   ✅ Jika server kepanasan, PD otomatis mindahin region             │
└─────────────────────────────────────────────────────────────────────┘
```

> 💡 **Inilah mengapa TiDB bisa scale-out!** Karena data dipecah ke banyak region dan disebar ke banyak server, menambah kapasitas cukup dengan menambah node baru.

---

## Fitur Unggulan TiDB

### 🧠 Smart Selection
TiDB secara otomatis memilih apakah query yang ditulis akan dieksekusi di **TiKV (row-based)** atau **TiFlash (columnar)** tergantung jenis query-nya:
- **Transaksi (INSERT/UPDATE/DELETE)** → TiKV
- **Analitik (Aggregasi, JOIN besar)** → TiFlash

### 🕐 MVCC (Multi-Version Concurrency Control)
Saat data dihapus, data tersebut **belum benar-benar hilang**. TiDB menyimpan versi-versi sebelumnya sehingga kita bisa melihat data yang sudah dihapus menggunakan fitur **AS OF TIMESTAMP**:

```sql
-- Melihat data yang sudah dihapus/diubah 5 menit lalu
SELECT * FROM orders AS OF TIMESTAMP (NOW() - INTERVAL 5 MINUTE);

-- Melihat data pada waktu tertentu
SELECT * FROM orders AS OF TIMESTAMP '2025-01-15 10:30:00';
```

---

## Prerequisites

| Komponen              | Minimum Requirement                  |
|-----------------------|--------------------------------------|
| **OS**                | Ubuntu 22.04 / Debian 12+           |
| **CPU**               | 2 Cores (4+ recommended)            |
| **RAM**               | 4 GB (8+ recommended)               |
| **Disk**              | 20 GB SSD                           |
| **Tool**              | curl, mysql-client                   |
| **TiDB Version**      | v8.5.6                              |

---

## Instalasi TiDB (Playground Mode)

> ⚠️ **Playground Mode** digunakan untuk **testing dan development** saja, **bukan untuk production**.

---

### 1. Install Dependencies

```bash
# Update repository
sudo apt update

# Install curl dan MySQL client
sudo apt install curl default-mysql-client -y
```

---

### 2. Install TiUP

TiUP adalah package manager untuk ekosistem TiDB. Semua komponen TiDB bisa di-manage melalui TiUP.

```bash
# Download dan install TiUP
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh

# Reload environment variable
source ~/.bashrc

# Verifikasi instalasi TiUP
tiup version
```

> ✅ Jika berhasil, akan muncul output versi TiUP yang terinstall.

---

### 3. Jalankan TiDB Playground

```bash
# Jalankan TiDB playground (lokal saja)
tiup playground v8.5.6

# ATAU jika ingin diakses dari network (selain localhost)
tiup playground v8.5.6 --host 0.0.0.0
```

> ⚠️ **Jika playground macet atau tidak bisa dijalankan**, stop dulu semua proses lalu coba lagi:
> ```bash
> tiup clean --all
> tiup playground v8.5.6
> ```

---

### 4. Akses TiDB

Setelah playground berhasil berjalan, kamu bisa mengakses beberapa service berikut:

| Service             | URL / Command                                                  | Credential                    |
|---------------------|----------------------------------------------------------------|-------------------------------|
| **MySQL Client**    | `mysql -h 192.168.4.100 -P 4000 -u root`                      | root (tanpa password)         |
| **MySQL Client**    | `mysql -h 192.168.4.100 -P 4000 -u admin -p`                  | admin : whnexp88              |
| **TiDB Dashboard** | `http://<IP>:2379/dashboard`                                    | root (tanpa password)         |
| **Grafana**         | `http://<IP>:3000`                                              | admin : whnexp88              |

> 💡 **Tips**: User `admin` dengan password `whnexp88` dibuat secara manual untuk memudahkan akses MySQL client.

---

## Deploy TiDB Cluster (Production)

> 🏭 Untuk **production** atau **develop**, gunakan mode **cluster deployment** yang lebih stabil dan bisa dikonfigurasi detail.

---

### 1. Generate Template Topology

```bash
# Generate template file topology
tiup cluster template > topology.yaml
```

> 📝 Edit file `topology.yaml` sesuai kebutuhan server (jumlah TiKV, TiFlash, PD, dll).

---

### 2. Pre-check Cluster

Sebelum deploy, lakukan pengecekan apakah semua konfigurasi dan server sudah siap:

```bash
tiup cluster check ./topology.yaml --user sherly -p
```

---

### 3. Deploy Cluster

```bash
tiup cluster deploy tidb-sherly v8.5.6 ./topology.yaml --user sherly -p
```

> ⏳ Proses deploy akan memakan waktu beberapa menit tergantung jumlah node dan kecepatan jaringan.

---

### 4. Start Cluster

```bash
# Start cluster dengan inisialisasi password root
tiup cluster start tidb-sherly --init
```

> ⚠️ **PENTING**: Saat `--init` dijalankan, TiDB akan **men-generate password root** secara otomatis. 
> **Catat dan simpan password ini di tempat yang aman!** Password hanya ditampilkan **sekali** dan **tidak bisa dilihat ulang**.

Contoh output:

```
The new password is: '4^R8@EjU6*Vh+5Wk21'.
Copy and record it to somewhere safe, it is only displayed once, and will not be stored.
The generated password can NOT be get and shown again.
```

---

### 5. Cek Status Cluster

```bash
tiup cluster display tidb-sherly
```

Contoh output:

```
Cluster type:       tidb
Cluster name:       tidb-sherly
Cluster version:    v8.5.6
Deploy user:        sherly
SSH type:           builtin
Dashboard URL:      http://192.168.4.100:2481/dashboard
Grafana URL:        http://192.168.4.100:3101
```

---

### 6. Akses Cluster

| Service             | URL / Command                                                         | Credential                         |
|---------------------|-----------------------------------------------------------------------|-------------------------------------|
| **TiDB Dashboard** | `http://192.168.4.100:2481/dashboard`                                  | root : `4^R8@EjU6*Vh+5Wk21`       |
| **Grafana**         | `http://192.168.4.100:3101/login`                                      | admin : admin (default)            |
| **MySQL Client**    | `mysql -h 192.168.4.100 -P 4101 -u root -p --skip-ssl`                | root : `4^R8@EjU6*Vh+5Wk21`       |

> 💡 **Flag `--skip-ssl`** diperlukan untuk menghindari error koneksi SSL saat connect via MySQL client.

---

## Migrasi Data menggunakan TiDB Data Migration (DM)

**TiDB Data Migration (DM)** adalah tool resmi untuk migrasi data dari MySQL/MariaDB ke TiDB secara **full + incremental** (CDC-like).

---

### Fase Migrasi

Proses migrasi menggunakan TiDB DM terdiri dari **3 fase** utama:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Fase Migrasi TiDB Data Migration                         │
│                                                                              │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────┐     │
│   │   ① FASE DUMP    │───▶│   ② FASE LOAD   │───▶│   ③ FASE SYNC       │     │
│   │                  │    │                  │    │                     │     │
│   │ Membaca &        │    │ Melempar data    │    │ Memantau setiap     │     │
│   │ menarik semua    │    │ dari file temp   │    │ perubahan (CDC)     │     │
│   │ data dari DB     │    │ ke TiDB          │    │ secara real-time    │     │
│   │ sumber ke file   │    │                  │    │                     │     │
│   │ sementara di     │    │                  │    │ INSERT / UPDATE /   │     │
│   │ worker           │    │                  │    │ DELETE diteruskan   │     │
│   │                  │    │                  │    │ otomatis ke TiDB    │     │
│   └─────────────────┘    └─────────────────┘    └─────────────────────┘     │
│                                                                              │
│   📌 DM memiliki checkpoint, jadi jika server mati proses bisa di-resume    │
└──────────────────────────────────────────────────────────────────────────────┘
```

| Fase       | Penjelasan                                                                                                                      |
|------------|---------------------------------------------------------------------------------------------------------------------------------|
| **Dump**   | Membaca dan menarik semua data yang ada di database sumber, lalu menyimpannya ke **file sementara (temporary file)** di worker. |
| **Load**   | Setelah data berhasil ditarik, DM melempar file/data tersebut ke TiDB.                                                         |
| **Sync**   | Setelah semua data berhasil masuk, proses Sync memantau **setiap perubahan (INSERT/UPDATE/DELETE)** yang terjadi di database sumber secara **real-time** dan langsung meneruskannya ke TiDB. |

> ⏱️ **Catatan Benchmark**: Proses load (fase 2) memakan waktu cukup lama tergantung ukuran data. Pada testing: mulai jam 15.30, setelah 1 jam progres mencapai 15.43%.

---

### Menjalankan DM (Playground)

#### 1. Jalankan DM Master

```bash
tiup dm-master \
  --name=master1 \
  --master-addr=127.0.0.1:8261 \
  --data-dir=./dm-data/master
```

#### 2. Jalankan DM Worker

```bash
tiup dm-worker \
  --name=worker1 \
  --join=127.0.0.1:8261 \
  --worker-addr=127.0.0.1:8262
```

#### 3. Monitoring & Control

```bash
# Cek status member DM
tiup dmctl --master-addr=127.0.0.1:8261 list-member
```

#### 4. Konfigurasi Source & Task

Buat 2 file YAML:
- **`source.yaml`** → konfigurasi koneksi ke database sumber
- **`task.yaml`** → konfigurasi task migrasi (database, tabel, mode, dsb.)

Contoh `source.yaml`:

```yaml
source-id: "mariadb-source"
from:
  host: "192.168.x.x"
  port: 3306
  user: "root"
  password: "password_kamu"
```

Contoh `task.yaml`:

```yaml
name: "migrasi"
task-mode: "all"            # full + incremental
source-config:
  source-id: "mariadb-source"
target-database:
  host: "127.0.0.1"
  port: 4000
  user: "root"
  password: ""
block-allow-list:
  bw-rule-1:
    do-dbs: ["nama_database"]
    do-tables:
      - db-name: "nama_database"
        tbl-name: "nama_tabel"
```

#### 5. Jalankan Task

```bash
tiup dmctl:v8.5.6 --master-addr=127.0.0.1:8261 start-task task.yaml
```

#### 6. Cek Status Task

```bash
tiup dmctl:v8.5.6 --master-addr=127.0.0.1:8261 query-status migrasi
```

#### 7. Resume Task (jika server mati)

```bash
# DM memiliki checkpoint, jadi proses bisa dilanjutkan tanpa mulai dari awal
tiup dmctl:v8.5.6 --master-addr=127.0.0.1:8261 resume-task migrasi
```

---

### Menjalankan DM (Develop Cluster)

#### 1. Jalankan DM Master & Worker

```bash
# Jalankan master
tiup dm-master \
  --name=master1 \
  --master-addr=127.0.0.1:8261 \
  --data-dir=./dm-data/master

# Jalankan worker
tiup dm-worker \
  --name=worker1 \
  --join=127.0.0.1:8261 \
  --worker-addr=127.0.0.1:8262
```

#### 2. Daftarkan Database Sumber

```bash
# Daftarkan source database
tiup dmctl --master-addr=127.0.0.1:8261 operate-source create source.yaml

# Cek koneksi source berhasil
tiup dmctl --master-addr=127.0.0.1:8261 operate-source show
```

#### 3. Jalankan Task Migrasi

```bash
tiup dmctl --master-addr=127.0.0.1:8261 start-task task.yaml
```

#### 4. Pantau Progress

```bash
tiup dmctl --master-addr=127.0.0.1:8261 query-status migrasi
```

#### 5. Matikan Worker (jika diperlukan)

```bash
tiup dmctl --master-addr=127.0.0.1:8261 operate-source stop mariadb_dtage
```

---

### Penambahan Tabel untuk Migrasi

Jika ada penambahan tabel baru yang perlu dimigrasi, ikuti langkah berikut:

```bash
# 1. Matikan task yang sedang berjalan
tiup dmctl --master-addr=127.0.0.1:8261 stop-task migrasi

# 2. Edit file task.yaml → tambahkan tabel baru di bagian block-allow-list

# 3. Jalankan kembali task
tiup dmctl --master-addr=127.0.0.1:8261 start-task migrasi
```

> ⚠️ **Penting**: Pastikan task **di-stop dulu** sebelum mengubah konfigurasi `task.yaml`, lalu start ulang task-nya.

---

## Troubleshooting

### Common Issues

| Problem                                    | Solusi                                                                                    |
|--------------------------------------------|-------------------------------------------------------------------------------------------|
| `Playground macet / tidak bisa start`      | Stop semua proses: `tiup clean --all`, lalu jalankan ulang                                |
| `Cannot connect via MySQL client`          | Pastikan TiDB berjalan, cek port `4000` (playground) atau `4101` (cluster)                |
| `SSL connection error`                     | Tambahkan flag `--skip-ssl` pada perintah MySQL client                                    |
| `DM task error / stuck`                    | Resume task: `tiup dmctl:v8.5.6 --master-addr=127.0.0.1:8261 resume-task migrasi`        |
| `Cluster tidak bisa start`                 | Cek log: `tiup cluster display tidb-sherly` dan periksa apakah semua node aktif           |
| `Lupa password root cluster`              | Password root hanya ditampilkan sekali saat `--init`. Jika hilang, perlu reset manual     |
| `Load migrasi sangat lambat`              | Normal untuk data besar. Pantau progress secara berkala via `query-status`                |

---

## Referensi

- [TiDB Official Documentation](https://docs.pingcap.com/tidb/stable)
- [TiUP Documentation](https://docs.pingcap.com/tidb/stable/tiup-overview)
- [TiDB Data Migration Guide](https://docs.pingcap.com/tidb/stable/dm-overview)
- [TiDB Architecture Overview](https://docs.pingcap.com/tidb/stable/tidb-architecture)
- [TiDB Playground Quick Start](https://docs.pingcap.com/tidb/stable/quick-start-with-tidb)
- [TiDB GitHub Repository](https://github.com/pingcap/tidb)

---
