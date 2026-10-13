# Deep Dive: Token Proofing & Session Hardening

## 1. Mekanisme Token Sekali Pakai (`proofNo`)

Untuk mencegah penembakan endpoint presensi kehadiran (`/mobile/attendanceSet`) tanpa verifikasi wajah yang sah, diterapkan sistem tiket bukti (*proof token*).

```
[ POST /mobile/checkFaceRecongnition ]
                 │
  (Validasi biometrik & anti-spoof lolos)
                 │
                 ▼
      Generate proofNo acak
(PROOF-YYYYMMDDHHMMSS-xxxxxxxx)
                 │
                 ▼
    Redis SETEX (TTL 300 detik)
 key: attendance:proof:<proofNo>
  payload: { userId, matched: 1 }
                 │
                 ▼
[ POST /mobile/attendanceSet (dengan proofNo) ]
                 │
                 ▼
    Validasi token via Redis GET
                 │
                 ▼
    Hapus token via Redis DEL (Konsumsi Atomik)
                 │
                 ▼
       Simpan Presensi ke Database
```

### Karakteristik Token:
1. **Masa Berlaku Terbatas (TTL 300 Detik):** Pengguna memiliki jendela waktu 5 menit antara verifikasi wajah berhasil hingga penekanan tombol simpan absensi.
2. **Konsumsi Sekali Pakai (Single-Use):** Begitu token divalidasi oleh `_check_attendance_proof()`, token langsung dihapus dari Redis menggunakan perintah `DEL`. Hal ini menjamin token yang sama tidak dapat dipakai ulang (*replay attack prevention*).
3. **Validasi Pemilik (User Binding):** Payload token menyimpan `userId`. Jika token digunakan oleh akun lain, backend menolak permintaan dengan kode galat pembajakan sesi.

---

## 2. Resolusi Identitas JWT-First (Commit `b6048de4`)

Pada kode lama, identitas karyawan diambil langsung dari parameter body permintaan HTTP (`idUser`), yang rentan dimanipulasi melalui modifikasi payload request.

### Urutan Prioritas Baru:
1. **Klaim Token JWT (`aud` claim):** Terlebih dahulu mengambil identitas dari token JWT sesi login yang sudah divalidasi kriptografis oleh middleware otentikasi.
2. **Kode Karyawan (`vCode`):** Fallback sekunder jika operasi dipanggil melalui konteks terotentikasi alternatif.
3. **Parameter Body (`idUser`):** Hanya berfungsi sebagai fallback terakhir untuk skenario administratif tertentu (misal admin mendaftarkan wajah karyawan lain).

Refaktorisasi ini diterapkan secara identik pada kedua endpoint utama: `insertFaceRecognition` (pendaftaran) dan `checkFaceRecongnition` (presensi).
