# Face Recognition Attendance System — Engineering Case Study & Internship Report

Repositori ini memuat dokumentasi teknis, analisis arsitektur, perjalanan refaktorisasi 22 commit, dan evaluasi implementasi sistem presensi kehadiran biometrik berbasis pengenalan wajah (*face recognition*) untuk aplikasi PWA SWB.

---

## Ringkasan

- **Sistem:** Face Recognition Attendance System (PWA + Sidecar AI + DCAF Perl Backend)
- **Periode Pengembangan:** 30 September 2026 – 15 Oktober 2026
- **Total Kontribusi:** 22 commit | 31 file terdampak
- **Fokus Utama:** Dual Anti-Spoofing, Server-Side Same-Pose Verification, Tokenisasi Single-Use, Optimasi Pemuatan Model WASM, dan Visual/Ergonomi Kamera.

---

## Daftar Isi Dokumentasi

1. [Gambaran Sistem & Masalah yang Diselesaikan](docs/01-overview-and-problem.md)
   - Latar belakang
   - Masalah titip absen (*buddy punching*) & spoofing
   - Kendala performa & bundle size di browser mobile
2. [Desain Arsitektur Sistem](docs/02-architecture-design.md)
   - Split architecture: Client Edge (WASM) vs Sidecar AI Container (Port 3100) vs Backend (DCAF Perl)
   - Diagram komponen & Sequence diagram end-to-end
3. [Perjalanan Evolusi & Refaktorisasi (22 Commits Journey)](docs/03-evolution-and-pivot.md)
   - 6 Fase pengembangan
   - Analisis keputusan pivot arsitektur (menanggalkan `@vladmandic/human`, migrasi ke `face-api` + `YuNet`)
   - Fase 6: Visual, Ergonomi Kamera, & Screen Fill Light
4. [Fitur-Fitur Mendalam (Deep Dive)](docs/04-deep-dive-features/)
   - [Dual Anti-Spoofing Pipeline](docs/04-deep-dive-features/dual-anti-spoofing.md) (MiniFASNetV2-SE WASM + Sidecar reflect padding)
   - [Server-Side Same-Pose Matching](docs/04-deep-dive-features/same-pose-matching.md) (Zero-leak biometrics, per-pose threshold)
   - [Multi-Face Rejection](docs/04-deep-dive-features/multi-face-rejection.md) (Pencegahan multi-wajah 3 lapis)
   - [Token Proofing & Session Hardening](docs/04-deep-dive-features/token-and-security.md) (Redis `proofNo` TTL 300s, JWT-first identity)
   - [Performa, Lifecycle, & Cache](docs/04-deep-dive-features/performance-and-cache.md) (Cache Storage browser, pose sampling 3-frame, retry pendaftaran)
5. [Benchmark & Hasil Pengujian](docs/05-benchmarks-and-results.md)
   - Perbandingan ukuran model, latensi inferensi, dan konsumsi memori heap
6. [Keterbatasan Sistem & Rencana Selanjutnya](docs/06-future-improvements.md)
   - Evaluasi pencahayaan ekstrem, rencana UI admin approval, dan re-enrollment
7. [Dokumen Laporan Lengkap (Monolitik)](docs/full-progress-report.md)
   - Naskah laporan komprehensif 904 baris

## Sorotan Fitur Utama

- **Pendaftaran Wajah Multi-Pose:** Perekaman 6 pose (lurus, kiri, kanan, atas, bawah, senyum) disimpan sebagai vektor 128-d dengan status `pending-approval`.
- **Tantangan Liveness Acak:** Setiap sesi memilih mode secara acak (toleh kiri/kanan, tengok atas/bawah, atau senyum) untuk menjamin kehadiran langsung pengguna asli.
- **Dual Anti-Spoofing:** MiniFASNetV2-SE dieksekusi di browser via ONNX Runtime Web WASM untuk umpan balik real-time, lalu diverifikasi ulang di sidecar Node.js (YuNet + MiniFASNet) sebagai keputusan final (ambang batas 0.50).
- **Tolak Multi-Wajah Berlapis:** Penyaringan wajah minoritas (<10% area) dan pembatalan otomatis jika terdeteksi lebih dari satu orang di kamera maupun server.
- **Server-Side Same-Pose Verification:** Template biometrik tidak pernah dibocorkan ke browser. Setiap pose tantangan hanya dicocokkan 1-ke-1 dengan pose pendaftaran yang identik.
- **Token `proofNo` Sekali Pakai:** Disimpan di Redis dengan TTL 300 detik dan langsung dihapus saat presensi tercatat, mencegah replay attack.
- **Identitas JWT-First:** Resolusi identitas pengguna diprioritaskan dari claim token JWT sesi login aktif.
- **Pemuatan Model Paralel & Persistent Cache Storage:** Berkas model disimpan di browser Cache Storage (`face-models-v1`) sehingga unduh ~13 MB hanya terjadi sekali.

---


