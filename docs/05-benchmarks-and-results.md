# 05. Benchmark & Hasil Pengujian

## 1. Perbandingan Arsitektur: `@vladmandic/human` vs `face-api + YuNet`

| Parameter Evaluasi | Fase 1 (`@vladmandic/human`) | Fase Terkini (`face-api` + `YuNet`) | Dampak Optimasi |
|---|---|---|---|
| **Ukuran Model di Sisi Klien** | > 25 MB (seluruh modul Human) | ~12.2 MB (face-api + MiniFASNet) | Penurunan ukuran aset ~51% |
| **Ukuran Model di Sisi Sidecar** | > 30 MB (Node TensorFlow Full) | ~830 KB (YuNet 230 KB + MiniFASNet 600 KB) | Penurunan ukuran model sidecar > 97% |
| **Waktu Cold Start Pertama** | 12 - 18 detik | 4 - 7 detik | 2.5x lebih cepat |
| **Waktu Start Berikutnya (Cache Hit)** | Tetap unduh / parse ulang (10s) | 1.2 - 2.5 detik (dari Cache Storage) | 4x - 8x lebih cepat |
| **FPS Inferensi di Android Entry-Level** | 1 - 3 FPS (tersendat) | 8 - 15 FPS (responsif) | Kelancaran interaksi meningkat drastis |
| **Keamanan Data Biometrik** | Template terekspos di browser | Zero-Leak (matching di server Perl) | Eliminasi celah pencurian biometrik |

---

## 2. Metrik Ambang Batas (Threshold Calibration)

| Komponen Validasi | Ambang Batas Default | Keterangan & Rationale |
|---|---|---|
| **Anti-Spoofing (Client & Server)** | `0.50` | Nilai probabilitas keaslian (*probReal*). Skor $\ge 0.50$ dinyatakan wajah asli hidup; $< 0.50$ ditolak sebagai serangan presentasi. |
| **Same-Pose: Pose Netral (`photoCenter`)** | `0.50` | Jarak Euclidean ketat antara pose lurus saat presensi terhadap pose lurus saat pendaftaran. |
| **Same-Pose: Pose Tantangan (`poseThreshold`)** | `0.55` | Jarak Euclidean dengan toleransi sudut gerak untuk pose toleh kiri, toleh kanan, tilt atas, tilt bawah, dan senyum. |
| **Deteksi Tilt Kepala (`tiltDelta`)** | `0.08` | Deviasi rasio geometrik vertikal hidung-mata-dagu terhadap baseline netral individu. |
| **Deteksi Senyum (`smileThreshold`)** | `0.60` | Rasio lebar mulut terhadap jarak antar-mata dengan hysteresis toleransi 0.57. |
| **Masa Aktif Token (`proofNo` TTL)** | `300 detik` | Jendela waktu maksimal antara verifikasi wajah dengan penyimpanan presensi. |

---

## 3. Hasil Pengujian Ketahanan Jaringan

Pada simulasi jaringan lambat (3G throttling dengan latensi 500 ms dan packet loss 10%):
- Helper `postEnrollWithRetry()` berhasil menyelesaikan pendaftaran pada percobaan ke-2 tanpa meminta pengguna mengulang gerakan pose.
- Pengguna mendapatkan informasi visual `"Koneksi lambat, mencoba ulang (2/3)..."` secara transparan.
