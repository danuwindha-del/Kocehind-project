# Kocehind Backup & Recovery Manager R1

Dokumen ini menjelaskan cara mencadangkan dan memulihkan data Kocehind v2.18.0,
termasuk akun pengguna, database sosial, saldo/ledger KOIN, media, rekaman LIVE,
website, source backend, konfigurasi rahasia, Nginx, TLS, dan service Linux.

> **Peringatan:** Restore adalah operasi destruktif. Database aktif akan diganti
dengan database dari backup dan file di lokasi Kocehind akan ditimpa. Selalu
verifikasi arsip, pastikan file backup aman, dan uji restore di VPS staging
sebelum menjalankannya di server produksi.

## 1. Cakupan backup

| Komponen | Cakupan |
|---|---|
| Akun dan autentikasi | Tabel akun, profil, password hash, sesi/token tersimpan di database |
| Sosial dan komunikasi | Follow, feed, komentar, like, inbox/pesan, notifikasi, aktivitas |
| KOIN dan pembayaran | Wallet, ledger, transaksi, pesanan, status pembayaran |
| Video/foto | `MEDIA_ROOT` (default `/var/lib/kocehind/media`) dan seluruh `/var/lib/kocehind` |
| LIVE | `STUDIO_RECORDING_ROOT` (default `/var/lib/kocehind/private-live`), replay, klip |
| Model dan aset runtime | Direktori model/moderasi yang berada di `/var/lib/kocehind` |
| Backend | `/opt/kocehind` kecuali virtualenv/staging yang dibangun ulang |
| Website | `/var/www/kocehind*`, rilis web dan file APK/link yang disimpan di sana |
| Rahasia dan konfigurasi | `/etc/kocehind`, kredensial provider, Firebase, KIPAY, Mailjet, Google Play |
| Web dan TLS | `/etc/nginx`, `/etc/letsencrypt`, `/etc/ssl/private` |
| Service Linux | Unit/drop-in `kocehind-*`, konfigurasi PostgreSQL, Redis, UFW dan Fail2ban |
| Database PostgreSQL | `pg_dump` custom format dan `pg_dumpall --globals-only` |
| Redis | Snapshot RDB opsional; konfigurasi Redis tetap dicadangkan |

Direktori virtualenv Python tidak disertakan karena harus dibuat ulang sesuai
OS/Python/arsitektur target. Source backend dan `requirements.txt` tetap dicadangkan.
`postgres-globals.sql` disimpan untuk referensi DBA; role aplikasi dibuat ulang
secara aman dari `DATABASE_URL` pada `backend.env` ketika restore.

## 2. Instalasi otomatis

Pada paket full source, jalankan sebagai root:

```bash
sudo bash backup_manager/install.sh
```

Manager juga dipasang otomatis oleh `backend/deploy/install_backend_ubuntu.sh`
(setelah backend dan website siap), dan dipastikan aktif sebelum backup pra-update
pada `scripts/vps2_update_aman.sh`.
Installer:

- memasang CLI ke `/usr/local/sbin/kocehind-backup`;
- memasang script runtime ke `/opt/kocehind/backup-manager`;
- membuat `/etc/kocehind/backup.conf` dengan permission `0600`;
- membuat folder `/var/backups/kocehind` dengan permission `0700`;
- mengaktifkan `kocehind-backup.timer` setiap hari sekitar pukul **02:00 waktu VPS**.

Instalasi timer **tidak langsung menjalankan backup pertama**. Buat backup manual
untuk memastikan ruang disk dan cakupan data sudah benar.

Periksa timer:

```bash
systemctl status kocehind-backup.timer --no-pager
systemctl list-timers kocehind-backup.timer
```

## 3. Backup manual

```bash
sudo kocehind-backup backup
```

Daftar backup:

```bash
sudo kocehind-backup list
sudo kocehind-backup status
```

Hasil default:

```text
/var/backups/kocehind/
  kocehind-full-YYYYMMDD-HHMMSS.tar.gz
  kocehind-full-YYYYMMDD-HHMMSS.tar.gz.sha256
```

Arsip dan checksum disimpan dengan permission `0600`; folder backup `0700`.
Salin arsip **beserta** file `.sha256` ke penyimpanan lain.

### Konsistensi dan downtime

Secara default `STOP_SERVICES=1`. Semua service `kocehind-*` yang sedang aktif
(dikecualikan service backup itu sendiri) dihentikan sementara selama dump dan
pengarsipan, lalu diaktifkan kembali sesuai keadaan sebelumnya. PostgreSQL dan
Redis tetap berjalan. Ini menjaga agar perubahan database/media tidak berlomba
dengan proses backup. Jika media sangat besar, proses dapat memakan waktu dan
website/API akan tidak tersedia selama pengarsipan.

Untuk mengurangi downtime, `STOP_SERVICES=0` dapat dipilih di
`/etc/kocehind/backup.conf`, tetapi snapshot database dan file media tidak lagi
terjamin berada pada satu titik waktu yang sama. Gunakan hanya jika menerima
risiko konsistensi tersebut.

### Retensi

Default `RETENTION_DAILY=7` dan `RETENTION_WEEKLY=4`: simpan tujuh backup terbaru,
lalu satu backup dari tiap minggu terdahulu hingga empat minggu. Retensi lokal
hanya menghapus file dengan pola nama backup Kocehind.

## 4. Verifikasi backup

```bash
sudo kocehind-backup verify \
  /var/backups/kocehind/kocehind-full-YYYYMMDD-HHMMSS.tar.gz
```

Verifikasi memeriksa checksum SHA-256 eksternal (jika sidecar tersedia), checksum
metadata internal, struktur arsip, dan integritas kompresi. Sidecar penting karena
melindungi seluruh byte arsip termasuk semua video/foto. Jangan memisahkan atau
mengubah nama arsip dan sidecar.

## 5. Enkripsi dan salinan off-site

Backup mencakup password hash, email, pesan, saldo, token/provider secrets,
private key TLS, dan kredensial pembayaran. File harus dianggap **sangat rahasia**.
Jangan mengunggah backup plaintext ke folder publik.

Pilihan A — rclone dengan remote bertipe `crypt` (disarankan untuk object storage):

1. Konfigurasikan rclone dan buat remote `crypt` yang membungkus remote tujuan.
2. Edit `/etc/kocehind/backup.conf`:

```ini
RCLONE_REMOTE=kocehind-crypt:kocehind-backups
```

3. Pastikan remote bisa diakses root dan lakukan uji upload/download manual.

Pilihan B — GPG public-key encryption:

1. Impor public key penerima ke keyring root (`gpg --import public-key.asc`).
2. Pastikan fingerprint penerima benar.
3. Atur `GPG_RECIPIENT=FINGERPRINT_ATAU_EMAIL` di `backup.conf`.
4. Backup berikutnya menjadi `.tar.gz.gpg`; plaintext dihapus setelah enkripsi.
5. Private key untuk decrypt harus disimpan di luar VPS dan tidak boleh ikut dibackup.

Jika GPG digunakan, perintah `verify` membutuhkan private key yang sesuai untuk
mendekripsi sementara arsip. Rclone remote dan GPG adalah pilihan; jangan mengaktifkan
keduanya tanpa memahami lapisan enkripsinya.

## 6. Restore pada VPS yang sama

Pastikan arsip dan sidecar benar, lalu:

```bash
sudo kocehind-backup verify /path/kocehind-full-YYYYMMDD-HHMMSS.tar.gz
sudo kocehind-backup restore /path/kocehind-full-YYYYMMDD-HHMMSS.tar.gz \
  --confirm RESTORE-KOCEHIND
```

Secara default manager membuat **safety backup** kondisi VPS sekarang di
`/var/backups/kocehind/pre-restore/` sebelum menimpa data. Pastikan ruang disk cukup.

Restore akan:

1. memverifikasi checksum eksternal dan internal serta struktur arsip;
2. membuat safety backup kondisi VPS saat ini (kecuali `--skip-safety-backup`);
3. menghentikan service Kocehind yang aktif, Redis, dan Nginx yang aktif;
4. mengembalikan media, rekaman LIVE, backend, website, rahasia, Nginx, TLS,
   dan konfigurasi service; `/etc/postgresql` tidak ditimpa secara default;
5. membuat ulang role PostgreSQL dari `DATABASE_URL` yang dipulihkan;
6. mengganti database dan menjalankan `pg_restore` + `ANALYZE`;
7. mengaktifkan Redis dan membersihkan DB cache lokal (kecuali snapshot Redis diminta),
   memvalidasi Nginx, lalu mengaktifkan service aplikasi yang tercatat aktif saat backup.

Jika restore gagal di tengah jalan, service aplikasi dibiarkan berhenti untuk
mencegah penulisan ke database/file yang belum pulih sempurna. Redis/Nginx yang
sebelumnya aktif dicoba dinyalakan kembali setelah validasi.

Redis tidak dipulihkan secara default karena isinya dapat berupa cache/sesi lama.
Secara default `/etc/postgresql` tidak ditimpa agar konfigurasi major version
VPS target tidak rusak. Hanya jika major version PostgreSQL sama dan Anda memang
ingin memulihkan konfigurasi PostgreSQL sumber, tambahkan `--restore-postgres-config`:

```bash
sudo kocehind-backup restore /path/kocehind-full-YYYYMMDD-HHMMSS.tar.gz \
  --confirm RESTORE-KOCEHIND --restore-postgres-config
```

Jika memang perlu memulihkan snapshot Redis:

```bash
sudo kocehind-backup restore /path/kocehind-full-YYYYMMDD-HHMMSS.tar.gz \
  --confirm RESTORE-KOCEHIND --restore-redis
```

Jangan gunakan `--restore-redis` kecuali konfigurasi Redis dan kebutuhan sesi lama
sudah dipahami.

## 7. Pemulihan ke VPS baru (R1.2)

Gunakan Ubuntu/Debian dan major version PostgreSQL yang kompatibel. Sediakan ruang disk cukup untuk database, media, backup, dan file sementara. Jangan alihkan DNS produksi sebelum server baru lulus pengujian.

1. Siapkan VPS baru dan firewall provider. Buka port yang diperlukan oleh Kocehind sesuai panduan deployment.
2. Ekstrak full source Kocehind v2.18.0 R1.2 pada VPS baru.
3. Jalankan instalasi backend dasar:

```bash
sudo bash scripts/vps2_instal_baru.sh
```

4. Salin arsip backup **dan** sidecar `.sha256` ke VPS baru, misalnya `/root/restore/`.
5. Verifikasi arsip:

```bash
sudo kocehind-backup verify /root/restore/kocehind-full-YYYYMMDD-HHMMSS.tar.gz
```

6. Pulihkan backup dengan **migration mode**. Pada VPS baru yang masih kosong, `--skip-safety-backup` dapat digunakan agar tidak membuat backup dari instalasi dasar yang belum berisi data pengguna:

```bash
sudo kocehind-backup restore \
  /root/restore/kocehind-full-YYYYMMDD-HHMMSS.tar.gz \
  --confirm RESTORE-KOCEHIND --skip-safety-backup --migration-mode
```

Migration mode juga mempertahankan runtime Backup Manager yang terpasang pada VPS tujuan agar restore backup lama tidak menurunkan versi CLI. Mode ini mempertahankan identitas VPS tujuan: hostname, `/etc/hosts`, resolver, firewall UFW, Fail2ban, SSH host keys, konfigurasi interface/network, dan `/etc/kocehind/backup.conf`. Database, media, konfigurasi aplikasi, rahasia Kocehind, Nginx virtual hosts, dan TLS tetap dipulihkan.

7. Jalankan audit migrasi:

```bash
sudo kocehind-backup migrate-check \
  --archive /root/restore/kocehind-full-YYYYMMDD-HHMMSS.tar.gz
```

Audit menampilkan IP sumber (jika metadata tersedia), alamat VPS tujuan, literal IP yang ditemukan pada konfigurasi arsip/target, serta daftar domain Cloudflare yang perlu diperiksa. Audit bersifat read-only.

8. Untuk backup R1.2 yang merekam tepat satu IP publik sumber, lakukan dry-run:

```bash
sudo kocehind-backup migrate-auto \
  --archive /root/restore/kocehind-full-YYYYMMDD-HHMMSS.tar.gz
```

Jika VPS memakai NAT dan IP publik tidak muncul di interface, isi `PUBLIC_IP_OVERRIDE` pada `/etc/kocehind/backup.conf` di VPS tujuan dengan IP publik dari panel provider, atau gunakan `--new-public-ip IP_BARU`. Setelah file yang akan diubah ditinjau, terapkan:

```bash
sudo kocehind-backup migrate-auto \
  --archive /root/restore/kocehind-full-YYYYMMDD-HHMMSS.tar.gz \
  --new-public-ip IP_BARU --apply --confirm AUTO-MIGRATE-KOCEHIND --reload-nginx
```

Untuk backup R1.1 atau lebih lama, metadata IP sumber mungkin tidak tersedia. Ambil IP lama dari panel provider VPS lama, lakukan dry-run, lalu apply secara eksplisit:

```bash
sudo kocehind-backup migrate-apply --old-ip IP_LAMA --new-ip auto
sudo kocehind-backup migrate-apply --old-ip IP_LAMA --new-ip auto \
  --apply --confirm 'REPLACE IP_LAMA WITH IP_BARU' --reload-nginx
```

Ganti teks `IP_LAMA` dan `IP_BARU` dengan angka IP yang sebenarnya pada perintah konfirmasi. Jika IP publik target berada di balik NAT, berikan IP target secara eksplisit menggunakan `--new-ip IP_BARU`.

9. Periksa layanan dan API:

```bash
sudo systemctl daemon-reload
sudo nginx -t
sudo systemctl --failed
curl -fsS http://127.0.0.1:8000/health
sudo kocehind-backup list
sudo kocehind-backup status
```

10. Uji login, profil, feed, video/foto, inbox, LIVE/replay, saldo KOIN, riwayat transaksi, upload, notifikasi, dan pembayaran sandbox. Uji domain ke IP baru sebelum mengalihkan DNS, misalnya dengan `curl --resolve` jika HTTPS sudah tersedia.

11. Setelah VPS baru sehat, perbarui record A/AAAA di Cloudflare untuk `kocehind.web.id`, `api.kocehind.web.id`, `web.kocehind.web.id`, `admin.kocehind.web.id`, dan `vt.kocehind.web.id`. DNS tidak diubah otomatis oleh Backup Manager. Periksa sertifikat HTTPS, payment-provider IP allowlist, webhook, SMTP, Firebase, dan layanan eksternal lainnya.

Jangan matikan VPS lama sampai trafik, login, media, pembayaran, dan proses latar belakang pada VPS baru sudah diverifikasi.

## 8. Pemeriksaan setelah restore

Checklist minimum:

- [ ] `curl http://127.0.0.1:8000/health` berhasil dan versi API sesuai.
- [ ] Seluruh service yang diperlukan berstatus active.
- [ ] Login dengan akun lama berhasil.
- [ ] Foto, avatar, dan video lama bisa diputar.
- [ ] Inbox dan notifikasi terbaca.
- [ ] Saldo KOIN dan ledger sesuai sebelum restore.
- [ ] LIVE/replay dapat diakses sesuai hak akses.
- [ ] Upload media baru berhasil.
- [ ] Nginx dan HTTPS valid.
- [ ] Pembayaran/webhook diuji menggunakan mode aman/sandbox.
- [ ] Backup otomatis berikutnya sukses dan salinannya berada di lokasi terpisah.

## 9. Batasan penting

- R1 ditujukan untuk PostgreSQL lokal pada VPS Kocehind; database eksternal ditolak.
- `postgres-globals.sql` disertakan untuk audit/pemulihan DBA; role aplikasi utama
  dibuat dari `DATABASE_URL`. Role/tablespace PostgreSQL tambahan harus ditinjau DBA.
- Virtualenv tidak disertakan; pasang ulang dependency dari `requirements.txt` pada
  VPS target yang kompatibel.
- Restore file konfigurasi PostgreSQL memerlukan OS/major version yang sesuai dan
  hanya dilakukan dengan opsi eksplisit `--restore-postgres-config`. Secara default
  konfigurasi PostgreSQL target tetap dipertahankan.
- Backup lokal bukan pengganti off-site backup. Simpan salinan terenkripsi di luar VPS.
- Uji restore di staging secara berkala. Backup yang belum pernah diuji restore
  belum boleh dianggap sebagai rencana pemulihan yang teruji.

## 10. Detail deteksi dan penyesuaian IP R1.2

- Backup R1.2 merekam hostname, alamat interface, rute default, dan alamat publik yang terlihat pada interface di `metadata/manifest.json`. Tidak ada panggilan ke layanan IP eksternal.
- Jika provider memakai NAT, alamat publik dapat tidak terlihat dari interface. Isi `PUBLIC_IP_OVERRIDE` pada `/etc/kocehind/backup.conf` di VPS sumber sebelum membuat backup untuk menyimpan IP publik yang ditampilkan panel provider. Di VPS tujuan, isi override baru hanya jika perlu. File `backup.conf` milik target dipertahankan saat restore migration mode.
- `migrate-check` adalah audit read-only. Untuk backup R1.1/lebih lama, IP sumber dapat ditinjau dari kandidat IP konfigurasi dalam arsip. Kandidat tidak otomatis dianggap IP VPS karena dapat berupa alamat layanan pihak ketiga.
- `migrate-auto` hanya berjalan jika metadata menunjukkan tepat satu IP publik sumber dan satu IP publik tujuan (atau tujuan diberikan eksplisit), serta IP lama benar-benar ditemukan pada file konfigurasi aktif yang diizinkan. Default-nya dry-run. Penerapan membutuhkan `--apply --confirm AUTO-MIGRATE-KOCEHIND --reload-nginx`.
- `migrate-apply` untuk backup lama memerlukan IP lama dan baru yang dipastikan operator. Default-nya dry-run; apply memerlukan konfirmasi teks persis. Loopback, unspecified, link-local, dan multicast dilindungi.
- Penggantian terbatas pada teks konfigurasi di `/etc/kocehind` (kecuali `backup.conf`), `/etc/nginx`, `/etc/livekit`, unit `kocehind-*`, file konfigurasi terpilih di `/opt/kocehind`, dan direktori website Kocehind. `/etc/hosts`, firewall, network config, SSH keys, database, sertifikat, media, dan Cloudflare tidak diubah.
- Sebelum apply, file yang akan diubah diarsipkan ke `/var/backups/kocehind/migration-safety/`. Nginx divalidasi dengan `nginx -t`; jika validasi gagal, perubahan file dikembalikan.
- Cloudflare DNS dan konfigurasi provider eksternal tetap diperbarui manual setelah pengujian. Ini disengaja agar trafik produksi tidak berpindah sebelum server baru siap.
