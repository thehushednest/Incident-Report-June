# Incident Report — June 2026

Laporan pasca-insiden keamanan pada DGX Spark `spark-1229`. Dua insiden cryptojacking dengan jeda enam hari — dengan dua pelajaran yang sangat berbeda.

---

## 📚 Dokumen

| # | File | Topik |
|---|---|---|
| 1 | **[INCIDENT-REPORT.md](./INCIDENT-REPORT.md)** | **7 Juni 2026** — Malam mesin saya dibajak. Cryptojacking via Redis tanpa autentikasi terekspos `0.0.0.0`. |
| 2 | **[INCIDENT-REPORT-PART-2.md](./INCIDENT-REPORT-PART-2.md)** | **13 Juni 2026** — Mereka tidak pernah pergi, mereka tertidur. Image Docker yang ter-poison via `docker commit` setelah insiden #1 — dibangunkan kembali oleh `docker compose up` rutin. SOC yang dipasang antara kedua insiden, menangkap mereka di menit pertama. |
| 3 | **[IOCs.md](./IOCs.md)** | Indicators of Compromise — file paths, build IDs, network IOCs, TTPs (MITRE ATT&CK), detection rules. Untuk konsumsi praktisi keamanan. |
| 4 | **[RUNBOOK.md](./RUNBOOK.md)** | Runbook operasional — *containment* reversible, forensik, analisa vektor, cleanup, verifikasi. Disarikan dari penanganan langsung. |

---

## 🎯 Yang ingin saya bagikan

Dua insiden, dua pengalaman yang berbeda. Tujuh hari saja jaraknya.

**Insiden pertama saya tulis dengan darah** — disk sekarat 100%, *load average* 37, satu malam berburu hantu. Tidak ada alarm. Yang menyadarkan saya cuma `df -h` di terminal.

**Insiden kedua saya tulis dengan tinta** — sore tenang, notifikasi pelan, satu cangkir kopi, dan 60 menit sejak alarm pertama sampai semua tertangani. Karena di antara keduanya, saya bangun **lapisan deteksi**.

Bukan kejeniusan keamanan. Cuma disiplin yang terlambat — tapi tidak terlambat *untuk* hari kedua.

Repo ini bukan ode untuk diri sendiri. Ini catatan dari seseorang yang pernah jatuh, lalu memutuskan jatuh sekali itu sudah cukup. Kalau ada satu hal yang ingin saya tinggalкan untuk siapa pun yang membaca:

> **Pasang detektormu sebelum kamu butuh. Lapisan pertahanan bukan paranoia — ia adalah perbedaan antara malam yang dirampas dan sore yang sekadar terganggu.**

---

## ⚠️ Catatan

- Seluruh kredensial yang terdampak telah dianggap bocor dan ditangani sesuai prinsip.
- Tidak ada *secret* aktif yang dimuat di repo ini.
- PII (IP publik, ID pribadi, dst) yang tidak relevan untuk pembelajaran sudah disanitasi atau diganti placeholder.
- Detail teknis yang dishare — IOC, *binary path*, deteksi rules, pola serangan — dimaksudкan sebagai bahan rujukan bagi komunitas keamanan.

---

— *thehushednest*
