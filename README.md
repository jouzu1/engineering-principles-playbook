# Engineering principles playbook

Playbook ini berisi kompendium prinsip rekayasa perangkat lunak, hukum komputasi, tangga karir engineering, kumpulan system prompt per role, serta panduan adaptasi domain industri.

## Formula kerja

```
Pola Pikir Kerja = 90% Sasis Universal (Hukum komputasi, cara ambil keputusan, tangga karir)
                 + 10% Plugin Domain (Regulasi industri, SLA latensi, batasan kepatuhan)
```

Kerangka kerja ini memisahkan hal-hal yang berlaku abadi di semua sistem komputasi (90% sasis universal) dari aturan spesifik yang dipaksakan oleh karakteristik bisnis atau regulator (10% plugin domain).

---

## Struktur dokumen

1. [engineering_principles_by_role.md](engineering_principles_by_role.md)
   Fondasi filosofis, hukum komputasi (Little's Law, Amdahl's Law, CAP, Conway's Law, Gall's Law), batas arsitektur, dan prinsip operasional baku dari level Junior sampai Fellow di IC Track, serta Tech Lead sampai CTO di Management Track.

2. [role_system_prompts.md](role_system_prompts.md)
   Kumpulan 12 template system prompt siap pakai untuk AI assistant, persona review PR, atau panduan evaluasi diri. Setiap prompt mengunci scope of influence, prinsip wajib, batasan teknis, dan hal-hal yang wajib ditolak untuk setiap level peran.

3. [domain_plugin_guide.md](domain_plugin_guide.md)
   Panduan praktis untuk memasang lapisan aturan 10% industri tertentu ke atas 90% sasis universal. Mencakup parameter evaluasi (RPO/RTO, SLA latensi, model konsistensi, auditabilitas) serta katalog plugin untuk FinTech, HealthTech, Gaming, AdTech, E-Commerce, dan GovTech.

---

## Cara penggunaan

### 1. Untuk pengembangan harian
Gunakan checklist pada peran Junior dan Mid-Level di `engineering_principles_by_role.md` saat merancang database dan endpoint API. Pastikan kodingan memenuhi proteksi boundary, idempotensi, dan isolasi konkurensi.

### 2. Untuk konfigurasi AI coding assistant
Salin template dari `role_system_prompts.md` ke dalam instruksi sistem (seperti Cursor rules, Claude Project, atau Copilot instructions) sesuai kebutuhan:
* Gunakan prompt Mid-Level untuk eksekusi fitur dan pembuatan endpoint.
* Gunakan prompt Senior untuk code review, failure analysis, dan observabilitas.
* Gunakan prompt Staff untuk penyusunan dokumen RFC atau evaluasi arsitektur lintas servis.

### 3. Untuk adaptasi industri baru
Ketika masuk ke industri dengan regulasi khusus, buka `domain_plugin_guide.md`, identifikasi lima parameter kuncinya, lalu tempelkan template domain plugin ke bagian akhir dari system prompt yang kamu gunakan.
