# 🦋 Ringkasan Percakapan — Proyek Identifikasi Kupu-kupu Beracun
**Mata Kuliah:** Pengolahan Citra Digital | **Kelompok 9**  
**Tanggal:** Maret 2026

---

## 1. Penjelasan Aposematism & Deteksi Warna

**Konsep:** *Aposematism* adalah fenomena evolusi di mana kupu-kupu **berbisa** mengembangkan warna mencolok (merah, oranye, kuning) sebagai sinyal peringatan kepada predator.

**Pendekatan Proyek:** Menggunakan ruang warna **HSV** (Hue, Saturation, Value) karena lebih stabil terhadap kondisi pencahayaan dibandingkan RGB.

**Fitur Warna yang Diekstrak:**
- `warning_color_ratio` — porsi piksel merah + oranye + kuning pada sayap
- Histogram HSV (Hue/Saturation/Value)
- Mean dan Std setiap channel warna

---

## 2. Grafik Bar (Warning Color Ratio)

Grafik batang horizontal membuktikan spesies beracun (merah) memiliki rasio warna peringatan yang lebih tinggi. Ini adalah visualisasi bukti *Aposematism*.

**Kasus Anomali — Atlas Moth:**  
Memiliki `warning_color_ratio` tinggi **tapi tidak beracun** — contoh ***Batesian Mimicry***: spesies aman yang berevolusi meniru tampilan spesies berbahaya.

---

## 3. Rencana Aplikasi Mobile

**Fitur yang direncanakan:**
- Upload gambar → Deteksi beracun/tidak
- Indeks koleksi: identifikasi spesies + informasi lengkap

**Teknologi yang direkomendasikan:**

| Komponen | Teknologi |
|---|---|
| Model ML | CNN (MobileNetV2 / EfficientNet) → TFLite |
| Database | Firebase Firestore |
| Storage | Firebase Storage |
| Mobile | Flutter / React Native |

**Tujuan aplikasi:**
> Membantu masyarakat umum dan peneliti lapangan mengidentifikasi kupu-kupu berpotensi berbahaya secara cepat di alam liar, serta membangun koleksi digital pribadi.

**Kupu-kupu paling berdampak ke manusia:**
- `IO MOTH` dan `SIXSPOT BURNET MOTH` — duri/sengat beracun
- `OLEANDER HAWK MOTH` — neurotoksin
- `PIPEVINE SWALLOW` — asam aristolochic
- `MONARCH` — glikosida jantung

---

## 4. Parameter Tambahan: 149 Fitur Lengkap

### Morfologi Bentuk Sayap (6 Fitur)

| Fitur | Penjelasan |
|-------|------------|
| `wing_area_ratio` | Persentase area sayap vs gambar |
| `aspect_ratio` | Lebar vs tinggi bounding box |
| `circularity` | Seberapa "bulat" kontur sayap (1.0 = lingkaran sempurna) |
| `convexity` | Seberapa cembung bentuk sayap |
| `extent` | Rasio area sayap vs bounding box |
| `spot_count` | Jumlah bintik/pola terdeteksi |

### Tekstur Sayap (36 Fitur)

| Fitur | Teknik | Arti |
|-------|--------|------|
| `tex_edge_mean` | Sobel Gradient | Rata-rata ketajaman tepi |
| `tex_edge_std` | Sobel Gradient | Variasi kekasaran tepi |
| `tex_edge_max` | Sobel Gradient | Tepi paling tajam |
| `tex_laplacian_var` | Laplacian | Kerumitan tekstur keseluruhan |
| `tex_grad_b0..b31` | Histogram (32 bin) | Distribusi kekasaran sayap |

**Total: 107 Warna + 6 Morfologi + 36 Tekstur = 149 fitur per gambar.**

---

## 5. Klasifikasi Gabungan Rule-Based (Simulasi Manual)

```
Skor_Gabungan = (Warna x 0.50) + (Tekstur x 0.30) + (Morfologi x 0.20)
```

Jika `Skor_Gabungan > 30%` maka DIPREDIKSI BERACUN.

> Kelemahan: batas 30% adalah tebakan manusia, mudah menghasilkan False Positive. Inilah alasan kita beralih ke Machine Learning sesungguhnya.

---

## 6. Model Machine Learning

### Pipeline

```
Gambar → Ekstraksi 149 Fitur → StandardScaler → Random Forest / SVM → Prediksi
```

### Ground Truth / Kunci Jawaban

Dikodekan manual di `TOXIC_SPECIES` (Sel 3) berdasarkan literatur Biologi:

```python
label_map = {sp: (1 if sp.upper() in TOXIC_SPECIES else 0) for sp in species_list}
```

- 20 Spesies → Label 1 (Beracun)
- 80 Spesies → Label 0 (Tidak Beracun)

---

## 7. Perbandingan: Random Forest vs SVM

| Kriteria | Random Forest | SVM |
|----------|:---:|:---:|
| Akurasi Keseluruhan | **83.4%** | 79.2% |
| Recall Beracun (temukan bahaya) | 65% | **77%** |
| Precision Beracun (tidak asal tuduh) | **58%** | 49% |
| Kecepatan Training | Cepat | Lambat |
| Rekomendasi | Dipilih | — |

**Pemenang: Random Forest** — lebih seimbang dan tidak mudah tertipu Batesian Mimicry.

---

## 8. Membaca Confusion Matrix (Heatmap Akurasi)

```
                    Tebakan ML
                 Aman      Beracun
Kenyataan Aman  [ 400 ]   [   0  ]
          Beracun[  83 ]   [  17  ]
```

| Kotak | Kenyataan | Tebakan | Status |
|-------|-----------|---------|--------|
| 400 (kiri atas) | Aman | Aman | True Negative — Tepat |
| 0 (kanan atas) | Aman | Beracun | False Positive = nol! |
| 17 (kanan bawah) | Beracun | Beracun | True Positive — Tepat |
| **83 (kiri bawah)** | **Beracun** | **Aman** | **False Negative — Berbahaya!** |

> Angka 17 dan 83 adalah jumlah lembar **foto**, bukan jumlah spesies. Dari total 100 foto kupu-kupu beracun di Data Uji, 83 foto gagal terdeteksi.

---

## 9. Saran Meningkatkan Akurasi Melewati 90%

### Teknik 1 — Turunkan Threshold (Termudah)
Buat model lebih "paranoid" dengan menurunkan ambang batas dari 50% ke 30%:
```python
y_pred = (rf_model.predict_proba(X_test)[:, 1] >= 0.30).astype(int)
```

### Teknik 2 — SMOTE (Penyeimbang Data)
Hasilkan data sintetis kupu-kupu beracun agar proporsi latihan menjadi 50:50:
```python
from imblearn.over_sampling import SMOTE
X_res, y_res = SMOTE().fit_resample(X_train, y_train)
```

### Teknik 3 — Feature Selection
Pilih hanya top-30 fitur terpenting menggunakan `rf.feature_importances_`.

### Teknik 4 — Deep Learning CNN (Solusi Ultimate)
Ganti pipeline CSV + Random Forest dengan CNN (MobileNetV2 / EfficientNet). Akurasi bisa mencapai **95%+** tanpa rekayasa fitur manual.

---

## 10. Arsitektur File Proyek

```
Kelompok 9/
├── main.ipynb                  <- Notebook utama (Run All)
├── Kelompok9_PCD/              <- Dataset gambar (100 spesies)
│   ├── train/
│   ├── valid/
│   └── test/
└── output_csv/                 <- Hasil ekstraksi 149 fitur
    ├── train_features.csv
    ├── train_scaled.csv        <- Digunakan ML (sudah StandardScaler)
    ├── valid_scaled.csv
    └── test_scaled.csv
```
