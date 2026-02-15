# Appendix — Source-to-Target Proof (OLTP → Kimball DW)

Dokumen ini membuktikan bahwa seluruh **dimensi** dan **fact/bridge** pada desain Kimball di Step #2 **benar-benar dapat dibangun** dari source OLTP yang tersedia **berdasarkan DDL yang diberikan** (tanpa asumsi kolom tambahan). Jika ada atribut yang bukan kolom OLTP (misalnya surrogate key, kalender, flag), dicatat sebagai **generated/derived** di bagian akhir.

**OLTP Tables (sesuai DDL):**
`customers`, `geolocation`, `order_items`, `order_payments`, `order_reviews`, `orders`, `product_category_name_translation`, `products`, `sellers`

> **NOTE!**
> Semua kolom tanggal pada OLTP disimpan sebagai `text`. Konversi tipe data (*casting*) ke `timestamp/date` dilakukan di staging. Ini bukan “kolom tambahan”, hanya transformasi tipe.

---

## Quick Checklist: Apakah semua kebutuhan Step #2 ada di OLTP?

* Dimensi customer (unik & alamat), seller, product, payment_type, order_status, location → ✅ tersedia
* Timestamp lifecycle order (purchase/approved/carrier/delivered/estimated) → ✅ tersedia di `orders` (text)
* Sales measures (price, freight) → ✅ tersedia di `order_items`
* Payment measures (payment_value, installments, payment_type) → ✅ tersedia di `order_payments`
* Review score & text & date → ✅ tersedia di `order_reviews`
* Category translation PT→EN → ✅ tersedia di `product_category_name_translation`
* Bridge review×item dan allocation factor berbasis price → ✅ bisa dihitung dari `order_reviews` + `order_items`

Satu-satunya elemen **non-OLTP** yang memang didesain opsional: `dim_review_topic` (hasil NLP).

---

## 1) OLTP Column Inventory (berdasarkan DDL)

### 1.1 `public.orders`

* `order_id` (PK candidate)
* `customer_id`
* `order_status`
* `order_purchase_timestamp` (text)
* `order_approved_at` (text)
* `order_delivered_carrier_date` (text)
* `order_delivered_customer_date` (text)
* `order_estimated_delivery_date` (text)

### 1.2 `public.order_items`

* `order_id`
* `order_item_id`
* `product_id`
* `seller_id`
* `shipping_limit_date` (text)
* `price` (real)
* `freight_value` (real)

### 1.3 `public.order_payments`

* `order_id`
* `payment_sequential`
* `payment_type`
* `payment_installments`
* `payment_value`

### 1.4 `public.order_reviews`

* `review_id`
* `order_id`
* `review_score`
* `review_comment_title`
* `review_comment_message`
* `review_creation_date` (text)

### 1.5 `public.customers`

* `customer_id`
* `customer_unique_id`
* `customer_zip_code_prefix`
* `customer_city`
* `customer_state`

### 1.6 `public.sellers`

* `seller_id`
* `seller_zip_code_prefix`
* `seller_city`
* `seller_state`

### 1.7 `public.products`

* `product_id`
* `product_category_name`
* `product_photos_qty`
* `product_weight_g`
* `product_length_cm`
* `product_height_cm`
* `product_width_cm`
* (ada juga: `product_name_lenght`, `product_description_lenght`)

### 1.8 `public.product_category_name_translation`

* `product_category_name`
* `product_category_name_english`

### 1.9 `public.geolocation`

* `geolocation_zip_code_prefix`
* `geolocation_lat`
* `geolocation_lng`
* `geolocation_city`
* `geolocation_state`

---

## 2) DIMENSIONS — Proof & Source Mapping

> **Aturan umum DW:** setiap dimensi akan memiliki `*_sk` (surrogate key). SK dibuat di DW (generated), sementara atribut dimensi diambil dari OLTP.

### 2.1 `dim_date` (Generated Calendar)

**Tujuan:** dimensi tanggal dipakai berulang untuk purchase/delivered/review/etc.
**Proof:** rentang tanggal dapat ditentukan dari kolom-kolom tanggal OLTP.

**Sumber tanggal yang tersedia:**

* `orders.order_purchase_timestamp`
* `orders.order_approved_at`
* `orders.order_delivered_carrier_date`
* `orders.order_delivered_customer_date`
* `orders.order_estimated_delivery_date`
* `order_reviews.review_creation_date`
* (opsional) `order_items.shipping_limit_date`

**Contoh query untuk menentukan min/max tanggal (staging):**

```sql
SELECT
  MIN(CAST(order_purchase_timestamp AS date)) AS min_date,
  MAX(CAST(order_estimated_delivery_date AS date)) AS max_date
FROM public.orders;
```

✅ `dim_date` dapat dibuat dari calendar generator berdasarkan range tanggal OLTP.

---

### 2.2 `dim_customer_person`

**Business Key:** `customer_unique_id`
**Source:** `public.customers`

**Mapping:**

* `customer_unique_id` ← `customers.customer_unique_id`

```sql
SELECT DISTINCT
  customer_unique_id
FROM public.customers
WHERE customer_unique_id IS NOT NULL;
```

✅ tersedia di OLTP.

---

### 2.3 `dim_customer_address`

**Business Key:** `customer_id`
**Source:** `public.customers`

**Mapping:**

* `customer_id` ← `customers.customer_id`
* `zip_code_prefix` ← `customers.customer_zip_code_prefix`
* `city` ← `customers.customer_city`
* `state` ← `customers.customer_state`

```sql
SELECT
  customer_id,
  customer_zip_code_prefix AS zip_code_prefix,
  customer_city            AS city,
  customer_state           AS state
FROM public.customers;
```

✅ tersedia di OLTP.

---

### 2.4 `dim_location` (Geospatial, aggregated by zip prefix)

**Business Key:** `zip_code_prefix`
**Source:** `public.geolocation` (perlu agregasi karena banyak titik per zip)

**Mapping:**

* `zip_code_prefix` ← `geolocation.geolocation_zip_code_prefix`
* `avg_lat` ← AVG(`geolocation_lat`)
* `avg_lng` ← AVG(`geolocation_lng`)
* `city` ← representative value (mis. MAX)
* `state` ← representative value (mis. MAX)

```sql
SELECT
  geolocation_zip_code_prefix AS zip_code_prefix,
  AVG(geolocation_lat)        AS avg_lat,
  AVG(geolocation_lng)        AS avg_lng,
  MAX(geolocation_city)       AS city,
  MAX(geolocation_state)      AS state
FROM public.geolocation
GROUP BY 1;
```

✅ tersedia di OLTP (butuh agregasi).

---

### 2.5 `dim_seller`

**Business Key:** `seller_id`
**Source:** `public.sellers`

**Mapping:**

* `seller_id` ← `sellers.seller_id`
* `zip_code_prefix` ← `sellers.seller_zip_code_prefix`
* `city` ← `sellers.seller_city`
* `state` ← `sellers.seller_state`

```sql
SELECT
  seller_id,
  seller_zip_code_prefix AS zip_code_prefix,
  seller_city            AS city,
  seller_state           AS state
FROM public.sellers;
```

✅ tersedia di OLTP.

---

### 2.6 `dim_product`

**Business Key:** `product_id`
**Source:** `public.products` + `public.product_category_name_translation`

**Mapping:**

* `product_id` ← `products.product_id`
* `category_name_pt` ← `products.product_category_name`
* `category_name_en` ← `product_category_name_translation.product_category_name_english`
* `photos_qty` ← `products.product_photos_qty`
* `weight_g` ← `products.product_weight_g`
* `length_cm` ← `products.product_length_cm`
* `height_cm` ← `products.product_height_cm`
* `width_cm` ← `products.product_width_cm`

```sql
SELECT
  p.product_id,
  p.product_category_name         AS category_name_pt,
  t.product_category_name_english AS category_name_en,
  p.product_photos_qty            AS photos_qty,
  p.product_weight_g              AS weight_g,
  p.product_length_cm             AS length_cm,
  p.product_height_cm             AS height_cm,
  p.product_width_cm              AS width_cm
FROM public.products p
LEFT JOIN public.product_category_name_translation t
  ON t.product_category_name = p.product_category_name;
```

✅ tersedia di OLTP (butuh join translation).

---

### 2.7 `dim_payment_type`

**Business Key:** `payment_type`
**Source:** `public.order_payments`

```sql
SELECT DISTINCT
  payment_type
FROM public.order_payments
WHERE payment_type IS NOT NULL;
```

✅ tersedia di OLTP.

---

### 2.8 `dim_order_status`

**Business Key:** `order_status`
**Source:** `public.orders`

```sql
SELECT DISTINCT
  order_status
FROM public.orders
WHERE order_status IS NOT NULL;
```

✅ `order_status` tersedia di OLTP.
⚠️ `status_group` bersifat derived (lihat catatan kolom tambahan).

---

### 2.9 `dim_review_topic` (Opsional)

**Source:** tidak ada di OLTP (hasil NLP pipeline).
✅ bersifat opsional dan tidak menghambat pembangunan DW inti.

---

## 3) FACTS & BRIDGE — Proof & Source Mapping

### 3.1 `fact_sales_item` (Transaction Fact)

**Grain:** 1 baris per order item (`order_id`, `order_item_id`)
**Sources:** `public.order_items` + `public.orders`

**Kolom yang dibutuhkan tersedia di OLTP:**

* `order_id`, `order_item_id` ← `order_items`
* `seller_id`, `product_id` ← `order_items`
* `item_price` ← `order_items.price`
* `freight_value` ← `order_items.freight_value`
* `purchase_ts` (proxy untuk purchase_date_sk) ← `orders.order_purchase_timestamp`
* `customer_id` ← `orders.customer_id` (untuk dim_customer_address & join ke customers)
* `order_status` ← `orders.order_status`

```sql
SELECT
  oi.order_id,
  oi.order_item_id,
  o.customer_id,
  oi.seller_id,
  oi.product_id,
  o.order_status,
  CAST(o.order_purchase_timestamp AS timestamp) AS purchase_ts,
  oi.price                                      AS item_price,
  oi.freight_value                              AS freight_value
FROM public.order_items oi
JOIN public.orders o
  ON o.order_id = oi.order_id;
```

✅ semua input fact tersedia di OLTP.

---

### 3.2 `fact_payments_txn` (Transaction Fact)

**Grain:** 1 baris per payment transaction (`order_id`, `payment_sequential`)
**Sources:** `public.order_payments` + `public.orders`

**Kolom yang dibutuhkan tersedia di OLTP:**

* `order_id`, `payment_sequential` ← `order_payments`
* `payment_type` ← `order_payments.payment_type`
* `installments` ← `order_payments.payment_installments`
* `payment_value` ← `order_payments.payment_value`
* `purchase_ts` (proxy) ← `orders.order_purchase_timestamp`
* `customer_id`, `order_status` ← `orders`

```sql
SELECT
  p.order_id,
  p.payment_sequential,
  p.payment_type,
  p.payment_installments AS installments,
  p.payment_value,
  o.customer_id,
  o.order_status,
  CAST(o.order_purchase_timestamp AS timestamp) AS purchase_ts
FROM public.order_payments p
JOIN public.orders o
  ON o.order_id = p.order_id;
```

✅ semua input fact tersedia di OLTP.

> **NOTE!** tidak ada timestamp pembayaran di OLTP → penggunaan purchase_ts sebagai proxy adalah aturan bisnis, bukan asumsi kolom.

---

### 3.3 `fact_order_lifecycle` (Accumulating Snapshot Fact)

**Grain:** 1 baris per order (`order_id`)
**Source:** `public.orders`

**Kolom timestamp lifecycle tersedia di OLTP:**

* purchase/approved/carrier/delivered/estimated

```sql
SELECT
  order_id,
  customer_id,
  order_status,
  CAST(order_purchase_timestamp      AS timestamp) AS purchase_ts,
  CAST(order_approved_at             AS timestamp) AS approved_ts,
  CAST(order_delivered_carrier_date  AS timestamp) AS carrier_ts,
  CAST(order_delivered_customer_date AS timestamp) AS delivered_ts,
  CAST(order_estimated_delivery_date AS timestamp) AS estimated_ts
FROM public.orders;
```

**Measures (durasi/delay) adalah derived dari timestamp OLTP:**

```sql
SELECT
  order_id,
  (CAST(order_approved_at AS date) - CAST(order_purchase_timestamp AS date))           AS days_to_approve,
  (CAST(order_delivered_carrier_date AS date) - CAST(order_approved_at AS date))       AS days_to_carrier,
  (CAST(order_delivered_customer_date AS date) - CAST(order_purchase_timestamp AS date)) AS days_to_deliver,
  (CAST(order_delivered_customer_date AS date) - CAST(order_estimated_delivery_date AS date)) AS delay_days,
  (CAST(order_delivered_customer_date AS date) > CAST(order_estimated_delivery_date AS date)) AS is_late
FROM public.orders;
```

✅ input tersedia di OLTP; measures derived dari kolom OLTP.

---

### 3.4 `fact_order_item_fulfillment` (Derived Fact @ Item)

**Grain:** 1 baris per order item (`order_id`, `order_item_id`)
**Sources:** `public.order_items` + `public.orders`

Tabel ini menempelkan metrik order-lifecycle ke item-level.

```sql
SELECT
  oi.order_id,
  oi.order_item_id,
  oi.seller_id,
  oi.product_id,
  o.customer_id,
  o.order_status,
  CAST(o.order_purchase_timestamp AS timestamp)      AS purchase_ts,
  CAST(o.order_delivered_customer_date AS timestamp) AS delivered_ts,
  CAST(o.order_estimated_delivery_date AS timestamp) AS estimated_ts,
  (CAST(o.order_delivered_customer_date AS date) - CAST(o.order_purchase_timestamp AS date)) AS days_to_deliver,
  (CAST(o.order_delivered_customer_date AS date) - CAST(o.order_estimated_delivery_date AS date)) AS delay_days,
  (CAST(o.order_delivered_customer_date AS date) > CAST(o.order_estimated_delivery_date AS date)) AS is_late
FROM public.order_items oi
JOIN public.orders o
  ON o.order_id = oi.order_id;
```

✅ sepenuhnya dari OLTP; “derived” karena perubahan grain.

---

### 3.5 `fact_reviews` (Transaction Fact)

**Grain:** 1 baris per review (`review_id`)
**Sources:** `public.order_reviews` + `public.orders` + `public.customers`

**Kolom tersedia di OLTP:**

* review: score + title + message + creation date
* join ke `orders` untuk `customer_id` (alamat transaksi)
* join ke `customers` untuk `customer_unique_id` (identitas person-level)

```sql
SELECT
  r.review_id,
  r.order_id,
  o.customer_id,
  c.customer_unique_id,
  CAST(r.review_creation_date AS timestamp) AS review_ts,
  r.review_score,
  r.review_comment_title   AS review_title,
  r.review_comment_message AS review_message
FROM public.order_reviews r
JOIN public.orders o
  ON o.order_id = r.order_id
LEFT JOIN public.customers c
  ON c.customer_id = o.customer_id;
```

> **NOTE!**
> `customer_id` dari `orders` digunakan untuk lookup ke `dim_customer_address` (konteks alamat/lokasi transaksi),
> sedangkan `customer_unique_id` dari `customers` digunakan untuk lookup ke `dim_customer_person` (identitas unik untuk analisis repeat/behavior).

- ✅ `customer_id` tersedia → mendukung `dim_customer_address`
- ✅ `customer_unique_id` tersedia → mendukung `dim_customer_person`
- ⚠️ `is_negative_flag` adalah derived dari `review_score` (misalnya score ≤ 2)
---

### 3.6 `bridge_review_item_allocation` (Allocation Bridge)

**Grain:** 1 baris per (review × order_item)
**Sources:** `public.order_reviews` + `public.order_items` (harga untuk bobot)

**Base join (review order-level → item-level):**

```sql
SELECT
  r.review_id,
  r.order_id,
  oi.order_item_id,
  oi.seller_id,
  oi.product_id,
  oi.price AS item_price
FROM public.order_reviews r
JOIN public.order_items oi
  ON oi.order_id = r.order_id;
```

**Allocation factor (price-weighted) bisa dihitung tanpa kolom tambahan:**

```sql
SELECT
  r.review_id,
  r.order_id,
  oi.order_item_id,
  oi.seller_id,
  oi.product_id,
  CASE
    WHEN SUM(oi.price) OVER (PARTITION BY r.order_id) > 0
      THEN oi.price / SUM(oi.price) OVER (PARTITION BY r.order_id)
    ELSE 1.0 / COUNT(*) OVER (PARTITION BY r.order_id)
  END AS allocation_factor
FROM public.order_reviews r
JOIN public.order_items oi
  ON oi.order_id = r.order_id;
```

✅ bridge bisa dibangun dari OLTP; `allocation_factor` adalah derived.

---

## 4) Proof untuk Customer Split & Location Join

### 4.1 Customer unik vs alamat transaksi

* `orders.customer_id` digunakan untuk alamat transaksi (shipping context).
* `customers.customer_unique_id` digunakan untuk identitas unik pelanggan (behavior context).

**Proof join dari OLTP:**

```sql
SELECT
  o.order_id,
  o.customer_id,
  c.customer_unique_id
FROM public.orders o
LEFT JOIN public.customers c
  ON c.customer_id = o.customer_id;
```

✅ keduanya tersedia di OLTP.

### 4.2 Location dari zip prefix

* `customers.customer_zip_code_prefix` → match ke `geolocation.geolocation_zip_code_prefix`
* `sellers.seller_zip_code_prefix` → match ke `geolocation.geolocation_zip_code_prefix`

✅ tersedia di OLTP.

---

## 5) Catatan Kolom Tambahan (Generated/Derived — bukan kolom mentah OLTP)

Berikut adalah elemen yang **boleh ada** di DW karena termasuk best practice dimensional modeling, namun **bukan kolom tambahan dari OLTP**:

1. **Surrogate keys** (`*_sk`, termasuk `date_sk`)

   * Dibuat di DW untuk performa join dan konsistensi.

2. **Atribut kalender** pada `dim_date` (year, month, day_name, is_weekend, dst.)

   * Dihasilkan dari generator kalender.

3. **Durasi & keterlambatan** pada lifecycle (`days_to_approve`, `days_to_carrier`, `days_to_deliver`, `delay_days`, `is_late`)

   * Diturunkan dari timestamp text di `orders` setelah casting.

4. **`status_group`** pada `dim_order_status`

   * Derived mapping dari `orders.order_status`, contoh rule:

     * `delivered` → Delivered
     * `canceled` / `unavailable` → Canceled
     * selain itu → In Progress

5. **`is_negative_flag`** pada `fact_reviews`

   * Derived dari `review_score` (contoh: score <= 2 → true).

6. **`allocation_factor`** pada `bridge_review_item_allocation`

   * Derived dari `order_items.price` (price-weighted) atau equal split fallback.

7. **`dim_review_topic`**

   * Tidak berasal dari OLTP; dihasilkan oleh pipeline NLP (opsional).

---

## 6) Kesimpulan

* Seluruh dimensi dan fact/bridge inti pada desain Kimball dapat dibangun dari source OLTP sesuai DDL yang diberikan.
* Tidak ada kebutuhan asumsi kolom tambahan di OLTP. Seluruh penambahan berada di kategori **generated/derived** (SK, kalender, durasi, grouping, flag, allocation factor).
* Elemen opsional seperti topic NLP tidak memblok pembangunan Data Warehouse inti.

---

<div align="center">

### Mentoring - Exercise 1 - Data Storage
Dokumen ini dibuat sebagai bagian dari pembelajaran di <strong>Pacmann Academy Bootcamp</strong>.

<a href="https://pacmann.io">
  <img src="https://img.shields.io/badge/BOOTCAMP%20%7C%20PACMANN%20ACADEMY-0D3B66?style=for-the-badge&logoColor=white" alt="Pacmann Academy">
</a>

<a href="https://pacmann.io">pacmann.io</a>

</div>

