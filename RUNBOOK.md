# Security Runbook — Cryptojacking Incident Response

Panduan operasional untuk menangani insiden kompromi container Docker via supply-chain — disarikan dari penanganan langsung dua insiden Juni 2026.

Filosofi: **Mode-2 reversible containment**. Hentikan pendarahan otomatis, tapi bukti dijaga. Manusia memutuskan langkah mematikan.

---

## 0. Tanda alarm

Notifikasi push (Telegram/Slack/dll) dari SOC. Yang **wajib direspons cepat**:

| Sinyal | Sumber | Artinya |
|---|---|---|
| `[Critical] [Suricata] ... supportxmr / minexmr / nanopool` | Suricata DNS rule | Ada yang query pool Monero |
| `[Critical] [Suricata] ET MALWARE CoinMiner` | ET Open ruleset | Threat intel match |
| `SOC Cryptominer Process Launched` | Falco rule | Proses miner dengan nama dikenal |
| `SOC Outbound To Mining Pool Port` | Falco rule | Koneksi ke stratum port (3333/4444/dll) |
| CPU container tiba-tiba 100%+ tanpa sebab | `docker stats` | Bisa miner — tapi cek dulu apakah workload normal |

Jangan langsung `docker rm` — bukti hilang.

---

## 1. Containment (1-2 menit) — REVERSIBLE

Tujuan: hentikan kerusakan TANPA merusak bukti.

```bash
# pause container tersangka (CPU langsung 0%, RAM tetap, state utuh)
docker pause <nama-container>

# kalau belum tahu mana persisnya, pause SEMUA container terkait sebagai precaution:
for c in $(docker ps --filter name=<project-prefix> --format '{{.Names}}'); do
  docker pause "$c"
done

# verifikasi CPU langsung 0%
docker stats --no-stream
```

Container *paused* = miner berhenti, koneksi pool putus, **filesystem container utuh sebagai bukti**.

---

## 2. Forensik (5-15 menit) — sebelum hapus apa pun

### a. Identifikasi proses pelaku
```bash
docker top <container>   # bisa di container paused
```
Catat PID, command, parent process.

### b. Ekstrak dropper + binary jahat (overlay container, butuh sudo)

Kunci: dapatkan path *merged overlay*-nya:
```bash
MERGED=$(docker inspect <container> --format '{{.GraphDriver.Data.MergedDir}}')
```

Lalu cari file IOC:
```bash
sudo find "$MERGED" -maxdepth 6 \( \
    -name 'qfhqe7cs.sh' -o -name 'ahtyxvwm' -o -name 'kubelet*' -o \
    -name '*.tvgudbkqwnex*' -o -name '.systemd-guard*' -o -name 'systemd-udevd.service' \
    \) 2>/dev/null
```

Atau pakai skrip terstruktur (template di [scripts/](./scripts/)).

### c. Simpan PERMANEN sebelum /tmp hilang
```bash
DATE=$(date +%Y-%m-%d)
sudo mkdir -p ~/incidents/${DATE}-<nama-incident>/
sudo cp -a /tmp/<forensic-dir> ~/incidents/${DATE}-<nama-incident>/
sudo chown -R $USER:$USER ~/incidents/${DATE}-*
```

### d. Identifikasi scope (mana yang kena, mana yang aman)
Scan container lain di project sama untuk IOC yang sama:
```bash
for c in $(docker ps -a --filter name=<project-prefix> --format '{{.Names}}'); do
  M=$(docker inspect -f '{{.GraphDriver.Data.MergedDir}}' "$c" 2>/dev/null)
  [ -z "$M" ] && continue
  sudo find "$M" -maxdepth 6 \( -name 'ahtyxvwm' -o -name 'kubelet*' -o -name '*tvgudbkqwnex*' \) 2>/dev/null \
    | sed "s|^|$c: |"
done
```

---

## 3. Analisis Vektor (15-30 menit)

Jawab pertanyaan ini sebelum hapus:

1. **Apakah HOST kena?** Jalankan deep scan ke seluruh sistem (lihat [scripts/incident-deep-scan.sh](./scripts/) — sanitasi placeholder dulu). Persistence di luar overlay container = serius.

2. **Image jadi/lokal-build?** `docker history <image>` — bandingкан dengan source.

3. **Source di host sehat?** Cek Dockerfile, dependency files (`package.json`, `requirements.txt`, lock files). Bandingкан `git diff` kalau di-version-control.

4. **Vektor terisolasi?** Cek dari scan-others apakah cuma 1 container atau banyak.

5. **Akses inbound?** Cek `ss -tlnp` apakah ada port yang harusnya tertutup tapi terekspos.

---

## 4. Cleanup (10-20 menit) — DESTRUKTIF, IRREVERSIBLE

**Lakukan SETELAH forensik tersimpan.**

### a. Hapus stack
```bash
cd /path/to/project
docker compose down                       # hapus container + network
# JANGAN -v kecuali volume juga kena (jarang)
```

### b. Hapus image yang terinfeksi (cek ID dari forensik)
```bash
docker image rm <image-name-atau-sha256>

# ragu? prune:
docker image prune -a                     # hapus semua image tak dipakai
```

### c. Audit deps sebelum rebuild
```bash
# Node
cd <project>/frontend && npm audit

# Python
cd <project>/backend && pip list --outdated && pip-audit  # kalau pip-audit terinstall
```

### d. Rebuild fresh dengan hardening (lihat [Part 2 — Hardening](./INCIDENT-REPORT-PART-2.md))
```bash
cd /path/to/project
docker compose build --no-cache <service>  # dari Dockerfile yang sudah pin digest + lockfile + ignore-scripts
```

---

## 5. Verifikasi image baru (5 menit) — SEBELUM REDEPLOY

```bash
# scan image baru utk IOC, TANPA jalankan CMD
docker run --rm --entrypoint /bin/sh <image> -c '
find / -maxdepth 6 \( \
    -name "qfhqe7cs.sh" -o -name "ahtyxvwm" -o -name "kubelet*" -o \
    -name "*.tvgudbkqwnex*" -o -name ".systemd-guard*" -o -name "systemd-udevd.service" \
\) 2>/dev/null
echo "/root:"; ls /root/
echo "/etc/cron.d:"; ls /etc/cron.d/
grep -E "nohup|tvgudbkqwnex" /root/.bashrc /root/.profile 2>/dev/null
'
# Kalau 0 hit + /root/ minimal + cron cuma e2scrub_all -> bersih
```

Baru `docker compose up -d`.

---

## 6. Post-Incident — wajib dilakukan

1. **Update incident memory / log internal** dengan tanggal, vektor, scope, cleanup, lessons.
2. **Update Dockerfile** kalau menemukan kelemahan baru (pin digest, `npm ci`, `--ignore-scripts`, lock files, dst).
3. **Update SOC rules** kalau ada IOC baru (Falco rules + Suricata rules).
4. **Tambah IOC ke deep-scan script** supaya scan berikutnya cek juga.
5. **Test pipeline SOC**: pause → notif → forensik flow harus selalu work.

---

## Filosofi Mode-2 (Reversible Containment)

**Yang boleh OTOMATIS:**
- `iptables DROP` (chain SOC_CONTAIN khusus, mudah di-flush)
- `kill -STOP` (freeze proses, reversible via `SIGCONT`)
- `docker pause` (container freeze, mudah di-unpause)
- Quarantine file (move ke `~/quarantine/` dengan backup, bukan delete)

**Yang HARUS minta konfirmasi manusia:**
- `kill -9` (irreversible)
- `docker rm` / `docker image rm` (irreversible)
- `rm` file (terutama di filesystem mounted)
- Modifikasi config sistem (`sshd_config`, `/etc/passwd`, dll)

**Pengaman wajib:**
- **Allowlist** proses kritis (yang TIDAK BOLEH disentuh otomasi: database, web server produksi, dll)
- **Rate-limit** aksi otomatis (max N aksi/menit — cegah loop)
- **Kill-switch** untuk mematikan seluruh active-response sementara
- **Audit log** tiap tindakan otomatis dengan timestamp

---

## Reference cepat

```bash
# Pause semua container project (containment cepat)
for c in $(docker ps --filter name=<prefix> --format '{{.Names}}'); do docker pause "$c"; done

# Cek overlay merged path
docker inspect <container> --format '{{.GraphDriver.Data.MergedDir}}'

# Kill switch SOC (kalau active-response salah)
sudo ~/soc/kill-switch.sh on
sudo ~/soc/kill-switch.sh off

# Restore container yang ke-pause
docker unpause <container>
```

---

## Pelajaran dari Juni 2026

1. **Bind port ke `127.0.0.1`**, NEVER `0.0.0.0` untuk service internal (vektor #1: Redis tanpa auth di `0.0.0.0` → RCE).
2. **Pin base image ke digest**, bukan tag mutable (vektor #2: image hasil `docker commit` attacker pakai tag yang sama).
3. **`npm ci --ignore-scripts`** untuk Node project (postinstall hook = supply-chain entry).
4. **`pip install --only-binary=:all:`** untuk Python (no `setup.py` execution).
5. **Verifikasi SHA256** untuk binari external yang di-download di Dockerfile.
6. **`docker compose up` BUKAN operasi aman** — selalu cek image fresh kalau ada kecurigaan.
7. **Container paused ≠ container dihapus** — Mode-2 reversible adalah golden path.
8. **SOC dipasang SEBELUM insiden**, bukan setelah — bedanya kerajaan vs ladang sampah.
