# 03. Perjalanan Evolusi & Refaktorisasi (18 Commits Journey)

Pengembangan sistem absensi wajah pada branch `face-attendance-human` berlangsung dalam 5 fase evolusi yang mencerminkan pembelajaran teknis dan optimasi berkelanjutan.

---

## 3.1 Ringkasan 5 Fase Evolusi

| Fase | Periode | Fokus Utama | Capaian Kunci |
|---|---|---|---|
| **Fase 1: Fondasi** | 30 Sep – 2 Okt 2026 | Eksplorasi spike pustaka `@vladmandic/human` | Integrasi model Human, pembuatan skema SQL biometrik, controller sesi, dan halaman enrollment awal. |
| **Fase 2: Pivot** | 5 Okt 2026 | Pembongkaran Human & migrasi arsitektur modular | Pustaka Human dibuang total (bundle terlalu berat). Mengadopsi `face-api` di peramban dan `YuNet` di sidecar. Penolakan multi-wajah diterapkan. |
| **Fase 3: Anti-Spoof** | 6 – 8 Okt 2026 | Dual pipeline anti-spoofing & UX feedback | Integrasi MiniFASNet ganda (browser WASM + sidecar), pembuatan Dockerfile sidecar, dan panduan visual oval ring. |
| **Fase 4: Liveness & Keamanan** | 8 – 9 Okt 2026 | Penyempurnaan liveness & backend hardening | Pose tilt (pitch) dan smile menggantikan mouth-open. Pencocokan biometrik dipindah ke server (zero-leak). Pengenalan token Redis `proofNo` dan JWT-first. |
| **Fase 5: Performa & Ketahanan** | 12 – 13 Okt 2026 | Optimasi runtime, cache, dan reliabilitas jaringan | Pemuatan model paralel sekali pakai, persistent Cache Storage, pose sampling 3-frame, auto-stop kamera, dan auto-retry jaringan saat pendaftaran. |
| **Fase 6: Visual & Ergonomi Kamera** | 15 Okt 2026 | Optimasi visual guide, ergonomi framing, dan pencahayaan layar | Perbaikan bug fogging oval (`evenodd`), solid white fill-light, reposisi oval ke atas, instruksi dinamis di bawah oval, screen wake lock, dan pembersihan UI. |

---

## 3.2 Linimasa Lengkap 22 Commit

| No | Hash | Tanggal | Pesan Commit | Kategori / Fase |
|---|---|---|---|---|
| C-01 | `982860cd` | 30 Sep 2026 | feat: add human liveness detection page and model integration | Fase 1 |
| C-02 | `1d1c6677` | 30 Sep 2026 | Add Face-Recognition Attendance System Queries and Controller | Fase 1 |
| C-03 | `5cc5c1d3` | 01 Okt 2026 | feat: add face enrollment page and routing | Fase 1 |
| C-04 | `7077fc60` | 01 Okt 2026 | fix: improve camera initialization and error handling | Fase 1 |
| C-05 | `8c8caf18` | 02 Okt 2026 | Enhance face detection and challenge handling | Fase 1 |
| C-06 | `928d5cfb` | 05 Okt 2026 | refactor(face): remove Human-based enroll/attendance flow, use YuNet in face-service | Fase 2 (PIVOT) |
| C-07 | `30124b61` | 05 Okt 2026 | refactor(face): reject multiple faces on enroll/attendance | Fase 2 |
| C-08 | `a5a2a833` | 06 Okt 2026 | feat: integrate anti-spoofing functionality and enhance face recognition flow | Fase 3 |
| C-09 | `27881b6d` | 08 Okt 2026 | feat: add Dockerfile and docker-compose configuration for face service | Fase 3 |
| C-10 | `aa51a845` | 08 Okt 2026 | Enhance anti-spoofing functionality and UI feedback | Fase 3 |
| C-11 | `66fd49df` | 08 Okt 2026 | feat: enhance face recognition liveness checks with tilt and smile detection | Fase 4 |
| C-12 | `b6048de4` | 09 Okt 2026 | feat: refactor user profile resolution logic and enhance attendance state handling | Fase 4 |
| C-13 | `3c5b89f9` | 12 Okt 2026 | fix: every check-in reloaded about 13 MB of face models from scratch | Fase 5 |
| C-14 | `ac588bd8` | 12 Okt 2026 | fix: prevent camera from running after component unmounts | Fase 5 |
| C-15 | `4962d6cb` | 12 Okt 2026 | feat: enhance anti-spoofing preprocessing and adjust threshold for improved accuracy | Fase 5 |
| C-16 | `4fcf749b` | 12 Okt 2026 | feat: prevent multiple simultaneous detections by introducing isStarting flag | Fase 5 |
| C-17 | `1d407b2e` | 12 Okt 2026 | feat: implement pose sampling for improved face recognition stability and caching of model files | Fase 5 |
| C-18 | `0b8b1ab0` | 13 Okt 2026 | feat: implement retry logic for face recognition enrollment to handle network issues | Fase 5 |
| C-19 | `c3a48645` | 15 Okt 2026 | fix(face-recog): fix foggy oval overlay and set solid white background | Fase 6 |
| C-20 | `d70dc9b5` | 15 Okt 2026 | refactor: simplify top bar layout and remove unused buttons in face recognition pages | Fase 6 |
| C-21 | `756d3f76` | 15 Okt 2026 | refactor: enhance hint display logic and improve styling in face recognition components | Fase 6 |
| C-22 | `b9af65a0` | 15 Okt 2026 | feat: implement wake lock functionality to enhance user experience during face recognition | Fase 6 |

---

## 3.3 Catatan Keputusan Arsitektur: Pivot C-06 (Architectural Decision Record)

### Konteks
Pada awal pengembangan (Fase 1), seluruh pipeline dirancang menggunakan pustaka `@vladmandic/human`. Pustaka ini menyediakan kemampuan komprehensif (FaceMesh, BlazeFace, Iris, Body, Liveness).

### Masalah yang Ditemukan
1. **Beban Bundel JavaScript:** Bundel kompilasi pustaka Human mencapai puluhan ribu baris kode dan mengunduh 8-10 berkas model terpisah secara berurutan.
2. **Kinerja Rendah di Smartphone Entry-Level:** Backend TensorFlow.js dari Human mengalami *throttling* berat pada chipset kelas menengah ke bawah, menghasilkan FPS rendah (<3 FPS) dan pemanasan perangkat yang tidak wajar.
3. **Rigiditas Pipeline:** Sulit menyematkan model anti-spoof kustom (MiniFASNetV2-SE) di dalam alur inferensi bawaan Human.

### Keputusan (ADR-01)
Pada commit `928d5cfb` (05 Oktober 2026), diputuskan untuk **menghapus total dependensi `@vladmandic/human`** dan menggantinya dengan arsitektur terpisah:
1. **Sisi Klien (PWA):** Menggunakan pustaka `@vladmandic/face-api` yang jauh lebih ringkas, dipadukan dengan modul mandiri `antiSpoof.js` berbasis ONNX Runtime Web WASM.
2. **Sisi Sidecar:** Menggunakan model **YuNet** (~230 KB) dari OpenCV Zoo yang terbukti sangat cepat, ringan, dan akurat dalam mendeteksi kotak wajah dan 5 titik landmark utama.

### Dampak Positif
- Penurunan drastis ukuran memori dan waktu inisialisasi awal.
- Komponen arsitektur menjadi modular: backend Perl, sidecar Node.js, dan antarmuka Vue dapat dikembangkan dan diuji secara independen.

---

## 3.4 Detail Perubahan Fase 6: Visual & Ergonomi Kamera (15 Okt 2026)

### 1. Perbaikan Bug Foggy Oval & Solid White Fill-Light (Commit C-19 — `c3a48645`)
- **What:** Mengganti compositing `destination-out` dengan `ctx.fill('evenodd')`. Area luar oval diisi putih solid `#FFFFFF`, dalam oval 100% transparan tanpa residu.
- **Why:** Pada kode lama, `ctx.fillStyle` tidak di-reset sebelum `destination-out`, menyisakan residu putih ~25% di dalam oval yang merusak kontras wajah dan memicu penolakan anti-spoof palsu (`0.01`). Area luar putih solid sekaligus berfungsi sebagai *screen fill light* (ring light alami dari layar HP).

### 2. Pembersihan UI & Judul Simetris (Commit C-20 — `d70dc9b5`)
- **What:** Menghapus chip debug (`Confidence`, `Wajah`, timestamp), menyembunyikan tombol flash/settings di topbar, merapikan judul topbar ke posisi tengah simetris, dan auto-hide dropdown tipe absen setelah dipilih.
- **Why:** Menghilangkan distraksi visual, memaksimalkan area vertikal untuk preview kamera, dan memberi tampilan antarmuka yang bersih.

### 3. Ergonomi Posisi Oval & Teks Panduan Lebih Jelas (Commit C-21 — `756d3f76`)
- **What:** Menggeser pusat oval ke area atas (`cy = ch * 0.38`), menempatkan pill instruksi tepat di bawah lengkungan bawah oval (`hintTop = cy + ovalH + 18px`), dan memperbesar font instruksi menjadi 15px tebal.
- **Why:** Pengguna memegang smartphone setinggi mata (bukan di tengah bawah). Penempatan instruksi tepat di bawah oval dengan font lebih besar memudahkan pengguna membaca instruksi pose tanpa memalingkan pandangan dari kamera.

### 4. Kontrol Kamera Eksplisit & Screen Wake Lock (Commit C-22 — `b9af65a0`)
- **What:** Kamera hanya aktif saat tombol "Mulai Kamera" ditekan, serta mengintegrasikan Screen Wake Lock API (`navigator.wakeLock`) dan panduan pencahayaan redup.
- **Why:** Memberikan kontrol penuh kepada pengguna sebelum streaming aktif, serta mencegah layar meredup/tidur (*auto-dimming/sleep*) saat proses verifikasi berlangsung.

