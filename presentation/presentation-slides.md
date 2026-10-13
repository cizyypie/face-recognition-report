---
marp: true
theme: default
paginate: true
header: "SWB - Face Recognition Absensi"
footer: "Laporan Kemajuan Magang - Branch `face-attendance-human`"
style: |
  section {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    font-size: 20px;
    padding: 40px;
  }
  h1 {
    color: #1e3a8a;
  }
  h2 {
    color: #2563eb;
    border-bottom: 2px solid #e2e8f0;
    padding-bottom: 8px;
  }
  .placeholder-box {
    border: 2px dashed #94a3b8;
    background-color: #f8fafc;
    border-radius: 8px;
    padding: 30px 20px;
    text-align: center;
    color: #475569;
    font-weight: 600;
    margin: 15px 0;
  }
  .highlight-yellow {
    background-color: #fef08a;
    padding: 2px 6px;
    border-radius: 4px;
  }
  .highlight-blue {
    background-color: #bae6fd;
    padding: 2px 6px;
    border-radius: 4px;
  }
  .grid-2 {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
    align-items: center;
  }
---

<!-- Slide 1: Title -->
# Face Recognition Absensi
### Laporan Penambahan & Refaktorisasi Fitur
**Branch:** `face-attendance-human` | **Aplikasi:** Wahana SWB App

---

<!-- Slide 2: Latar Belakang & Masalah -->
## Latar Belakang & Masalah

* **Tujuan Utama:** Presensi nirsentuh berbasis PWA Mobile untuk mencegah <span class="highlight-yellow">titip absen (buddy punching)</span>.
* **Tantangan Awal:**
  * Dependency library `@vladmandic/human` terlalu berat (puluhan ribu baris).
  * Cold-start lambat dan frame rate rendah pada smartphone spesifikasi standar.
  * Risiko manipulasi spoofing (foto cetak, layar ponsel) & manipulasi API.

---

<!-- Slide 3: Pivot Arsitektur -->
## Pivot Arsitektur: Lebih Ringan & Responsif

<div class="grid-2">
<div>

* **Keputusan:** Melepas `@vladmandic/human` sepenuhnya.
* **Client-Side:** Menggunakan `@vladmandic/face-api` yang fokus pada deteksi esensial.
* **Sidecar Microservice:** Menggunakan model **YuNet (ONNX)** via container Node.js.
* **Hasil:** Ukuran bundle berkurang signifikan, load time instan, dan inferensi stabil.

</div>
<div>

<div class="placeholder-box">
[AREA GAMBAR: Perbandingan bundle size atau grafik penurunan waktu load time sebelum vs sesudah pivot]
</div>

</div>
</div>

---

<!-- Slide 4: Arsitektur 3-Tier Terkini -->
## Arsitektur Sistem Terkini

* **Tier 1 - Client (PWA):** Vue 2, Quasar, `@vladmandic/face-api`, `onnxruntime-web` (MiniFASNetV2-SE).
* **Tier 2 - Backend:** Perl DCAF (`Mobile.pm`), validasi JWT & proof token, Redis, MySQL.
* **Tier 3 - AI Sidecar:** Node.js Express (`face-service:3100`), YuNet Face Detector, MiniFASNetV2-SE Anti-Spoof.

<div class="placeholder-box">
[AREA GAMBAR: Diagram Arsitektur 3-Tier (PWA Browser -> Perl DCAF -> AI Sidecar & Redis/MySQL)]
</div>

---

<!-- Slide 5: Dual Anti-Spoofing Pipeline -->
## Dual Anti-Spoofing Pipeline

Perlindungan ganda terhadap serangan foto cetak dan layar digital:

1. **Client-Side Anti-Spoof:**
   * Inferensi lokal via `onnxruntime-web` WASM.
   * Visual feedback lingkaran: <span class="highlight-yellow">Merah (Spoof)</span> / <span class="highlight-blue">Hijau (Real)</span>.
2. **Server-Side Verification:**
   * Verifikasi ulang di sidecar container `face-service`.
   * Evaluasi probabilitas keaslian fisik wajah sebelum verifikasi identitas.

---

<!-- Slide 6: Screenshot UI Deteksi & Anti-Spoofing -->
## Tampilan UI Presensi & Anti-Spoofing

<div class="grid-2">
<div>

<div class="placeholder-box">
[AREA GAMBAR: Screenshot UI PWA saat deteksi wajah real (lingkaran panduan hijau)]
</div>

</div>
<div>

<div class="placeholder-box">
[AREA GAMBAR: Screenshot UI PWA saat deteksi spoofing / foto layar (lingkaran panduan merah)]
</div>

</div>
</div>

---

<!-- Slide 7: Peningkatan Liveness & Pose Tantangan -->
## Peningkatan Liveness: Pose Tantangan

Tantangan gerakan acak menggunakan 68-Point Facial Landmark:

* **Toleh Kiri / Kanan (Head Yaw):** Rasio simetri hidung terhadap mata dan kontur rahang.
* **Tengok Atas / Bawah (Head Tilt/Pitch):** Rasio vertikal posisi hidung terhadap jarak mata-ke-dagu.
* **Senyum (Smile):** Rasio pelebaran sudut bibir terhadap jarak pupil mata.
* **Multi-Face Rejection:** Otomatis membatalkan frame jika terdeteksi lebih dari satu wajah.

---

<!-- Slide 8: Server-Side Same-Pose Matching -->
## Server-Side Same-Pose Matching

<div class="grid-2">
<div>

* **Zero-Leak Biometrics:** Data template biometrik **tidak pernah** dikirim ke browser.
* **Validasi Satu-ke-Satu (1:1):**
  * Pose tantangan live dicocokkan spesifik dengan template pose pendaftaran yang bersesuaian.
  * Contoh: Live `turn_left` dicocokkan langsung ke data enroll `photoLeft`.
* **Metrik:** Euclidean Distance dengan threshold ketat di backend Perl.

</div>
<div>

<div class="placeholder-box">
[AREA GAMBAR: Ilustrasi pencocokan pose Live vs Enroll (Toleh Kiri Live vs Toleh Kiri Template)]
</div>

</div>
</div>

---

<!-- Slide 9: Screenshot Registrasi Wajah -->
## Registrasi Biometrik (Face Enrollment)

Pengambilan template wajah multi-pose saat pertama kali didaftarkan:

<div class="placeholder-box">
[AREA GAMBAR: Screenshot halaman pendaftaran wajah multi-pose (PTrIAbsenFaceRecognition.vue)]
</div>

* Menyimpan vektor deskriptor per pose: Center, Left, Right, Up, Down, Smile.
* Disimpan aman dalam tabel database internal (`EACU.AACUATR`).

---

<!-- Slide 10: Pengamanan Transaksi Presensi -->
## Pengamanan Transaksi: Single-Use Token

Mencegah serangan manipulasi request dan replay attack:

1. **Evaluasi Wajah Berhasil:** Backend menerbitkan token unik `proofNo` (`PROOF-<timestamp>-<hex>`).
2. **Penyimpanan Redis:** Disimpan sementara dengan <span class="highlight-yellow">TTL 300 detik (5 menit)</span>.
3. **Submit Presensi:** Endpoint `/mobile/attendanceSet` memvalidasi keberadaan `proofNo`.
4. **Konsumsi Sekali Pakai:** Token langsung **dihapus atomik** dari Redis setelah presensi tercatat.

---

<!-- Slide 11: Alur Kerja Lengkap -->
## Alur Presensi End-to-End

<div class="placeholder-box">
[AREA GAMBAR: Sequence Diagram alur presensi dari buka kamera hingga pencatatan di database]
</div>

* **Kamera & Liveness:** Validasi pose acak dan pengecekan keaslian wajah lokal.
* **Verifikasi Wajah:** Auto-capture frame, kirim ke server, verifikasi sidecar & DB.
* **Pencatatan Presensi:** Kirim GPS, foto watermark, dan token `proofNo`.

---

<!-- Slide 12: Ringkasan Dampak & Rencana Selanjutnya -->
## Ringkasan & Langkah Selanjutnya

* **Pencapaian Utama:**
  * Performa PWA jauh lebih ringan dan stabil tanpa dependensi berat.
  * Perlindungan menyeluruh dari fraud foto/layar dan replay request.
  * Keamanan biometrik terjamin dengan validasi server-side.
* **Langkah Selanjutnya:**
  * Optimasi toleransi pencahayaan rendah (low-light conditions).
  * Pengujian performa pada variasi perangkat mobile entry-level.

