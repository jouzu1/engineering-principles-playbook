# Universal canon of software engineering and technical leadership
## Fondasi filosofis, hukum komputasi, dan prinsip baku berdasarkan tangga peran (career ladder)

### 0. Meta-prinsip: sifat rekayasa sistem

Hukum-hukum ini berlaku umum bagi siapa saja yang merancang software dan sistem komputasi:

1. Tesler's Law (Conservation of complexity): Setiap sistem memiliki kompleksitas bawaan minimum yang tidak dapat dihilangkan. Kompleksitas tersebut hanya dapat dipindahkan: diserap oleh pengguna, diserap oleh kode aplikasi, atau diserap oleh infrastruktur.
2. Fred Brooks' No Silver Bullet: Tidak ada satu pun teknologi, bahasa pemrograman, atau metodologi manajemen yang menjanjikan peningkatan produktivitas atau keandalan sebesar sepuluh kali lipat dalam waktu singkat.
3. Halting problem dan ketidaklengkapan (Turing dan Godel): Tidak semua perilaku program dapat diprediksi secara mekanis atau dibuktikan kebenarannya di dalam satu sistem tertutup. Analisis otomatis selalu menghadapi kompromi antara false positive dan false negative.
4. Hukum kedua dinamika software (Software entropy): Sistem software yang terus dimodifikasi akan mengalami peningkatan ketidakteraturan, kecuali ada alokasi kerja terencana untuk menyederhanakan arsitektur dan membersihkan technical debt.

---

### Track 1: Individual Contributor (IC Track)

```
[FELLOW / SR. FELLOW]        -> Pengaruh: Industri global dan fondasi komputasi
       ^
[DISTINGUISHED ENGINEER]     -> Pengaruh: Seluruh perusahaan dan standar eksternal
       ^
[PRINCIPAL ENGINEER]         -> Pengaruh: 1 Organisasi besar / Business unit
       ^
[STAFF / SR. STAFF]          -> Pengaruh: Multi-tim / Domain lintas skuad
       ^
[SENIOR ENGINEER]            -> Pengaruh: 1 Tim dan 1 sistem domain
       ^
[MID-LEVEL ENGINEER]         -> Pengaruh: 1 Fitur penuh (End-to-end delivery)
       ^
[JUNIOR / ASSOCIATE]         -> Pengaruh: 1 Task / 1 Unit kerja terisolasi
```

#### 1. Junior / Associate Engineer
Scope of influence: 1 unit kerja terdefinisi.  
Ukuran keberhasilan: ketepatan implementasi instruksi, determinisme kode, dan kecepatan belajar.

Mental model:
* Determinisme eksekusi: kode yang baik adalah kode yang perilakunya dapat diprediksi saat dijalankan dengan berbagai variasi input.
* Kerendahan hati kognitif: menyadari batas pengetahuan diri tentang edge cases sistem, lalu bertanya atau memverifikasi asumsi sebelum menulis implementasi panjang.

Prinsip operasional:
1. Validasi boundary zero-trust: periksa semua data yang masuk di layer controller (tipe data, format, dan batas ukuran payload). Anggap data dari client selalu berisiko sampai divalidasi.
2. KISS dan YAGNI: buat implementasi yang hanya menjawab kebutuhan tiket saat ini. Jangan menambahkan fungsi spekulatif untuk kebutuhan yang belum diminta.
3. Enkapsulasi: fungsi luar tidak perlu tahu implementasi internal suatu modul. Jaga data privat tetap tertutup.
4. Penanganan error eksplisit: dilarang menulis blok catch kosong. Tangani error secara terencana atau teruskan ke handler terpusat.
5. Pengujian berbasis perilaku: unit test harus memverifikasi output berdasarkan variasi input (termasuk input kosong, null, atau nilai batas), bukan menguji variabel internal.

Pantangan:
* Menyalin kode dari luar tanpa memahami alurnya baris per baris.
* Hanya menguji skenario sukses (happy path bias).
* Diam berjam-jam saat terhalang masalah tanpa meminta arahan setelah batas waktu satu sampai dua jam.

#### 2. Mid-Level Engineer
Scope of influence: 1 fitur penuh (lifecycle end-to-end).  
Ukuran keberhasilan: kemandirian pengiriman fitur dari skema database hingga integrasi API, dengan regresi minimal.

Mental model:
* Pemisahan state dan logika: komputasi stateless mudah diskalakan, sedangkan state persisten di database atau cache adalah sumber utama masalah konkurensi.
* Otonomi eksekusi: memecah kebutuhan bisnis menjadi skema database yang rapi, kontrak API yang konsisten, dan kode yang siap jalan di production.

Prinsip operasional:
1. Idempotensi: mutasi data (terutama transaksi finansial dan pembuatan entitas) harus kebal terhadap pengiriman ganda akibat network retry. Terapkan database unique constraint atau idempotency key.
2. Disiplin akses database: eliminasi masalah N+1 query. Setiap query filter, join, dan sorting harus memiliki database index yang terverifikasi melalui query planner. Hindari pemanggilan kolom yang tidak digunakan.
3. Isolasi konkurensi: mutasi data sensitif seperti saldo atau stok wajib diselesaikan secara atomik di level database (misal: query update berkondisi) atau locking eksplisit. Jangan menghitung nilai mutasi di memori aplikasi.
4. Batas timeout wajib: setiap pemanggilan I/O keluar (database pool, HTTP client, API pihak ketiga) wajib memiliki batas timeout eksplisit.
5. Kompatibilitas kontrak API: pertahankan kompatibilitas field response yang masih digunakan oleh client aktif.

Pantangan:
* Membiarkan skema internal database bocor langsung ke antarmuka response JSON publik.
* Mengabaikan kemungkinan adanya dua request konkuren yang tiba pada waktu bersamaan.
* Membangun framework buatan sendiri di dalam aplikasi untuk fitur yang use case-nya baru ada satu.

#### 3. Senior Engineer
Scope of influence: 1 tim dan 1 sistem domain.  
Ukuran keberhasilan: uptime sistem, efisiensi review kode, mitigasi insiden, dan peningkatan kapabilitas tim.

Mental model:
* Desain berbasis kegagalan: sistem dirancang dengan asumsi bahwa setiap komponen atau dependensi luar dapat mengalami gangguan sewaktu-waktu.
* Operasional jangka panjang: kualitas kode dinilai dari kemudahan pemeliharaan, kejelasan observabilitas, dan kemudahan mitigasi saat terjadi insiden.

Prinsip operasional:
1. Realitas sistem terdistribusi: jaringan selalu memiliki latensi, bandwidth terbatas, dan risiko kegagalan transmisi.
2. Teori antrean dan kapasitas (Little's Law): utilisasi sistem harus dijaga di bawah 80%. Waktu proses yang naik sedikit akan melipatgandakan antrean secara eksponensial.
3. Pengendalian blast radius: kegagalan satu sub-sistem tidak boleh menumbangkan sistem inti. Terapkan circuit breaker, isolasi bulkhead, dan response cadangan.
4. Standar observabilitas: gunakan structured JSON logging dengan trace_id, pantau metrik berbasis rate, error, duration (RED), serta pisahkan healthcheck liveness dan readiness.
5. Pola expand-contract: migrasi database tanpa downtime dilakukan bertahap (tambah skema baru, dual-write, migrasi data lama, alihkan pembacaan, hapus skema lama).
6. Prioritas teknologi stabil: gunakan teknologi dan library yang sudah teruji pola kegagalannya di lingkungan produksi dibanding teknologi baru yang belum matang.

Pantangan:
* Menangani insiden sendirian tanpa menulis post-mortem dan tanpa membagikan perbaikan pencegahannya ke tim.
* Menggunakan review PR untuk memaksakan gaya sintaks personal daripada fokus pada kebenaran logika, keamanan, dan performa.

#### 4. Staff Engineer / Senior Staff
Scope of influence: multi-tim dan domain teknis lintas skuad.  
Ukuran keberhasilan: percepatan delivery lintas tim, standarisasi arsitektur, dan pengurangan duplikasi sistemik.

Mental model:
* Pengaruh tidak langsung: dampak kerja diukur dari peningkatan kecepatan dan keselamatan kerja puluhan engineer lain melalui standar dan blueprint yang dirancang.
* Penyelesaian akar sistemik: insiden dianalisis hingga ke level proses kerja, batasan organisasi, atau celah arsitektur yang memungkinkan insiden tersebut terjadi.

Prinsip operasional:
1. Conway's Law: struktur sistem software mengikuti jalur komunikasi organisasi. Susun batas servis sesuai batas kepemilikan tim agar tidak terjadi konflik dependensi berkelanjutan.
2. Gall's Law: sistem kompleks yang berhasil selalu berevolusi dari sistem sederhana yang berjalan dengan baik. Tolak perancangan arsitektur terdistribusi rumit dari nol sebelum model dasarnya terbukti.
3. Reversibilitas keputusan: bedakan keputusan yang sulit dibatalkan (one-way doors) dan keputusan yang mudah dibatalkan (two-way doors). Keputusan one-way doors membutuhkan kajian tertulis mendalam melalui RFC. Keputusan two-way doors diselesaikan dengan cepat.
4. Budaya dokumentasi teknis: perubahan arsitektur lintas tim wajib memiliki catatan Architecture Decision Record (ADR) yang memuat konteks, alternatif yang ditolak, dan konsekuensi operasional.
5. Pencegahan overengineering: tolak adopsi teknologi kompleks yang tidak didukung oleh kebutuhan skala nyata.

Pantangan:
* Menghasilkan dokumen arsitektur tanpa memahami kendala teknis implementasi di lapangan.
* Memecah monolit menjadi puluhan service tanpa justifikasi beban kerja atau tanpa kejelasan kepemilikan tim.

#### 5. Principal Engineer
Scope of influence: 1 organisasi besar atau business unit.  
Ukuran keberhasilan: keberlanjutan strategi teknologi jangka panjang, efisiensi biaya infrastruktur (FinOps), dan mitigasi risiko kelangsungan bisnis.

Mental model:
* Penyelarasan komputasi dan bisnis: menerjemahkan target pertumbuhan perusahaan menjadi batasan teknis yang terukur dan realistis.
* Efisiensi modal: performa sistem diukur dari biaya komputasi per unit transaksi bisnis dan dampaknya terhadap margin perusahaan.

Prinsip operasional:
1. Batasan konsistensi (CAP dan PACELC): tetapkan trade-off konsistensi data secara sadar. Tentukan bagian yang memerlukan konsistensi absolut dan bagian yang dapat menggunakan konsistensi bertahap demi latensi serta ketersediaan.
2. Waktu kausal di sistem terdistribusi: hindari ketergantungan pada jam server untuk pengurutan transaksi kritis. Gunakan kausalitas logis atau infrastruktur waktu khusus.
3. Praktik FinOps: audit efisiensi penggunaan sumber daya cloud secara berkala. Pastikan biaya infrastruktur proporsional terhadap nilai bisnis yang diproses.
4. Verifikasi ketahanan bencana: jamin pencapaian Recovery Point Objective (RPO) dan Recovery Time Objective (RTO) melalui pengujian failover terencana dan simulasi kegagalan.
5. Perlindungan metrik (Goodhart's Law): awasi agar metrik teknis tidak dimanipulasi menjadi target penilaian personal yang merusak kualitas rekayasa.

Pantangan:
* Mempertahankan sistem lama yang boros biaya hanya karena enggan menghapus investasi masa lalu (sunk cost fallacy).
* Membangun platform internal berskala besar tanpa membandingkan biaya dan manfaatnya terhadap solusi yang sudah ada di pasar.

#### 6. Distinguished Engineer
Scope of influence: seluruh perusahaan dan standar eksternal industri.  
Ukuran keberhasilan: penyelesaian masalah komputasi yang belum memiliki solusi standar di pasar, serta perumusan arah riset institusi.

Mental model:
* Pendekatan first-principles: membedah masalah komputasi hingga ke batas fisika perangkat keras: alokasi memori, hierarki cache, dan batas transmisi data.
* Penasihat independen korporat: memberikan masukan teknis objektif kepada jajaran direksi untuk mencegah kekeliruan arsitektur yang mengancam kelangsungan bisnis.

Prinsip operasional:
1. Batas kompleksitas komputasi: mengenali masalah yang tidak dapat diselesaikan secara eksak dalam waktu singkat (NP-Hard), lalu mengarahkan penggunaan algoritma aproksimasi atau heuristik.
2. Batas teori informasi: menerapkan batasan kompresi data, kapasitas kanal, dan pemrosesan throughput pada skala data perusahaan.
3. Ketahanan konsensus ekstrem: merancang arsitektur yang tetap beroperasi meskipun terjadi manipulasi data atau kegagalan node yang tidak terduga.
4. Partisipasi standar terbuka: membawa solusi internal yang matang ke dalam forum standarisasi industri terbuka untuk memperluas ekosistem.

Pantangan:
* Menjalankan riset yang terisolasi dari kebutuhan strategis dan daya saing perusahaan.

#### 7. Fellow / Senior Fellow
Scope of influence: industri komputasi global dan sejarah rekayasa perangkat lunak.  
Ukuran keberhasilan: perumusan paradigma baru yang menjadi fondasi bagi ekosistem software dunia.

Mental model:
* Perintis fondasi komputasi: merancang sistem operasi, model data, atau algoritma yang menjadi acuan standar lintas generasi perangkat keras.
* Penyederhanaan skala masif: mengubah masalah komputasi yang rumit menjadi abstraksi universal yang mudah digunakan.

Prinsip operasional:
1. Ketelitian ilmiah dan pembuktian matematis: fondasi sistem dibangun di atas logika formal yang terbukti kebenarannya.
2. Nilai utilitas universal: menciptakan karya teknologi yang bermanfaat bagi komunitas rekayasa global secara berkelanjutan.

---

### Track 2: Management Track

```
[CHIEF TECHNOLOGY OFFICER (CTO)]   -> Pengaruh: Kelangsungan perusahaan dan posisi pasar
       ^
[VP OF ENGINEERING (VPE)]          -> Pengaruh: Seluruh organisasi engineering
       ^
[DIRECTOR OF ENGINEERING]          -> Pengaruh: Departemen / Multi-tribes
       ^
[ENGINEERING MANAGER (EM)]         -> Pengaruh: Manusia, karir, dan eksekusi skuad
       ^
[TECH LEAD / TEAM LEAD]            -> Pengaruh: Eksekusi delivery 1 skuad
```

#### 1. Tech Lead / Team Lead
Scope: 1 skuad pengembang.  
Fokus: kelancaran delivery sprint, pemecahan tugas, dan penanganan hambatan harian.

Prinsip operasional:
1. Brooks' Law: penambahan anggota tim ke dalam proyek yang sedang terlambat akan memperlambat penyelesaian proyek. Tangani keterlambatan dengan memangkas cakupan fitur.
2. Pembatasan Work In Progress (WIP): batasi jumlah tiket yang dikerjakan secara paralel dalam satu waktu agar tim fokus menyelesaikan pekerjaan yang sudah dimulai.
3. Psychological safety: ciptakan lingkungan kerja yang terbuka agar kendala teknis dan kesalahan kode dilaporkan sedini mungkin tanpa rasa takut.
4. Dekomposisi tugas atomik: pecah kebutuhan besar menjadi tugas teknis independen yang dapat diverifikasi dalam satu hingga dua hari kerja.

#### 2. Engineering Manager (EM)
Scope: pembinaan manusia, proses, dan eksekusi pada 1 sampai 2 skuad.  
Fokus: pengembangan talenta, pengelolaan kapasitas, dan evaluasi kinerja yang adil.

Prinsip operasional:
1. Managerial leverage (Andy Grove): keberhasilan manajer diukur dari total output tim yang dipimpin dan dipengaruhinya.
2. Umpan balik langsung (Radical Candor): sampaikan apresiasi dan koreksi secara spesifik dan tepat waktu tanpa menunda evaluasi kritis.
3. Disiplin alokasi kapasitas (aturan 70-20-10): alokasikan 70% kapasitas untuk delivery fitur bisnis, 20% untuk perbaikan technical debt dan infrastruktur, serta 10% untuk eksplorasi teknis.
4. Kalibrasi kompetensi: bimbing anggota tim yang belum menyadari batas kemampuannya, dan dukung anggota tim handal yang mengalami sindrom imposter.

#### 3. Director of Engineering
Scope: multi-departemen (manager of managers).  
Fokus: penyelarasan lintas fungsi, restrukturisasi organisasi, dan kepemimpinan manajerial.

Prinsip operasional:
1. Inverse Conway Maneuver: sesuaikan struktur organisasi tim terlebih dahulu untuk membentuk arsitektur software yang modular.
2. Batas komunikasi organisasi (Dunbar's Number): pecah kelompok kerja yang melampaui batas komunikasi efektif (di atas 50 hingga 100 orang) menjadi unit-unit otonom dengan batas tanggung jawab yang tegas.
3. Arbitrase produk dan teknologi: seimbangkan target kecepatan peluncuran fitur dengan kebutuhan pemeliharaan stabilitas jangka panjang.
4. Rubrik karier transparan: bangun standar promosi dan evaluasi karier yang didasarkan pada dampak nyata terhadap sistem dan bisnis.

#### 4. VP of Engineering (VPE)
Scope: seluruh divisi rekayasa dan operasional teknologi.  
Fokus: kecepatan organisasi, kepatuhan hukum, tata kelola keamanan, dan efisiensi operasional.

Prinsip operasional:
1. Metrik efisiensi DORA: pantau efektivitas divisi engineering melalui frekuensi deployment, lead time for changes, change failure rate, dan waktu pemulihan insiden (MTTR).
2. Manajemen risiko vendor: pantau ketergantungan pada penyedia cloud dan third-party kritis, serta siapkan rencana mitigasi jika terjadi eskalasi biaya.
3. Tata kelola keamanan enterprise: pastikan kepatuhan terhadap standar regulasi hukum dan perlindungan data pribadi sebagai bentuk proteksi institusional.
4. Penanganan insiden tanpa kompromi: terapkan proses audit pasca-insiden yang mendalam untuk menutup celah kelemahan operasional.

#### 5. Chief Technology Officer (CTO)
Scope: dewan direksi, posisi pasar, dan pemegang saham.  
Fokus: visi teknologi jangka panjang, keunggulan kompetitif, dan alokasi modal R&D.

Prinsip operasional:
1. Teknologi sebagai pembeda kompetitif: bangun sendiri teknologi yang menjadi inti keunggulan produk di pasar. Untuk fungsi pendukung umum, gunakan solusi yang sudah tersedia di pasar.
2. Tanggung jawab alokasi modal: pertanggungjawabkan setiap pengeluaran biaya rekayasa dan infrastruktur terhadap nilai bisnis dan valuasi perusahaan.
3. Radar disrupsi teknologi: amati arah perkembangan teknologi tiga sampai lima tahun ke depan dan lakukan pembaruan sistem sebelum didahului oleh kompetitor.
4. Komunikasi dewan direksi: sampaikan risiko dan pencapaian teknologi menggunakan bahasa finansial, dampak reputasi, dan kepatuhan hukum.

---

### Matriks ringkasan peran

| Role / Level | Scope of Influence | Hukum / Prinsip Utama | Metrik Keberhasilan Terpenting | Pantangan Terbesar |
|---|---|---|---|---|
| Junior IC | 1 Task | Zero-trust boundary, KISS | Kebenaran logika kode, kelulusan test | Diam berjam-jam saat terhalang masalah, salin kode tanpa paham |
| Mid IC | 1 Fitur penuh | Idempotensi, isolasi konkurensi | Pengiriman fitur otonom tanpa regresi | Kebocoran skema internal, overengineering prematur |
| Senior IC | 1 Tim / 1 Sistem | Fallacies of dist. computing, Little's Law | Uptime sistem, MTTR, kualitas review kode | Menangani insiden sendirian tanpa dokumentasi, dogmatisme gaya kode |
| Staff IC | Multi-tim | Conway's Law, Gall's Law, Reversibilitas | Kecepatan delivery lintas tim, adopsi ADR | Desain arsitektur menara gading, pemecahan servis prematur |
| Principal IC | 1 Business unit | CAP/PACELC, waktu kausal, FinOps | Efisiensi biaya komputasi, ketahanan bencana | Terjebak sunk-cost fallacy, membangun platform tanpa kalkulasi biaya |
| Distinguished | Seluruh korporasi | Kompleksitas komputasi, teori informasi | Penyelesaian masalah tak berstandar | Riset yang tidak terhubung dengan daya saing bisnis |
| Fellow | Seluruh dunia | Paradigma komputasi baru | Karya teknologi yang menjadi standar global | Dogmatisme akademis tertutup |
| Tech Lead | 1 Skuad | Brooks' Law, WIP limits, Safety | Kelancaran delivery sprint, penghapusan blocker | Menambah orang pada proyek yang terlambat |
| Eng Manager | 1-2 Skuad (People)| Managerial leverage, Radical candor | Retensi talenta, kepatuhan alokasi 20% tech-debt | Menunda pemberian evaluasi kritis performa |
| Director | Departemen | Inverse Conway, batas Dunbar | Keseimbangan output produk dan stabilitas | Membiarkan birokrasi silo antar-divisi |
| VPE | Organisasi Eng | DORA metrics, keamanan enterprise | Kecepatan rekayasa, kepatuhan audit regulasi | Mengorbankan standar keamanan demi kecepatan rilis |
| CTO | Pasar dan dewan | Alokasi modal, teknologi pembeda | Nilai valuasi bisnis, pertahanan kompetitif | Mengadopsi teknologi baru tanpa model bisnis yang jelas |
