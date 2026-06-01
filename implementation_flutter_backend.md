# Integrasi Frontend Flutter dengan Backend Laravel (MOTHRA)

Plan ini merangkum langkah-langkah untuk mengganti *mock data* (data statis) di Flutter menjadi data dinamis yang diambil dari API Laravel yang telah kita buat.

## User Review Required
> [!IMPORTANT]
> - **API Base URL**: Secara default, emulator Android menggunakan `http://10.0.2.2:8000/api`. Jika Anda menjalankan Flutter sebagai aplikasi Windows (Desktop), kita akan menggunakan `http://127.0.0.1:8000/api`. Saya akan mengatur konfigurasinya agar bisa disesuaikan dengan mudah.
> - **State Management**: Karena aplikasi saat ini menggunakan `InheritedWidget` (AppState) yang sederhana, saya akan tetap mempertahankan arsitektur ini dan menambahkan `ApiService` berskala global yang mengatur token (menyimpannya dengan `SharedPreferences`).

## Proposed Changes

### 1. Dependencies & Konfigurasi Dasar
Akan menambahkan package HTTP untuk komunikasi dengan API.

#### [MODIFY] pubspec.yaml
- Menambahkan package `http: ^1.2.0` ke dalam dependencies.

### 2. API Service Layer
Membuat layer service untuk menangani semua HTTP request ke backend Laravel.

#### [NEW] lib/app/services/api_service.dart
- Kelas `ApiService` dengan metode:
  - Manajemen Token (login, simpan token ke SharedPreferences).
  - Auth: `login`, `logout`.
  - Data Kupu-kupu: `getButterflies`, `createButterfly`, `updateButterfly`, `deleteButterfly`.
  - Scanning: `scanImage` (mengirim multipart file), `saveScan`.
  - Riwayat & Koleksi: `getHistory`, `getCollection`.
  - Statistik: `getDashboardStats`, `getAdminStats`.

### 3. Pembaruan Models
Menyesuaikan agar model bisa parsing data dengan tepat dari JSON response Laravel.

#### [MODIFY] lib/app/models/butterfly_model.dart
- Menghapus list `sampleButterflies` statis.
- Memastikan field null-safety sesuai dengan output backend Laravel.

#### [MODIFY] lib/app/models/scan_result_model.dart
- Menghapus list `sampleScanHistory` statis.
- Menambahkan factory `fromJson` yang mendukung *nested* relasi butterfly dari Laravel resource.

### 4. Pembaruan UI Screens (Menggunakan FutureBuilder)
Setiap halaman akan diperbarui untuk mengambil data dari `ApiService` dan menampilkannya dengan `FutureBuilder` beserta state loading (`CircularProgressIndicator`).

#### [MODIFY] lib/app/screens/auth/login_screen.dart
- Menghubungkan proses login dengan `ApiService.login()`. Menampilkan alert jika gagal. Menyimpan user state ke `AppState`.

#### [MODIFY] lib/app/screens/dashboard/dashboard_screen.dart
- Mengambil dan menampilkan data statistik asli (jumlah scan, jumlah spesies yang dikoleksi, dll) dari endpoint `/api/dashboard/stats` atau `/api/dashboard/admin-stats`.

#### [MODIFY] lib/app/screens/scan/scan_screen.dart & result_screen.dart
- Mengganti fungsi *mock delay 3 detik* menjadi request multipart file sungguhan ke endpoint `/api/scan`.
- Menggunakan endpoint `/api/scan/{id}/save` untuk menyimpan spesies ke koleksi jika tombol "Simpan ke Koleksi" ditekan.

#### [MODIFY] lib/app/screens/history/history_screen.dart
- Menggunakan `FutureBuilder` untuk me-render daftar riwayat scan yang diambil dari `/api/history`.

#### [MODIFY] lib/app/screens/collection/collection_screen.dart
- Menggunakan endpoint `/api/butterflies` (karena API ini menyertakan nilai boolean `is_collected` secara spesifik per-user). 

#### [MODIFY] lib/app/screens/admin/admin_panel_screen.dart & species_form_screen.dart
- Menghubungkan fungsionalitas Tambah, Edit, dan Hapus (CRUD) spesies menggunakan fungsi API khusus Admin di `ApiService`.

## Verification Plan

### Automated Tests
- Menjalankan `flutter pub get` untuk memastikan dependencies terinstall dengan baik.

### Manual Verification
- Menjalankan aplikasi di emulator atau desktop.
- Menguji alur Login dengan akun user dan admin.
- Membuka halaman Beranda dan melihat metrik dari database.
- Melakukan percobaan "Scan Gambar", memastikan gambar terkirim ke Laravel dan mengembalikan response mock CNN dari backend.
- Mengecek status *Unlock* di halaman Koleksi jika spesies berhasil disimpan.
