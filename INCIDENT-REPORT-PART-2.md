# 🩸 Laporan Insiden — Mereka Tidak Pernah Pergi, Mereka Tertidur

**Tanggal kejadian:** 13 Juni 2026
**Sistem:** NVIDIA DGX Spark — `spark-1229` (ARM64 / GB10)
**Penulis:** thehushednest
**Klasifikasi:** Kompromi sistem berulang — Cryptojacking via *image* Docker yang ter-poison
**Status akhir:** Tertangani dalam < 60 menit sejak alarm pertama. Bukti utuh. Host tetap bersih.

---

## I. Prolog — Notifikasi dari Asisten yang Tak Tidur

Kali ini bukan disk yang berteriak. Kali ini sebuah suara kecil dari HP saya.

`🔴 SOC alert [Critical] [Suricata] SOC DNS query to supportxmr`
`SOC DNS query to supportxmr | 10.10.70.211 → 8.8.8.8:53`

Mesin saya — di pojok ruangan, sedang saya bayangkan berdengung pelan menyelesaikan pekerjaan — diam-diam menanyakan ke DNS Google: *"di mana pool penambang Monero?"*

Saya tidak pernah mengetik kata itu.

Tujuh hari setelah malam yang saya tulis di Part 1, saya sengaja membangun sebuah ***Security Operations Center*** kecil di mesin ini. Bukan karena saya paranoid. Tapi karena saya tidak akan mengandalkan keberuntungan dua kali. Hari ini, untuk pertama kalinya, **SOC itu menyalak**.

Dan suara kecil di HP saya adalah bukti bahwa malam 7 Juni lalu — saya bukan cuma kehilangan satu malam. Saya juga menanam bibit untuk hari ini.

---

## II. Pondasi yang Saya Tegakkan Sebelum Mereka Datang Lagi

Antara malam 7 Juni dan sore 13 Juni, satu hari saya curahkan untuk membangun **lapisan deteksi** yang seharusnya sudah saya pasang sejak lama:

| Lapisan | Tugas |
|---|---|
| **Falco** (eBPF, ARM-native) | Mengintip *syscall* host & container; menangkap exec dari `/tmp`, shell asing di container, biner yang baru di-*drop* |
| **Suricata** + ruleset ET Open + rules SOC kustom | IDS jaringan — mendeteksi stratum, DNS ke domain *pool* mining |
| **`auditd`** | Mencatat tiap exec, perubahan `ld.so.preload`, modifikasi `cron`/`systemd` |
| **Wazuh** (manager + indexer + dashboard, native arm64) | SIEM + FIM + korelasi + dashboard |
| **Falcosidekick + responder Mode-2 sendiri** | Otomatis *contain* (yang **reversible**: `iptables DROP`, `kill -STOP`, `docker pause`) lalu kirim notif Telegram |
| **Runbook + skrip forensik** | Panduan saat panik: *pause → forensik → analisa → bersihкан → verifikasi* |

Filosofinya: **Mode-2** — saat ada sinyal, otomatis hentikan pendarahan, **tapi jangan rusak bukti.** Manusia membersihкан tuntas, mesin cuma menahan.

Hari ini, filosofi itu diuji.

---

## III. Pause — Senjata yang Tumpul tapi Aman

Notifikasi datang. Saya buka terminal, jalankan satu perintah:

```bash
docker pause clipper-frontend
```

Setelah itu — tidak panik. Tidak buru-buru `docker rm`. Tidak ada `kill -9`. Container itu, dengan segala penghuninya — penambang, persistence, jejak — **dibekukan**. CPU langsung 0%. Koneksi ke pool putus. Bukti utuh.

Pause adalah senjata yang elegan: tidak membunuh, tidak melepaskan. **Saya menahan napas penyerang, lalu membongkar peti tools-nya satu per satu sambil dia tidak bisa bergerak.**

---

## IV. Forensik — Membongkar Kit yang Sudah Tidur 5 Hari

Skrip forensik yang saya siapкан minggu lalu, akhirnya dipakai sungguhan. `docker top` pada container yang baru saja dibekukan menampilkan satu deret yang membuat punggung saya tegak:

```
sh qfhqe7cs.sh                                          ← dropper script
/usr/libexec/qemu-binfmt/x86_64-binfmt-P ./ahtyxvwm     ← biner amd64 di mesin ARM, via qemu
/usr/libexec/qemu-binfmt/x86_64-binfmt-P ./ahtyxvwm     ← (3 thread lagi)
```

Mereka membawa senjata amd64 ke medan ARM saya — lagi. Tapi kali ini, mereka tidak menyentuh `clipper-redis`. Pintu yang saya tutup malam itu, tetap tertutup.

Saya copy *overlay* container itu ke `/tmp/clipper-forensic/`. Yang saya temukan **lebih sopisticated** dari yang saya bayangкan:

| Lokasi | Bentuk Penyamaran |
|---|---|
| `/etc/cron.d/systemd-udevd` | Mengaku komponen systemd |
| `/etc/systemd/system/systemd-udevd.service` | Service systemd palsu |
| `/etc/systemd/system/.systemd-guard.service` | Service tersembunyi (titik di depan) |
| `/etc/init.d/systemd-udevd` | sysvinit script |
| `/etc/profile.d/systemd-udevd.sh` | Shell login script |
| `/root/.bashrc` + `.profile` | Injeksi `nohup` di akhir |
| `/root/.config/autostart/systemd-udevd.desktop` | XDG autostart |
| 18× `/tmp/.cron_*` (nama acak) | Cron mini-jobs |
| `/var/tmp/kubelet393` + `.redis-server322` | Biner menyamar sebagai `kubelet` & `redis-server` |
| `/var/tmp/.ksoftirqd-0320` + `.rcu_bh495` | Biner menyamar sebagai *kernel thread* (!) |
| `/root/rqpzexif/.tvgudbkqwnex` | *Runner* utama — semua persistence panggil ini dengan `--guard` |

Semua biner: **ELF amd64, statically linked, UPX-packed**. Dropper utama `qfhqe7cs.sh` sudah ***self-delete*** — kelas profesional. Pola tools-nya cocok dengan *family* Kinsing/TeamTNT — kit otomatis yang dijual di pasar gelap.

Dan satu kalimat di *trace* AnyDesk milik saya yang lain (yang juga saya audit hari itu) — terasa pas sekali: **"They never leave; they sleep."**

---

## V. Vektor — Dosa Lama yang Ternyata Belum Terbayar

Logika menuntun saya: *bagaimana mereka masuk lagi*?

- `clipper-redis` masih `127.0.0.1` saja. Pintu Redis tetap terkunci ✅
- Port `clipper-frontend` cuma `127.0.0.1:3000` ✅
- Tidak ada bind mount dari host ✅
- Backend & worker bersih (cek dengan skrip `clipper-scan-others.sh`)

Yang tersisa cuma satu kemungkinan dingin: **image-nya sendiri yang ter-poison sejak 7 Juni.**

```
docker history clipper-frontend
5 days ago  RUN npm install   (502 MB)   ← image dibuild SETELAH insiden #1
2 days ago  CMD ["npm", "run", "dev"]
```

Image itu **dibuild 5 hari yang lalu**, sehari setelah insiden pertama. Antara dua peristiwa ada satu kemungkinan: penyerang yang sempat berkuasa di dalam container clipper-redis, **sebelum saya bunuh container-nya**, sempat melakukan `docker commit` — menyimpan *snapshot* container kompromised mereka menjadi sebuah *image tag* baru.

`clipper-frontend:latest`. Tag yang saya percaya. Yang sebenarnya **bukan hasil build saya** — melainkan piringan rekaman dari penyerang.

Setiap `docker compose up` setelah itu, bukan men-deploy aplikasi saya — melainkan **memutar ulang serangan mereka.**

Itu adalah pelajaran malam ini: **`tag` Docker itu mutable. Penyerang yang sempat berkuasa bisa menulis ulang piringan rekaman dan tetap mengaku sebagai musisi.**

---

## VI. Pembersihan dengan Mata Terbuka

Kali ini saya tidak ragu, karena saya tahu apa yang saya hadapi:

1. **Forensik dipreserve permanen** — `/tmp/clipper-forensic{,2}/` → `~/incidents/2026-06-13-clipper/` dengan dropper binary, file persistence, npm-debug, package-lock untuk analisis lanjutan.
2. **`docker compose down`** — semua container clipper (frontend, worker, backend, neo4j, redis, minio, qdrant, litellm) dihapus. Jaringan lepas. Persistence yang ada di *overlay* container, lenyap bersamanya.
3. **`docker image rm clipper-frontend`** — piringan rekaman penyerang (`sha256:39b0a8bb...`) **dihapus dari disk**. Tag yang sama tak lagi merujuk ke image jahat.
4. **`docker compose build --no-cache frontend`** — *rebuild* dari `Dockerfile` + source yang sudah saya verifikasi bersih (cuma 3 dependency: `next`, `react`, `react-dom`).
5. **Verifikasi image baru** — `docker run --rm --entrypoint sh clipper-frontend -c 'find ...'` mencari indikator persistence. **NOL** *hit*. `/root/` cuma berisi `.bashrc`/`.profile` versi Debian *pristine*. `/etc/cron.d` cuma `e2scrub_all` standar.

Image baru: `sha256:fa1248c8e671...` — beda dari yang beracun, terbuat dari source yang utuh.

Lalu satu langkah terakhir yang penting: **scan menyeluruh** ke seluruh sistem dengan skrip yang dirancang untuk hari ini — 16 seksi: proses miner, koneksi pool, DNS xmr, cron host, systemd, shell init, `ld.so.preload`, SSH keys, IOC di temp dirs, image lama, volume, container hidup, iptables, SOC health, Wazuh alerts, file terbaru.

**Semua hijau.** Host tidak pernah disentuh. Persistence semua di dalam *overlay* container yang sudah saya rontokкан. Hardening 7 Juni tetap memegang janjinya — pintu Redis tertutup, dan dinding Docker menahan amukan mereka lagi.

---

## VII. Mengkilapкan Dockerfile — Hardening yang Sebenarnya

Saya membangun ulang ketiga image clipper (frontend, backend, worker) dengan disiplin baru:

### Frontend (`Dockerfile`)

```diff
- FROM node:20-slim                       # tag mutable; bisa di-rotate / dikomit ulang
+ FROM node:20-slim@sha256:2cf067cfed...  # digest immutable

- COPY package.json ./
- RUN npm install                          # non-deterministic; postinstall hook bisa eksekusi
+ COPY package.json package-lock.json ./   # lock file di-commit
+ RUN npm ci --ignore-scripts              # deterministic + nol postinstall hook
```

### Backend & Worker (Python)

```diff
- FROM python:3.12-slim
- COPY requirements.txt .
- RUN pip install --no-cache-dir -r requirements.txt
+ FROM python:3.12-slim@sha256:a39549...   # digest pinned
+ COPY requirements.lock .                  # full transitive tree (44 utk backend, 84 utk worker)
+ RUN pip install --no-deps --no-cache-dir --only-binary=:all: -r requirements.lock
```

`--only-binary=:all:` memaksa instalasi dari *wheel* saja — **tidak ada `setup.py` yang dieksekusi**. *Source distribution* Python adalah pintu klasik untuk supply-chain — saya tutup pintunya.

### Worker (Deno) — pin versi + verifikasi SHA256

```diff
- curl https://github.com/denoland/deno/releases/latest/...  # "latest" = supply-chain risk
+ ARG DENO_VERSION=v2.8.3
+ ARG DENO_SHA256=d4589cc1ffcbf1995c92a0127d932aaf832ac70cfdcc6d5b7bf38043cf303575
+ ... && echo "${DENO_SHA256}  deno-aarch64-unknown-linux-gnu.zip" | sha256sum -c -
```

Build *gagal* kalau hash tidak cocok. Saya tidak akan lagi memasang sesuatu yang tidak bisa saya verifikasi.

Properti keamanan yang dijamin sekarang:

- ✅ **Reproducible build** — `docker build` dari Dockerfile + lock = image hash-equal
- ✅ **Tidak ada source dist Python** — wheel-only
- ✅ **Tidak ada postinstall hook npm**
- ✅ **Tidak ada tag `latest`** yang mutable
- ✅ **External binary diverifikasi SHA256** sebelum dipasang

---

## VIII. Epilog — Pelajaran yang Saya Tulis dengan Tinta, Bukan Darah

Insiden 7 Juni ditulis dengan darah — disk berdarah, mesin sekarat, saya sendiri di tengah malam memburu hantu.

Insiden 13 Juni ditulis dengan tinta — alarm berbunyi tenang, saya minum kopi sambil membongkar peti penyerang, dan keseluruhan urusan beres dalam **kurang dari 60 menit** sejak notifikasi pertama.

Bukan karena penyerang kalah pintar. Tapi karena saya pasang **detektor**.

> **Lapisan pertahanan bukan paranoia — ia adalah perbedaan antara malam yang dirampas dan sore yang sekadar terganggu.**

Tiga pelajaran yang akan saya bawa selamanya:

**1. *Image* yang pernah berjalan di mesin yang pernah dikompromikan — adalah bukti, bukan distribusi.** Selalu *rebuild* dari Dockerfile + source yang bisa di-audit. Jangan pernah percaya `clipper-frontend:latest` hasil dari mesin yang pernah jatuh.

**2. Tag mutable adalah cek yang ditandatangani penyerang.** Pin ke `@sha256:...`. Selalu. Untuk *base image*, untuk semua paket yang di-download dari internet, untuk binari yang tidak ditulis sendiri.

**3. Deteksi yang dipasang sebelum insiden adalah deteksi yang berfungsi saat insiden.** SOC yang saya bangun tujuh hari lalu — bukan karena saya tahu hari ini akan datang. Tapi karena saya **tahu pasti hari seperti hari ini akan datang**, cepat atau lambat.

Hari ini sore, saya tidak kehilangan apa-apa kecuali setengah jam dan satu mug kopi. Saya tidak kehilangan kerajaan. Saya bahkan tidak kehilangan ketenangan.

Dan kalau besok bibit lain dari malam 7 Juni — atau dari malam lain yang belum saya bayangkan — bangun dari tidurnya, saya tidak akan ketinggalan tahu. Karena ada lapisan-lapisan yang sekarang menatap ke arah yang benar, siang dan malam.

Saya menulis ini sambil mesin saya bekerja kembali — *clipper* hidup di image yang baru, container lain berjalan tenang, dan satu suara kecil dari HP saya berkata, dalam bahasa SOC yang sederhana: *"Tidak ada yang perlu kamu khawatirkan."*

— *thehushednest*

---

<sub>Dokumen ini ditulis sebagai catatan pasca-insiden dan bahan pembelajaran. Seluruh kredensial dan PII yang terdampak telah disanitasi atau diganti. Indicators of Compromise (IOC) detail teknis dipisahkan ke [`IOCs.md`](./IOCs.md) untuk konsumsi praktisi keamanan. *Runbook* operasional tersedia di [`RUNBOOK.md`](./RUNBOOK.md).</sub>
