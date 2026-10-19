# Universal canon of distributed systems resilience and failure mode engineering
## Fondasi ilmiah, hukum konsistensi, arsitektur mitigasi kegagalan, dan standar ketahanan sistem skala tinggi

Dokumen ini merupakan acuan formal rekayasa sistem terdistribusi, konsistensi data, dan pola ketahanan (resilience patterns) yang independen terhadap bahasa pemrograman, framework, maupun platform cloud.

---

## 0. Meta-aksioma dan teorema dasar sistem terdistribusi

Hukum-hukum ini berlaku mutlak pada setiap sistem yang terdiri dari lebih dari satu proses atau mesin yang berkomunikasi melalui jaringan:

### 1. Delapan kekeliruan komputasi terdistribusi (L. Peter Deutsch & James Gosling)
Setiap engineer yang merancang arsitektur microservices atau multi-node wajib menyadari bahwa asumsi-asumsi berikut adalah salah:
1. Jaringan itu andal (The network is reliable).
2. Latensi adalah nol (Latency is zero).
3. Bandwidth tidak terbatas (Bandwidth is infinite).
4. Jaringan itu aman (The network is secure).
5. Topologi jaringan tidak pernah berubah (Topology doesn't change).
6. Hanya ada satu administrator (There is one administrator).
7. Biaya transportasi data adalah nol (Transport cost is zero).
8. Jaringan bersifat homogen (The network is homogeneous).

### 2. Teorema CAP dan PACELC (Eric Brewer & Daniel Abadi)
* **CAP Theorem**: Pada sistem penyimpanan data terdistribusi yang mengalami partisi jaringan (**P** - Partition), sistem hanya dapat memilih mempertahankan Konsistensi (**C** - Consistency) ATAU Ketersediaan (**A** - Availability). Menolak request (mengorbankan A) demi data benar, atau melayani request dengan potensi data usang (mengorbankan C).
* **PACELC Theorem**: Jika terjadi Partisi (**P**), pilih antara Availability (**A**) atau Consistency (**C**); **E**lse (jika sistem berjalan normal tanpa partisi), pilih antara Latency (**L**) atau Consistency (**C**). Sistem yang menuntut konsistensi absolut di kondisi normal pasti membayar harga berupa latensi jaringan antar-node yang lebih tinggi.

### 3. Teorema ketidakmungkinan FLP (Fischer, Lynch, Paterson - 1985)
Dalam model sistem terdistribusi asinkron di mana pesan jaringan dapat tertunda tanpa batas waktu yang pasti, tidak ada algoritma konsensus deterministik yang dapat menjamin mencapai kesepakatan (consensus) jika terdapat potensi satu node mengalami crash. Oleh karena itu, semua algoritma konsensus praktis di dunia nyata (seperti Raft dan Paxos) memperkenalkan konsep timeout (partial synchrony) atau randomisasi.

---

## 1. Dual-write anomaly dan Transactional Outbox Pattern

### A. Masalah kegagalan atomisitas ganda (The Dual-Write Trap)
Pola anti-pattern paling umum dalam arsitektur modern adalah mencoba menyimpan state ke database lokal sekaligus mempublikasikan event ke message broker (seperti Kafka atau RabbitMQ) di dalam satu fungsi aplikasi:

```text
Aplikasi -> 1. UPDATE DB (Sukses)
Aplikasi -> 2. PUBLISH Event ke Message Broker (Gagal / Network Timeout)
=> AKIBAT: Data tersimpan di DB, tapi event hilang selamanya. Sistem downstream tidak pernah sinkron (Silent Financial/Data Leakage).
```
Sebaliknya, jika publish didahulukan sebelum commit DB:
```text
Aplikasi -> 1. PUBLISH Event ke Message Broker (Sukses)
Aplikasi -> 2. UPDATE DB (Gagal / Deadlock / DB Crash)
=> AKIBAT: Event terkirim ke downstream, tapi data lokal di-rollback. Downstream memproses transaksi fiktif (Phantom Transaction).
```

### B. Solusi baku: Transactional Outbox Pattern
Jadikan penulisan data entitas bisnis dan pencatatan event keluar sebagai **satu transaksi database lokal yang atomik**:

1. Tambahkan tabel `outbox` di dalam schema database yang sama dengan entitas bisnis:
   ```sql
   CREATE TABLE outbox_events (
       id VARCHAR(64) PRIMARY KEY,
       aggregate_type VARCHAR(64) NOT NULL,
       aggregate_id VARCHAR(64) NOT NULL,
       event_type VARCHAR(64) NOT NULL,
       payload JSONB NOT NULL,
       created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
       processed_at TIMESTAMP WITH TIME ZONE NULL
   );
   ```
2. Aplikasi menulis ke tabel bisnis dan tabel `outbox_events` di dalam satu blok transaksi database lokal:
   ```sql
   BEGIN TRANSACTION;
   INSERT INTO orders (id, user_id, amount) VALUES (...);
   INSERT INTO outbox_events (id, aggregate_type, aggregate_id, event_type, payload) VALUES (...);
   COMMIT;
   ```
3. Pemancaran event dilakukan oleh komponen terpisah melalui dua pendekatan:
   * **Log-Based Change Data Capture (CDC)** (Rekomendasi Utama): Menggunakan alat seperti Debezium yang membaca transaction log database (Postgres Write-Ahead Log / MySQL Binary Log) dan mem-publish event ke broker tanpa membebani query database.
   * **Polling Publisher**: Worker background terjadwal yang membaca baris `processed_at IS NULL`, mengirimkan event ke broker, lalu menandai baris tersebut sebagai terproses.

---

## 2. Thundering herd, retry storm, dan Exponential Backoff dengan Full Jitter

### A. Bahaya retry serentak (Thundering Herd)
Ketika sebuah dependensi atau service hilir (downstream) mengalami downtime sesaat, ribuan client yang gagal akan mencoba mengirim ulang request secara otomatis. Jika retry dilakukan dengan interval konstan atau exponential backoff biasa tanpa randomisasi, seluruh client akan melakukan retry pada gelombang waktu yang persis sama. Gelombang serentak ini (*retry storm*) akan langsung merobohkan service hilir yang baru saja mulai pulih.

### B. Algoritma Exponential Backoff dengan Full Jitter
Sesuai riset Sally Floyd & Van Jacobson serta AWS Architecture Lab, pendekatan terbaik untuk meratakan beban retry adalah **Full Jitter**:

```text
Interval Maksimum:
sleep_cap = min(max_backoff, base_backoff * (2 ^ attempt))

Interval Aktual:
actual_sleep = random_between(0, sleep_cap)
```

Perbandingan perilaku:
* **No Jitter**: Seluruh client memukul server di detik ke-1, 2, 4, 8, 16 bersamaan (puncak traffic sangat tinggi).
* **Full Jitter**: Beban terdistribusi merata sepanjang garis waktu, menghilangkan puncak spike dan memberikan ruang bagi downstream untuk pulih stabil.

### C. Batasan retry budget
* Terapkan batas maksimal percobaan (maksimal 3 sampai 5 kali).
* Gunakan *Retry Budget* (misal: alokasikan maksimal 10% dari total bandwidth request untuk retry). Jika rasio retry melebihi 10%, tolak retry langsung dengan status error agar tidak memperburuk kondisi sistem yang sedang overload.

---

## 3. Backpressure, load shedding, dan saturasi antrean

### A. Ilusi antrean tanpa batas (The Unbounded Queue Trap)
Mengizinkan antrean request atau message masuk membengkak tanpa batas di dalam memori server adalah resep pasti menuju bencana:
1. Latensi meroket karena request terjebak mengantre selama puluhan detik sebelum dieksekusi.
2. Ketika request akhirnya diproses, client atau upstream HTTP gateway sudah melakukan timeout dan memutus koneksi. Server membuang kapasitas CPU berharga untuk memproses request basi (*wasted work*).
3. Memori server habis dan memicu crash Out-Of-Memory (OOM) atau jeda Garbage Collection (GC) yang sangat panjang.

### B. Kontrol saturasi: Load Shedding proaktif
Ketika utilisasi sistem mendekati batas aman (> 80% kapasitas sesuai Little's Law):
* **Fail Fast**: Tolak request baru yang masuk dengan status HTTP 429 Too Many Requests atau HTTP 503 Service Unavailable, disertai header `Retry-After`.
* **Prioritas Beban (Shedding Tier)**:
  - *Tier 1 (Kritis)*: Pemrosesan pembayaran dan mutasi data inti tetap diproses.
  - *Tier 2 (Sekunder)*: Fitur rekomendasi, pencarian lanjutan, atau logging detail dinonaktifkan sementara (*graceful degradation*).
  - *Tier 3 (Background)*: Job analitik dan pengiriman email ditunda sepenuhnya.
* **Controlled Delay (CoDel)**: Pantau waktu tunggu request di dalam antrean antarmuka. Jika waktu tunggu di antrean melebihi threshold tertentu (misal: > 50ms), buang request tersebut di pintu masuk sebelum menyentuh worker thread pool.

---

## 4. Saga Pattern untuk transaksi terdistribusi

### A. Mengapa Two-Phase Commit (2PC) dihindari di cloud
Two-Phase Commit (2PC) membutuhkan koordinator terpusat dan mengunci sumber daya (*distributed lock*) di seluruh database partisipan sampai fase voting dan commit selesai:
* Ketersediaan sistem adalah perkalian ketersediaan tiap node ($A_{total} = A_1 \times A_2 \times ... \times A_n$). Jika satu node lambat atau hang, seluruh transaksi menggantung dan mengunci database lain.
* Menyebabkan bottleneck latensi ekstrem dan risiko kebuntuan (*distributed deadlock*).

### B. Prinsip kerja Saga Pattern
Saga memecah transaksi bisnis global menjadi serangkaian transaksi lokal independen pada setiap servis. Setiap langkah memperbarui database lokal dan menerbitkan event untuk memicu langkah berikutnya:

```text
[Langkah 1: Servis Order] -> Buat Order (PENDING)
       |
[Langkah 2: Servis Payment] -> Potong Saldo (SUKSES)
       |
[Langkah 3: Servis Inventory] -> Potong Stok (GAGAL: Stok Habis!)
       |
       v (Memicu Transaksi Kompensasi Balik)
[Kompensasi: Servis Payment] -> Refund Saldo Pengguna
       |
[Kompensasi: Servis Order] -> Ubah Status Order menjadi CANCELLED
```

### C. Dua model implementasi Saga
1. **Choreography (Koreografi Event)**:
   * Setiap servis mendengarkan event dari servis lain dan memutuskan tindakannya sendiri secara independen.
   * Cocok untuk alur sederhana (2 sampai 4 langkah).
   * Risiko: Alur bisnis sulit dipantau jika langkah transaksi semakin banyak (spaghetti event loop).
2. **Orchestration (Orkestrasi Terpusat)**:
   * Satu servis koordinator (Saga Orchestrator) secara eksplisit mengirimkan perintah ke setiap partisipan dan mengelola state machine transaksi.
   * Sangat direkomendasikan untuk transaksi kompleks dengan banyak skenario kompensasi.
   * Memberikan kejelasan audit mengenai posisi status transaksi yang sedang berjalan.

### D. Aksi kompensasi wajib idempoten
Transaksi kompensasi (seperti pengembalian saldo atau pembatalan reservasi tiket) wajib dirancang **idempoten dan kebal terhadap network retry**, karena pesan kompensasi dapat dikirim ulang berkali-kali oleh koordinator.

---

## 5. Topologi Read-Replica dan Read-After-Write Consistency

### A. Masalah kelambatan replikasi (Replication Lag Anomaly)
Pada arsitektur database modern, satu primary node menangani penulisan data (Write), sementara beberapa replica node menangani pembacaan data (Read) secara asinkron. Terdapat jeda waktu replikasi (*replication lag*) antara milidetik hingga beberapa detik.

Skenario kegagalan umum:
1. User mengupdate profil atau membuat pesanan baru (tulis ke Primary).
2. Halaman aplikasi langsung me-refresh dan memanggil endpoint query data (baca dari Replica).
3. Karena data belum selesai disalin ke Replica, halaman menampilkan data lama atau error 404 (Entitas Tidak Ditemukan).
4. User mengira aksinya gagal dan menekan tombol submit berkali-kali.

### B. Strategi mitigasi: Read-Your-Writes Consistency
1. **Sticky User Routing**: Setelah user melakukan mutasi data (POST / PUT / PATCH / DELETE), tandai sesi user tersebut (misal via cookie atau token cache) untuk mengarahkan seluruh pembacaan data user tersebut langsung ke database Primary selama jendela waktu tertentu (misal: 3 sampai 5 detik).
2. **Version / Timestamp Vector**: Kirimkan versi data atau timestamp penulisan terakhir ke client. Client menyertakan token ini pada request pembacaan; proxy database hanya mengarahkan query ke replica yang telah mengonfirmasi bahwa replikasi log-nya telah melampaui timestamp tersebut.

---

## 6. Konsensus terdistribusi, kuorum, dan pencegahan Split-Brain

### A. Aturan kuorum mayoritas (Majority Quorum Rule)
Untuk mencegah terjadinya *split-brain* (kondisi di mana partisi jaringan membagi cluster menjadi dua kelompok yang masing-masing mengklaim dirinya sebagai pemimpin resmi), sistem konsensus (Raft, Paxos, Zookeeper) mewajibkan aturan kuorum:

$$Q = \left\lfloor \frac{N}{2} \right\rfloor + 1$$

Di mana $N$ adalah jumlah total node dalam cluster.
* Cluster 3 node membutuhkan minimal 2 node untuk membentuk keputusan. Toleransi kegagalan: 1 node.
* Cluster 5 node membutuhkan minimal 3 node untuk membentuk keputusan. Toleransi kegagalan: 2 node.
* **Aturan Baku**: Jumlah node cluster penyedia konsensus wajib berjumlah **ganjil** (3, 5, atau 7). Menambahkan node genap (misal dari 3 menjadi 4) menaikkan overhead jaringan tanpa meningkatkan toleransi kegagalan (keduanya sama-sama hanya mentoleransi 1 node crash).

### B. Fencing token pada distributed lock
Jangan pernah mempercayai distributed lock (seperti Redis Redlock) tanpa mekanisme proteksi pemisah (*fencing token*):
1. Client A memperoleh lock dengan waktu sewa 10 detik.
2. Client A mengalami jeda komputasi panjang tak terduga (misal: Full Garbage Collection Pause selama 15 detik).
3. Waktu sewa lock habis; lock dialokasikan ke Client B.
4. Client A bangun dari jeda dan mengira masih memegang lock, lalu menulis ke penyimpanan bersamaan dengan Client B -> Terjadi korupsi data.

**Solusi**: Server lock wajib menerbitkan angka urut monotonik yang selalu bertambah (*fencing token*, misal: token = 34). Penyimpanan akhir (database / file store) wajib menolak penulisan jika token yang dibawa lebih kecil dari token transaksi terakhir yang berhasil diproses.

---

## 7. Matriks failure mode dan penangkal produksi (3 AM Incident Cheat Sheet)

| Gejala Insiden Produksi | Penyebab Utama di Lapangan | Penangkal Arsitektur Baku (Antidote) |
|---|---|---|
| **Downstream crash berulang saat baru restart** | Retry storm serentak dari ribuan client (Thundering Herd) | Terapkan Exponential Backoff dengan Full Jitter dan pasang Circuit Breaker |
| **Data di database tersimpan, tapi event Kafka hilang** | Dual-write anomaly (menulis DB dan mem-publish broker secara terpisah) | Wajibkan Transactional Outbox Pattern atau CDC (Debezium) |
| **User komplain data yang baru diinput hilang saat refresh** | Replikasi asinkron lambat pada Read-Replica (Replication lag) | Arahkan pembacaan pasca-mutasi ke Primary DB (Read-Your-Writes sticky session) |
| **Server kehabisan memori (OOM) saat traffic memuncak** | Antrean request memori tanpa batas (Unbounded Queue) | Pasang bounded queue dan aktifkan Load Shedding proaktif (HTTP 429/503) |
| **Transaksi lintas servis menggantung dan mengunci tabel** | Penggunaan Two-Phase Commit (2PC) atau ketiadaan timeout | Ganti dengan Saga Pattern orkestrasi dan aksi kompensasi teruji |
| **Dua transaksi memotong saldo atau kuota yang sama** | Read-modify-write di memori aplikasi tanpa isolasi DB | Gunakan atomic conditional update di DB atau SELECT FOR UPDATE eksplisit |
| **Proses background menulis data lama menimpa data baru** | Jeda GC panjang membuat distributed lock kadaluwarsa | Terapkan Fencing Token bertingkat (monotonically increasing version) |
