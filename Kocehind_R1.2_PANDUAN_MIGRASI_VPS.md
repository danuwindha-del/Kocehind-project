# Panduan Migrasi VPS Kocehind R1.2

Panduan ini untuk memindahkan Kocehind ke VPS baru dengan IP publik berbeda tanpa menimpa identitas jaringan VPS tujuan. Jangan alihkan Cloudflare DNS sebelum pengujian selesai.

## Urutan sederhana

1. Siapkan VPS Ubuntu/Debian baru dan buka port yang dibutuhkan.
2. Ekstrak full source Kocehind v2.18.0 R1.2 dan jalankan `sudo bash scripts/vps2_instal_baru.sh`.
3. Salin arsip backup dan file `.sha256` ke VPS baru.
4. Verifikasi: `sudo kocehind-backup verify /root/restore/kocehind-full-....tar.gz`.
5. Restore dengan migration mode:

```bash
sudo kocehind-backup restore /root/restore/kocehind-full-....tar.gz \
  --confirm RESTORE-KOCEHIND --skip-safety-backup --migration-mode
```

6. Audit IP dan konfigurasi:

```bash
sudo kocehind-backup migrate-check --archive /root/restore/kocehind-full-....tar.gz
```

7. Jika backup dibuat oleh R1.2 dan satu IP publik sumber/tujuan terdeteksi, lihat dry-run:

```bash
sudo kocehind-backup migrate-auto --archive /root/restore/kocehind-full-....tar.gz
```

8. Jika daftar file yang akan berubah benar, terapkan dengan konfirmasi:

```bash
sudo kocehind-backup migrate-auto --archive /root/restore/kocehind-full-....tar.gz \
  --apply --confirm AUTO-MIGRATE-KOCEHIND --reload-nginx
```

Jika provider memakai NAT, tambahkan `--new-public-ip IP_PUBLIK_BARU`. Alternatifnya isi `PUBLIC_IP_OVERRIDE` di `/etc/kocehind/backup.conf` pada VPS tujuan.

## Backup R1.1 atau lebih lama

Backup lama tidak memiliki metadata IP sumber. Jalankan `migrate-check`, lihat kandidat IP, pastikan IP lama dari panel VPS lama, lalu lakukan dry-run:

```bash
sudo kocehind-backup migrate-apply --old-ip IP_LAMA --new-ip auto
```

Jika sudah benar, terapkan:

```bash
sudo kocehind-backup migrate-apply --old-ip IP_LAMA --new-ip auto \
  --apply --confirm 'REPLACE IP_LAMA WITH IP_BARU' --reload-nginx
```

`--new-ip auto` memerlukan tepat satu IP publik target yang terdeteksi. Jika NAT, ganti `auto` dengan IP publik dari panel provider.

## Yang dipertahankan pada VPS baru

Restore selalu mempertahankan runtime Backup Manager yang terpasang pada VPS tujuan. Dengan `--migration-mode`, restore juga tidak menimpa hostname, `/etc/hosts`, resolver, UFW, Fail2ban, SSH host keys, konfigurasi interface/network, dan `/etc/kocehind/backup.conf` milik target. Database, akun, saldo, media, LIVE, konfigurasi aplikasi, Nginx virtual hosts, dan TLS dipulihkan.

## Batas aman

- Penggantian IP hanya untuk file teks konfigurasi yang diizinkan.
- Loopback seperti `127.0.0.1` dan `::1` tidak boleh diganti.
- File yang diubah disalin ke safety archive terlebih dahulu.
- Nginx harus lolos `nginx -t`; jika gagal, file yang diubah dikembalikan.
- Cloudflare, firewall provider, IP allowlist payment gateway, dan webhook eksternal tidak diubah otomatis.
- Setelah VPS baru lolos tes, ubah record A/AAAA Cloudflare untuk `kocehind.web.id`, `api.kocehind.web.id`, `web.kocehind.web.id`, `admin.kocehind.web.id`, dan `vt.kocehind.web.id`.
- Jangan matikan VPS lama sampai login, video, LIVE, inbox, KOIN, pembayaran dan proses background tervalidasi.
