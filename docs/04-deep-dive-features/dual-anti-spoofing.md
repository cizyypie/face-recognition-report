# Deep Dive: Dual Anti-Spoofing Pipeline

## 1. Konsep Desain

Serangan spoofing biometrik (*presentation attack*) umumnya berupa:
- **Foto Cetak (Print Attack):** Foto wajah pada kertas foto resolusi tinggi.
- **Layar Digital (Replay Attack):** Menampilkan foto atau video wajah dari layar smartphone atau tablet lain.
- **Topeng 3D / Cutout:** Topeng dengan lubang mata/mulut.

Untuk menepis serangan tersebut tanpa membebani server secara berlebihan, sistem menerapkan **Dual-Layer Anti-Spoofing Architecture**:
1. **Lapisan 1 (Client Edge - Browser):** Memberikan respon instan kepada pengguna per-frame via ONNX Runtime Web WASM.
2. **Lapisan 2 (Server Authoritative - Sidecar Container):** Mengevaluasi citra resolusi penuh sebelum token presensi dicetak.

---

## 2. Lapisan 1: Browser WASM (`antiSpoof.js`)

- **Model:** `MiniFASNetV2-SE quantized` (~600 KB ONNX).
- **Runtime:** `onnxruntime-web` menggunakan WebAssembly execution provider.
- **Frekuensi Eksekusi:** Setiap 450 ms selama deteksi wajah aktif di kamera.

### Tahapan Pemrosesan Per Frame:
1. **Crop 1.5x Konteks:** Kotak pembatas wajah (*bounding box*) dari face-api diperluas sebesar 1.5x untuk mencakup area dahi, telinga, leher, dan latar belakang sekitar (fitur mikro pantulan cahaya sangat bergantung pada konteks luar wajah).
2. **Render Offscreen Canvas:** Citra crop dirender pada canvas tak terlihat berukuran 128x128 piksel dengan `imageSmoothingQuality = 'high'`.
3. **Konversi Tensor NCHW Float32:** Data citra dinormalisasi ke rentang `[0, 1]` dan diatur dalam format kanal `[1, 3, 128, 128]`.
4. **Softmax Output:** Model menghasilkan logit `[real_logit, spoof_logit]`, di mana probabilitas keaslian dihitung dengan formula softmax:
   $$probReal = \frac{e^{logit_{real}}}{e^{logit_{real}} + e^{logit_{spoof}}}$$
5. **Windowed Smoothing:** Skor akhir dihitung dari rata-rata bergerak 4 frame terakhir untuk mengeliminasi kedipan (*flicker*) status UI.

---

## 3. Lapisan 2: Sidecar Container (`face-service:3100`)

Mengevaluasi foto tunggal resolusi penuh yang dikirim saat pengguna menuntaskan tantangan liveness.

### Pipeline Pemrosesan di `server.js`:
1. **Deteksi YuNet:** Menentukan lokasi wajah terbesar dan titik landmark dalam koordinat gambar asli.
2. **Reflect/Mirror Padding:** Jika perluasan kotak crop 1.5x keluar dari batas citra, piksel tepi tidak dipotong melainkan dicerminkan (*reflect padding*) agar tidak menimbulkan artefak batas hitam.
3. **Downscale Setara INTER_AREA:** Gambar di-downscale menggunakan kombinasi resize bilinear ke kelipatan 128 diikuti oleh `avgPool(k, k)` valid. Metode ini mempertahankan integritas tekstur frekuensi tinggi yang krusial bagi MiniFASNet.
4. **Eksekusi Tf.tidy():** Seluruh operasi tensor dibungkus dalam `tf.tidy()` untuk mencegah kebocoran memori pada lingkungan Node.js.

---

## 4. Evaluasi & Ketahanan (Circuit Breaker)

- **Ambang Batas Keputusan:** Ditetapkan `0.50` secara konsisten antara client dan server (sejak commit `4962d6cb`). Jika skor server `< 0.50`, transaksi absensi langsung ditolak.
- **Circuit Breaker Redis:**
  Jika sidecar gagal merespons atau mengembalikan galat sebanyak 3 kali berturut-turut dalam durasi 60 detik, controller backend secara otomatis mengaktifkan mode *circuit breaker* (`attendance:facesvc:down` TTL 60 detik) untuk mencegah penumpukan antrean koneksi.

