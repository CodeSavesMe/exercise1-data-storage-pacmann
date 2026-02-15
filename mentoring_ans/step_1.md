# Data Warehouse Requirements Gathering

**Studi Kasus:** Olist E-Commerce

## 1. Tujuan Tahap Ini

Tahap *Requirements Gathering* bertujuan menerjemahkan kebutuhan bisnis menjadi spesifikasi teknis desain *Data Warehouse*. Fokus utamanya adalah mengumpulkan informasi mengenai laporan yang diperlukan, definisi metrik yang akurat, level detail data (*granularity*), serta mendeteksi potensi anomali data seperti *double counting* atau bias akibat relasi data.

**Sumber Data Tersedia (OLTP):**
`orders`, `order_items`, `order_payments`, `order_reviews`, `customers`, `sellers`, `products`, `product_category_name_translation`, `geolocation`.

---

## 2. Analisis Kebutuhan & Implikasi Teknis

Berikut adalah pertanyaan kunci yang mensimulasikan diskusi antara tim data dan stakeholder, dikelompokkan berdasarkan area teknis.

### A. Cakupan & Granularity (Kedetailan Data)

#### 1. Cakupan Analisis

**Pertanyaan:** "Analisis apa saja yang diharapkan tersedia dari Data Warehouse?"

**Jawaban Stakeholder:**
"Kebutuhan utama mencakup 4 domain:

1. **Sales:** GMV, AOV, performa seller/produk/wilayah.
2. **Delivery:** Lead time, delay, dan bottleneck.
3. **Reviews:** Skor kepuasan dan analisis teks.
4. **Payments:** Tipe pembayaran, cicilan, dan rekonsiliasi."

**NOTE!**
Jawaban ini membantu mengelompokkan kebutuhan menjadi “Domain Bisnis”. Biasanya tiap domain dimodelkan menjadi satu atau beberapa tabel fakta (*fact table*) terpisah agar analisis lebih stabil dan tidak mudah salah hitung.

---

#### 2. Level Detail Data (Grain)

**Pertanyaan:** "Untuk analisis Sales, level detail seperti apa yang diperlukan: per order atau per item?"

**Jawaban Stakeholder:**
"Analisis perlu *drill-down* sampai **produk dan seller**. Jadi minimal harus tersedia data di level **Order Item**. Satu order bisa berisi banyak item dari seller berbeda."

**NOTE!**
Ini berkaitan dengan penentuan *Grain*.

* Jika data hanya disimpan per Order, sistem sulit menjawab “Berapa GMV Seller A?” karena satu order bisa mengandung item Seller A dan Seller B.

---

### B. Definisi Metrik & Logika Bisnis

#### 3. Definisi GMV (Gross Merchandise Value)

**Pertanyaan:** "Bagaimana definisi GMV yang disepakati? Apakah ongkos kirim (*freight*) termasuk GMV?"

**Jawaban Stakeholder:**
"GMV dihitung murni dari **total harga item** (`price`). Ongkos kirim (`freight_value`) ditampilkan terpisah. Total transaksi adalah `price + freight`."

**NOTE!**
Penting untuk mengunci definisi metrik (“Kamus Data”) di awal agar angka tidak berbeda antar laporan.

* Tanpa kesepakatan ini, GMV bisa “overstated” jika dianggap `price + freight`.

---

#### 4. Filter Status Order (KPI Finansial)

**Pertanyaan:**
"Status order apa yang dianggap valid untuk KPI finansial (GMV)?"

**Jawaban Stakeholder:**
"Hanya order yang berstatus **'delivered'**. Status lain (canceled, shipped, processing) hanya dipakai untuk metrik operasional, bukan finansial."

**NOTE!**
Tanpa aturan filtering status, GMV dapat tidak valid karena memasukkan transaksi yang batal atau belum selesai.

**Best Practice:**
Agar filter dashboard lebih konsisten dan mudah dipakai, status sering dikelompokkan menjadi *status group*, misalnya:

* Delivered
* Canceled
* In Progress

> **NOTE!**
> `orders.order_status` adalah nilai mentah dari OLTP (mis. delivered, canceled, unavailable, dll).
> Di Data Warehouse dibuat `dim_order_status.status_group` (mis. Delivered/Canceled/In Progress) sebagai governance layer untuk filter KPI yang konsisten.

---

### C. Kompleksitas Pengiriman & Pembayaran

#### 5. Analisis Pengiriman (Delivery Performance)

**Pertanyaan:**
"Apakah cukup metrik level order atau perlu sampai level seller/produk?"

**Jawaban Stakeholder:**
"Keduanya. KPI kecepatan kirim ada di level Order, tapi analisis keterlambatan seller ada di level Produk/Item."

**NOTE!**
Menggabungkan dua level detail yang berbeda berisiko data terduplikasi. Strategi pemodelan diperlukan karena:

* Timestamp delivery ke customer ada di `orders.order_delivered_customer_date` (order-level)
* Timestamp serah-terima ke carrier ada di `orders.order_delivered_carrier_date` (order-level)
* `seller_id` dan `product_id` ada di `order_items` (item-level)


---

#### 6. Multi-Payment per Order (Grain Pembayaran)

**Pertanyaan:** "Apakah satu order selalu punya satu pembayaran, atau bisa lebih dari satu transaksi pembayaran?"

**Jawaban Stakeholder:**
"Satu order bisa punya **lebih dari satu record pembayaran** (misalnya cicilan atau kombinasi metode). Biasanya dibedakan oleh urutan transaksi seperti `payment_sequential`."

**NOTE!**
Ini menjelaskan bahwa pembayaran tidak selalu 1 baris per order. Jika pembayaran digabung mentah dengan data item penjualan, jumlah baris dapat “meledak” (multi-item × multi-payment) dan berisiko menyebabkan perhitungan ganda.

---

#### 7. Tren Waktu Pembayaran (Proxy Date)

**Pertanyaan:** "Bagaimana tren waktu pembayaran dihitung jika tidak ada timestamp pembayaran?"

**Jawaban Stakeholder:**
"Gunakan **tanggal purchase order** sebagai acuan (*proxy*)."

**NOTE!**
Penggunaan **Proxy Date** menjaga konsistensi tren waktu ketika timestamp asli tidak tersedia di tabel pembayaran.

---

#### 8. Rekonsiliasi Data

**Pertanyaan:** "Apakah diperlukan rekonsiliasi antara sales dan payments?"

**Jawaban Stakeholder:**
"Ya. Diperlukan pengecekan selisih antara GMV delivered dan total payments delivered."

**NOTE!**
Ini bagian dari *Data Quality*. Selisih dapat terjadi karena diskon, voucher, pembulatan, atau karakteristik data—yang penting selisih terukur dan dapat dipantau.

---

### D. Tantangan Relasi One-to-Many, Customer, & Lifecycle

#### 9. Perspektif Customer: Customer Unik vs Alamat Transaksi

**Pertanyaan:** "Analisis customer yang dibutuhkan fokus pada customer unik (repeat behavior) atau alamat per transaksi?"

**Jawaban Stakeholder:**
“Keduanya dibutuhkan:

* Perspektif **customer unik** untuk memahami repeat order/behavior,
* Perspektif **alamat transaksi** untuk analisis wilayah pengiriman per order.”

**NOTE!**
Satu entitas “customer” sering punya dua sudut pandang: identitas unik untuk perilaku pelanggan, dan identitas transaksi untuk konteks alamat/lokasi pengiriman. Keduanya sering dibutuhkan dalam pelaporan.

---

#### 10. Risiko Analisis Review (Order-level vs Item-level)

**Pertanyaan:** "Apa risiko analisis review per seller jika satu order berisi banyak item?"

**Jawaban Stakeholder:**
"Risikonya adalah **bias**. Ulasan diberikan per order. Jika order punya 3 item, ulasan tersebut tidak boleh dihitung 3 kali secara mentah."

**NOTE!**
Ini risiko **Double Counting** akibat one-to-many. Review perlu dialokasikan dengan aturan yang konsisten agar analisis per seller/kategori tetap adil.

---

#### 11. Aturan Alokasi Review

**Pertanyaan:** "Bagaimana aturan alokasi nilai review ke setiap item?"

**Jawaban Stakeholder:**
"Preferensi utama adalah **berbobot berdasarkan harga item**. Jika tidak, bagi rata (*equal split*)."

**NOTE!**
Aturan **Allocation Logic** diperlukan agar laporan review-by-seller tidak bias dan tidak menghasilkan ranking yang berubah-ubah antar dashboard.

---

#### 12. Data Terlambat (Late Arriving Data)

**Pertanyaan:** "Apakah terdapat kondisi data yang datang terlambat, terutama tanggal delivered?"

**Jawaban Stakeholder:**
"Ada. Status delivered sering baru terisi belakangan."

**NOTE!**
Implikasi pada pipeline ETL/ELT: tidak cukup append-only. Diperlukan mekanisme **update/backfill** untuk memperbarui data historis.

**Tambahan (Best Practice):**
Gunakan *backfill window* rutin, misalnya **7–14 hari** ke belakang, untuk memastikan perubahan status/timestamp yang datang terlambat tetap tercatat dan metrik tetap akurat.

---

## 3. Ringkasan Keputusan Teknis

| Aspek               | Keputusan                                         | Alasan Teknis                                             |
| ------------------- | ------------------------------------------------- | --------------------------------------------------------- |
| **Cakupan**         | Sales, Delivery, Reviews, Payments                | Mewakili kebutuhan analitik utama stakeholder.            |
| **Grain Sales**     | **Order Item**                                    | Mendukung drill-down akurat per Seller & Produk.          |
| **Grain Payments**  | **Payment Transaction** (multi-payment per order) | Menghindari ledakan baris dan double counting.            |
| **Metrik GMV**      | **Price Only**                                    | Definisi bisnis baku (tanpa ongkir).                      |
| **Validasi KPI**    | **Delivered-only** + *status grouping*            | Mengunci KPI finansial dan memudahkan filter dashboard.   |
| **Tanggal Payment** | **Proxy: purchase date**                          | Tidak ada timestamp payment di sumber.                    |
| **Logic Review**    | **Weighted Allocation** (fallback equal)          | Menghindari bias/double counting pada review-by-seller.   |
| **Rekonsiliasi**    | GMV delivered vs total payments delivered         | Kontrol kualitas dan monitoring anomali.                  |
| **Pipeline**        | **Support update/backfill** (7–14 hari)           | Mengakomodasi late-arriving lifecycle (delivered date).   |
| **Sumber Produk**   | Termasuk `product_category_name_translation`      | Konsistensi kategori (PT→EN) untuk reporting.             |
| **Customer View**   | Customer unik & alamat transaksi                  | Mendukung analisis repeat behavior dan lokasi pengiriman. |

---


## Glossary — Data Warehouse & Dimensional Modeling


| Istilah                                  | Penjelasan singkat                                                                                        |
|------------------------------------------| --------------------------------------------------------------------------------------------------------- |
| **Stakeholder**                          | Pihak pengguna data yang menentukan kebutuhan laporan dan KPI (Finance, Ops, CX, BI).                     |
| **Requirements Gathering**               | Tahap mengumpulkan kebutuhan: laporan apa, definisi metrik, filter, dan level detail.                     |
| **OLTP**                                 | Database operasional transaksi (tabel sumber seperti `orders`, `order_items`, dll).                       |
| **Data Warehouse (DW)**                  | Sistem penyimpanan terstruktur untuk analitik: konsisten, siap agregasi, mudah diaudit.                   |
| **Dimensional Modeling (Kimball-style)** | Metode desain DW: memisahkan **Fact** (angka) dan **Dimension** (konteks).                                |
| **Star Schema**                          | Pola relasi fact di tengah dengan banyak dimension di sekelilingnya.                                      |
| **Fact Constellation (Multi-fact)**      | Desain dengan beberapa fact table yang berbagi dimensi (sales, payments, lifecycle, reviews).             |
| **Business Process**                     | Domain analitik: **Sales**, **Delivery**, **Reviews**, **Payments**.                                      |
| **Definition Locks**                     | Kesepakatan baku definisi metrik & filter (GMV, delivered-only, proxy payment date).                      |
| **KPI**                                  | Metrik performa bisnis (GMV, AOV, late rate, avg review score, dll).                                      |
| **GMV (Gross Merchandise Value)**        | Total nilai barang terjual; pada desain: `SUM(order_items.price)` (tanpa freight).                        |
| **Freight / Freight Value**              | Ongkos kirim (`order_items.freight_value`), ditampilkan terpisah dari GMV.                                |
| **AOV (Average Order Value)**            | Rata-rata nilai order (umumnya dihitung pada order yang delivered).                                       |
| **Grain (Granularity)**                  | Definisi “1 baris fact mewakili apa” (order item / order / payment transaction / review).                 |
| **Fact Table**                           | Tabel berisi angka/measures pada grain tertentu, memakai FK ke dimensi.                                   |
| **Dimension Table**                      | Tabel konteks untuk filter/grouping (date, product, seller, customer, status, location).                  |
| **Measure**                              | Nilai numerik untuk agregasi (SUM/AVG/COUNT), mis. `item_price`, `payment_value`, `delay_days`.           |
| **Attribute**                            | Kolom deskriptif pada dimensi (city/state/kategori) untuk segmentasi dan filter.                          |
| **Business Key (BK)**                    | Kunci alami dari OLTP (mis. `product_id`, `seller_id`, `customer_id`).                                    |
| **Surrogate Key (SK)**                   | Kunci buatan di DW (biasanya integer) untuk join cepat dan konsisten.                                     |
| **Foreign Key (FK)**                     | Kolom di fact yang menunjuk ke dimensi (mis. `product_sk`, `customer_address_sk`).                        |
| **Degenerate Dimension (DD)**            | ID yang disimpan langsung di fact tanpa dimensi terpisah (mis. `order_id`, `review_id`).                  |
| **Conformed Dimension**                  | Dimensi yang dipakai bersama oleh banyak fact dengan definisi yang sama.                                  |
| **Role-Playing Dimension**               | Satu dimensi dipakai untuk beberapa peran; contoh `dim_date` untuk purchase/delivered/review.             |
| **Transaction Fact**                     | Fact kejadian detail (mis. 1 baris per `order_item`, 1 baris per `payment_sequential`).                   |
| **Accumulating Snapshot Fact**           | Fact 1 baris per proses (order lifecycle) yang kolomnya bisa ter-update saat status berubah.              |
| **Periodic Snapshot Fact**               | Ringkasan periodik (harian/mingguan/bulanan); disebut sebagai konsep, tidak wajib di desain inti.         |
| **Derived Fact**                         | Fact turunan untuk mengubah grain; contoh: order-level delivery ditempel ke item-level.                   |
| **Bridge Table**                         | Tabel penghubung untuk relasi kompleks/alokasi; contoh: review (order-level) ke item/seller.              |
| **Allocation Factor**                    | Bobot pembagian nilai (mis. review) saat 1 event terkait banyak item.                                     |
| **Price-weighted Allocation**            | Alokasi berbobot harga: `item_price / SUM(item_price)` per order.                                         |
| **Equal Split**                          | Alokasi rata: `1 / jumlah_item` per order (fallback jika pembobotan tidak valid).                         |
| **One-to-Many**                          | Relasi 1 baris ke banyak baris (order → items, order → payments).                                         |
| **Double Counting**                      | Angka terhitung ganda akibat join one-to-many tanpa kontrol grain.                                        |
| **Bias (Analitik)**                      | Hasil analisis menjadi tidak adil/salah (contoh: review order-level dihitung berulang untuk banyak item). |
| **Payment Transaction**                  | Satu record pembayaran pada `order_payments` ditandai `payment_sequential`.                               |
| **Multi-payment per Order**              | Satu order bisa punya banyak pembayaran; alasan pemisahan `fact_payments_txn` dari sales.                 |
| **Proxy Date**                           | Tanggal pengganti saat timestamp asli tidak tersedia; payment trend memakai `order_purchase_timestamp`.   |
| **Lead Time**                            | Durasi antar milestone proses pengiriman (purchase→approved→carrier→delivered).                           |
| **Delay / Delay Days**                   | Selisih antara delivered dan estimated delivery date (`delay_days`).                                      |
| **Late Rate / Is Late**                  | Indikator keterlambatan (`is_late`) jika delivered melewati estimated date.                               |
| **Late-Arriving Data**                   | Data penting datang belakangan (mis. delivered date/status terisi setelah beberapa hari).                 |
| **Backfill Window**                      | Rentang muat ulang historis (7–14 hari) untuk menangkap update late-arriving.                             |
| **MERGE/UPSERT**                         | Pola load yang mendukung insert+update berdasarkan key (utama untuk accumulating snapshot).               |
| **Casting**                              | Mengubah tipe data; pada dataset ini tanggal OLTP `text` → `timestamp/date` di staging.                   |
| **Staging Layer**                        | Lapisan sebelum DW untuk casting, cleaning, standardisasi kolom OLTP.                                     |
| **Auditability / Traceability**          | Kemampuan menelusuri angka ke sumber (dibantu DD seperti `order_id`).                                     |
| **Reconciliation**                       | Pengecekan selisih metrik lintas domain (GMV delivered vs total payments delivered).                      |
| **Data Quality Checks**                  | Validasi seperti missing keys, nilai aneh, dan rekonsiliasi untuk monitoring kualitas.                    |
| **Unknown Member (SK=0)**                | Baris default di dimensi untuk menjaga FK valid saat key belum tersedia/NULL.                             |
| **Customer Unique vs Address**           | `customer_unique_id` untuk identitas unik (repeat behavior), `customer_id` untuk alamat transaksi.        |
| **Geospatial Aggregation**               | Agregasi `geolocation` per zip prefix (avg lat/lng) agar stabil untuk peta/heatmap.                       |
| **Category Translation (PT→EN)**         | Join `products.product_category_name` ke `product_category_name_translation` untuk kategori Inggris.      |
| **NLP Topic Dimension   **               | `dim_review_topic`: hasil pipeline NLP (bukan OLTP), boleh nullable di `fact_reviews`.                    |

---


<div align="center">

### Mentoring - Exercise 1 - Data Storage
Dokumen ini dibuat sebagai bagian dari pembelajaran di <strong>Pacmann Academy Bootcamp</strong>.

<a href="https://pacmann.io">
  <img src="https://img.shields.io/badge/BOOTCAMP%20%7C%20PACMANN%20ACADEMY-0D3B66?style=for-the-badge&logoColor=white" alt="Pacmann Academy">
</a>

<a href="https://pacmann.io">pacmann.io</a>

</div>


