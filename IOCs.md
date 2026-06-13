# Indicators of Compromise — Cryptojacking Kit (Juni 2026)

Indicator teknis yang ditemukan saat insiden Cryptojacking di sebuah container Docker pasca *image* yang di-poison via `docker commit`. Dibagikan untuk pembelajaran komunitas keamanan.

**Threat family:** Otomatisasi miner toolkit, mirip pola Kinsing/TeamTNT.

---

## File / Path IOCs (di dalam container yang terinfeksi)

### Dropper & Payload
| Path | Catatan |
|---|---|
| `qfhqe7cs.sh` | Dropper utama (8-char random). **Self-delete** setelah eksekusi |
| `ahtyxvwm` (di working dir + `/root/`) | UPX-packed ELF amd64, 41 KB — dropper kompak |

### Runner Utama (Guard)
| Path | Catatan |
|---|---|
| `/root/rqpzexif/.tvgudbkqwnex` | *Runner* utama — dipanggil oleh semua persistence dengan flag `--guard` |

### Persistence Multi-Vector
| Lokasi | Tipe |
|---|---|
| `/etc/cron.d/systemd-udevd` | Cron sistem, disamarkan jadi service systemd |
| `/etc/systemd/system/systemd-udevd.service` | Service systemd palsu (mengaku komponen udev) |
| `/etc/systemd/system/.systemd-guard.service` | Service tersembunyi (titik di depan) |
| `/etc/init.d/systemd-udevd` | SysV init script |
| `/etc/profile.d/systemd-udevd.sh` | Shell login startup |
| `/root/.bashrc` (akhir file) | Injeksi `nohup /root/rqpzexif/.tvgudbkqwnex >/dev/null 2>&1 &` |
| `/root/.profile` (akhir file) | Injeksi sama dengan `.bashrc` |
| `/root/.config/autostart/systemd-udevd.desktop` | XDG autostart entry |
| `/tmp/.cron_*` (18× file, nama acak 8-char) | Mini-cron jobs (1 menit interval) panggil runner dgn `--guard` |

### Binary Pengelola (menyamar service legit)
| Path | Nama Samaran | Ukuran |
|---|---|---|
| `/var/tmp/kubelet393` | `kubelet` (Kubernetes) | 2.84 MB, statically-linked ELF amd64 |
| `/root/.local/bin/kubelet393` | Copy sama | (sama) |
| `/var/tmp/.redis-server322` | `redis-server` | 2.84 MB (hash sama dengan kubelet393) |
| `/var/tmp/.ksoftirqd-0320` | `ksoftirqd` (kernel thread!) | 5.49 MB |
| `/var/tmp/.rcu_bh495` | `rcu_bh` (kernel thread) | 5.49 MB (hash sama dengan ksoftirqd) |
| `/root/.cpcxpdmeajnv` & `.iizhlwxrgffm` | (random) | 2.55 MB, *with debug_info*, not stripped — likely runner |

### Build IDs (SHA1)
| Build ID | Binary |
|---|---|
| `3c90bfa968b8195b86900149dd341221ba9e358b` | `ahtyxvwm` (UPX-packed dropper) |
| `620c3639b20866ca93b9bfe7fd1518b131735cc5` | `kubelet393` / `.redis-server322` family |
| `f53de5a61070a31f1c0e24719048bfcd4f77d587` | `.cpcxpdmeajnv` / `.iizhlwxrgffm` family (runner) |

### File Lain (sisa investigasi)
| Path | Catatan |
|---|---|
| `/root/.local/bin/qemu_kubelet393_*.core` | *Core dump* dari miner (attacker tidak adapt ke ARM target) |
| `/usr/bin/qemu-x86_64-static/.nomedia` | Tampering — `qemu-x86_64-static` seharusnya biner, jadi direktori. Mencurigakan |

---

## Network IOCs

### Domain Pool Monero (dari DNS query)
- `pool.supportxmr.com` (151+ query dalam 5 menit)
- `fr02.supportxmr.com`
- `fr04.supportxmr.com`
- `pool-global.supportxmr.com`

### Pola Karakteristik
- Connect ke pool **belum tercapai** saat di-contain (cuma tahap DNS)
- DNS via UDP/53 ke DNS publik (8.8.8.8) dan ISP resolver
- Miner pakai pool yang resolusi via DNS — bukan IP hardcoded (lebih sulit di-block via firewall)

---

## Process Tree Signature

Saat aktif, *process tree* di dalam container:
```
npm run dev
└─ sh qfhqe7cs.sh                                            ← dropper (self-delete)
   └─ /usr/libexec/qemu-binfmt/x86_64-binfmt-P ./ahtyxvwm    ← entry (qemu emulation amd64 di arm64)
      ├─ thread x N (miner threads)
      └─ child workers
```

**Tanda kunci:**
- Eksekusi via `qemu-binfmt` = mismatch arsitektur (host ARM, payload amd64)
- Parent `npm run dev` = vektor masuk via Node.js runtime
- 35 menit *delay* sejak container start sebelum dropper muncul — anti-sandbox / timer

---

## Tactics, Techniques, Procedures (TTPs — MITRE ATT&CK)

| Phase | Technique | Bukti |
|---|---|---|
| Initial Access | T1525 — Implant Internal Image | Image `clipper-frontend:latest` di-poison via `docker commit` setelah kompromi awal |
| Execution | T1059.004 — Unix Shell | Dropper `qfhqe7cs.sh` |
| Execution | T1053.003 — Scheduled Task: Cron | 18× cron entries panggil runner |
| Persistence | T1543.002 — Create or Modify System Process: Systemd Service | `systemd-udevd.service` + `.systemd-guard.service` |
| Persistence | T1546.004 — Event Triggered Execution: Unix Shell Configuration Modification | Injeksi `.bashrc` / `.profile` |
| Defense Evasion | T1036.005 — Masquerading: Match Legitimate Name | Binary `kubelet`, `redis-server`, `ksoftirqd`, `rcu_bh` |
| Defense Evasion | T1564.001 — Hidden Files and Directories | Persistence file diawali `.` (`.systemd-guard.service`, `.cron_*`) |
| Defense Evasion | T1070.004 — Indicator Removal: File Deletion | Dropper `qfhqe7cs.sh` self-delete |
| Defense Evasion | T1027.002 — Obfuscated Files: Software Packing | UPX-packed dropper |
| Impact | T1496 — Resource Hijacking | Cryptojacking (Monero pool query) |

---

## Detection Rules

### Falco (yang menangkap)

```yaml
- rule: SOC Outbound To Mining Pool Port
  condition: outbound and fd.sport != 0 and fd.rport in (3333,4444,5555,7777,8888,9999,14433,14444,45560,45700)
            and fd.ip != "0.0.0.0" and fd.net != "127.0.0.0/8"
            and not fd.snet in (rfc_1918_addresses)
  priority: CRITICAL
  tags: [mining]

- rule: SOC Execute From Temp Dir
  condition: spawned_process and (proc.exepath startswith /tmp/ or
             proc.exepath startswith /var/tmp/ or proc.exepath startswith /dev/shm/)
  priority: WARNING
  tags: [dropper]
```

### Suricata (yang men-trigger alert)

```
# SOC custom DNS rules
alert dns $HOME_NET any -> any any (msg:"SOC DNS query to known mining pool";
    dns.query; content:"pool.minexmr"; nocase; sid:1000020;)
alert dns $HOME_NET any -> any any (msg:"SOC DNS query to supportxmr";
    dns.query; content:"supportxmr.com"; nocase; sid:1000022;)
```

ET Open ruleset juga matches: `ET MALWARE CoinMiner Domain in DNS Lookup`

---

## Mitigations

1. **Pin Docker base image ke digest** — bukan tag mutable (lihat hardening di Part 2)
2. **`npm ci --ignore-scripts`** — matikan postinstall hooks (Node)
3. **`pip install --only-binary=:all:`** — wheel-only, no `setup.py` exec (Python)
4. **Verifikasi SHA256** untuk binari external (Deno, dll)
5. **Container hardening** (next layer): `read_only`, `cap_drop: ALL`, `no-new-privileges`, non-root user
6. **Egress filtering**: block outbound ke domain mining pool dikenal di DNS/firewall
7. **Image scanning** (Trivy/Grype) di CI/CD pipeline
8. **Container runtime monitoring** (Falco/Tracee) untuk detect *drop-and-execute* pattern

---

<sub>Daftar IOC ini dibagikan untuk kemudahan deteksi komunitas. *Hash* dan *path* dapat berubah pada *variant* yang berbeda — gunakan sebagai *starting point*, bukan signature tetap.</sub>
