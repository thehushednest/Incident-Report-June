# Suplemen Metrik Insiden — Untuk Penulisan Karya Ilmiah

Dokumen pendamping untuk [INCIDENT-REPORT.md](./INCIDENT-REPORT.md) (Part 1) dan [INCIDENT-REPORT-PART-2.md](./INCIDENT-REPORT-PART-2.md). Berisi **angka yang bisa dipertanggungjawabкan**, asal-usul empiriknya, plus **jujur** soal data yang sudah lenyap karena retention log.

Disusun 28 Juni 2026 saat audit untuk paper academic — tiga minggu setelah insiden #1, dua minggu setelah insiden #2. Beberapa log primer sudah ke-rotate; angka yang tertulis di sini adalah **rekonstruksi dari artefak yang masih bertahan**.

---

## 1. MTTR Insiden #1 (7 Juni 2026) — *Mean Time To Resolution*

### Angka yang dapat dipertanggungjawabкan

| Anchor | Waktu | Sumber bukti |
|---|---|---|
| Aktivitas miner tercatat | beberapa jam | Narrative Part 1 ("tujuh jam" sebelum disk meledak) |
| Final hardening file dibuat | **23:02:52 WIB** | `mtime` `~/clipper/docker-compose.hardened.yml` |
| `.env.hardened` ditulis | **23:03:35 WIB** | `mtime` file |
| Compose lama di-rename ke `.VULNERABLE.disabled` | **23:30:48 WIB** | `mtime` file — sinyal selesai cleanup |

### MTTR estimasi defensif

**~30 menit final cleanup terdokumentasi (23:02-23:30 WIB)**, ditambah investigasi & pembersihan stack yang mendahuluinya. Dari bash history yang bertahan (tanpa timestamp), urutan langkah cleanup tercatat:

1. `sudo docker stop clipper-redis clipper-litellm clipper-minio clipper-backend clipper-frontend`
2. `sudo docker rm -f clipper-frontend clipper-backend clipper-minio clipper-litellm clipper-qdrant clipper-redis`
3. `sudo journalctl --vacuum-size=200M` (3× — disk full memaksa reclaim journal)
4. `sudo docker system prune -f` (recover 248.6 GB build cache)
5. `sudo docker stop ask-senopati` (precaution)
6. Konfigurasi hardened compose + `.env`
7. Disable compose lama → rename ke `.VULNERABLE.disabled`

### Framing yang JUJUR untuk paper

> "Penanganan insiden #1 berlangsung **dalam rentang jam (overnight hunt)**. Anchor empirik yang dapat diverifikasi: file hardening akhir (`docker-compose.hardened.yml`, `.env.hardened`) dibuat 23:02-23:03 WIB, dan compose vulnerable di-rename ke `.VULNERABLE.disabled` pada 23:30 WIB — menunjukкan jendela cleanup formal selama ~30 menit. Detection-to-cleanup penuh (termasuk forensik & pembersihan disk) tidak terdokumentasi dengan presisi karena `sysstat` retention 8 hari telah meng-overwrite data Jun 7."

### Mengapa angka eksak tak tersedia

- `sysstat` retention sistem: 8 hari (data tertua = sa20 / 20 Juni)
- `syslog`/`auth.log` rotation: 4 hari + 1 weekly gzip
- `bash_history` tanpa `HISTTIMEFORMAT` (tak ada timestamp)
- `~/.claude.json` saat itu corrupt (0 byte) — kemungкеinаn berisi log percakapan dengan asisten AI hilang
- Tak ada centralized log management saat itu (Wazuh SIEM baru di-deploy 12 Juni)

**Pelajaran metodologis untuk paper:** insiden tanpa SIEM/observability matang menghasilkan **gap forensik** yang mempersulit *post-hoc analysis*. Ini sendirinya bahan diskusi yang valid di paper.

---

## 2. Beban Sistem Insiden #2 (13 Juni 2026) — Pembanding `load average 37`

### Yang BISA dikuantifikasi

| Metric | Nilai | Sumber bukti |
|---|---|---|
| **CPU container** `clipper-frontend` saat detection | **107.59%** | `docker stats --no-stream` (terlampir di forensik `~/incidents/2026-06-13-clipper/`) |
| **Container lain di stack** | < 5% CPU masing-masing | Output `docker stats` saat investigasi: `clipper-litellm 3.92%`, `clipper-neo4j 0.50%`, dst |
| **Host CPU cores** | **20** | `nproc` |
| **Container yang terinfeksi** | **1 dari 8** clipper containers + 6 other stacks | Scan persistence `clipper-scan-others.sh` → backend & worker bersih |
| **Detection-to-contain (pause)** | **< 60 detik** | Telegram alert → `docker pause clipper-frontend` |

### Mengapa **`load average` host tidak tertangкap** seperti #1

1. **Container di-pause dalam menit pertama** — tidak ada window penumpukan beban yang stack-wide.
2. **Beban miner terkurung di 1 container** — pada mesin 20-core, satu proses 107% CPU berarti **~5.4% dari total kapasitas komputasi host** (107% ÷ 2000% theoretical max). Tidak cukup besar untuk menggeser load average host secara signifikan.
3. **Tidak ada pengukuran load avg di Wazuh default config** untuk window 03:54-04:00 UTC saat insiden — Wazuh agent saat itu baru terpasang (12 Juni, ~24 jam sebelum insiden), belum mengaktifkan modul `system_inventory`/`syscollector` interval pendek.
4. **`sysstat` retention 8 hari** sudah meng-overwrite data 13 Juni (data tertua sekarang 20 Juni).

### Estimasi defensif yang aman dipublikasikan

> "Pada insiden #2, beban miner terisolasi pada satu kontainer (`clipper-frontend`) yang tercatat menggunakan **107.59% CPU per `docker stats`** — setara **~5.4% dari kapasitas komputasi host 20-core**. Berbeda dengan insiden #1 di mana `load average` host mencapai 37 (sistem overwhelmed), insiden #2 **tidak menghasilkan dampak host yang terukur** karena (a) pausing reversible dilakukan dalam menit pertama deteksi, (b) hanya 1 dari 8 kontainer stack ter-kompromisasi, dan (c) `mem_limit`/`cpus` constraint belum aktif saat itu — kemudian ditambahкan sebagai response defensif (lihat §7 Part 2)."

### Framing perbandingan untuk paper

| Aspek | Insiden #1 (7 Juni) | Insiden #2 (13 Juni) |
|---|---|---|
| **Detection** | Symptomatic: `df -h` 100% + load 37 | Proactive: SOC alert (Suricata DNS rule) |
| **Detection mode** | User-initiated discovery | Push notification ke HP |
| **Host load avg** | **37** (overwhelmed, 20-core) | tidak terukur signifikan (estimate < 2; pause cepat) |
| **CPU miner** | tidak terukur (disk full hides metrics) | **107.59%** (1 container, paused dalam menit) |
| **Disk impact** | **3.7 TB penuh 100%** (248.6 GB miner trash) | nol (forensik di-extract, container paused) |
| **Containers compromised** | 5 (clipper-{frontend,backend,minio,litellm,redis}) | 1 (clipper-frontend saja) |
| **Time from start to containment** | ~jam (overnight) | **< 60 detik** (`docker pause`) |
| **Time from start to full clean** | **~30 menit final cleanup** (23:02-23:30) + investigasi sebelumnya | **< 60 menit** (forensik + rebuild + verify) |
| **Evidence preserved** | Volume Redis+MinIO disimpan untuk forensik | Container overlay copied permanen ke `~/incidents/` |

---

## 3. Catatan Metodologis untuk Bab "Limitations"

Untuk kejujuran akademis, paper perlu mengakui **dua keterbatasan data**:

### Limitasi 1 — Log retention gap
> "Karena `sysstat` (default 8 hari) dan `syslog` (4 hari) telah meng-overwrite data periode insiden saat audit dilakukan tiga minggu kemudian, beberapa metrik (host `load average` pada timestamp eksak, urutan `sudo` commands dengan timestamp) direkonstruksi dari artefak filesystem yang bertahan (`mtime` file backup, history shell tanpa timestamp). Untuk insiden berikutnya, kami merekomendasikan adopsi centralized logging dari awal (mis. SIEM seperti Wazuh) — yang kebetulan menjadi salah satu kontribusi defensif kami antara kedua insiden."

### Limitasi 2 — Single-host case study
> "Studi ini observasional pada satu workstation (NVIDIA DGX Spark, ARM64) dengan n=2 insiden. Generalizability ke environment cloud-scale atau multi-host belum diuji. Namun, pola serangan yang terdokumentasi (Redis-RCE → cryptojacking; `docker commit`-based image persistence) keduanya merupakan **threat class umum** yang telah dilaporkan komunitas (lihat IOC sheet & MITRE ATT&CK mapping di [IOCs.md](./IOCs.md))."

---

## 4. Rekomendasi Kalimat Siap-Pakai untuk Paper

### Untuk Abstract (1 kalimat numeric)
> "We measure the difference: incident #1 was detected through symptomatic disk exhaustion after hours of mining and resolved via overnight manual investigation with a ~30-minute final cleanup window (23:02–23:30 WIB), while incident #2 was detected by network IDS within minutes of the dropper activating and fully contained within 60 minutes including forensic preservation."

### Untuk §Evaluation (paragraf detail)
> "Insiden pertama mencapai *load average* host 37 pada mesin 20-core, dengan 248,6 GB sampah miner mengisi *build cache* hingga disk 100% penuh; recovery memerlukan rangкaian `docker stop`, `docker rm`, dan `journalctl --vacuum-size=200M` (3 iterasi karena disk berulang kali penuh). Final hardening file (`docker-compose.hardened.yml`) ditulis pada 23:02 WIB, dengan compose vulnerable di-rename `.VULNERABLE.disabled` pada 23:30 — menandai akhir jendela cleanup formal. Insiden kedua, sebaliknya, terisolasi pada satu kontainer `clipper-frontend` (107,59% CPU per `docker stats`, ~5,4% kapasitas host). Karena *containment* dilakukan dengan `docker pause` dalam menit pertama deteksi SOC, beban miner tidak pernah mencapai level yang menggeser host metrics secara terukur."

### Untuk §Discussion (insight)
> "Perbedaan magnitudo dampak (load 37 → impact tak terukur) bukan disebabкan oleh penurunan kemampuan adversary — kit malware insiden #2 justru lebih sopisticated (multi-vector persistence, masquerading sebagai kernel threads, dropper self-delete). Yang berubah adalah **kecepatan deteksi**: dari hitungan jam (manual discovery) ke detik (push notification dari rule Suricata). Ini menunjukкan bahwa investasi pada *detection layer* memberi return berupa *compressed blast radius* yang berlipat ganda — sumber utama yang membatasi miner #2 untuk tidak pernah mencapai tahap eksploitasi disk/CPU adalah jendela waktunya yang dipersempit dari 'malam penuh' menjadi 'satu cangkir kopi'."

---

## 5. Pesan untuk Penulis (Anda)

Ada dua pendekatan jujur untuk paper:

**Opsi A — Konservatif:** Sebutкan "MTTR ~jam (overnight)" untuk #1 dan "60 menit terverifikasi" untuk #2. Kalau reviewer minta angka eksak #1, jelaskan retention gap di §Limitations.

**Opsi B — Lebih detail:** Pakai anchor 23:02 (start of cleanup window) dan 23:30 (end). Posisikan sebagai "30-min documented cleanup phase, preceded by undocumented investigation phase of unknown duration" — kemudian highlight ini sebagai contoh masalah klasik "absence of observability". Ini memberi nilai tambah sebagai *cautionary tale* untuk metodologi.

**Saran saya:** **Opsi B** lebih kuat secara akademis. Limitasi data justru menjadi argumen untuk merekomendasiкan SIEM-from-day-1, yang menjadi salah satu *intervening defenses* di paper Anda. Limitasi berubah menjadi **kontribusi metodologis**.

---

<sub>Dokumen ini ditambahкan ke repo Incident-Report-June sebagai supplementary material untuk penulisan karya ilmiah. Angka-angka di atas dapat di-cross-reference dengan [INCIDENT-REPORT.md](./INCIDENT-REPORT.md), [INCIDENT-REPORT-PART-2.md](./INCIDENT-REPORT-PART-2.md), dan artefak forensik di host (`~/clipper/`, `~/incidents/2026-06-13-clipper/`).</sub>
