# Deep Dive: Server-Side Same-Pose Face Matching

## 1. Masalah Kebocoran Biometrik pada Arsitektur Lama

Sebelum commit `66fd49df`, sistem menggunakan pola perbandingan di sisi klien:
1. Browser memanggil endpoint `checkFaceRecongnition`.
2. Server mengembalikan array deskriptor biometrik pengguna (`storedDescriptors`) ke browser dalam format JSON.
3. Browser menghitung Euclidean Distance di JavaScript.

Pola ini memiliki risiko keamanan kritis:
- **Zero-Leak Violation:** Vektor embedding 128 dimensi milik karyawan dapat diekstraksi dari Developer Tools atau sniffing lalu lintas jaringan.
- **Cross-Pose Mismatching:** Browser membandingkan pose tantangan secara acak ke semua pose tersimpan untuk mencari kemiripan terdekat, yang memungkinkan pose yang salah tetap lolos verifikasi.

---

## 2. Arsitektur Same-Pose Verification (`_match_face_descriptors`)

Pencocokan biometrik dipindahkan sepenuhnya ke dalam backend Perl (`Mobile.pm`).

```
Enrolled Template (di Database):
[ photoCenter | photoLeft | photoRight | photoUp | photoDown | photoSmile ]
       ^             ^            ^           ^          ^           ^
       | (wajib)     |            |           |          |           |
       | (center)    | (same)     | (same)    | (same)   | (same)    | (same)
       v             v            v           v          v           v
[ photoCenter | photoLeft | photoRight | photoUp | photoDown | photoSmile ]
Live Captured Pose (saat Presensi):
```

### Aturan Perbandingan 1-ke-1:
1. **Pose Netral / Tengah (`photoCenter`):** Wajib ada. Jarak Euclidean terhadap template netral terdaftar harus $< 0.50$.
2. **Pose Tantangan Tantangan Acak:**
   - `photoLeft` (live toleh kiri) **hanya** dibandingkan dengan `photoLeft` (terdaftar).
   - `photoRight` (live toleh kanan) **hanya** dibandingkan dengan `photoRight` (terdaftar).
   - `photoUp` (live tengok atas) **hanya** dibandingkan dengan `photoUp` (terdaftar).
   - `photoDown` (live tengok bawah) **hanya** dibandingkan dengan `photoDown` (terdaftar).
   - `photoSmile` (live tersenyum) **hanya** dibandingkan dengan `photoSmile` (terdaftar).
   - Ambang batas untuk pose tantangan: $< 0.55$.

---

## 3. Prinsip Toleransi Nol (Zero-Tolerance Policy)

Pencocokan menerapkan aturan *fail-fast*:
- Jika terdapat salah satu pose tantangan yang menghasilkan jarak $\ge 0.55$, maka status verifikasi dinyatakan **gagal total** (`isMatch: 0`).
- Sistem mencatat pose spesifik mana yang gagal (`failedPose`) untuk audit dan perbaikan posisi wajah pengguna.
- Template biometrik yang tersimpan di database **tidak pernah dikirimkan kembali** ke browser. Respon server hanya mengembalikan status boolean `isMatch`, jarak per pose untuk visualisasi, dan token bukti `proofNo`.
