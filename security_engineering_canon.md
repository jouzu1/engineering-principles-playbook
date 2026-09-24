# Universal canon of security engineering and defensive architecture
## Fondasi ilmiah, hukum komputasi keamanan, standar pertahanan pentest, dan tata kelola sistem

Dokumen ini merupakan acuan universal rekayasa keamanan (security engineering) yang independen terhadap bahasa pemrograman, vendor cloud, maupun domain industri.

---

## 0. Meta-aksioma dan teorema dasar keamanan sistem

Hukum-hukum ini dirumuskan oleh perintis ilmu komputer dan kriptografi, serta berlaku mutlak di semua sistem komputasi:

### 1. Delapan prinsip desain aman Saltzer dan Schroeder (1975)
1. Economy of mechanism: rancang mekanisme keamanan sesederhana mungkin. Kompleksitas adalah musuh utama keamanan karena menyembunyikan celah dan membuat audit menjadi tidak efektif.
2. Fail-safe defaults: izin akses awal harus berupa penolakan (deny by default). Terapkan whitelist eksplisit untuk entitas yang diizinkan, hindari pendekatan blacklist.
3. Complete mediation: setiap akses ke setiap objek (file, database, endpoint, memori) harus divalidasi otorisasi dan autentikasinya pada setiap pemanggilan. Dilarang mengandalkan cache otorisasi yang tidak terverifikasi.
4. Open design (Prinsip Kerckhoffs): keamanan sistem tidak boleh bergantung pada kerahasiaan algoritma atau kode sumber. Asumsikan pihak penyerang memiliki seluruh source code dan diagram arsitektur.
5. Separation of privilege: operasi yang berisiko tinggi harus membutuhkan lebih dari satu kondisi atau kunci verifikasi (seperti otentikasi dua faktor atau persetujuan ganda).
6. Least privilege: setiap pengguna, proses, container, dan service hanya boleh beroperasi dengan hak akses minimum yang diperlukan untuk menyelesaikan tugas saat itu.
7. Least common mechanism: minimalkan penggunaan sumber daya bersama antar-pengguna yang berbeda. Terapkan isolasi data, memori, dan pool koneksi antar-tenant untuk mencegah kebocoran data silang.
8. Psychological acceptability: antarmuka keamanan harus mudah digunakan secara wajar oleh developer dan pengguna. Mekanisme yang terlalu rumit akan memicu pengguna mencari jalan pintas yang merusak keamanan.

### 2. Teorema ketidakpastian keamanan (Rice's Theorem on Security)
Secara matematis mustahil membuat program otomatis yang dapat memutuskan secara sempurna apakah suatu program arbitrer bebas dari kerentanan keamanan tanpa menghasilkan false positive atau false negative. Pertahanan harus dibangun secara berlapis di dalam kode aplikasi itu sendiri (defense in depth), bukan sekadar bergantung pada alat pemindai luar atau firewall.

### 3. Ken Thompson Hack (Reflections on Trusting Trust - 1984)
Kamu tidak dapat sepenuhnya mempercayai kode yang tidak kamu buat sendiri dari tingkat paling dasar. Compiler, build tools, dan dependensi pihak ketiga dapat disusupi untuk menyuntikkan pintu belakang (backdoor) ke dalam file biner tanpa meninggalkan jejak di source code. Keamanan rantai pasok software (supply chain) membutuhkan verifikasi kriptografis independen.

### 4. Model integritas Biba dan Clark-Wilson
* Model Biba (Integritas Data): data dari subjek atau sumber yang tingkat integritasnya lebih rendah tidak boleh mencemari state bersih sistem (no read down, no write up). Input dari client luar wajib disanitasi sebelum diolah sistem.
* Model Clark-Wilson: data hanya boleh dimodifikasi melalui transaksi yang terstruktur rapi (well-formed transactions) dan menerapkan pemisahan tugas (separation of duties). Orang yang menginput data transaksi tidak boleh menjadi orang yang menyetujui eksekusinya.

---

## 1. Application security dan defensive coding

Standar pertahanan di level kode aplikasi dan API untuk menggagalkan eksploitasi penetration testing:

### A. Broken Object-Level Authorization (BOLA / IDOR)
BOLA adalah kerentanan nomor satu pada Web API, di mana pengguna yang memiliki token valid mengakses data milik pengguna lain dengan memanipulasi ID pada URL atau payload.

Aturan baku pertahanan:
* Scope-Enforced Query: setiap query database wajib menyertakan ID akun atau ID tenant pengguna yang sedang login.
  ```sql
  -- Pola aman:
  SELECT * FROM transactions 
  WHERE id = :transaction_id 
    AND tenant_id = :auth_tenant_id 
    AND user_id = :auth_user_id;
  ```
* Gunakan format pengidentifikasi non-sekuensial (UUID v4 atau ULID) alih-alih angka integer berurutan (1, 2, 3) untuk mencegah penyerang menebak ID secara otomatis (enumeration attack).

### B. Pemisahan mutlak kode dan data (Anti-Injection)
Serangan injection (SQLi, NoSQLi, OS Command Injection, Server-Side Template Injection) terjadi ketika parser menafsirkan data dari input pengguna sebagai instruksi mesin.

Aturan baku pertahanan:
* SQL: 100% menggunakan Parameterized Queries atau Prepared Statements. Data diperlakukan murni sebagai nilai literal, bukan teks perintah SQL.
* Command execution: dilarang memanggil interpreter shell (seperti `exec("sh -c " + input)`). Jika harus menjalankan binary eksternal, gunakan eksekusi langsung dengan array parameter terpisah tanpa melibatkan shell.
* Sanitasi berbasis konteks: lakukan encoding karakter khusus sesuai tempat data tersebut ditampilkan (HTML entity encoding, URL encoding, JSON string escaping).

### C. Server-Side Request Forgery (SSRF) dan proteksi metadata cloud
SSRF terjadi saat backend mengunduh data dari URL yang dimasukkan oleh pengguna, lalu penyerang mengarahkan URL tersebut ke jaringan privat atau instance metadata service cloud (169.254.169.254) untuk mencuri kredensial server.

Aturan baku pertahanan:
* Resolusikan domain menjadi IP address di level aplikasi sebelum request dikirim.
* Tolak request jika IP hasil resolusi berada pada rentang IP privat atau loopback (RFC 1918: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.1, serta 169.254.169.254).
* Wajibkan penggunaan IMDSv2 (Instance Metadata Service v2) pada cloud provider, yang mensyaratkan token sesi berbasis HTTP header dengan batas time-to-live paket (hop-limit = 1).

### D. Standar kriptografi terapan
* Hashing password: wajib menggunakan algoritma yang memiliki resistensi terhadap komputasi paralel GPU/ASIC, yaitu Argon2id (standar utama) atau bcrypt dengan work factor memadai. Dilarang menggunakan hash cepat seperti MD5, SHA-1, SHA-256, atau SHA-512 untuk password.
* Enkripsi simetris data: gunakan algoritma Authenticated Encryption with Associated Data (AEAD), yaitu AES-256-GCM atau ChaCha20-Poly1305. AEAD menjamin kerahasiaan sekaligus integritas data, sehingga data yang dimanipulasi akan langsung ditolak saat dekripsi.
* Constant-Time comparison: semua evaluasi perbandingan token, tanda tangan HMAC, atau nilai rahasia wajib dijalankan dalam waktu konstan (constant-time) untuk mencegah timing attack nanodetik.

### E. Pencegahan Mass Assignment dan kebocoran informasi
* Terapkan Data Transfer Object (DTO) dengan aturan whitelist ketat. Jangan pernah memetakan payload request mentah langsung ke model database (mencegah manipulasi field seperti `is_admin: true`).
* Tangani error secara terpusat. Jangan pernah mengirimkan database error, stack trace, atau path direktori internal ke client dalam response HTTP 500.

---

## 2. System, container, dan software supply chain security

Standar pengerasan lingkungan eksekusi dan integritas rantai pasok software:

### A. Pengerasan runtime container
1. Rootless container: container harus berjalan menggunakan user non-root (`USER nonroot`). Penyerang yang mengeksploitasi celah aplikasi tidak boleh mendapatkan hak akses administrator di dalam container.
2. Read-only filesystem: pasang opsi `readOnlyRootFilesystem: true`. Jika aplikasi memerlukan penyimpanan sementara, sediakan volume memori terpisah (tmpfs) tanpa izin eksekusi (`noexec`).
3. Drop Linux capabilities: buang semua capability default kernel Linux dengan konfigurasi `cap-drop: [ALL]`. Berikan kembali hanya capability yang benar-benar esensial.
4. Non-escalation privilege: pastikan `allowPrivilegeEscalation: false` untuk mencegah proses anak mengambil hak akses lebih tinggi.

### B. Pengurangan attack surface dengan base image minimal
* Hindari penggunaan image Linux lengkap (seperti Ubuntu atau Debian standar) untuk menjalankan aplikasi di production.
* Gunakan Distroless image atau Chainguard. Image distroless hanya berisi runtime bahasa dan dependensi aplikasi kamu. Image ini tidak memiliki package manager, shell (bash/sh), curl, atau compiler, sehingga mematikan kemampuan penyerang untuk mengunduh dan mengeksekusi script eksploitasi.

### C. Keamanan rantai pasok software (Software Supply Chain / SLSA)
1. Software Bill of Materials (SBOM): buat dokumen inventaris seluruh komponen dan dependensi pustaka open source pada setiap proses build rilis.
2. Kunci versi dependensi: gunakan lockfile biner (`package-lock.json`, `poetry.lock`, `go.sum`) dan verifikasi cryptographic hash dari setiap paket yang diunduh.
3. Penandatanganan artefak (Cryptographic Signing): tanda tangani image container dan file biner rilis menggunakan alat seperti Sigstore Cosign. Cluster Kubernetes hanya boleh menjalankan image yang lolos verifikasi tanda tangan digital resmi.
4. Pencegahan dependency confusion: gunakan namespace privat pada private package registry untuk mencegah eksekusi paket publik palsu bertrik typosquatting.

---

## 3. Cloud, network, dan infrastructure hardening

Arsitektur isolasi jaringan dan pengelolaan kunci rahasia terpusat:

### A. Arsitektur VPC Subnet Triad
Bagi jaringan cloud menjadi tiga lapis subnet terpisah dengan aturan routing yang ketat:

```
[INTERNET PUBLIK]
       |
       v (Port 443 / TLS 1.3 / WAF / DDoS Mitigation)
+--------------------------------------------------------+
| 1. PUBLIC SUBNET (Reverse Proxy / Load Balancer)       |
+---------------------------+----------------------------+
                            | (Strict Security Group / mTLS)
                            v
+--------------------------------------------------------+
| 2. PRIVATE SUBNET (Backend API / Stateless Pods)       |
|    * Akses internet keluar hanya melalui NAT Gateway   |
|    * Tidak memiliki Public IP                          |
+---------------------------+----------------------------+
                            | (Internal Peering / Zero Internet Access)
                            v
+--------------------------------------------------------+
| 3. ISOLATED DATA SUBNET (Database / Redis / Message)   |
|    * Terisolasi total dari internet luar               |
|    * Akses database dibatasi per subnet backend        |
+--------------------------------------------------------+
```

### B. Arsitektur Envelope Encryption (KMS)
Jangan pernah menyimpan master encryption key langsung di server aplikasi atau file konfigurasi. Terapkan pola Envelope Encryption:
1. Data Encryption Key (DEK) dibuat secara lokal untuk mengenkripsi data mentah.
2. Key Encryption Key (KEK) disimpan secara aman di dalam Hardware Security Module (AWS KMS, GCP Cloud KMS, atau HashiCorp Vault).
3. DEK dienkripsi menggunakan KEK, dan server hanya menyimpan ciphertext data bersama dengan ciphertext DEK.
4. Jika database dicuri, data tidak dapat didekripsi tanpa akses sah ke API KMS.

### C. Kredensial dinamis berumur pendek (Zero Static Keys)
* Hapus penggunaan static access key jangka panjang untuk layanan cloud.
* Gunakan federasi identitas OIDC atau IAM Roles for Service Accounts (IRSA). Server aplikasi menerima token otorisasi temporer yang diperbarui secara otomatis dan kedaluwarsa dalam satu jam.

### D. Egress traffic filtering dan isolasi outbound
* Batasi koneksi keluar (egress) dari server backend. Pasang proxy firewall keluar yang hanya mengizinkan domain pihak ketiga terdaftar.
* Penyerang yang berhasil mendapatkan akses shell tidak dapat menghubungkan server kamu ke Command and Control (C2) server eksternal jika seluruh port keluar diblokir secara default.

---

## 4. Advanced identity, authentication, dan access control

Tata kelola otorisasi modern melampaui sistem role sederhana:

### A. ReBAC (Relationship-Based Access Control) dan Google Zanzibar
Pada sistem dengan skala pengguna dan entitas data yang masif, model RBAC (Role-Based) sederhana tidak lagi memadai. Terapkan model ReBAC yang memetakan otorisasi berdasarkan graf relasi antar-objek:
* Izin akses ditentukan oleh relasi: "Pengguna A dapat mengedit File B karena Pengguna A adalah anggota Grup C yang memiliki relasi editor terhadap Folder D tempat File B berada".
* Pisahkan evaluasi izin menjadi engine independen yang deterministik dan konsisten.

### B. WebAuthn dan FIDO2 (Passkeys)
* Password dan SMS OTP rentan terhadap serangan phishing, Man-in-the-Middle (reverse proxy phishing), dan pembajakan SIM card.
* FIDO2/WebAuthn menggunakan kriptografi kunci publik asimetris yang terikat secara kriptografis pada domain web (domain-bound). Kunci privat disimpan di dalam hardware secure enclave perangkat pengguna. Serangan phishing otomatis gagal karena browser menolak mengirimkan bukti otentikasi ke domain palsu.

### C. Token Binding dan DPoP (Demonstrating Proof-of-Possession)
Pada protokol OAuth 2.0 / 2.1 standar, token JWT yang dicuri dari browser atau jaringan dapat disalahgunakan oleh siapa saja (bearer token). Terapkan DPoP: setiap request API mewajibkan client membuktikan kepemilikan private key lokal melalui tanda tangan digital pada header request, sehingga token yang dicuri tidak dapat digunakan di perangkat lain.

### D. Break-Glass Procedure dan Four-Eyes Principle
* Developer tidak boleh memiliki akses langsung ke database production secara default.
* Jika terjadi insiden darurat, akses ke production harus melalui prosedur Break-Glass: akses diberikan sementara, dicatat dalam audit log permanen, dan membutuhkan persetujuan dari minimal dua pihak yang berwenang (Four-Eyes Principle).

---

## 5. Threat detection, deception, dan defense in depth

Mendeteksi keberadaan penyusup sebelum kerusakan data meluas:

### A. The Pyramid of Pain (David Bianco)
Ukuran efektivitas deteksi keamanan didasarkan pada seberapa besar kesulitan yang ditimbulkan kepada penyerang saat indikator serangannya kita netralkan:
1. Hash values (Mudah bagi penyerang untuk mengubah hash file malware).
2. IP addresses (Mudah bagi penyerang untuk mengganti proxy atau VPN).
3. Domain names (Mudah bagi penyerang untuk mendaftarkan domain baru).
4. Network / Host artifacts (Sedang).
5. Tools (Sulit bagi penyerang jika alat eksploitasi mereka terdeteksi dan diblokir).
6. Tactics, Techniques, and Procedures / TTPs (Sangat menyakitkan bagi penyerang: ketika taktik dasar mereka dalam MITRE ATT&CK dinetralkan, mereka harus mempelajari metode serangan baru dari awal).

### B. Deception Technology (Canary Tokens dan Honeypots)
* Pasang penanda jebakan (canary token) di dalam sistem: misalnya membuat tabel database bernama `backup_admin_passwords` yang tidak digunakan oleh aplikasi normal, atau menyisipkan file konfigurasi palsu yang memuat token AWS khusus.
* Begitu ada query atau ada upaya penggunaan token palsu tersebut, sistem peringatan otomatis langsung memicu alarm darurat: dapat dipastikan 100% ada pihak tidak sah yang sedang menjelajahi sistem internal.

---

## 6. Incident response, forensik digital, dan tata kelola celah

Prosedur baku saat menghadapi insiden dan penanganan hasil temuan keamanan:

### A. Siklus penanganan insiden (NIST SP 800-61)
1. Preparation: persiapkan tools, akses darurat, dan tim yang terlatih sebelum insiden terjadi.
2. Detection and analysis: identifikasi anomali, tentukan ruang lingkup insiden, dan verifikasi apakah ancaman bersifat nyata.
3. Containment: lakukan isolasi sistem yang terdampak (misalnya memutuskan akses jaringan pada pod yang terinfeksi) untuk mencegah penyebaran lateral.
4. Eradication: bersihkan komponen malware, cabut kredensial yang terpapar, dan tutup celah pintu masuk.
5. Recovery: pulihkan sistem dari backup bersih yang terverifikasi dan pantau lalu lintas data secara ketat.
6. Post-incident activity: susun dokumen post-mortem tanpa menyalahkan individu (blameless post-mortem) dan perbaiki kelemahan arsitektur agar insiden tidak terulang.

### B. Prinsip forensik dan Order of Volatility (RFC 3227)
* Locard's Exchange Principle: setiap kontak yang dilakukan penyerang di sistem pasti meninggalkan jejak digital.
* Order of Volatility: data digital memiliki tingkat kerapuhan yang berbeda terhadap kehilangan daya:
  `CPU Registers / Cache -> RAM (Memory) -> Network State / ARP Cache -> Disk Storage -> Remote Logs`
* Aturan Larangan Kritis: jangan langsung me-restart atau mematikan server yang sedang diserang. Tindakan restart menghapus seluruh data malware dan jejak koneksi penyerang yang berada di dalam RAM. Lakukan live memory acquisition atau network isolation terlebih dahulu sebelum mematikan mesin.

### C. Manajemen celah keamanan dan SLA remediasi (CVSS v3.1 / v4.0)
Setiap celah yang ditemukan melalui penetration testing atau audit wajib diberi skor Common Vulnerability Scoring System (CVSS) dan diselesaikan sesuai batasan waktu baku:

| Tingkat Keparahan | Rentang Skor CVSS | Batas Waktu Remediasi (SLA) | Contoh Kerentanan |
|---|---|---|---|
| **Critical** | 9.0 - 10.0 | Maksimal 24 - 48 Jam | Remote Code Execution (RCE) tanpa otentikasi, SQL Injection bypass login, SSRF ke cloud metadata |
| **High** | 7.0 - 8.9 | Maksimal 7 Hari Kalender | BOLA / IDOR data finansial, Stored XSS di dashboard admin, kegagalan otorisasi privilege escalation |
| **Medium** | 4.0 - 6.9 | Maksimal 30 Hari Kalender | CSRF pada aksi sensitif, kelemahan cipher TLS, rate limiting absen pada endpoint publik |
| **Low** | 0.1 - 3.9 | Masuk sprint reguler (90 Hari) | Information disclosure minor (header versi server), cookie tanpa flag SameSite |

### D. Karantina data produksi dan pemisahan lingkungan
* Dilarang keras menyalin atau mengekspor database production yang memuat data asli pengguna (PII, nomor kontak, data transaksi) ke lingkungan development atau staging.
* Lingkungan non-production wajib menggunakan data sintetis (dummy data) atau data produksi yang telah melalui proses anonymization dan data masking satu arah yang tidak dapat dipulihkan.

---

## 7. Matriks evaluasi penetration testing

| Skenario Pengujian Pentest | Teknik yang Digunakan Penyerang | Standar Kontrol Mitigasi Wajib |
|---|---|---|
| **BOLA / IDOR Testing** | Memodifikasi ID entitas pada URL atau body request | Verifikasi kepemilikan data langsung di query database (`WHERE id = :id AND user_id = :auth_user`) |
| **Authentication Brute Force** | Dictionary attack pada endpoint `/login` | Rate limit berbasis IP dan akun, algoritma hashing Argon2id, lockout progresif |
| **Session Hijacking** | Mencuri session cookie melalui XSS atau network sniffing | Cookie dengan atribut `HttpOnly; Secure; SameSite=Strict`, implementasi DPoP atau token rotation |
| **SQL / Command Injection** | Menyuntikkan karakter kutip atau operator boolean pada input | 100% Parameterized Queries, penolakan karakter berbahaya di layer validasi skema |
| **SSRF Cloud Exploitation** | Mengirimkan input URL mengarah ke `169.254.169.254` | Resolusi DNS lokal dengan filter IP privat (RFC 1918), wajibkan IMDSv2 hop-limit=1 |
| **Privilege Escalation** | Mengubah parameter peran pada JWT atau request body | Verifikasi izin peran terpusat di server, tolak parameter yang tidak terdaftar di DTO |
| **Supply Chain Poisoning** | Menyusupkan malware melalui dependensi library pihak ketiga | Kunci hash dependensi pada lockfile, scan CVE otomatis di pipeline CI dengan batas CVSS |
| **Log Forgery / Injection** | Memasukkan karakter newline (`\r\n`) untuk memalsukan data log | Structured JSON logging murni dengan sanitasi otomatis seluruh nilai string |
