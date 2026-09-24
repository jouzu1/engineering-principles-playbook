# Domain plugin guide: Menyesuaikan prinsip rekayasa ke industri spesifik

Dokumen ini menjelaskan cara menerapkan formula kerja:

```
Pola Pikir Kerja = 90% Sasis Universal (Hukum komputasi, tangga karir, acuan keputusan)
                 + 10% Plugin Domain (Regulasi industri, SLA latensi, batasan kepatuhan)
```

Dua dokumen utama (`engineering_principles_by_role.md` dan `role_system_prompts.md`) berfungsi sebagai sasis universal. Bagian ini menjelaskan cara membangun dan memasang lapisan 10% plugin untuk industri atau model bisnis tertentu.

---

## 1. Lima parameter penentu plugin industri

Setiap kali berpindah industri atau menangani produk baru, identifikasi lima parameter berikut:

### A. Toleransi kehilangan data (Data Loss Tolerance / RPO)
* RPO = 0 (Zero Data Loss): Setiap mutasi data harus tersimpan permanen di disk dan terkonfirmasi sebelum mengembalikan status sukses. Berlaku untuk transaksi uang dan pembukuan akun.
* RPO > 0 (Loss-Tolerant): Kehilangan data beberapa detik atau menit dapat diterima demi mempertahankan throughput tinggi atau latensi rendah. Berlaku untuk sensor metrik, view count, dan sinyal analytics.

### B. Target batas waktu (Latency SLA)
* Sub-10ms (Real-time murni): Memerlukan pemrosesan in-memory, protokol UDP/gRPC, dan pembatasan logging pada jalur kritis.
* 50-200ms (Web application standar): Dapat menggunakan protokol REST/HTTP dengan query database relasional yang terindeks baik.
* Asynchronous (Batch/Background): Operasi dapat memakan waktu beberapa detik hingga menit melalui sistem antrean.

### C. Model konsistensi data
* Konsistensi ketat (Strict Consistency / Serializability): Menolak anomali pembacaan data lama demi kebenaran saldo atau kuota.
* Konsistensi bertahap (Eventual Consistency): Memberikan respons cepat dan membiarkan data sinkron beberapa detik kemudian di node lain.

### D. Kepatuhan hukum dan auditabilitas
* Audit trail permanen: Log transaksi tidak boleh diubah atau dihapus oleh siapa pun, termasuk admin database.
* Data residency: Data pengguna wajib disimpan di server fisik dalam wilayah yurisdiksi tertentu.
* Hak penghapusan data: Kemampuan menghapus atau menganonimkan data pribadi pengguna sesuai regulasi perlindungan data.

### E. Karakteristik konkurensi (Contention Rate)
* Low Contention: Sumber daya jarang diakses bersamaan oleh banyak user. Cukup menggunakan optimistic locking (kolom version).
* High Contention: Ribuan user berebut satu item yang sama (flash sale, tiket konser). Memerlukan antrean terdistribusi atau partisi inventori di memory.

---

## 2. Katalog plugin per industri

### A. Plugin FinTech dan perbankan
Karakteristik:
* Mengutamakan kebenaran data mutlak di atas performa.
* Penerapan akuntansi double-entry: setiap mutasi wajib memiliki baris debit dan kredit yang seimbang.
* Log audit bersifat append-only: data audit tidak boleh dapat di-update atau di-delete.
* Kepatuhan standar PCI-DSS untuk data kartu dan regulasi otoritas moneter lokal.
* Idempotency key wajib diterapkan di seluruh endpoint transaksi.

### B. Plugin HealthTech dan data medis
Karakteristik:
* Kerahasiaan data pasien diatur oleh standar HIPAA atau regulasi privasi kesehatan.
* Enkripsi field-level: data rekam medis terenkripsi sebelum disimpan di database, sehingga admin database tidak dapat membaca data sensitif secara langsung.
* Akses data berbasis izin peran yang ketat (Role-Based Access Control). Setiap aksi membaca data medis harus tercatat dalam log akses.
* Waktu retensi data log dan rekam medis umumnya wajib disimpan antara 5 sampai 10 tahun.

### C. Plugin Gaming real-time dan simulasi
Karakteristik:
* Mengutamakan latensi rendah dan throughput tinggi dibanding konsistensi data absolut.
* Komunikasi menggunakan protokol UDP atau WebSocket biner.
* Menerapkan teknik client prediction dan server reconciliation untuk pergerakan entitas.
* Paket data posisi yang hilang di jaringan diabaikan (fire-and-forget), karena data posisi terbaru akan tiba dalam hitungan milidetik berikutnya.

### D. Plugin AdTech dan Real-Time Bidding (RTB)
Karakteristik:
* Batas waktu respons lelang iklan sangat ketat (sering kali di bawah 50 milidetik).
* Database menggunakan in-memory cache terdistribusi (Aerospike, Redis, ScyllaDB).
* Logging detail dalam format JSON dihindari pada jalur kritis. Data aktivitas dikumpulkan secara batch atau melalui sampling metrik.
* Kehilangan data klik atau impresi dalam persentase sangat kecil ditoleransi demi menjaga ketersediaan layanan.

### E. Plugin E-Commerce dan Retail berskala besar
Karakteristik:
* Menggabungkan dua model: konsistensi bertahap pada katalog produk dan pencarian, namun konsistensi ketat pada checkout dan pemotongan inventori.
* Menangani flash sale menggunakan reservasi kuota sementara di cache memory sebelum menulis transaksi permanen ke database relasional.
* Mengisolasi kegagalan sistem rekomendasi agar tidak mengganggu jalur utama transaksi pembayaran.

### F. Plugin GovTech dan infrastruktur kritis
Karakteristik:
* Penerapan lingkungan terisolasi (air-gapped) tanpa sambungan langsung ke jaringan internet publik.
* Larangan penggunaan layanan cloud pihak ketiga yang tidak tersertifikasi oleh badan keamanan negara.
* Verifikasi keamanan rantai pasok software (Software Bill of Materials / SBOM) untuk semua library pihak ketiga.

---

## 3. Template prompt plugin domain

Tambahkan blok instruksi berikut ke bagian akhir dari system prompt role yang kamu gunakan:

```markdown
## DOMAIN PLUGIN OVERLAY: [NAMA DOMAIN / INDUSTRI]
1. Target SLA Latensi: Maksimal [masukkan angka, misal: 150ms pada p99].
2. Toleransi Kehilangan Data: RPO = [masukkan nilai, misal: 0 untuk finansial / 5 menit untuk analitik].
3. Regulasi Wajib: Sistem harus mematuhi standar [misal: PCI-DSS, UU PDP, HIPAA, SOC 2].
4. Aturan Integritas Data Spesifik:
   - [Tuliskan aturan domain, contoh: Semua mutasi saldo wajib menerapkan double-entry ledger].
   - [Contoh: Data sensitif pasien wajib dienkripsi di level field].
5. Jalur Kritis vs Non-Kritis:
   - Jalur Kritis: [Sebutkan fitur yang tidak boleh gagal, misal: Pemrosesan transaksi].
   - Jalur Non-Kritis: [Sebutkan fitur yang boleh terdegradasi, misal: Riwayat notifikasi].
```
