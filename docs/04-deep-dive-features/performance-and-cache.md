# Deep Dive: Performa, Lifecycle Kamera, & Ketahanan Jaringan

Fase 5 difokuskan pada pengujian di perangkat fisik nyata dengan koneksi internet terbatas dan spesifikasi perangkat keras menengah ke bawah.

---

## 1. Persistent Model File Caching (`modelCache.js`)

Sebelum commit `3c5b89f9` dan `1d407b2e`, setiap kali pengguna membuka halaman presensi, peramban mengunduh ulang berkas model AI sebesar ~13 MB dari server:
- SSD MobileNetV1: ~5.4 MB
- FaceRecognitionNet: ~6.2 MB
- MiniFASNetV2-SE: ~600 KB
- Runtime WASM: belasan megabyte

### Solusi Cache Storage:
Modul `modelCache.js` memanfaatkan antarmuka standar browser **Cache Storage API** dengan nama cache `face-models-v1`:
1. Saat memuat model, sistem memeriksa ketersediaan berkas di Cache Storage lokal.
2. Jika ditemukan (*cache hit*), model dibaca langsung dari penyimpanan lokal tanpa lalu lintas jaringan.
3. Jika belum tersedia (*cache miss*), berkas diunduh dari server dan langsung disimpan ke Cache Storage untuk sesi-sesi berikutnya.
4. Terdapat mekanisme fallback otomatis ke `fetch` reguler jika peramban tidak mendukung Cache Storage.

Hasil: Proses *cold start* pengunduhan belasan megabyte hanya terjadi satu kali per perangkat.

---

## 2. Pemuatan Model Paralel & Warm-Up (`faceModels.js`)

- **Pemuatan Konkuren:** Ketiga model face-api dan sesi ONNX dimuat secara bersamaan menggunakan `Promise.all()`, bukan berurutan satu per satu.
- **Model Warm-Up:** Begitu model selesai dimuat ke memori WebAssembly, sistem langsung mengeksekusi inferensi dummy pada canvas kosong. Hal ini mengkompilasi graf komputasi WebAssembly sebelum kamera aktif, sehingga frame pertama kamera tidak mengalami *stutter* atau *freeze*.
- **Preload Latar Belakang:** Diimplementasikan pada `PlainLayout.vue` untuk mulai memuat model 3 detik setelah layout aplikasi aktif, selagi pengguna masih bernavigasi di menu utama.

---

## 3. Pose Sampling 3-Frame (`poseSampler.js`)

Mengambil deskriptor wajah dari satu frame tunggal saat pengguna sedang bergerak (misal memutar kepala) sering kali menghasilkan citra kabur (*motion blur*), yang menyebabkan jarak Euclidean membengkak dan verifikasi gagal.

### Mekanisme Sampling:
- Begitu pose tantangan terdeteksi valid, `poseSampler.js` mengumpulkan 3 frame berturut-turut pada pose yang sama dengan interval maksimum 700 ms antar-frame.
- Deskriptor biometrik final adalah rata-rata vektor dari 3 frame tersebut:
  $$\vec{v}_{final} = \frac{\vec{v}_1 + \vec{v}_2 + \vec{v}_3}{3}$$
- Jika jeda antar-frame melebihi 700 ms (misal wajah bergerak keluar frame), sampel dibatalkan dan diulang otomatis.

---

## 4. Lifecycle Kamera & Proteksi Konkurensi

- **Penghentian Kamera Saat Unmount (Commit `ac588bd8`):**
  Pada hook `beforeDestroy()` kedua halaman Vue, seluruh stream media kamera dihentikan eksplisit:
  ```javascript
  if (this.stream) {
    this.stream.getTracks().forEach(track => track.stop())
  }
  ```
  Ini mencegah kamera terus menyala di latar belakang saat pengguna berpindah halaman (menghemat baterai dan menjaga privasi).
- **Flag Proteksi `isStarting` (Commit `4fcf749b`):**
  Mencegah pemanggilan ganda pada fungsi `startDetection()` jika tombol ditekan berulang kali secara cepat, yang sebelumnya dapat membuka dua stream kamera bersamaan.

---

## 5. Toleransi Kegagalan Jaringan saat Pendaftaran (`postEnrollWithRetry`)

Pengiriman data pendaftaran wajah (`POST /mobile/insertFaceRecognition`) kini dibungkus dengan helper retry:
- Maksimal 3 kali percobaan dengan jeda bertingkat (2 detik, lalu 4 detik).
- **Hanya Diulang pada Masalah Jaringan:** Request hanya diulang jika terjadi *timeout*, *network error*, atau respons HTTP 5xx. Jika penolakan berasal dari logika bisnis (skor anti-spoof rendah atau wajah tidak cocok), permintaan tidak diulang.
- Data foto dan deskriptor yang sudah direkam tetap tersimpan di memori Vue, sehingga jika koneksi gagal total pengguna cukup menekan tombol coba kirim kembali tanpa perlu mengulang pengambilan pose wajah dari awal.
