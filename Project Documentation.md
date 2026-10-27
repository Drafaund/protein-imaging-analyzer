# Dokumentasi Lengkap Fitur, Alur Kerja, dan Mitigasi Risiko
## Protein Crystallization Imaging Analyzer

Dokumen ini merupakan catatan teknis dan panduan operasional proyek dalam Bahasa Indonesia. Dokumen ini merangkum secara mendalam fungsionalitas dari setiap fitur, arsitektur perangkat lunak, potensi risiko teknis, serta strategi mitigasi yang diterapkan.

---

## 1. Ringkasan Eksekutif & Arsitektur Sistem

Aplikasi ini adalah platform analisis citra berbasis web (*full-stack*) yang dirancang khusus untuk eksperimen kristalisasi protein di laboratorium bioteknologi (sejalan dengan produk instrumen Formulatrix seperti *Rock Imager* dan *Rock Maker*).

### Teknologi yang Digunakan (Tech Stack):
* **Backend:** C# .NET 8 Web API (didukung ASP.NET Core SignalR untuk *real-time progress & metrics streaming*)
* **Arsitektur:** Clean Architecture (Domain, Application, Infrastructure, Web API) yang mematuhi prinsip SOLID secara ketat.
* **Frontend:** React dengan TypeScript (TSX).
* **AI & Imaging Engine:** Model berbasis Python (PyTorch/TensorFlow) untuk tahap *training*, diekspor ke format **ONNX**, dan dieksekusi secara *in-process* di backend .NET 8 menggunakan **Microsoft.ML.OnnxRuntime** serta **OpenCvSharp**.
* **Database & Storage:** SQLite untuk persistensi relasional (metadata *plate*, koordinat *well*, label prediksi, riwayat *override* manual).
* **CI/CD & Code Quality:** GitLab CI/CD (GitLab Runner), laporan cakupan pengujian Cobertura (target *unit test coverage* 100% pada *core business logic*), dan *static analysis* Qodana (target 0 *warning*).

### Diagram Alur Siklus Tertutup (*Closed-Loop Flywheel*):
```
┌────────────────────────────────────────────────────────┐
│  Fitur 1: Dataset Annotation & Model Training Engine   │
│  • Labeling citra tetesan kristal (React UI)           │
│  • .NET 8 memicu training Python di background         │
│  • Hasil training diekspor ke artefak model ONNX       │
└───────────────────────────┬────────────────────────────┘
                            │ Model ONNX Baru Aktif
                            ▼
┌────────────────────────────────────────────────────────┐
│  Fitur 2: Automated Plate Inference & Lab Reporting    │
│  • Unggah citra plate & auto-mapping ke 96-well (A1-H12)│
│  • Inferensi berkecepatan tinggi via ONNX Runtime      │
│  • Simpan ke SQLite & ekspor ringkasan inspeksi (CSV)  │
└───────────────────────────┬────────────────────────────┘
                            │ Hasil Prediksi & Confidence
                            ▼
┌────────────────────────────────────────────────────────┐
│  Fitur 3: Active Learning Review & Feedback Loop       │
│  • Triage visual: beri tanda pada well meragukan /     │
│    ber-confidence rendah atau kategori Crystal         │
│  • Peneliti mengoreksi label yang salah (override)     │
│  • Sampel koreksi dialirkan otomatis ke dataset latih  │
│    (kembali ke Fitur 1 untuk continuous retraining)    │
└────────────────────────────────────────────────────────┘
```

---

## 2. Rincian dan Spesifikasi Fitur Utama

### FITUR 1: Dataset Annotation & Automated Model Training Pipeline

#### Deskripsi & Tujuan
Menyediakan modul bagi peneliti untuk mengelola dataset citra lokal, melabeli tetesan kristal secara interaktif di browser, dan memicu pelatihan model AI baru langsung dari aplikasi tanpa perlu menjalankan perintah terminal/skrip secara manual.

#### Fungsionalitas Rinci:
1. **Antarmuka Pelabelan Gambar (*Image Labeling UI*):**
   * Menampilkan kumpulan citra tetesan protein kristalisasi yang belum memiliki label.
   * Peneliti dapat memberikan anotasi ke dalam 4 kategori standar industri:
     - **Clear:** Tetesan cairan jernih, belum ada kristal atau endapan.
     - **Precipitate:** Terjadi pengendapan zat amorf (keruh/berbutir halus tapi bukan kristal).
     - **Crystal:** Terbentuk kristal protein (kondisi paling berharga/kritis yang dicari peneliti).
     - **Other:** Tetesan rusak, terdapat gelembung udara, debu, atau artefak visual lainnya.
   * Dilengkapi tombol navigasi cepat (keyboard shortcut) untuk mempercepat proses pelabelan data massal.
2. **Orkestrasi Pelatihan Model via .NET 8 (*Training Orchestrator*):**
   * Pengguna cukup menekan tombol *"Start Training"* pada antarmuka web.
   * Backend .NET 8 secara otomatis memvalidasi kelengkapan dataset yang sudah dilabeli.
   * .NET 8 menjalankan proses eksternal di sistem operasi (`System.Diagnostics.Process`) untuk memanggil skrip Python pelatihan.
3. **Pemantauan & Streaming Progres Asinkron (Real-Time Percentage & Metrics):**
   * **Perhitungan Persentase Progres:** Skrip Python menghitung persentase kemajuan pelatihan berdasarkan rasio epoch/langkah yang telah selesai terhadap total target:
     $$\text{Progress (\%)} = \left(\frac{\text{Current Epoch}}{\text{Total Epochs}}\right) \times 100$$
   * **Streaming Data Terstruktur (stdout):** Pada setiap akhir epoch atau batch, Python mencetak baris JSON ke saluran output standar dengan pembersihan buffer seketika (`flush=True`), contoh:
     `{"progress": 50.0, "epoch": 10, "total_epochs": 20, "loss": 0.2415, "acc": 0.925}`
   * **Orkestrasi SignalR di .NET 8:** Backend .NET menangkap output teks secara asinkron (`OutputDataReceived`), mem-parsing nilai JSON, dan langsung mem-broadcast data pembaruan tersebut ke klien web menggunakan **ASP.NET Core SignalR (WebSocket)**.
   * **Visualisasi Reaktif di React TSX:** Komponen antarmuka web menerima event dari SignalR dan secara *real-time* memperbarui bilah progres (*Live Progress Bar*) dari 0% hingga 100%, lengkap dengan metrik *loss*, *accuracy*, estimasi sisa waktu, dan tombol pembatalan (*Cancel Training*).
   * **State Resiliency (Reconnect Support):** Jika pengguna me-refresh atau menutup browser sementara proses training berjalan, backend tetap melacak status pekerjaan di memori/database. Saat halaman dibuka kembali, frontend langsung meminta status terkini (`GET /api/training/status`) dan menyambung ulang ke stream SignalR tanpa mengganggu proses pelatihan yang sedang berlangsung.
4. **Ekspor & Kompilasi Artefak ONNX:**
   * Setelah pelatihan selesai dan metrik performa memenuhi syarat, skrip Python mengekspor bobot model (*weights*) ke format terbuka **ONNX** (`.onnx`).
   * Backend .NET memverifikasi keabsahan file `.onnx` tersebut dan menyimpannya ke direktori model yang siap digunakan untuk inferensi.

---

### FITUR 2: Automated Plate Inference, Grid Mapping & Inspection Reporting

#### Deskripsi & Tujuan
Mengotomasi proses inspeksi *plate* kristalisasi skala besar. Mengeliminasi kebiasaan lama di mana peneliti harus menatap layar dan mencatat status ratusan sumur (*well*) satu per satu ke lembar kerja Excel manual.

#### Fungsionalitas Rinci:
1. **Ingesti Citra & Pemetaan Koordinat Grid 96-Well (*Auto-Grid Mapping*):**
   * Mengunggah satu set gambar *plate* kristalisasi.
   * Sistem secara otomatis membaca metadata atau pola penamaan file (misalnya `Plate01_A01.jpg` s/d `Plate01_H12.jpg`) dan memetakannya secara presisi ke koordinat standar mikrotiter plate 96-well (baris A–H, kolom 1–12).
2. **Inferensi Asinkron Berkecepatan Tinggi via ONNX Runtime:**
   * Inferensi tidak memanggil runtime Python, melainkan dijalankan langsung di dalam proses C# .NET 8 menggunakan pustaka **`Microsoft.ML.OnnxRuntime`**.
   * Prapemrosesan citra (penyesuaian ukuran/resize dan normalisasi matriks warna) dilakukan secara cepat menggunakan **OpenCvSharp**.
   * Menghasilkan prediksi kategori (*Clear*, *Precipitate*, *Crystal*, atau *Other*) beserta nilai tingkat keyakinan (*confidence score*, contoh: *Crystal 94.2%*).
   * Proses berjalan secara asinkron dengan dukungan pembatalan (*CancellationToken*) sehingga aplikasi web tetap responsif.
3. **Persistensi Data ke SQLite:**
   * Seluruh data tersimpan terstruktur di database SQLite: ID *plate*, tanggal inspeksi, versi model yang digunakan, posisi sumur, label prediksi, dan nilai keyakinan.
4. **Mesin Pelaporan & Ekspor Lab (*Lab Inspection Export Engine*):**
   * Pengguna dapat mengekspor hasil inspeksi ke format **CSV / Excel** terstruktur dengan satu klik.
   * Berkas ekspor memuat rekapitulasi data: daftar koordinat sumur yang berhasil menghasilkan kristal (*hit list*), persentase distribusi hasil uji, serta catatan inspeksi untuk laporan riset resmi laboratorium.

---

### FITUR 3: Interactive Review & Active Learning Feedback Loop

#### Deskripsi & Tujuan
Sebagai jembatan antara evaluasi manusia (*Human-in-the-Loop*) dan kecerdasan buatan. Fitur ini mengatasi keterbatasan alami AI (yang tidak mungkin 100% sempurna) dan memastikan data yang salah diprediksi dapat dikoreksi serta dimanfaatkan kembali untuk membuat model AI semakin pintar.

#### Fungsionalitas Rinci:
1. **Visualisasi Interaktif Plate 96-Well:**
   * Menampilkan representasi visual utuh dari plate mikrotiter ($8 \times 12$) dengan kode warna yang jelas untuk setiap status:
     - 🟢 Hijau: *Crystal*
     - 🟡 Kuning: *Precipitate*
     - ⚪ Abu-abu/Putih: *Clear*
     - 🔴 Merah: *Other* / Anomali
2. **Mekanisme Triage Cerdas (*Confidence Triage*):**
   * Sistem otomatis memberikan penanda khusus (*flagging/highlight*) pada sumur-sumur yang:
     - Memiliki nilai keyakinan rendah (misal: *confidence* $< 70\%$).
     - Terdeteksi sebagai *Crystal* (karena kristal adalah target utama yang paling bernilai dan wajib diverifikasi keasliannya oleh manusia).
   * Manfaat: Peneliti tidak perlu memeriksa ke-96 gambar sumur secara acak; peneliti cukup memfokuskan perhatian pada sumur-sumur yang diberi tanda peringatan oleh AI.
3. **Koreksi Manual Satu-Klik (*One-Click Manual Override*):**
   * Mengklik sumur akan menampilkan gambar resolusi penuh dari tetesan kristal tersebut.
   * Jika prediksi model salah (misalnya AI memprediksi *Precipitate*, padahal aslinya *Crystal* kecil), peneliti dapat langsung mengubah labelnya dengan satu klik.
   * Database SQLite mencatat log audit: label asli dari AI, label hasil koreksi manusia, nama operator, dan waktu koreksi.
4. **Siklus Pembelajaran Aktif (*Active Learning Re-Ingestion Loop*):**
   * Citra yang labelnya telah dikoreksi atau divalidasi oleh peneliti secara otomatis dialirkan kembali (*auto-routed*) ke dalam folder dataset latih (kembali ke Fitur 1).
   * Data koreksi ini menjadi bahan bakar baru (*hard examples*) untuk siklus pelatihan berikutnya, sehingga model berikutnya tidak akan mengulangi kesalahan yang sama.

---

## 3. Identifikasi Risiko Teknis, Potensi Bahaya, dan Cara Mengatasinya (Mitigasi)

Berikut adalah kajian risiko rekayasa perangkat lunak yang dihadapi serta solusi mitigasinya:

---

### RISIKO 1: Orkestrasi Proses & Streaming Log Real-time (.NET ↔ Python)

#### Apa Masalahnya?
.NET 8 dan Python adalah dua runtime independen. Backend C# bertindak sebagai koordinator yang memanggil skrip Python di level sistem operasi untuk melatih model AI.

#### Potensi Bahaya:
1. **I/O Deadlock (Aplikasi Macet Total):**  
   Saat skrip Python melakukan `print()` untuk mencatat log epoch, teks masuk ke buffer *Standard Output* (`stdout`) sistem operasi. Jika antrean buffer OS ini penuh (sekitar 4 KB) dan aplikasi .NET terlambat atau tidak membacanya secara asinkron, skrip Python akan berhenti mendadak (*freeze*) selamanya menunggu buffer dikosongkan.
2. **Zombie / Orphan Process (Proses Siluman Penguras CPU/GPU):**  
   Jika pengguna membatalkan pelatihan (*Cancel*) atau server .NET mati mendadak, skrip Python di OS tidak otomatis berhenti. Proses Python akan tetap berjalan di background, terus memakan 100% daya prosesor/GPU, dan memicu error kehabisan memori jika ada pelatihan baru yang dipicu.
3. **Silent Crash:**  
   Jika Python mengalami error (misal: CUDA out-of-memory atau file gambar rusak), namun .NET tidak menangkap saluran *Standard Error* (`stderr`), antarmuka web pengguna akan terus menampilkan animasi *loading* tanpa henti.

#### Strategi Mitigasi & Solusi di Kode:
* **Prinsip Clean Architecture:** Logika eksekusi proses OS dibungkus ke dalam interface di *Application Layer* (misal: `ITrainingProcessRunner`) dan diimplementasikan secara terisolasi di *Infrastructure Layer*. Logika bisnis tidak terikat langsung pada perintah terminal.
* **Pembacaan Buffer Asinkron (*Async Event Stream*):** Menggunakan event handler `OutputDataReceived` dan `ErrorDataReceived` dari C# `System.Diagnostics.Process` dengan panggilan `BeginOutputReadLine()` dan `BeginErrorReadLine()`. Ini memastikan setiap karakter yang dicetak Python langsung dikonsumsi secara *non-blocking*, menghilangkan kemungkinan terjadinya *deadlock*.
* **Pembersihan Pohon Proses Utuh (*Process Tree Kill*):** Menghubungkan proses eksternal dengan `CancellationToken`. Jika pembatalan terjadi, .NET mengeksekusi metode modern:
  ```csharp
  process.Kill(entireProcessTree: true);
  ```
  Ini menjamin seluruh proses induk dan *sub-process* Python/CUDA dihentikan hingga tuntas dari sistem operasi tanpa meninggalkan *orphan process*.
* **Validasi Exit Code:** Sistem selalu mengecek `process.ExitCode`. Jika nilainya bukan 0, isi `stderr` dibaca dan diterjemahkan menjadi pesan error yang ramah pengguna.

---

### RISIKO 2: Thread-Safe Batch Inference & Manajemen Memori Unmanaged

#### Apa Masalahnya?
Satu *plate* kristalisasi menghasilkan 96 gambar beresolusi tinggi. Melakukan inferensi pada 96 gambar sekaligus dapat membebani memori dan menyebabkan *UI thread* menjadi lambat atau macet.

#### Potensi Bahaya:
1. **Kebocoran Memori & Fragmentasi Native Heap (*Memory Leaks*):**  
   Pustaka seperti OpenCvSharp dan ONNX Runtime mengalokasikan memori native C++ (di luar kontrol otomatis .NET Garbage Collector). Jika penanganan matriks gambar (`Mat`) tidak ditutup dengan benar, memori RAM server akan bocor dan cepat habis (*Out of Memory*).
2. **Konflik Akses Thread (*Race Condition*):**  
   Jika beberapa permintaan inferensi dijalankan secara paralel tanpa sinkronisasi yang aman, sesi inferensi model AI dapat mengalami kerusakan memori internal (*internal state corruption*).

#### Strategi Mitigasi & Solusi di Kode:
* **Penerapan Ketat Blok `using` dan `IDisposable`:** Seluruh objek matriks citra OpenCvSharp (`Mat`, `VideoCapture`) dan tensor ONNX Runtime selalu dibungkus dalam blok `using` atau dipanggil dengan `.Dispose()` eksplisit setelah selesai diproses.
* **Pembatasan Konkurensi dengan Semaphore:** Untuk mencegah lonjakan beban memori saat memproses 96 gambar sekaligus, gunakan `SemaphoreSlim` (misalnya dibatasi maksimum 4–8 gambar diproses bersamaan tergantung jumlah inti CPU):
  ```csharp
  private readonly SemaphoreSlim _throttler = new SemaphoreSlim(maxParallelism);
  ```
* **Pola Singleton untuk Sesi Inferensi (*Thread-Safe Session*):** Instansiasi `InferenceSession` dari ONNX Runtime dibuat sebagai objek tunggal (*Singleton*) yang digunakan bersama secara *read-only*. Sesi ini aman dipanggil dari banyak thread (*thread-safe for concurrent Run calls*), sehingga menghemat waktu *startup* dan menghindari overhead alokasi ulang memori.

---

### RISIKO 3: Ketidakseimbangan Kelas (Class Imbalance) & Kualitas Feedback pada Active Learning

#### Apa Masalahnya?
Dalam eksperimen kristalisasi protein di dunia nyata, pembentukan kristal (*Crystal*) adalah peristiwa langka (kurang dari 5% dari total tetesan). Mayoritas tetesan adalah cairan bening (*Clear*) atau endapan amorf (*Precipitate*).

#### Potensi Bahaya:
1. **Paradoks Akurasi Semu (*Accuracy Paradox*):**  
   Jika model selalu menebak "Clear" atau "Precipitate", model tersebut bisa memiliki akurasi di atas 90%, namun sebenarnya gagal total karena tidak mampu mendeteksi keberadaan kristal sama sekali.
2. **Degradasi Performa (*Catastrophic Overfitting* saat Retraining):**  
   Jika pengguna hanya melabeli beberapa gambar baru dan langsung melakukan pelatihan ulang secara naif tanpa menyeimbangkan data, model baru justru bisa kehilangan kemampuan generalisasinya dan menjadi bias terhadap kelas yang baru dimasukkan.

#### Strategi Mitigasi & Solusi di Kode:
* **Fungsi Kerugian Berbobot (*Class-Weighted Loss / Focal Loss*):** Pada skrip pelatihan Python, kelas *Crystal* diberi bobot penalti yang jauh lebih tinggi ketika salah ditebak dibanding kelas lainnya.
* **Metrik Evaluasi yang Relevan:** Pengukuran kualitas model tidak hanya melihat *overall accuracy*, melainkan memantau *Recall* dan *F1-Score* spesifik untuk kelas *Crystal*.
* **Pengambilan Sampel Berstrata (*Stratified Sampling*):** Dataset pelatihan otomatis dibagi dengan proporsi kelas yang seimbang antara data lama yang sudah terbukti valid dengan data baru hasil koreksi manual.
* **Pencegahan Regresi (*Benchmark Validation Gate*):** Model hasil pelatihan ulang harus diuji secara otomatis terhadap sekumpulan data tolok ukur (*benchmark set*). Jika skor deteksi *Crystal* pada model baru lebih rendah daripada model lama, sistem akan menolak mengaktifkan model baru tersebut dan memberi peringatan kepada pengguna.

---

### RISIKO 4: Dependensi Native pada Pipeline CI/CD (GitLab Runner)

#### Apa Masalahnya?
OpenCvSharp dan ONNX Runtime memerlukan berkas biner pustaka native C++ sistem operasi (misalnya `libonnxruntime.so` di Linux atau `.dll` di Windows). Pipeline CI/CD di GitLab Runner sering kali berjalan di dalam kontainer Docker minimalis (misal: `mcr.microsoft.com/dotnet/sdk:8.0-alpine` atau `-jammy`) yang belum tentu memiliki pustaka grafis sistem yang lengkap.

#### Potensi Bahaya:
Pengujian otomatis (*Unit Test*) di GitLab Runner gagal (*broken pipeline*) bukan karena ada bug di kode logika bisnis, melainkan karena *runtime loader* tidak menemukan dependensi native C++.

#### Strategi Mitigasi & Solusi di Kode:
* **Abstraksi Penuh dengan Interface (Clean Architecture):**  
  Seluruh operasi OpenCV dan ONNX dibungkus di balik antarmuka seperti `IInferenceEngine` dan `IImagePreprocessor`. Dengan demikian, 100% pengujian logika domain, validasi format, alur plate 96-well, dan kalkulasi confidence score dapat diuji menggunakan *Mock Object* (Moq / NSubstitute) tanpa menyentuh pustaka native sama sekali di unit test.
* **Pemisahan Kategori Pengujian:** Memisahkan pengujian menjadi dua kategori:
  1. *Unit Tests:* Ringan, berjalan instan, murni menguji logika C#, tidak butuh pustaka native, dan diwajibkan mencapai *coverage* 100%.
  2. *Integration Tests:* Menguji inferensi nyata dengan model uji berukuran kecil di lingkungan Runner yang telah dipasangi paket native.

---

## 4. Rangkuman Pertanyaan & Jawaban Cepat untuk Diskusi Mentor

| Pertanyaan Penguji / Senior Engineer | Jawaban Singkat & Tegas |
| :--- | :--- |
| **Kenapa memilih ONNX Runtime di .NET daripada panggil Python saat inferensi?** | *"Inferensi via ONNX Runtime berjalan langsung di dalam proses .NET 8 (in-process). Ini mengeliminasi latensi inter-process communication (IPC), menghemat konsumsi RAM, tidak memerlukan runtime Python pada mesin produksi, dan mempermudah deployment."* |
| **Bagaimana jika proses training Python memakan waktu 30 menit dan user menutup browser?** | *"Proses pelatihan dikelola sebagai background job di level backend .NET yang terikat dengan ID tugas. Ketika user membuka kembali aplikasi, frontend cukup melakukan query status atau terhubung kembali ke stream WebSocket untuk melihat progres yang sedang berjalan."* |
| **Bagaimana menjamin Clean Architecture benar-benar dipatuhi?** | *"Domain Layer sama sekali tidak memiliki referensi ke proyek lain atau pustaka pihak ketiga. Semua akses ke database (SQLite), sistem operasi (Python Process), dan engine AI (ONNX) berada di Infrastructure Layer dan hanya diakses melalui Interface (Dependency Inversion Principle)."* |
| **Bagaimana mencapai target 100% unit test coverage dengan adanya dependensi OpenCV/ONNX?** | *"Dengan menerapkan Strategy Pattern dan Dependency Injection, seluruh logika alur plate, scoring, filter, dan active learning dapat diuji secara terisolasi menggunakan mock/fake. Hanya lapisan tipis wrapper infrastruktur yang berinteraksi langsung dengan native binary."* |
