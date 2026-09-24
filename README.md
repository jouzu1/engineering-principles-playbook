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
   Fondasi filosofis, hukum komputasi (Little's Law, Amdahl's Law, CAP, Conway's Law, Gall's Law, Hyrum's Law), batas arsitektur, dan prinsip operasional baku dari level Junior sampai Fellow di IC Track, serta Tech Lead sampai CTO di Management Track.

2. [role_system_prompts.md](role_system_prompts.md)
   Kumpulan 12 template system prompt siap pakai untuk AI assistant, persona review PR, atau panduan evaluasi diri. Setiap prompt mengunci scope of influence, prinsip wajib, batasan teknis, dan hal-hal yang wajib ditolak untuk setiap level peran.

3. [domain_plugin_guide.md](domain_plugin_guide.md)
   Panduan praktis untuk memasang lapisan aturan 10% industri tertentu ke atas 90% sasis universal. Mencakup parameter evaluasi (RPO/RTO, SLA latensi, model konsistensi, auditabilitas) serta katalog plugin untuk FinTech, HealthTech, Gaming, AdTech, E-Commerce, dan GovTech.

4. [threat_modeling_and_ai_security.md](threat_modeling_and_ai_security.md)
   Modul keamanan formal dan pemodelan ancaman. Berisi Prinsip Kerckhoffs, 10 Immutable Laws of Security (Microsoft), proses 4 langkah Adam Shostack, matriks mitigasi STRIDE klasik, mitigasi STRIDE-AI, serta Agent Trust Boundary Model (AakashX) untuk arsitektur AI Agent yang aman.

5. [security_engineering_canon.md](security_engineering_canon.md)
   Kompendium rekayasa keamanan sistem, aplikasi, dan infrastruktur. Memuat prinsip Saltzer-Schroeder, Rice's theorem, penanganan OWASP Pentest defense, pengerasan container (rootless/distroless), arsitektur cloud VPC triad dan envelope KMS, otorisasi ReBAC Zanzibar, teknologi deception (canary tokens), protokol forensik digital (Order of Volatility), SLA perbaikan celah CVSS, dan kepatuhan audit data.

6. [security_role_system_prompts.md](security_role_system_prompts.md)
   Kumpulan 6 template system prompt spesialis keamanan siber (AppSec Engineer, Cloud SecOps Architect, Red Team Pentester, Blue Team Incident Responder, Product Security Lead, dan CISO Governance Director) untuk AI coding assistant, security review, atau simulasi audit.

---

## Cara penggunaan

### 1. Untuk pengembangan harian
Gunakan checklist pada peran Junior dan Mid-Level di `engineering_principles_by_role.md` saat merancang database dan endpoint API. Pastikan kodingan memenuhi proteksi boundary, idempotensi, isolasi konkurensi, dan The Beyonce Rule.

### 2. Untuk konfigurasi AI coding assistant
Salin template dari `role_system_prompts.md` ke dalam instruksi sistem (seperti Cursor rules, Claude Project, atau Copilot instructions) sesuai kebutuhan:
* Gunakan prompt Mid-Level untuk eksekusi fitur dan pembuatan endpoint.
* Gunakan prompt Senior untuk code review, failure analysis, dan observabilitas.
* Gunakan prompt Staff untuk penyusunan dokumen RFC atau evaluasi arsitektur lintas servis.

### 3. Untuk simulasi keamanan dan persiapan penetration testing
Gunakan `security_engineering_canon.md` dan salin persona spesialis dari `security_role_system_prompts.md`:
* Gunakan prompt Red Team Pentester untuk meminta AI mencari celah dan membuat proof of concept serangan pada kodingan kamu.
* Gunakan prompt AppSec Engineer untuk mengaudit source code terhadap celah BOLA/IDOR, injection, dan cryptographic hygiene.
* Gunakan prompt Cloud SecOps Architect untuk mengaudit Terraform, Dockerfile, dan kebijakan IAM least privilege.

### 4. Untuk adaptasi industri baru
Ketika masuk ke industri dengan regulasi khusus, buka `domain_plugin_guide.md`, identifikasi lima parameter kuncinya, lalu tempelkan template domain plugin ke bagian akhir dari system prompt yang kamu gunakan.

### 5. Untuk audit arsitektur AI Agent
Gunakan `threat_modeling_and_ai_security.md` saat merancang backend kritis atau mengintegrasikan AI Agent. Terapkan pemisahan 4 batas kepercayaan (Instruction, Data, Tool, Action) dengan prinsip bahwa model probabilistik boleh mengusulkan, namun runtime deterministik yang mengeksekusi.
