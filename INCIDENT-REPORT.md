# 🩸 Laporan Insiden — Malam Mesin Saya Dibajak

**Tanggal kejadian:** 7 Juni 2026
**Sistem:** NVIDIA DGX Spark — `spark-1229` (ARM64 / GB10)
**Penulis:** thehushednest
**Klasifikasi:** Kompromi sistem — Cryptojacking via container terekspos
**Status akhir:** Tertangani. Terkontainmen. Tidak ada eskalasi ke host.

---

## I. Prolog — Tanda Pertama

Malam itu mesin saya megap-megap.

Bukan kiasan. Satu per satu service yang biasanya patuh mulai berjatuhan — gagal *start*, gagal nulis log, gagal hidup. `~/.claude.json` saya bahkan terpotong jadi **nol byte**, seolah ada tangan tak kasat mata yang meremas mesin ini sampai kehabisan napas.

Saya tarik satu perintah. `df -h`.

```
/dev/nvme0n1p2   3.7T   3.7T   0   100%   /
```

**Penuh. Seratus persen.** Disk 3,7 terabyte yang kemarin masih lega, malam itu tersumbat sampai ke tetes terakhir. Sesuatu sedang menulis ke mesin saya — cepat, rakus, dan tidak diundang.

Beban sistem yang biasanya mendekati nol, malam itu menyentuh **load average 37**. Dua puluh inti prosesor mesin riset saya menjerit di bawah beban yang bukan milik saya.

Ada penumpang gelap di dalam mesin ini. Dan dia sedang menambang.

---

## II. Perburuan

Saya buka `top`. Di sanalah dia — sebuah proses berjalan **sebagai root**, di dalam sebuah container Docker, menyamar di balik emulasi `qemu` arsitektur x86 di mesin ARM saya. Penyusup yang cukup cerdas untuk membawa senjatanya sendiri ke medan yang asing.

Dua biner tersembunyi, namanya dirancang agar terlihat seperti bawaan sistem:

- `.smartd838`
- `.automount389`

Titik di depan nama, angka acak di belakang — kamuflase klasik. Tapi tidak ada `smartd` yang menambang Monero. Tidak ada `automount` yang mengisi 2,7 terabyte sampah ke *writable layer* container dan *build cache* dalam hitungan jam.

Saya ikuti jejaknya mundur. Dari proses, ke container, ke port, ke pintu yang terbuka.

Dan di sanalah saya menemukannya — luka yang saya buat sendiri.

---

## III. Pintu yang Saya Tinggalkan Terbuka

Tujuh jam sebelum mesin mulai sekarat, saya menggelar sebuah *stack* eksperimen bernama **`clipper`** — perkakas AI pemotong video. Dalam terburu-buru, file `docker-compose` menerbitkan port-port internalnya bukan ke `127.0.0.1`, melainkan ke **`0.0.0.0`** — ke seluruh dunia.

Di antaranya, yang paling fatal:

```
clipper-redis  →  0.0.0.0:6381   (TANPA password)
```

Sebuah Redis tanpa autentikasi, telanjang menghadap internet.

Bagi penyerang otomatis yang menyapu internet siang-malam, ini bukan sekadar pintu terbuka — ini undangan dengan karpet merah. Teknik bukunya: sambung ke Redis, tulis muatan, suruh Redis menulis muatan itu ke disk sebagai *cron* atau *module*, lalu — **eksekusi**. *Remote Code Execution* klasik yang umurnya sudah belasan tahun, tetap mematikan karena kita, manusianya, tetap saja lalai.

Mereka masuk lewat celah itu. Menanam penambang. Dan membiarkannya melahap mesin saya sampai disk berteriak penuh.

---

## IV. Garis yang Tidak Berhasil Mereka Lewati

Di sinilah cerita berbelok dari bencana menjadi pelajaran.

Saya periksa konfigurasi container itu dengan tangan gemetar — bukan karena takut, tapi karena marah pada diri sendiri. Lalu saya membaca sesuatu yang membuat saya bisa bernapas lagi:

- Container itu **tidak** *privileged*.
- Container itu **tidak** memetakan `docker.sock`.
- Container itu **tidak** menambatkan satu pun direktori host.

Artinya: penyerang punya *root* — **tapi hanya root di dalam tempurung container.** Dinding isolasi Docker menahan mereka. Mereka mengamuk di dalam sel, tapi tidak pernah memegang kunci rumah.

Saya buktikan satu per satu, sampai keringat dingin reda:

| Yang Saya Periksa | Putusan |
|---|---|
| Kunci SSH asing di `authorized_keys` | ❌ Tidak ada |
| *Crontab* root & user yang ditanami | ❌ Bersih |
| Unit `systemd` jahat yang baru | ❌ Tidak ada |
| `/etc/ld.so.preload` (rootkit klasik) | ❌ Kosong |
| Akun pengguna baru / ber-UID 0 | ❌ Tidak ada |
| Sisa biner penambang di seluruh disk | ❌ Lenyap bersama container |
| Stack lain (`senopati-*`, `iris-*`) | ✅ Tak tersentuh — terikat `127.0.0.1`, jaringan terpisah, kredensial unik |
| Antarmuka publik (open-webui) | ✅ Bersih — tanpa akun baru, pendaftaran tertutup, tanpa kunci cloud |

**Tidak ada kredensial cloud yang bocor. Tidak ada proyek lain yang dijebol. Tidak ada pintu belakang yang tertinggal.**

Penyerang menang satu pertempuran kecil di satu container. Mereka tidak pernah menyentuh kerajaan.

---

## V. Pembersihan

Saya tidak tidur sampai mesin ini bersih.

1. **Eksekusi balik** — penambang dihentikan paksa; lima container `clipper` (frontend, backend, minio, redis, litellm) di-`stop` lalu di-`rm` tanpa ampun.
2. **Reklamasi** — `docker system prune` mengembalikan **248,6 GB** *build cache* yang dirampas penyerang.
3. **Ruang darurat** — log Docker, `syslog`, dan `kern.log` dipangkas; `journalctl` di-*vacuum* ke 200 MB; *reserved block* sempat diturunkan ke 1% demi sekejap napas, lalu **dipulihkan ke 5%** setelah badai reda.
4. **Memadamkan yang sekarat** — sebuah service yang *crash-loop* **2.748 kali** dimatikan; satu service usang lain yang menunjuk ke proyek yang sudah lama saya hapus juga dihentikan.
5. **Forensik** — *volume* Redis & MinIO milik `clipper` **sengaja saya simpan**, beku, untuk analisis lanjutan.

Hasil akhir, diverifikasi dengan mata kepala sendiri:

```
/dev/nvme0n1p2   3.7T   545G   3.0T   16%   /
```

Penambang hilang dari daftar proses. Disk bernapas lega. Mesin hidup kembali.

---

## VI. Memperkuat Benteng

Stack `clipper` saya bangun ulang dari nol dengan disiplin yang seharusnya saya pegang sejak awal:

- ✅ **Semua port** hanya menghadap `127.0.0.1` — tidak ada lagi yang telanjang ke internet.
- ✅ **Redis wajib berkata-sandi** — tidak ada lagi pintu tanpa kunci.
- ✅ **Rahasia kuat & acak** — kata sandi 5 karakter yang memalukan itu sudah saya kubur.

---

## VII. Dua Pengakuan yang Tidak Saya Sembunyikan

Laporan yang jujur tidak berhenti di kemenangan. Ada dua luka yang masih menganga, dan saya catat di sini agar tidak pernah saya lupakan:

1. **`fail2ban` saya buta.** Ada **12.613 percobaan tembus SSH** tercatat — namun penjaga gerbang saya melaporkan **nol** pelanggaran, **nol** pemblokiran. Karena semua serangan datang ter-*NAT* lewat gateway, IP itu masuk daftar tepercaya, dan setiap pukulan lolos tanpa tercatat. Penjaga yang menatap ke arah yang salah.

2. **SSH masih menerima kata sandi.** Selama itu masih hidup, gerombolan bot akan terus mengetuk. Pertahanan sejati hanya satu: **kunci publik saja, password mati.**

Keduanya akan ditutup. Bukan besok. Segera.

---

## VIII. Epilog — Pelajaran yang Ditulis dengan Darah

Penyerang itu tidak jenius. Mereka tidak memecahkan kriptografi saya, tidak menemukan *zero-day*, tidak menembus dinding yang kokoh.

Mereka cuma menemukan **satu pintu yang saya lupa kunci.**

Itulah seluruh kisahnya. Sebuah Redis tanpa sandi, terbuka ke internet selama tujuh jam, di tengah sebuah eksperimen yang saya kira "cuma sebentar". Tujuh jam sudah cukup bagi mesin internet untuk menemukannya, menjebolnya, dan memerasnya.

> **Jangan pernah, sekali pun, menerbitkan layanan backend ke `0.0.0.0`.**
> `127.0.0.1` plus *reverse proxy* berautentikasi. Selalu. Tanpa pengecualian. Bahkan — terutama — saat Anda hanya "mencoba sebentar".

Mesin ini bertahan karena lapisan-lapisan pertahanan lain berdiri tegak: isolasi container, pengikatan `127.0.0.1` di semua stack lain, kredensial yang tidak pernah dipakai ulang. Pertahanan berlapis bukan paranoia — ia adalah alasan mengapa malam itu berakhir dengan laporan ini, bukan dengan kehancuran.

Saya kehilangan satu malam. Saya tidak kehilangan kerajaan.

Dan lain kali, pintu itu akan terkunci sebelum saya berpaling.

— *thehushednest*

---

<sub>Dokumen ini ditulis sebagai catatan pasca-insiden dan bahan pembelajaran. Seluruh kredensial yang terdampak telah dianggap bocor dan ditangani sesuai prinsip. Tidak ada rahasia aktif yang dimuat dalam laporan ini.</sub>
