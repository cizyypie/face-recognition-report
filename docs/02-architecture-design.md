# 02. Desain Arsitektur Sistem

## 2.1 Arsitektur Tiga Lapisan (Three-Tier Topology)

Sistem mengadopsi arsitektur terpisah antara Client Edge PWA, Backend Controller Perl (DCAF Framework), dan Microservice AI Sidecar Container.

```mermaid
graph TD
    subgraph BrowserPWA["BROWSER PWA (Mobile Client)"]
        P1["PTrIAbsenFaceRecognitionAbsen.vue\nHalaman Presensi Utama"]
        P2["PTrIAbsenFaceRecognition.vue\nHalaman Pendaftaran Wajah"]
        FA["@vladmandic/face-api\nSSD MobileNetV1 + Landmark 68-pt\n+ FaceRecognitionNet 128-d"]
        AS["antiSpoof.js\nonnxruntime-web WASM\nMiniFASNetV2-SE preview real-time"]
        P1 --> FA
        P1 --> AS
        P2 --> FA
        P2 --> AS
    end

    subgraph BackendPerl["BACKEND DCAF (Perl)"]
        MP["DCAF::Controller::Mobile\nMobile.pm"]
        HLP["Fungsi Helper Internal\n_call_face_service\n_verify_face_photo\n_match_face_descriptors\n_check_attendance_proof"]
        CB["Circuit Breaker Redis\nattendance:facesvc:fail_count"]
        MP --> HLP
        HLP --> CB
    end

    subgraph SidecarAI["SIDECAR CONTAINER (face-service:3100)"]
        EX["Express.js Server (Node.js 18)"]
        YU["YuNet ONNX\nface_detection_yunet_2023mar.onnx\nDeteksi Cepat + 5 Landmark"]
        MF["MiniFASNetV2-SE ONNX\nface_antispoof_quantized.onnx\nKlasifikasi Keaslian (probReal)"]
        EX --> YU
        YU --> MF
    end

    subgraph DataLayer["DATA LAYER"]
        RD["Redis\nKey attendance:proof:proofNo\nTTL 300s (Single-use Token)"]
        MY["MySQL - Database EACU\nKolom AACUATR (JSON)\nTemplate Biometrik 6 Pose"]
    end

    P1 -->|"POST /mobile/checkFaceRecongnition\nFoto plain + deskriptor pose live"| MP
    P1 -->|"POST /mobile/attendanceSet\nproofNo + foto watermark + GPS"| MP
    P2 -->|"POST /mobile/insertFaceRecognition\nFoto pendaftaran + 6 deskriptor"| MP
    HLP -->|"POST /analyze\nbase64 frame beresolusi penuh"| EX
    HLP <-->|"SETEX / GET / DEL\nattendance:proof"| RD
    HLP <-->|"Query attGetFaceProfile\nQuery attSaveFaceRecognitionFull"| MY
```

---

## 2.2 Sequence Diagram Alur Presensi Kehadiran

Diagram berikut mengilustrasikan interaksi seluruh komponen dari inisiasi kamera hingga pencatatan transaksi kehadiran:

```mermaid
sequenceDiagram
    participant U as Pengguna (Karyawan)
    participant FE as Vue PWA (Client)
    participant WASM as WebAssembly Engine
    participant BE as Backend Mobile.pm
    participant SVC as face-service Sidecar (3100)
    participant RD as Redis Cache
    participant DB as Database MySQL

    U->>FE: Pilih Tipe Absen (Datang / Pulang)
    FE->>WASM: Pemuatan Model (faceModels.js dari Cache Storage)
    FE->>U: Tampilkan Viewfinder & Panduan Oval

    loop Render Loop requestAnimationFrame
        FE->>WASM: Deteksi Wajah, Landmark 68-pt, & Embedding 128-d
        FE->>WASM: Evaluasi MiniFASNet client-side tiap 450ms
        WASM-->>FE: Status Keaslian (probReal rata-rata 4 frame)
        FE->>FE: Filter Multi-Wajah (tolak jika wajah > 1)
    end

    alt Verifikasi Liveness Tantangan Acak
        Note over FE: Memilih tantangan acak: yaw / tilt / smile
        FE->>U: Tampilkan instruksi tantangan (contoh: tengok atas/bawah)
        U->>FE: Pengguna melakukan gerakan pose
        FE->>FE: Evaluasi geometri landmark (pitchRatio / smileRatio)
        FE->>FE: Sampling 3 frame -> simpan deskriptor pose
        FE->>FE: Generate foto bukti dengan watermark GPS & waktu
    end

    FE->>BE: POST /mobile/checkFaceRecongnition\n{photo, currentFaceDescriptor: {center, up, down, ...}}
    BE->>DB: Ambil template biometrik terdaftar via JWT aud claim
    DB-->>BE: Data template 6 pose tersimpan

    BE->>SVC: POST /analyze {frame: base64, kind: frontal}
    SVC->>SVC: Deteksi YuNet & Anti-spoof MiniFASNet (reflect padding)
    SVC-->>BE: Hasil skor {detected: true, faces: 1, spoofScore: 0.88}

    Note over BE: Validasi Skor Anti-Spoof (>= 0.50)
    BE->>BE: _match_face_descriptors() perbandingan 1-ke-1 per pose
    Note over BE: Validasi Ambang Batas (center < 0.50, pose tantangan < 0.55)

    alt Verifikasi Biometrik Lolos
        BE->>BE: Cetak token proofNo (PROOF-YYYYMMDDHHMMSS-hex8)
        BE->>RD: SETEX attendance:proof:proofNo 300s {userId, matched:1}
        BE-->>FE: Respons sukses {code: 0, proofNo, isMatch: true}
        FE->>U: Tampilkan preview foto & tombol Konfirmasi Absen
    else Verifikasi Gagal
        BE-->>FE: Respons gagal {code: 1, message, failedPose}
        FE->>U: Tampilkan pesan kesalahan & tombol coba lagi
    end

    U->>FE: Tekan tombol Konfirmasi Absen
    FE->>BE: POST /mobile/attendanceSet\n{vAttendanceType, GPS coords, foto watermark, proofNo}
    BE->>RD: GET attendance:proof:proofNo
    RD-->>BE: Data payload token
    BE->>BE: Validasi kepemilikan userId & status matched
    BE->>RD: DEL attendance:proof:proofNo (token langsung dihapus)
    BE->>BE: Validasi radius lokasi kantor (100 meter)
    BE->>DB: Simpan transaksi absensi ke absTrnCreate
    BE-->>FE: Respons sukses {code: 0, status: success}
    FE->>U: Notifikasi popup presensi berhasil dicatat
```

---

## 2.3 Peran Setiap Lapisan

### 1. Sisi Klien (PWA di Browser Ponsel)
- Mengakses kamera perangkat melalui `navigator.mediaDevices.getUserMedia()`.
- Menjalankan inferensi ringan secara lokal: deteksi landmark 68-titik, ekstraksi embedding, dan evaluasi anti-spoofing awal via WebAssembly.
- Mengelola antarmuka pengguna: instruksi tantangan acak, visual guide ring (merah/hijau), penghitung waktu mundur sesi (45 detik), dan pembentukan watermark koordinat serta waktu pada foto.

### 2. Sisi Sidecar AI (`face-service:3100`)
- Microservice mandiri berbasis Node.js 18 yang dikemas dalam Docker container terisolasi.
- Menjalankan model **YuNet** untuk deteksi bounding box berkecepatan tinggi (~230 KB) dan **MiniFASNetV2-SE** untuk klasifikasi citra asli vs spoof.
- Melakukan normalisasi citra masukan dengan konteks 1.5x *reflect padding* dan *downscale* INTER_AREA agar identik dengan distribusi data saat model dilatih.
- Mengembalikan keputusan authoritative `spoofScore` dan penolakan multi-wajah.

### 3. Sisi Backend Controller (`Mobile.pm` DCAF Perl)
- Menjadi gerbang otentikasi utama dan penegak kebijakan keamanan (*security enforcer*).
- Menghubungkan komunikasi antar-layanan menggunakan *circuit breaker* berbasis Redis.
- Menjalankan algoritma pencocokan biometrik pose-ke-pose (*same-pose matching*) dengan prinsip *zero-leak biometrics*.
- Menerbitkan dan mengonsumsi token sekali pakai `proofNo` berumur 300 detik.
