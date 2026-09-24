# Threat modeling and AI agent security boundaries
## Panduan pemodelan ancaman, taksonomi STRIDE, dan arsitektur keamanan AI Agent

Dokumen ini melengkapi prinsip rekayasa perangkat lunak dengan kerangka kerja keamanan formal, pemodelan ancaman sistemik, dan arsitektur pengamanan agen AI.

---

## 1. Fondasi keamanan sistem

### A. Prinsip Kerckhoffs
Keamanan suatu sistem kriptografi atau arsitektur komputasi tidak boleh bergantung pada kerahasiaan algoritma atau kode sumbernya, melainkan hanya pada kerahasiaan kunci privat atau rahasia enkripsi. Sistem harus tetap aman meskipun seluruh diagram arsitektur dan source code diketahui publik.

### B. 10 Hukum Keamanan Tak Terubahkan (Microsoft)
1. Hukum 1: Jika penyerang dapat membujuk kamu menjalankan kodenya di komputermu, itu bukan komputermu lagi.
2. Hukum 2: Jika penyerang dapat mengubah sistem operasi di komputermu, itu bukan komputermu lagi.
3. Hukum 3: Jika penyerang memiliki akses fisik tanpa batas ke komputermu, itu bukan komputermu lagi.
4. Hukum 4: Jika kamu mengizinkan penyerang mengunggah program ke situs webmu, itu bukan situs webmu lagi.
5. Hukum 5: Kata sandi yang lemah membatalkan arsitektur keamanan yang kuat.
6. Hukum 6: Komputer hanya seaman tingkat integritas administratornya.
7. Hukum 7: Data terenkripsi hanya seaman kunci dekripsinya.
8. Hukum 8: Alat pendeteksi malware yang kedaluwarsa hanya sedikit lebih baik daripada tidak ada proteksi sama sekali.
9. Hukum 9: Anonimitas mutlak tidak realistis di jaringan komputer.
10. Hukum 10: Teknologi bukanlah obat mujarab; manusia dan prosedur tetap faktor penentu.

---

## 2. Proses 4 langkah threat modeling (Adam Shostack)

Pemodelan ancaman dilakukan pada tahap desain arsitektur sebelum koding dimulai:

```
[1. MODEL THE SYSTEM] 
         |  (Data Flow Diagram: entitas, proses, data store, trust boundary)
         v
[2. FIND THREATS]
         |  (Gunakan taksonomi STRIDE)
         v
[3. ADDRESS THREATS]
         |  (Mitigate, Eliminate, Transfer, atau Accept)
         v
[4. VALIDATE WORK]
            (Verifikasi efektivitas mitigasi pada implementasi)
```

1. Model the system: petakan aliran data menggunakan Data Flow Diagram (DFD). Identifikasi siapa aktor luar, proses apa yang berjalan, di mana data disimpan, dan di mana batas kepercayaan (trust boundary) berada.
2. Find threats: gunakan kerangka STRIDE untuk menguji setiap elemen sistem yang melintasi trust boundary.
3. Address threats: tentukan tindakan mitigasi untuk setiap ancaman yang ditemukan:
   * Mitigasi: menambahkan kontrol proteksi (seperti enkripsi atau rate limit).
   * Eliminasi: menghapus fitur berbahaya yang tidak esensial.
   * Transfer: memindahkan risiko ke penyedia spesialis (seperti payment gateway bersertifikasi PCI-DSS).
   * Terima (Accept): mendokumentasikan risiko yang dinilai rendah dengan persetujuan pimpinan bisnis.
4. Validate work: uji apakah kontrol yang diimplementasikan benar-benar menutup ancaman yang dimodelkan.

---

## 3. Klasifikasi ancaman STRIDE dan kontrol mitigasi

| Kategori Ancaman | Properti yang Dilanggar | Contoh Kasus Backend | Kontrol Mitigasi Standar |
|---|---|---|---|
| **S**poofing | Autentisitas (Authenticity) | Penyerang mengaku sebagai user lain menggunakan token palsu | Autentikasi berbasis tanda tangan kriptografis, verifikasi JWT ketat, mTLS antar-servis |
| **T**ampering | Integritas (Integrity) | Penyerang mengubah parameter harga atau saldo di request payload | Checksum payload, validasi skema ketat di controller, transaksi atomik di database |
| **R**epudiation | Non-repudiation | Pengguna menyangkal telah melakukan transfer dana | Log audit append-only yang tidak bisa diedit admin, pencatatan IP, trace_id, dan timestamp |
| **I**nformation Disclosure | Kerahasiaan (Confidentiality) | Pesan error menampilkan database connection string atau data kartu | Enkripsi data at-rest dan in-transit, sanitasi error response, larangan mencatat PII di log |
| **D**enial of Service | Ketersediaan (Availability) | Penyerang membanjiri endpoint pencarian berat dengan jutaan request | Rate limiting, circuit breaker, timeout eksplisit, isolasi queue background worker |
| **E**levation of Privilege | Otorisasi (Authorization) | User biasa mengakses endpoint admin dengan memanipulasi role di token | Role-Based Access Control (RBAC) di sisi server, validasi kepemilikan data (anti-IDOR) |

---

## 4. Keamanan AI Agent: Agent Trust Boundary Model (AakashX)

Sistem berbasis Large Language Model (LLM) bersifat probabilistik dan rentan terhadap manipulasi instruksi. Untuk membangun agen yang aman, terapkan empat batas isolasi berikut:

```
                  [USER INPUT / EXTERNAL DATA]
                               |
                   (1) INSTRUCTION BOUNDARY
                               v
                         [LLM AGENT]
                        /           \
  (2) DATA BOUNDARY    /             \    (3) TOOL BOUNDARY
                      v               v
             [CONTEXT WINDOW]    [TOOL CALL VALIDATOR]
                                      |
                           (4) ACTION BOUNDARY
                                      v
                               [CRITICAL RUNTIME]
```

### Aturan emas arsitektur AI Agent
> *"LLM may propose, the runtime must authorize."*  
> Model bahasa hanya bertugas mengusulkan rencana tindakan. Keputusan eksekusi wajib divalidasi dan diotorisasi oleh runtime kode deterministik.

### 4 Batas kepercayaan agen (Agent Trust Boundaries)

1. **Instruction Boundary:**
   Pisahkan instruksi sistem terpercaya (system instructions) dari teks masukan pengguna atau data web pihak ketiga. Anggap semua data dari luar berpotensi memuat serangan prompt injection tidak langsung. Jangan biarkan data luar mengubah aturan dasar perilaku agen.

2. **Data Boundary:**
   Batasi data yang boleh dibaca dan dimasukkan ke dalam context window model. Bersihkan informasi pribadi (PII), kredensial, dan data sensitif sebelum dikirim ke penyedia LLM eksternal. Gunakan isolasi konteks per sesi pengguna.

3. **Tool Boundary:**
   Terapkan hak akses minimal (least privilege) pada setiap tools yang disediakan untuk agen. Jangan pernah memberikan akses shell terminal bebas atau akses query SQL mentah. Setiap pemanggilan tool wajib memiliki validasi skema parameter yang ketat di runtime.

4. **Action Boundary:**
   Terapkan gerbang otorisasi ganda untuk tindakan yang berdampak permanen (seperti transfer uang, penghapusan database, atau pengiriman pesan massal). Tindakan dengan konsekuensi tinggi wajib meminta persetujuan manusia secara eksplisit (Human-in-the-Loop).

---

## 5. STRIDE-AI: Adaptasi ancaman untuk sistem AI (Tsafac et al.)

1. Model Impersonation (Spoofing): Penyerang membuat antarmuka tiruan yang meniru output agen resmi untuk memanipulasi keputusan pengguna.
2. Data dan Model Poisoning (Tampering): Penyerang menyuntikkan data berbahaya ke dalam database referensi Retrieval-Augmented Generation (RAG) atau dataset fine-tuning untuk membelokkan output agen.
3. Provenance Loss (Repudiation): Ketidakmampuan melacak rantai penalaran agen karena hilangnya log prompt, context data, dan parameter model saat tindakan dieksekusi.
4. Model Inversion dan Data Extraction (Information Disclosure): Penyerang memancing agen melalui jailbreak bertingkat untuk membocorkan data pelatihan sensitif atau system prompt rahasia.
5. Resource Exhaustion / Sponge Attacks (Denial of Service): Penyerang mengirimkan prompt kompleks yang memaksa model melakukan reasoning berlebihan atau looping pemanggilan tools tanpa henti untuk menguras kuota token dan membebani server.
6. Alignment Bypass / Jailbreak (Elevation of Privilege): Penyerang menembus batasan keamanan model menggunakan teknik persona switching atau encoding teks untuk memaksa agen menjalankan tindakan terlarang.

---

## 6. Checklist evaluasi keamanan sebelum rilis

Sebelum merilis fitur backend atau AI agent ke production, verifikasi enam pertanyaan ini:

1. Apakah seluruh input dari luar divalidasi skemanya sebelum diproses oleh business logic atau LLM?
2. Apakah otorisasi kepemilikan resource (IDOR check) diverifikasi di sisi server untuk setiap endpoint mutasi?
3. Apakah aksi berisiko tinggi pada AI agent dilindungi oleh gerbang konfirmasi manusia (Human-in-the-Loop)?
4. Apakah ada rate limit dan batas token untuk mencegah serangan eksploitasi Denial of Service dan sponge attacks?
5. Apakah data sensitif dan kredensial disaring agar tidak pernah masuk ke log atau context window publik?
6. Apakah log audit transaksi dicatat secara append-only dengan trace_id yang dapat ditelusuri end-to-end?
