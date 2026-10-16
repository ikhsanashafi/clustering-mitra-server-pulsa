# LAPORAN PROYEK MACHINE LEARNING
# CLUSTERING PERILAKU MITRA SERVER PULSA MENGGUNAKAN ALGORITMA K-MEANS

**Dosen Pengampu:** ASEP MAULANA S.Kom., M.Kom  
**Disusun Oleh:**  
- **Nama:** Ikhsan Ashafi  
- **NIM:** 241011400424  
- **Program Studi:** Teknik Informatika S1  
- **Fakultas:** Ilmu Komputer  
- **Perguruan Tinggi:** Universitas Pamulang  
- **Periode Data:** 15 Oktober 2026 (Dataset Transaksi Aktual Server Pulsa)  

---

## Ringkasan Eksekutif

Laporan ini menyajikan analisis komprehensif segmentasi perilaku mitra (reseller) server pulsa menggunakan algoritma unsupervised learning **K-Means**. Analisis dilakukan berdasarkan dataset log transaksi aktual berformat Excel (`ikhsan.xlsx`). Setelah baris header dikenali secara tepat, dataset memuat **2,272 baris transaksi** yang berlangsung pada rentang waktu **15 Oktober 2026 pukul 14:56:29 hingga 23:28:07 WIB** (~8.53 jam operasional server). 

Seluruh transaksi pada file berstatus **"Sukses"** (100%), mencakup **359 kode produk unik**, dan melibatkan **451 kode reseller unik**. Transaksi diringkas per kode reseller menjadi 10 fitur perilaku utama yang merefleksikan volume transaksi, total omzet penjualan, biaya modal beli, akumulasi komisi downline, estimasi laba server, harga rata-rata dan median produk, keragaman produk, rata-rata jam aktivitas, serta proporsi transaksi malam (pukul 21.00 ke atas).

Fitur numerik dengan kemiringan distribusi (*skewness*) tinggi ditransformasikan menggunakan fungsi `log1p` lalu distandardisasi menggunakan `StandardScaler`. Pengujian K-Means dilakukan pada rentang $K = 2$ hingga $K = 6$ dengan `random_state=42` dan `n_init=20`. Evaluasi menggunakan **Elbow Method** dan **Silhouette Score** menetapkan **$K = 3$ sebagai baseline optimal** dengan skor kohesi-separasi tertinggi sebesar **0.272** (Inertia: 2934.78).

### Ringkasan Temuan Kunci
1. **Integritas Data:** Total 2,272 transaksi dengan 2,272 ID transaksi unik (0 duplikasi) dan 0 nilai kosong (*missing values*) pada 14 kolom fungsional.
2. **Karakteristik Keuangan:** Total harga beli dari supplier adalah Rp178,815,873; total harga jual ke reseller adalah Rp178,870,477; selisih margin jual-beli adalah Rp54,604; total komisi downline tercatat Rp54,502; dan total kolom laba server tercatat Rp102.
3. **Ketimpangan Aktivitas (Prinsip Pareto):** Distribusi transaksi sangat miring ke kanan (*right-skewed*). Reseller tunggal teratas (`DTD18293 - WHY RELOAD`) melakukan 580 transaksi (25.5% dari seluruh transaksi server). Pada hasil clustering, **Cluster 2 (157 reseller / 34.8%)** menguasai **89.1% total omzet penjualan server** (Rp 158.8 Juta).
4. **Reseller Prospek Downline:** Ditemukan 145 reseller yang aktif menghasilkan komisi. Mitra `DTD12886 - Pabuaran` meraih komisi downline tertinggi sebesar Rp 5,532 dengan 35 transaksi dan 12 variasi produk, menjadikannya kandidat Master Dealer pembina wilayah paling potensial.

---

## BAB I — Pendahuluan

### 1.1 Latar Belakang
Industri server pulsa dan payment point online bank (PPOB) di Indonesia beroperasi dengan volume transaksi harian yang sangat tinggi, margin keuntungan tipis per transaksi, serta jaringan keagenan bertingkat (*multi-tier downline*). Dalam arsitektur server pulsa (seperti platform OtomaX), setiap detik server memproses permintaan isi ulang pulsa reguler, paket data internet, token listrik PLN, hingga saldo dompet digital (DANA, OVO, GoPay).

Catatan transaksi yang tersimpan di basis data server sering kali hanya diperlakukan sebagai arsip laporan keuangan harian. Akibatnya, pemilik server kesulitan mengenali pola perilaku mitra secara objektif: siapa mitra yang bertransaksi dengan volume tinggi, siapa mitra eceran yang hanya bertransaksi di jam malam, dan siapa mitra yang berhasil membangun jaringan downline produktif.

Teknik Machine Learning, khususnya *unsupervised learning*, memungkinkan pengelompokan mitra secara otomatis berdasarkan kemiripan vektor perilaku transaksi tanpa membutuhkan label manual sebelumnya. Proyek ini mengimplementasikan algoritma K-Means untuk menghasilkan segmentasi mitra yang dapat diandalkan guna mendukung keputusan operasional bisnis server pulsa.

### 1.2 Identifikasi Masalah
1. Data transaksi mentah berukuran besar dan memiliki variasi aktivitas yang sangat lebar antar-mitra.
2. Ketimpangan volume transaksi sangat ekstrem, di mana sebagian besar mitra bertransaksi 1-2 kali sementara segelintir mitra bertransaksi ratusan kali.
3. Tidak adanya label segmentasi bawaan pada sistem server pulsa untuk membedakan mitra ritel kecil, mitra grosir (H2H), dan master agen.
4. Kolom komisi dan laba memiliki relasi bisnis khusus dengan selisih harga jual dan harga beli yang memerlukan verifikasi semantik teliti.
5. Keterbatasan rentang waktu data (satu hari) menuntut kehati-hatian metodologis agar kesimpulan tidak digeneralisasi berlebihan.

### 1.3 Rumusan Masalah
1. Bagaimana struktur, sebaran waktu, dan kualitas dataset transaksi server pulsa pada tanggal 15 Oktober 2026?
2. Bagaimana merancang *feature engineering* tingkat reseller yang relevan dan aman dari kebocoran informasi pribadi (*data leakage*)?
3. Berapa jumlah cluster ($K$) yang paling optimal berdasarkan metrik Elbow Method dan Silhouette Score?
4. Bagaimana profil karakteristik setiap cluster yang terbentuk dan bagaimana interpretasi bisnisnya?
5. Siapa saja mitra yang paling prospektif untuk dipromosikan sebagai Master Dealer atau pembina jaringan downline?

### 1.4 Tujuan
1. Melakukan audit kualitas data secara komprehensif terhadap dataset transaksi server pulsa `ikhsan.xlsx`.
2. Mengekstraksi 10 fitur perilaku transaksi agregat per reseller serta menerapkan transformasi `log1p` dan `StandardScaler`.
3. Menguji dan mengevaluasi K-Means pada rentang $K = 2$ hingga $K = 6$.
4. Memvisualisasikan cluster ke dalam ruang 2D menggunakan Principal Component Analysis (PCA).
5. Mengidentifikasi reseller unggulan yang berpotensi tinggi menghasilkan komisi downline dan omzet besar bagi server.

### 1.5 Manfaat
- **Bagi Akademik:** Sebagai implementasi praktis metodologi data science end-to-end yang memenuhi kaidah ilmiah pada tugas mata kuliah Machine Learning Teknik Informatika S1 Universitas Pamulang.
- **Bagi Bisnis Server Pulsa:** Memberikan wawasan empiris mengenai segmentasi reseller, alokasi prioritas modul supplier (H2H), serta perancangan insentif komisi agen downline.

---

## BAB II — Landasan Teori

### 2.1 Machine Learning
Machine Learning adalah cabang kecerdasan buatan yang memungkinkan sistem komputer mempelajari representasi pola dari data historis guna melakukan inferensi, klasifikasi, atau pengelompokan tanpa diprogram secara eksplisit aturan peraturannya (Mitchell, 1997).

### 2.2 Unsupervised Learning
Unsupervised learning digunakan saat dataset tidak memiliki label target ($y$). Algoritma bertugas menemukan struktur intrinsik, korelasi tersembunyi, atau pengelompokan (*clustering*) pada matriks fitur masukan ($X$). Clustering mempartisi $N$ objek ke dalam $K$ kelompok sedemikian rupa sehingga objek dalam kelompok yang sama memiliki tingkat kemiripan tinggi, sementara objek antar-kelompok memiliki perbedaan signifikan.

### 2.3 Algoritma K-Means
K-Means (MacQueen, 1967; Lloyd, 1982) adalah algoritma partisi berbasis jarak yang meminimalkan nilai *Within-Cluster Sum of Squares* (WCSS) atau inertia:

$$J = \sum_{j=1}^{K} \sum_{i \in C_j} \|x_i - \mu_j\|^2$$

di mana $\mu_j$ adalah centroid (rata-rata posisi) dari cluster $C_j$. Algoritma bekerja secara iteratif:
1. Inisialisasi $K$ centroid awal (menggunakan metode k-means++).
2. Menetapkan setiap observasi $x_i$ ke centroid terdekat berdasarkan jarak Euclidean.
3. Menghitung ulang posisi centroid $\mu_j$ sebagai rata-rata observasi yang terdaftar di dalamnya.
4. Mengulangi langkah 2 dan 3 hingga posisi centroid konvergen atau perubahan inertia berada di bawah batas toleransi.

### 2.4 Standardisasi dan Transformasi Fitur
K-Means sangat sensitif terhadap skala fitur karena perhitungan jarak Euclidean memberi bobot lebih besar pada fitur berskala besar (misalnya omzet jutaan rupiah dibanding jumlah transaksi bernilai satuan). 
- **Transformasi log1p:** $y = \ln(1 + x)$ diterapkan pada variabel miring positif untuk mereduksi pengaruh outlier ekstrem tanpa membuang baris data.
- **StandardScaler:** Melakukan transformasi z-score pada matriks fitur:

$$z = \frac{x - \mu}{\sigma}$$

sehingga seluruh fitur memiliki rata-rata $\mu = 0$ dan variansi $\sigma^2 = 1$.

### 2.5 Elbow Method dan Silhouette Score
1. **Elbow Method:** Memplot kurva inertia terhadap penambahan nilai $K$. Titik di mana penurunan inertia mulai melandai secara drastis (*elbow point*) menjadi kandidat jumlah cluster ideal.
2. **Silhouette Score:** Mengukur kualitas pengelompokan untuk setiap sampel $i$:

$$s(i) = \frac{b(i) - a(i)}{\max(a(i), b(i))}$$

di mana $a(i)$ adalah rata-rata jarak sampel $i$ ke seluruh titik dalam cluster yang sama, dan $b(i)$ adalah rata-rata jarak terdekat sampel $i$ ke titik dalam cluster tetangga terdekat. Rentang nilai dari -1 hingga +1, di mana nilai mendekati 1 menunjukkan cluster terpisah dengan sangat baik.

### 2.6 Mengapa Clustering Dipilih
Dataset transaksi operasional server pulsa yang ada tidak memiliki label status kualitas mitra. Selain itu, karena seluruh transaksi berstatus "Sukses", pendekatan supervised learning (seperti prediksi transaksi gagal) tidak memungkinkan untuk dilatih. Oleh sebab itu, clustering perilaku transaksi adalah pendekatan analitik yang paling tepat dan bermakna secara bisnis.

---

## BAB III — Metodologi

### 3.1 Sumber dan Cakupan Data
Dataset bersumber dari file buku kerja Excel `ikhsan.xlsx` lembar kerja `Sheet1`. Baris ke-0 merupakan baris kosong, baris ke-1 berisi nama kolom resmi sistem, dan baris ke-2 hingga 2,273 memuat catatan transaksi.

| Parameter Audit | Nilai Aktual |
|---|---|
| **Jumlah Baris Transaksi** | 2,272 baris |
| **Jumlah Kolom Mentah** | 14 kolom |
| **Rentang Waktu Transaksi** | 15 Oktober 2026 14:56:29 s.d. 23:28:07 WIB |
| **Durasi Operasional** | ~8 Jam 31 Menit 38 Detik |
| **ID Transaksi Unik (TrxID)** | 2,272 unik (0 duplikasi) |
| **Kode Reseller Unik** | 451 reseller |
| **Kode Produk Unik** | 359 produk |
| **Variasi Status** | 2,272 transaksi "Sukses" (100%) |

### 3.2 Kamus Data dan Perlakuan Fitur

| Kolom Mentah | Tipe Data | Makna Operasional | Perlakuan dalam Machine Learning |
|---|---|---|---|
| `TrxID` | int64 | Pengenal unik transaksi | Dikeluarkan (identitas baris) |
| `Tgl. Entri` | object/datetime | Waktu entri transaksi server | Diekstrak menjadi jam fraksional & indikator malam |
| `Produk` | object/str | Kode produk pulsa/data/PLN | Dihitung jumlah produk unik per reseller |
| `Tujuan` | int64 | Nomor tujuan pelanggan (MSISDN) | **Sensitif:** Dikeluarkan & disamarkan |
| `Kode Reseller` | object/str | ID akun reseller mitra | Kunci agregasi data (*grouping key*) |
| `Nama Reseller` | object/str | Nama konter/gerai pulsa | Identitas tampilan, tidak masuk ke fitur model |
| `Modul` | object/str | Jalur gateway supplier H2H | Dianalisis untuk distribusi operasional |
| `Status` | object/str | Status eksekusi transaksi | Tidak bervariasi (seluruhnya "Sukses") |
| `Hrg. Beli` | int64 | Harga modal supplier | Diagregasikan (total harga beli) |
| `Hrg. Jual` | int64 | Harga jual ke reseller | Diagregasikan (total, rata-rata, median) |
| `Komisi` | int64 | Komisi downline terpotong | Diagregasikan (total komisi) |
| `Laba` | int64 | Laba bersih tercatat server | Diagregasikan (total laba) |
| `Jawaban Provider` | object/str | Log respons API supplier | **Sensitif:** Dikeluarkan dari pemodelan |
| `SN` | object/str | Serial number / Ref token PLN | **Sensitif:** Dikeluarkan & disamarkan |

### 3.3 Pemeriksaan Kualitas dan Integritas Finansial
Pemeriksaan nilai keuangan menghasilkan ringkasan agregat berikut:

| Metrik Keuangan Server | Nilai Rupiah |
|---|---|
| **Total Harga Beli (Modal Supplier)** | Rp 178,815,873 |
| **Total Harga Jual (Omzet Penjualan)** | Rp 178,870,477 |
| **Selisih Margin Kotor (Jual - Beli)** | Rp 54,604 |
| **Total Komisi Downline Terbayar** | Rp 54,502 |
| **Total Laba Bersih Tercatat** | Rp 102 |
| **Konsistensi Persamaan Finansial** | Margin Kotor (Rp 54,604) = Komisi (Rp 54,502) + Laba (Rp 102) |

Temuan ini membuktikan bahwa pembukuan server pulsa konsisten: keuntungan kotor selisih harga jual dan harga beli dialokasikan sebesar **Rp 54,502 (99.8%) untuk komisi reseller/downline**, dan sisanya **Rp 102 (0.2%) menjadi laba bersih langsung server**.

### 3.4 Feature Engineering (10 Fitur Perilaku Reseller)
Data transaksi diagregasikan per `Kode Reseller` menghasilkan matriks berisi 451 baris dengan 10 fitur:
1. `jumlah_transaksi`: Frekuensi transaksi per reseller.
2. `total_harga_beli`: Total modal belanja reseller.
3. `total_harga_jual`: Total nilai perputaran omzet reseller.
4. `total_komisi`: Total komisi yang diperoleh dari jaringan agen.
5. `total_laba_tercatat`: Total laba bersih server dari transaksi reseller.
6. `rata_rata_harga_jual`: Rata-rata nominal per transaksi.
7. `median_harga_jual`: Nilai tengah harga transaksi reseller.
8. `jumlah_produk_unik`: Keragaman variasi produk yang dijual.
9. `jam_transaksi_rata_rata`: Rata-rata waktu bertransaksi.
10. `proporsi_transaksi_jam_21_ke_atas`: Rasio aktivitas transaksi pada jam malam (pukul 21.00 ke atas).

---

## BAB IV — Hasil dan Pembahasan

### 4.1 Hasil Pengujian Jumlah Cluster
Pengujian K-Means pada rentang $K = 2$ hingga $K = 6$ dengan standardisasi menghasilkan metrik evaluasi berikut:

| K | Inertia (WCSS) | Silhouette Score | Sebaran Anggota Cluster |
|:---:|:---:|:---:|:---|
| **2** | 3358.53 | 0.266 | C0: 184, C1: 267 |
| **3** | **2934.78** | **0.272** | **C0: 240, C1: 54, C2: 157** |
| **4** | 2571.18 | 0.229 | C0: 178, C1: 52, C2: 174, C3: 47 |
| **5** | 2296.99 | 0.219 | C0: 177, C1: 89, C2: 48, C3: 131, C4: 6 |
| **6** | 1995.47 | 0.226 | C0: 151, C1: 117, C2: 6, C3: 52, C4: 4, C5: 121 |

**Justifikasi Pemilihan $K = 3$:**
$K = 3$ menghasilkan **Silhouette Score tertinggi (0.272)** di antara seluruh konfigurasi yang diuji. Pada kurva Elbow Method, titik $K = 3$ merupakan belokan utama di mana penurunan inertia mulai stabil. Secara substansi bisnis, $K = 3$ membagi mitra secara jelas menjadi tiga kelompok perilaku yang operasional.

### 4.2 Profil Karakteristik Setiap Cluster ($K = 3$)

| Metrik Karakteristik | Cluster 0 | Cluster 1 | Cluster 2 |
|---|:---:|:---:|:---:|
| **Label Profil** | **Aktivitas Rendah (Dominan Sore)** | **Aktivitas Rendah (Dominan Malam)** | **Aktivitas Tinggi (Mitra Utama)** |
| **Jumlah Reseller** | 240 (53.2%) | 54 (12.0%) | 157 (34.8%) |
| **Total Transaksi** | 422 (18.6%) | 121 (5.3%) | 1,729 (76.1%) |
| **Rata-rata Trx / Reseller** | 1.76 transaksi | 2.24 transaksi | 11.01 transaksi |
| **Rata-rata Produk Unik** | 1.54 produk | 2.02 produk | 4.49 produk |
| **Porsi Transaksi $\ge$ 21.00** | 1.0% | 79.0% | 18.0% |
| **Total Omzet Penjualan** | Rp 12,492,789 (7.0%) | Rp 7,534,892 (4.2%) | Rp 158,842,796 (88.8%) |
| **Rata-rata Nilai Trx** | Rp 29,603 | Rp 62,271 | Rp 91,869 |

### 4.3 Analisis Ketimpangan dan Prinsip Pareto
Data menunjukkan berlakunya **Hukum Pareto (80/20)** secara nyata pada bisnis server pulsa:
- **Cluster 2** hanya terdiri dari **34.8% reseller (157 mitra)**, namun menguasai **76.1% total transaksi** dan menghasilkan **88.8% total omzet penjualan server** (Rp 158.8 Juta dari total Rp 178.8 Juta).
- Reseller dengan transaksi tertinggi adalah `DTD18293 - WHY RELOAD` dengan 580 transaksi (rata-rata 1 transaksi per 53 detik). Ini mengindikasikan koneksi API Host-to-Host (H2H) antar server.
- Sebaliknya, **Cluster 0** menaungi lebih dari separuh total reseller (53.2%), tetapi hanya menyumbang 7.0% omzet penjualan dengan frekuensi rata-rata 1.76 transaksi per mitra.

### 4.4 Analisis Rekomendasi Reseller Prospek Downline
Sesuai arahan praktis bisnis pulsa, reseller yang memiliki akumulasi komisi tinggi membuktikan keberadaan downline aktif di bawah jaringannya. Dari 451 reseller, terdapat **145 reseller (32.2%) yang aktif menghasilkan komisi**.

Berikut adalah **Top 10 Reseller Paling Prospektif untuk Dikembangkan Menjadi Master Dealer**:

| No | Kode Reseller | Nama Reseller | Cluster | Total Komisi | Jumlah Trx | Total Omzet | Produk Unik |
|:---:|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| 1 | **DTD12886** | **Pabuaran** | Cluster 2 | **Rp 5,532** | 35 | Rp 981,723 | 12 |
| 2 | **DTD18234** | **Al fattah cell** | Cluster 2 | **Rp 1,956** | 5 | Rp 149,968 | 5 |
| 3 | **DTD18357** | **Juriah** | Cluster 1 | **Rp 1,508** | 2 | Rp 41,070 | 1 |
| 4 | **CRM2912** | **1RB Yayah Cell** | Cluster 2 | **Rp 1,500** | 5 | Rp 188,914 | 3 |
| 5 | **DTD17618** | **Mimi Cell 002.Tanah tinggi** | Cluster 2 | **Rp 1,500** | 5 | Rp 109,218 | 5 |
| 6 | **DTD18296** | **Master Why** | Cluster 2 | **Rp 1,431** | 11 | Rp 235,883 | 7 |
| 7 | **DTD10010** | **LULU** | Cluster 2 | **Rp 1,400** | 9 | Rp 288,758 | 7 |
| 8 | **DTD18218** | **Blur cell** | Cluster 2 | **Rp 1,203** | 24 | Rp 1,200,225 | **17** |
| 9 | **DTD15947** | **fatih cell** | Cluster 2 | **Rp 1,200** | 3 | Rp 907,875 | 2 |
| 10 | **DTD10037** | **1RB OPIC NEW** | Cluster 2 | **Rp 1,200** | 6 | Rp 130,617 | 5 |

**Rekomendasi Manajerial Keagenan:**
1. **Mitra `DTD12886 - Pabuaran`:** Juara komisi server (Rp 5,532) dengan 12 variasi produk. Layak diberikan status *Master Dealer VIP* dan batas markup harga khusus agar jaringan downline-nya semakin berkembang pesat.
2. **Mitra `DTD18218 - Blur cell`:** Memiliki 17 variasi produk unik (tertinggi di server) dan omzet Rp 1.2 Juta dengan komisi Rp 1,203. Menunjukkan konter fisik ritel yang sangat lengkap dan ramai pembeli.
3. **Mitra Raksasa Non-Komisi (`DTD12773 - bumiayu22` & `DTD12861 - lion 2 cell`):** Memiliki omzet di atas Rp 10 Juta namun komisinya masih mendekati nol. Sangat prospektif diedukasi untuk merekrut agen downline baru di wilayahnya guna melipatgandakan omzet server.

### 4.5 Visualisasi Hasil Model
Semua grafik hasil visualisasi disimpan secara otomatis di direktori `outputs/figures/`:
1. `01_distribusi_transaksi_per_jam.png`: Grafik batang distribusi jam transaksi dengan puncak di pukul 21.00 (628 transaksi).
2. `02_distribusi_aktivitas_dan_omzet.png`: Histogram perbandingan distribusi jumlah transaksi sebelum dan sesudah transformasi `log1p`.
3. `03_elbow_method.png`: Kurva Elbow penurunan inertia dari K=2 hingga K=6.
4. `04_silhouette_score.png`: Grafik batang Silhouette Score yang membuktikan K=3 sebagai nilai tertinggi.
5. `05_cluster_scatter_pca_2d.png`: Scatter plot proyeksi PCA 2D yang memperlihatkan separasi antar-cluster dan posisi centroid masing-masing.
6. `06_jumlah_reseller_per_cluster.png`: Diagram batang perbandingan jumlah reseller vs kontribusi total omzet penjualan.
7. `07_perbandingan_fitur_utama_antar_cluster.png`: Perbandingan rata-rata frekuensi transaksi, produk unik, dan porsi jam malam.
8. `08_top_reseller_prospek_downline.png`: Bar horizontal ranking top reseller penghasil komisi downline terbesar.

### 4.6 Keterbatasan Hasil Analisis
1. **Jendela Waktu Sempit:** Data hanya mencakup satu tanggal (15 Oktober 2026 selama ~8.5 jam). Pola perilaku yang teramati merefleksikan snapshot jangka pendek dan belum memperhitungkan variasi transaksi akhir pekan (*weekend*), awal bulan (*payday*), atau hari libur nasional.
2. **Tidak Ada Transaksi Gagal:** Seluruh 2,272 transaksi berstatus sukses, sehingga model tidak dapat digunakan untuk memprediksi probabilitas kegagalan transaksi atau menganalisis modul provider bermasalah.
3. **Ukuran Laba Server Kecil:** Kolom laba tercatat Rp 102 karena sebagian besar keuntungan kotor dialirkan sebagai komisi downline keagenan (Rp 54,502).

---

## BAB V — Kesimpulan dan Saran

### 5.1 Kesimpulan
1. Dataset transaksi server pulsa `ikhsan.xlsx` memiliki kualitas teknis yang sangat baik tanpa nilai kosong dan tanpa duplikasi TrxID. Seluruh 2,272 transaksi berhasil dieksekusi dengan total perputaran omzet mencapai Rp 178,870,477.
2. Penerapan transformasi `log1p` berhasil menormalkan skala fitur yang sangat miring, dan standardisasi `StandardScaler` memungkinkan algoritma K-Means mempartisi data secara adil tanpa terdistorsi oleh perbedaan satuan.
3. Evaluasi K-Means menetapkan **$K = 3$ sebagai jumlah cluster baseline optimal** dengan nilai Silhouette Score tertinggi sebesar **0.272** dan penurunan inertia yang melandai pada Elbow Method.
4. Terbentuk tiga profil cluster yang jelas:
   - **Cluster 0 (53.2%):** Mitra ritel aktivitas rendah pada jam sore.
   - **Cluster 1 (12.0%):** Mitra spesialis aktivitas malam (79% transaksi di atas pukul 21.00).
   - **Cluster 2 (34.8%):** Mitra whale / kontributor utama yang menguasai 89.1% omzet penjualan server.
5. Analisis keagenan berhasil menemukan **145 mitra downline aktif**, dengan `DTD12886 - Pabuaran` dan `DTD18218 - Blur cell` sebagai reseller paling prospektif untuk dijadikan Master Dealer pembina jaringan.

### 5.2 Saran Pengembangan
1. **Pengumpulan Data Multi-Periode:** Disarankan mengekspor data transaksi selama 30 hari penuh (1 bulan) untuk mengamati pola retensi, kebiasaan belanja di tanggal gajian, dan stabilitas cluster.
2. **Integrasi Data Gangguan Supplier:** Menyertakan log transaksi gagal (*Gagal*, *Pending*, *Timeout*) agar dapat dibangun model prediksi keandalan rute modul gateway supplier (H2H).
3. **Penyempurnaan Kebijakan Bisnis:** Menggunakan hasil segmentasi ini untuk memberikan penawaran limit tempo/bon saldo hanya kepada Cluster 2 yang memiliki rekam jejak omzet terbukti dan stabil.

---

## Lampiran dan Catatan Reproduksibilitas

### Lampiran A — Kamus Fitur Model Clustering
- `jumlah_transaksi`: Frekuensi kemunculan TrxID per reseller.
- `total_harga_beli`: Penjumlahan kolom `Hrg. Beli` per reseller.
- `total_harga_jual`: Penjumlahan kolom `Hrg. Jual` per reseller.
- `total_komisi`: Penjumlahan kolom `Komisi` per reseller.
- `total_laba_tercatat`: Penjumlahan kolom `Laba` per reseller.
- `rata_rata_harga_jual`: Nilai rata-rata hitung `Hrg. Jual`.
- `median_harga_jual`: Nilai median (persentil 50) `Hrg. Jual`.
- `jumlah_produk_unik`: Penghitungan jumlah produk unik (`Produk.nunique()`).
- `jam_transaksi_rata_rata`: Nilai rata-rata jam desimal transaksi.
- `proporsi_transaksi_jam_21_ke_atas`: Rasio transaksi dengan jam $\ge 21.00$.

### Lampiran B — Catatan Reproduksibilitas dan Kode Sumber
Seluruh kode program dapat direproduksi secara konsisten dengan konfigurasi:
- **Bahasa Pemrograman:** Python 3.13
- **Pustaka Utama:** `pandas`, `numpy`, `scikit-learn`, `matplotlib`, `seaborn`, `openpyxl`, `streamlit`, `joblib`
- **Random Seed:** `random_state=42`, `n_init=20`
- **Skrip Eksekusi:** Cukup jalankan perintah `python main.py` untuk menghasilkan seluruh tabel dan grafik, serta `streamlit run app.py` untuk membuka dashboard analitik interaktif.

### Referensi
1. Lloyd, S. (1982). Least squares quantization in PCM. *IEEE Transactions on Information Theory*, 28(2), 129–137.
2. MacQueen, J. (1967). Some methods for classification and analysis of multivariate observations. *Proceedings of the Fifth Berkeley Symposium on Mathematical Statistics and Probability*.
3. Pedregosa, F., et al. (2011). Scikit-learn: Machine Learning in Python. *Journal of Machine Learning Research*, 12, 2825–2830.
4. Rousseeuw, P. J. (1987). Silhouettes: a graphical aid to the interpretation and validation of cluster analysis. *Journal of Computational and Applied Mathematics*, 20, 53–65.
