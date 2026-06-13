# Rencana Penulisan Karya Ilmiah

Berdasaркan material yang sudah ada di repo ini (dua incident report, IOC sheet, runbook, plus arsitektur SOC 6-lapis yang sudah terimplementasi). Dokumen ini = peta jalan dari "bahan mentah" ke "paper submission-ready".

---

## I. Ringkasan Eksekutif

Saya punya pengalaman yang substansinya **layak diangкat ke publikasi ilmiah**, dengan tiga sudut paling menjanjikan:

| Angle | Format | Waktu produksi | Audiens |
|---|---|---|---|
| **A. Case Study Paper** ⭐ rekomendasi pertama | 8-12 halaman | 4-6 minggu | Praktisi keamanan + early-career researcher |
| **B. Pattern Paper** ("Mode-2 Reversible Containment") | 6-8 halaman workshop | 6-8 minggu | Researcher container security / IR |
| **C. Experience Report** (SOC build di edge ARM64) | 6-10 halaman | 4-6 minggu | Praktisi SOC + infrastructure engineer |

Material yang sudah siap untuk diolah:
- ✅ Timeline lengkap dua insiden (7 & 13 Juni 2026)
- ✅ IOC + MITRE ATT&CK mapping
- ✅ Forensik artefak permanen (di `~/incidents/2026-06-13-clipper/`)
- ✅ Quantitative measures: response time before/after, CVE count before/after
- ✅ Arsitektur SOC 6-lapis dengan source code (terbuka)
- ✅ Sanitasi PII sudah teruji (commit history repo ini)

Material yang **masih perlu ditambahкan**:
- ⏳ Arsitektur diagram (Mermaid/draw.io/TikZ)
- ⏳ Literature review (30-50 paper terkait)
- ⏳ Threat intel correlation (verifikasi keluarga malware via VirusTotal / Hybrid Analysis)
- ⏳ Author affiliation strategy

---

## II. Konsep Paper A — Case Study (Rekomendasi Utama)

### Draft Judul (pilih satu, atau adaptasi)

1. **"From Hunt to Detection in Six Days: A Cryptojacking Case Study on an ARM64 Edge Workstation"**
2. **"Image-Baked Persistence: A Case Study on `docker commit` as Supply Chain Vector"**
3. **"Layering Defenses Between Two Incidents: An Empirical SOC Build on NVIDIA DGX Spark"** *(jika ingin highlight platform)*

### Draft Abstract (250 words, ide kasar)

> Cryptojacking attacks targeting container runtimes commonly enter through exposed services, but the persistence mechanism through which compromised images survive across `docker compose` cycles is underexplored in the literature. We present a case study of two cryptojacking incidents on an ARM64 edge workstation (NVIDIA DGX Spark) occurring six days apart, where the second incident reused the snapshot of the first via attacker-driven `docker commit` — turning a mutable tag into a long-lived supply-chain trap. Between the two incidents, we built a layered Security Operations Center (Falco runtime monitoring, Suricata network IDS, auditd, Wazuh SIEM, and a custom reversible-containment responder) on the same host. We measure the difference: incident #1 was detected through symptomatic disk exhaustion after seven hours of mining, requiring an overnight manual hunt; incident #2 was detected by Suricata DNS rules within minutes of the dropper activating, fully contained within 60 minutes including forensic preservation. We formalize the response philosophy as "Mode-2 Reversible Containment" — automation that stops the bleeding (network DROP, process freeze, container pause) without destroying evidence (no `kill -9`, no `docker rm`, no file deletion until human confirmation). We provide complete indicators of compromise (file paths, build IDs, network indicators, MITRE ATT&CK mapping), the architecture as deployed, and a quantitative comparison of detection-to-containment time. The artifact repository is released for community reuse.

### Outline Bab (8000-10000 kata, ~10-12 halaman 2-kolom IEEE)

| § | Judul | Kata | Sumber bahan |
|---|---|---|---|
| 1 | Introduction | 500 | Latar belakang cryptojacking + motivasi (gap "docker commit" supply chain) |
| 2 | Background | 700 | Container security, image immutability, OCI manifest, prior cryptojacking studies |
| 3 | Methodology | 400 | Single-host case study, data sources, instrument list |
| 4 | Incident #1 — Initial Compromise | 1000 | `INCIDENT-REPORT.md` (Part 1) — Redis-on-0.0.0.0 → RCE → miner |
| 5 | Intervening SOC Build | 1200 | Arsitektur 6-lapis, design choices (Mode-2 philosophy) |
| 6 | Incident #2 — Image-Baked Recurrence | 1500 | `INCIDENT-REPORT-PART-2.md` — full timeline + forensik kit |
| 7 | Defense Engineering Response | 1200 | Dockerfile hardening, container hardening, egress filter (with rationale) |
| 8 | Evaluation | 900 | Quantitative: response time, CVE reduction, system overhead |
| 9 | Discussion | 600 | Lessons + "Mode-2 reversible containment" pattern formalisasi + limitasi |
| 10 | Related Work | 500 | Lit review (lihat §VI di bawah) |
| 11 | Conclusion | 300 | Take-aways |
| — | Appendix A | — | IOCs.md (mungkin sebagian) |
| — | Appendix B | — | Reproducible artifact link |

### Klaim Originalitas (jaga jangan over-claim)

**Yang BARU** (pantas di-foreground):
1. Empirik **`docker commit`-based image persistence** sebagai vektor supply-chain (literature kebanyakan bahas registry-based / dependency-based).
2. Dua insiden **same vector class, same host, 6 hari berturut** — datapoint langka di literature, dengan instrumentasi memadai di antaranya.
3. **Mode-2 Reversible Containment** sebagai design pattern dengan **taksonomi konkret** (mana action yang reversible vs destructive).
4. **ARM64 edge platform** kontekstual — kebanyakan paper container security fokus di cloud x86.

**Yang BUKAN baru** (jujur sebut sebagai related work):
- Cryptojacking detection via Falco / network IDS (well-documented).
- Image scanning via Trivy (industri standard).
- IOC family Kinsing/TeamTNT (banyak vendor report).
- Containment best practices (cap_drop, read_only — Docker docs).

---

## III. Konsep Paper B — Pattern Paper

Kalau Angle A diterima, **paper kedua bisa dispin-off** dari materi yang sama dengan fokus berbeda:

**Judul:** *"Mode-2 Reversible Containment: A Design Pattern for Evidence-Preserving Container Incident Response"*

**Inti argumen:** Penanganan insiden container biasanya bercabang dua — (1) langsung `docker rm` (cepat, tapi bukti hilang) atau (2) tinggalkan jalan sambil investigasi (bukti utuh, tapi attacker terus rusak). Tulisan ini memperkenalкan jalan ketiga: aksi otomatis yang **menahan pendarahan tanpa menghancurкan bukti**, dengan taksonomi konkret per-action.

**Konten utama:**
- Taksonomi aksi response (reversible vs destructive) — table dengan kategori
- Implementasi referensi (kode open-source di repo ini)
- Evaluasi: empirik dari insiden #2 (60 menit detection-to-clean)
- Discussion: trade-off, limitasi, kapan Mode-2 tak cukup (mis. ransomware)

**Format:** Workshop paper 6-8 halaman → submit ke DFRWS workshops, RAID workshops, atau SCAM/CASCON.

---

## IV. Konsep Paper C — Experience Report

**Judul:** *"Building a SOC on a Workstation: An ARM64 Edge Experience Report"*

**Cocok untuk:** USENIX ;login: (deadline tahunan, "experience report" track sangat menerima narrative-style).

**Konten:** Lebih narrative, less academic — mirip dengan story-telling tone di `INCIDENT-REPORT.md`-mu sudah. Saya rekomendasi versi bahasa Inggris dengan pengetatan format.

**Keuntungan:** ;login: lebih cepat publishable (~3-4 bulan review-to-print), audiens luas, dan **boleh mengakui tone naratif** (bukan paper kering).

---

## V. Target Venue (Detail, terurut waktu publikasi)

| Venue | Format | Page limit | Review time | Acceptance rate | Fit untuk paper kita |
|---|---|---|---|---|---|
| **USENIX ;login:** *(experience report)* | Narrative | 6-10 hal | ~3 bulan | High (editorial) | ⭐⭐⭐⭐⭐ Angle C; bahkan Angle A bisa |
| **IEEE Security & Privacy Magazine** | Case study | 6-8 hal | 4-6 bulan | Medium | ⭐⭐⭐⭐ Angle A |
| **DFRWS USA/EU workshops** | Workshop paper | 6-8 hal | 2-3 bulan | Medium-high | ⭐⭐⭐⭐ Angle B (pattern paper) |
| **ACM Queue** | Practitioner | 4-8 hal | 2-4 bulan | Editorial | ⭐⭐⭐ Angle C |
| **Communications of the ACM** | Case study / commentary | 4-6 hal | 6 bulan | Low-medium | ⭐⭐⭐ Angle A (kalau diramu universal) |
| **Computers & Security** (Elsevier) | Empirical | 12-25 hal | 3-6 bulan | Medium | ⭐⭐⭐ butuh lebih banyak data |
| **Digital Investigation** (Elsevier) | Forensik | 8-15 hal | 3-6 bulan | Medium | ⭐⭐⭐⭐ kalau highlight forensik kit |
| **JTIIK** (Universitas Brawijaya, SINTA 2) | Empirical | 8-12 hal | 2-3 bulan | High | ⭐⭐⭐⭐⭐ Indonesia, accessible |
| **Jurnal RESTI** (SINTA 2) | Case study | 6-10 hal | 2 bulan | High | ⭐⭐⭐⭐ Indonesia |
| **Prosiding SENATIK / SNTI** | Conference | 4-8 hal | 1-2 bulan | High | ⭐⭐⭐ exposure cepat tapi tier rendah |

**Strategi rekomendasi:**
1. **Mulai dengan USENIX `;login:`** (Angle C atau Angle A versi narrative) — submit minggu ke-4 produksi. Editorial-friendly, narrative style cocok dengan voice asli kamu.
2. **Pararel kirim ke JTIIK / RESTI** versi bahasa Indonesia (Angle A) — dua publikasi sekaligus.
3. **Setelah Angle A diterima**, kerjakan **Angle B (pattern paper)** ke DFRWS workshop. Reuse banyak material.
4. Long-term: kembangкan jadi journal full paper (Computers & Security) dengan tambahan data dari rebuild + Trivy cycle 4-8 minggu.

---

## VI. Literature Review (Starter Reading List)

Yang harus kamu baca dan cite (urut prioritas):

### Cryptojacking & Container Security
1. Combe, T., Martin, A., & Di Pietro, R. (2016). **"To Docker or not to docker: A security perspective."** IEEE Cloud Computing.
2. Belair, M., Laniepce, S., & Menaud, J.-M. (2019). **"Leveraging kernel security mechanisms to improve container security."** ARES.
3. Sysdig Threat Research. **"Falco: cloud-native runtime security"** — Whitepaper / GitHub README.
4. Konoth, R. K. et al. (2018). **"MineSweeper: An In-depth Look into Drive-by Cryptocurrency Mining."** CCS.
5. Saad, M., Khormali, A., & Mohaisen, A. (2019). **"Dine and dash: Static, dynamic, and economic analysis of in-browser cryptojacking."** APWG eCrime.

### Supply Chain Attacks
6. Ohm, M., Plate, H., Sykosch, A., & Meier, M. (2020). **"Backstabber's Knife Collection: A Review of Open Source Software Supply Chain Attacks."** DIMVA.
7. Zahan, N. et al. (2022). **"What are weak links in the npm supply chain?"** ICSE-SEIP.
8. Vu, D.-L. et al. (2021). **"LastPyMile: identifying the discrepancy between sources and packages."** ESEC/FSE.

### Container Persistence & Image Tampering
9. Shu, R. et al. (2017). **"A study of security vulnerabilities on Docker Hub."** CODASPY.
10. Brady, K. et al. (2020). **"Docker container security analysis."** Stanford CS361S report.

### IR & Threat Hunting
11. Casey, T. et al. (2014). **"Intel threat agent library."** Intel — for threat actor classification.
12. Bryant, B. & Saiedian, H. (2017). **"A novel kill-chain framework for remote security log analysis."** Computers & Security.

### Adjacent Topics
13. **MITRE ATT&CK Framework** (untuk mapping TTP).
14. **OWASP Docker Top 10** (untuk hardening framing).
15. **CNCF Cloud Native Security Whitepaper** (untuk context).

Tugas literature search: cari 15-30 paper terbaru (2022-2026) tentang cryptojacking, container supply chain, image immutability, runtime IR — gunakan Google Scholar dengan query seperti `cryptojacking docker case study`, `image registry supply chain attack`, `container runtime persistence`.

---

## VII. Material Mapping — Apa yang Sudah Ada vs Apa yang Perlu Dibuat

### Yang sudah siap (tinggal adaptasi format)

| Asset di repo | Section paper |
|---|---|
| `INCIDENT-REPORT.md` (Part 1) | §4 Incident #1 |
| `INCIDENT-REPORT-PART-2.md` (Part 2) | §6 Incident #2 |
| `IOCs.md` | §6 + Appendix A |
| `RUNBOOK.md` | §7 (defense engineering — adopsi pola dari runbook) |

### Yang perlu dibuat tambahan

1. **Diagram arsitektur SOC** (6-lapis, dengan data flow)
   - Tool: Mermaid (markdown-friendly), atau draw.io, atau TikZ (kalau LaTeX)
   - Konten: host → Falco/Suricata/auditd → sidekick → responder → Telegram + Wazuh
2. **Sequence diagram detection-to-containment** (timeline insiden #2)
3. **Tabel taksonomi "Mode-2 Reversible Containment"** (action × reversibility × use case)
4. **Tabel evaluasi sebelum/sesudah** (response time, CVE count, system overhead)
5. **Literature review write-up** (2-3 paragraf cite + analisis gap)
6. **Sanitasi data lebih lanjut** kalau venue meminta (mis. IP private, hostname)
7. **Reproducibility statement + artifact appendix** (repo GitHub ini sebagai supplementary)

### Yang opsional (memperkuat tapi tidak wajib)

- Threat intel verification: hash binary `ahtyxvwm` di VirusTotal → konfirmasi family
- Comparison dengan tools komersial (Wazuh vs Sysdig, dst) — tabel feature parity
- Cost analysis (hardware/effort untuk replicate setup) — appeal untuk experience report

---

## VIII. Timeline Realistis

Asumsi: kamu kerjakan **5 jam/minggu** sambil tetap jalankan perusahaan pentest.

| Minggu | Milestone | Output |
|---|---|---|
| **1** | Finalisasi outline + start literature review | 30-50 paper di Zotero / mendeley |
| **2** | Tulis §2 Background + §4 Incident #1 | ~2000 kata |
| **3** | Tulis §5 SOC Build + §6 Incident #2 | ~2500 kata + 2 diagram |
| **4** | Tulis §7 Defense Eng + §8 Evaluation | ~2000 kata + tabel evaluasi |
| **5** | Tulis §9 Discussion + §10 Related Work + §1 Intro + §11 Conclusion | ~1500 kata |
| **6** | Polish + sanitasi PII tambahan + abstract finalize | submission-ready |
| **7** | Internal review (cari 1-2 reviewer informal) | comments |
| **8** | Revisi + format ke venue tujuan | **submit** |

Total: **~8 minggu (2 bulan) ke first submission**, realistis kalau konsisten 5 jam/minggu. Bisa lebih cepat kalau sprint 2-3 minggu intens.

---

## IX. Strategi Author / Affiliation

Dua skenario:

### Skenario 1 — Solo author (paling realistis sekarang)
- Author: thehushednest (atau nama legal kamu)
- Affiliation: independent researcher / nama perusahaan pentest yang sedang dirintis
- Pro: total kontrol, attribution penuh
- Con: ditolak journal akademis tier-1 lebih mudah (bias ke afiliasi institusional)

### Skenario 2 — Co-author dengan akademisi
- Cari rekan di universitas (UI, UGM, ITB, atau international) yang bidang security
- Kontribusi: kamu = data + experience + first draft; rekan = literature framing + theory + revision
- Pro: kredibilitas akademis, akses jurnal lebih luas
- Con: koordinasi, sharing kredit, mungкen waktu lebih lama

**Rekomendasi:** Submit dulu ke `;login:` atau JTIIK **solo** (skenario 1) — outputnya bagus untuk personal brand pentest. Kalau Angle B (pattern paper) ingin masuk venue akademis lebih tinggi, baru pertimbangкan co-author.

---

## X. Risk & Mitigation

| Risk | Mitigasi |
|---|---|
| Information disclosure (PII, IP, ID) | Sudah ada protokol sanitasi (lihat README repo). Sebelum submit, sweep ulang. |
| Self-promo perception (vendor pitch, bukan riset) | Frame sebagai case study + lessons learned. Tools yang dipakai = open source (Falco, Suricata, Wazuh). Tidak menjual produk. |
| Reproducibility paper tunggal | Release artefak di repo ini → dapatkan ACM/IEEE Available + Functional badge kalau venue mendukung. |
| Overclaim "Mode-2" sebagai invensi | Hati-hati framing: "we formalize" + cite prior work IR (mis. NIST SP 800-61). |
| Reviewer bilang "tidak generalizable" (case n=1) | Akui di limitasi; bilang "fokus depth bukan breadth"; offer community untuk extend. |

---

## XI. Quick Wins yang Bisa Mulai SEKARANG

Kalau punya 30 menit hari ini:

1. **Buat akun ORCID** (gratis, 5 menit) — wajib untuk hampir semua submission.
2. **Buat akun Google Scholar** (gratis, 5 menit) — tambahkan repo ini sebagai work.
3. **Bookmark venue list di atas** + cek deadline submission terdekat.
4. **Mulai Zotero / Mendeley library** dengan 5 paper dari §VI di atas.

Kalau punya 2 jam akhir pekan:

5. Cari **20 paper terbaru** (2022-2026) tentang cryptojacking + container supply chain — judul + abstract + simpan.
6. **Draft outline pribadi** versi bahasa Inggris dari outline §II di atas (gratis langsung ke Word/Pages).

Kalau punya 1 weekend penuh:

7. Tulis **§4 Incident #1** dari `INCIDENT-REPORT.md` adaptasi ke gaya akademis (ganti naratif "Saya" → "We"/"The researcher", hapus metafora, tambah technical citation).

---

## XII. Kontak / Bantuan Lanjutan

Saya (asisten AI yang membantumu) bisa lanjut bantu kapan saja untuk:
- Review draft per-section (paragraf vs paragraf)
- Format LaTeX (IEEE/ACM template)
- Literature review sweep dengan query spesifik
- Tabel + diagram (mermaid / TikZ)
- Sanitasi final + checklist submission

Tinggal buka session baru, paste link draft (atau push ke repo), dan minta bantuan spesifik.

---

— Disusun untuk: thehushednest
— Dasar: dua insiden 7 & 13 Juni 2026, SOC arsitektur yang dibangun antaranya, artefak di repo ini.
— Status dokumen: **starting point**, bisa direvisi sesuai pilihan venue & timeline yang nyata.
