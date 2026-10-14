# 06. Keterbatasan Sistem & Rencana Selanjutnya

## 1. Keterbatasan Sistem yang Teridentifikasi

### 1.1 Kondisi Pencahayaan Ekstrem
Model `MiniFASNetV2-SE` mengandalkan tekstur mikro frekuensi tinggi untuk membedakan pantulan kulit asli dengan layar/kertas. Pada kondisi berikut, akurasi klasifikasi dapat menurun:
- **Kondisi Sangat Gelap (Underexposed):** Kamera depan tanpa flash pada area gudang minim cahaya menghasilkan *noise* tinggi, yang berpotensi menyebabkan wajah asli terklasifikasi sebagai spoof.
- **Kondisi Backlight Kuat (Siluet):** Cahaya matahari langsung atau lampu terang di belakang kepala menyebabkan wajah tampak gelap dan titik landmark sulit terdeteksi secara presisi.

### 1.2 Oklusi Fisik Wajah
Penggunaan masker medis atau kacamata hitam pekat menutupi titik landmark penting (mata atau bibir), yang secara otomatis membatalkan kalkulasi pose serta menyebabkan jarak Euclidean melebihi batas toleransi.

---

## 2. Perlu Tindakan Selanjutnya (Action Items)

### Prioritas Tinggi:
1. **Uji Lapangan Multi-Lokasi:**
   - Melakukan uji coba komprehensif pada berbagai titik operasional (gudang logistik, counter gerai, pool kendaraan) untuk mengkalibrasi nilai `spoofThreshold` dan `tiltDelta` pada kondisi riil.
2. **UI Web Administrasi untuk Approval Template:**
   - Membangun antarmuka bagi admin HR / Operation untuk mereview dan menyetujui template pendaftaran wajah baru yang saat ini berstatus `pending-approval`.
3. **Mekanisme Retry pada Presensi (`attendanceSet`):**
   - Menerapkan mekanisme retry yang aman pada proses presensi kehadiran agar tahan terhadap sinyal lemah, dengan tetap memastikan sifat sekali pakai token `proofNo` tidak menimbulkan absensi ganda.

### Prioritas Menengah:
4. **Alur Pendaftaran Ulang Wajah (Re-Enrollment):**
   - Menyediakan fitur pembaruan template bagi karyawan yang mengalami perubahan fisik signifikan (misal perubahan berat badan drastis atau luka fisik).
5. **Kompresi Resolusi Foto Adaptif:**
   - Menyesuaikan resolusi citra yang dikirim ke server secara otomatis berdasarkan kecepatan koneksi pengguna yang terdeteksi.

