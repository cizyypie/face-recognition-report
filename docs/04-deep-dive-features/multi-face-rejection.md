# Deep Dive: Multi-Face Rejection (Tolak Multi-Wajah)

## 1. Latar Belakang & Ancaman

Dalam skenario operasional nyata di lapangan (gudang, counter, pool kendaraan):
- Terdapat potensi orang lain berada di belakang atau di samping karyawan yang sedang melakukan absensi.
- Terdapat kemungkinan upaya penipuan dengan memegang foto wajah orang lain di samping wajah asli.
- Terdapat poster, foto dinding, atau gambar manusia pada latar belakang yang dapat membingungkan model pengenalan wajah.

Untuk memastikan absensi dilakukan murni oleh satu individu yang sah, mekanisme penolakan multi-wajah (*multi-face rejection*) diterapkan pada 3 lapisan sistem yang independen.

---

## 2. Lapisan 1: Client Render Loop (`sortAndFilterFaces`)

Di sisi antarmuka pengguna (`PTrIAbsenFaceRecognitionAbsen.vue` dan `PTrIAbsenFaceRecognition.vue`):

```javascript
// Memilah dan menyaring wajah sekunder yang tidak relevan
const sorted = detections.sort((a, b) => area(b) - area(a))
const minArea = area(sorted[0]) * 0.10 // 10% dari luas wajah utama
const validFaces = sorted.filter(d => area(d) >= minArea)

if (validFaces.length > 1) {
  this.statusLabel = `LEBIH DARI 1 WAJAH (${validFaces.length})`
  this.guideRingColor = 'red'
  if (this.sessionDeadline) this.stopTimers() // Hentikan hitung mundur sesi
}
```

- **Filter Wajah Kecil:** Menghindari salah deteksi terhadap gambar kecil pada poster di kejauhan dengan mengabaikan wajah dengan luas $< 10\%$ dari wajah terbesar.
- **Interupsi Interaktif:** Jika terdapat lebih dari 1 wajah yang lolos filter, seluruh proses liveness dibekukan seketika hingga kamera hanya menyorot satu orang.

---

## 3. Lapisan 2: Sidecar Container (`server.js`)

Saat foto final dikirim ke microservice sidecar, algoritma deteksi YuNet mengevaluasi seluruh area gambar resolusi penuh:
- Setelah proses Non-Maximum Suppression (NMS), jumlah wajah yang memenuhi kriteria ukuran dievaluasi.
- Jika terdeteksi lebih dari 1 wajah (`faceCount > 1`), sidecar mengembalikan respons galat eksplisit:
  ```json
  {
    "detected": true,
    "faces": 2,
    "error": "MULTIPLE_FACES"
  }
  ```

---

## 4. Lapisan 3: Backend Controller (`Mobile.pm`)

Di sisi penegakan aturan bisnis backend:
- Subroutine `_verify_face_photo` memeriksa respons dari sidecar.
- Jika menerima galat `MULTIPLE_FACES`, permintaan langsung dihentikan sebelum komputasi biometrik dilakukan.
- Backend mengembalikan respons HTTP terstandarisasi dengan kode kegagalan dan pesan peringatan: `"Lebih dari satu wajah terdeteksi."`.

