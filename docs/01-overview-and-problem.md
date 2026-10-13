# 01. Gambaran Sistem & Masalah yang Diselesaikan

## 1.1 Konteks dan Latar Belakang

Sistem absensi berbasis pengenalan wajah (*face recognition*) diinisiasi pada aplikasi PWA SWB untuk menggantikan sistem presensi kehadiran berbasis kode QR. Mekanisme QR Code sebelumnya memiliki celah keamanan signifikan dalam lingkungan kerja operasional logistik, di antaranya:
- **Praktik Titip Absen (*Buddy Punching*):** Karyawan dapat membagikan foto atau tangkapan layar kode QR kepada rekan kerja untuk memindai atas namanya.
- **Manipulasi Lokasi (Fake GPS):** Penggunaan aplikasi mocking lokasi tanpa validasi fisik keberadaan orang secara visual.

Untuk mengatasi permasalahan tersebut, dirancang sistem presensi nirsentuh yang menggabungkan:
1. Verifikasi biometrik pengenalan wajah pengguna.
2. Uji keaslian fisik secara langsung (*liveness challenge*).
3. Deteksi anti-spoofing berbasis *deep learning* untuk menangkal foto cetak, layar ponsel, dan video rekaman.
4. Validasi koordinat GPS dan pembuktian kriptografis berbasis token di sisi server.

---

## 1.2 Masalah Teknis yang Dihadapi

### 1. Masalah Performa dan Ukuran Bundel AI di Browser Mobile
Pada fase eksplorasi awal, pustaka `@vladmandic/human` diuji sebagai mesin AI di sisi peramban (*client-side*). Namun ditemukan sejumlah kendala fundamental:
- **Ukuran Bundel Berlebihan:** Pustaka Human mengemas seluruh pipeline deteksi (BlazeFace, FaceMesh, FaceEmbedding, Iris, Liveness) dalam satu bundel besar berukuran puluhan ribu baris JavaScript dan belasan megabyte berkas model.
- **Cold Start Lambat:** Pada peramban ponsel, proses unduh dan inisialisasi awal membutuhkan waktu lebih dari 10 detik.
- **Beban Komputasi TensorFlow.js:** Inferensi per frame di atas TensorFlow.js pada perangkat Android *entry-level* (RAM 2 GB, CPU Cortex-A55) menyebabkan latensi tinggi (200-400 ms per frame) sehingga pengalaman interaksi menjadi patah-patah.

### 2. Celah Keamanan Client-Side Matching
Pada implementasi awal, pencocokan vektor wajah dihitung di peramban JavaScript. Hal ini menimbulkan dua risiko serius:
- **Kebocoran Data Biometrik:** Vektor template pengguna (`storedDescriptors`) dikirim dari server ke peramban, mengekspos data biometrik di memori JavaScript dan lalu lintas jaringan.
- **Manipulasi Hasil Verifikasi:** Keputusan kelulusan presensi ditentukan di peramban, memungkinkan manipulasi script sebelum data presensi dikirim ke server.

### 3. Masalah Replay Attack dan Pemanggilan Endpoint Tanpa Validasi
Tanpa mekanisme pengikatan bukti verifikasi, endpoint pencatatan absensi dapat ditembak langsung melalui HTTP request tanpa melewati proses pemindaian wajah.

---

## 1.3 Solusi Teknis yang Diterapkan

1. **Pivot Arsitektur Modular:** Mengganti `@vladmandic/human` dengan kombinasi `@vladmandic/face-api` (khusus deteksi wajah & landmark di peramban) dan model deteksi **YuNet** (~230 KB) di microservice sidecar.
2. **Dual Anti-Spoofing:** Model pasif **MiniFASNetV2-SE quantized** (~600 KB) dieksekusi di browser melalui WebAssembly (ONNX Runtime Web) untuk feedback langsung, lalu divalidasi ulang di sidecar container sebagai keputusan final (ambang batas 0.50).
3. **Server-Side Same-Pose Matching:** Seluruh proses pencocokan biometrik dipindahkan ke backend controller. Template biometrik tidak pernah dikirim ke peramban (*zero-leak*), dan perbandingan dilakukan spesifik pose-ke-pose.
4. **Tokenisasi Redis `proofNo`:** Penerbitan token sekali pakai dengan masa aktif 300 detik yang langsung dikonsumsi (dihapus) saat absensi dicatat.
5. **Persistent Cache Storage & Pose Sampling:** Berkas model disimpan di Cache Storage browser untuk memangkas *cold start*, dan deskriptor pose dihitung dari rata-rata 3 frame untuk kestabilan vektor.
