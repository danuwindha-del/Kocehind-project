# Instalasi backend dan seluruh layanan Kocehind 2.18.0 pada VPS baru

Gunakan paket asli **Kocehind_v2.18.0_Sampul_Profil_Email.zip** yang Anda kirim. Panduan ini untuk **Ubuntu 24.04 LTS x86_64, akses root, VPS khusus Kocehind yang belum mempunyai instalasi/data Kocehind**. Instalasi membuat database kosong. Data VPS yang mati tidak ikut pulih dari source.

Command disusun dari installer, unit systemd dan setup yang benar-benar ada pada paket asli. Tidak ada command instalasi yang telah dijalankan pada VPS Anda dalam percakapan ini. Jalankan per bagian dan hentikan bila ada error. Jika VPS cadangan sudah berisi Kocehind atau layanan penting lain, jangan menjalankan installer fresh ini di atasnya.

## 1. Periksa VPS dan unggah paket

Masuk SSH sebagai root. Jika memakai akun sudo, jalankan `sudo -i` dahulu.

```bash
cat /etc/os-release
uname -m
nproc
free -h
df -h /
```

Jika OS/arsitektur berbeda dari Ubuntu 24.04 x86_64, command model/Python perlu dicocokkan terlebih dahulu. Backend, encoding, LIVE, TensorFlow/PyTorch dan recording bersaing memakai resource. Dokumentasi LiveKit merekomendasikan setidaknya 4 CPU dan RAM 4 GB untuk satu instance Egress saja; itu bukan ukuran cukup untuk seluruh Kocehind beserta model AI pada server yang sama.

Unggah ZIP asli melalui SFTP ke `/root/Kocehind_v2.18.0_Sampul_Profil_Email.zip`. Sesuaikan nama berkas jika hasil unduhan Anda berbeda.

```bash
apt-get update
apt-get install -y unzip curl ca-certificates python3 python3-venv python3-pip
cd /root
unzip -n Kocehind_v2.18.0_Sampul_Profil_Email.zip
cd /root/Kocehind_v2.18.0_Sampul_Profil_Email
sha256sum -c SHA256SUMS.txt
unzip -n SOURCE_LENGKAP_v2.18.0.zip
export KOCEHIND_PACKAGE=/root/Kocehind_v2.18.0_Sampul_Profil_Email
export KOCEHIND_SRC="$KOCEHIND_PACKAGE/kocehind_realtime_v2"
```

Semua baris checksum harus OK. Variabel `KOCEHIND_PACKAGE` dan `KOCEHIND_SRC` dipakai pada bagian berikutnya. Jika membuka sesi SSH baru, jalankan kembali dua baris `export` tersebut.

## 2. DNS dan firewall provider

Arahkan record A berikut ke IPv4 publik VPS baru. Jika tidak memakai IPv6 pada server ini, pastikan AAAA lama tidak mengarahkan pengguna ke server lain.

| Host | Kegunaan |
|---|---|
| `kocehind.web.id` | Website utama |
| `api.kocehind.web.id` | API Android dan callback |
| `web.kocehind.web.id` | Web sosial |
| `admin.kocehind.web.id` | Admin |
| `vt.kocehind.web.id` | Tautan/bagikan yang disediakan source |
| `live.kocehind.web.id` | Signaling LiveKit |
| `turn.kocehind.web.id` | TURN |

Gunakan DNS-only untuk `live` dan `turn` bila pengelola DNS Anda juga menyediakan proxy HTTP. HTTP proxy biasa tidak menggantikan jalur media WebRTC/UDP.

Izinkan pada firewall provider dan firewall OS yang Anda pakai:

| Protokol/port | Fungsi |
|---|---|
| TCP port SSH Anda | Administrasi SSH |
| TCP 80, 443 | HTTP, sertifikat, HTTPS dan WebSocket |
| TCP 7881 | Media LIVE melalui TCP |
| UDP 7882 | Media LIVE; installer Kocehind memakai satu port UDP ini |
| UDP 3478 | TURN UDP |
| TCP 5349 | TURN TLS |

Daftar media di atas mengikuti konfigurasi installer Kocehind, bukan rentang port default LiveKit. Biarkan PostgreSQL 5432, Redis 6379, API internal 8000 dan signaling internal 7880 privat.

Jika UFW sudah digunakan, tambahkan aturan berikut. Jangan mengaktifkan firewall baru sebelum port SSH yang benar diizinkan.

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 7881/tcp
ufw allow 7882/udp
ufw allow 3478/udp
ufw allow 5349/tcp
ufw status
```

## 3. Instal backend dasar

```bash
cd "$KOCEHIND_SRC"
bash scripts/vps2_instal_baru.sh
curl -fsS http://127.0.0.1:8000/health
systemctl is-active postgresql redis-server nginx kocehind-api kocehind-streams kocehind-mycha kocehind-studio-worker
```

Installer memasang PostgreSQL, Redis, Nginx, FFmpeg, virtualenv, source backend, migrasi dan layanan dasar. `health` harus menunjukkan versi **2.18.0**. Secret JWT dan password database dibuat otomatis pada `/etc/kocehind/backend.env`; pertahankan nilainya.

Jika installer melaporkan instalasi sudah ada, jangan menghapus database atau file konfigurasi untuk memaksakan pemasangan. Pesan tersebut berarti kondisi server perlu ditangani sebagai instalasi existing.

## 4. Pasang web dan HTTPS

Sesudah DNS A sudah menuju VPS ini dan TCP 80/443 dapat dicapai:

```bash
cd "$KOCEHIND_SRC"
bash scripts/vps2_pasang_website.sh
certbot --nginx -d kocehind.web.id -d api.kocehind.web.id --redirect
certbot --nginx -d web.kocehind.web.id -d admin.kocehind.web.id -d vt.kocehind.web.id --redirect
nginx -t
systemctl reload nginx
```

Certbot meminta email dan persetujuan layanan saat diperlukan. Sertifikat `live` dan `turn` dibuat oleh installer LIVE pada bagian 6. Tombol unduh Android belum menyediakan APK sampai APK signed benar-benar dipublikasikan.

## 5. Aktifkan email/OTP dan isi Google OAuth

```bash
nano /etc/kocehind/backend.env
```

Tambahkan/isi baris berikut dengan nilai milik Anda. Jangan membuat baris duplikat untuk key yang sudah ada. Pertahankan `DATABASE_URL`, `JWT_SECRET`, `REDIS_URL` dan `MEDIA_ROOT` yang dibuat installer.

```dotenv
GOOGLE_WEB_CLIENT_ID='CLIENT_ID_WEB_DARI_GOOGLE'
WEB_PUBLIC_URL='https://web.kocehind.web.id'
WEB_ALLOWED_ORIGINS='https://web.kocehind.web.id,https://admin.kocehind.web.id'
WEB_COOKIE_SECURE=1
MAILJET_API_KEY='KEY_MAILJET_ANDA'
MAILJET_SECRET_KEY='SECRET_MAILJET_ANDA'
MAILJET_FROM_EMAIL='EMAIL_PENGIRIM_TERVERIFIKASI'
MAILJET_FROM_NAME='Kocehind'
STUDIO_RECORDING_ROOT=/var/lib/kocehind/private-live
LIVE_RECORDING_ENABLED=0
```

Google Client ID bukan Client Secret. Gunakan project Google/Firebase yang sesuai dengan aplikasi Android Anda. Mailjet memakai API HTTPS, bukan SMTP cPanel. Pengirim/domain Mailjet harus sudah diverifikasi dan record SPF/DKIM mengikuti petunjuk akun Mailjet.

Pasang unit email lalu validasi konfigurasi dalam subshell agar nilai environment CLI selalu terbaru:

```bash
install -m 644 "$KOCEHIND_SRC/backend/deploy/kocehind-email.service" /etc/systemd/system/
chown root:kocehind /etc/kocehind/backend.env
chmod 640 /etc/kocehind/backend.env
(
  set -a
  source /etc/kocehind/backend.env
  set +a
  cd /opt/kocehind/backend
  /opt/kocehind/venv/bin/python mailjet_preflight.py
)
systemctl daemon-reload
systemctl enable --now kocehind-email
systemctl restart kocehind-api
systemctl is-active kocehind-api kocehind-email
```

Preflight Mailjet menggunakan sandbox tanpa mengirim email. Pengiriman nyata tetap perlu diuji melalui pendaftaran dan OTP akun Anda. Tanpa Mailjet yang berfungsi, alur verifikasi akun tidak selesai.

## 6. Pasang server LIVE dan TURN

Pastikan `live` dan `turn` sudah menuju IPv4 VPS dan port pada bagian 2 dibuka. Jalankan:

```bash
read -rp 'IPv4 publik VPS baru: ' KOCEHIND_PUBLIC_IP
read -rp 'Email untuk sertifikat: ' KOCEHIND_CERT_EMAIL
cd "$KOCEHIND_SRC"
bash scripts/vps2_pasang_live.sh "$KOCEHIND_PUBLIC_IP" "$KOCEHIND_CERT_EMAIL"
systemctl is-active kocehind-live kocehind-live-worker
curl -fsS http://127.0.0.1:8000/live/config
```

Installer menyiapkan binary LiveKit versi yang dipin source, sertifikat, key/secret acak, konfigurasi webhook dan worker rekonsiliasi. Jangan mengganti key/secret itu secara manual untuk menebak konfigurasi lama. Uji host dan penonton pada dua jaringan berbeda setelah layanan siap.

## 7. Rekaman LIVE/Studio: pasang Docker dan Egress

Jika Docker belum terpasang, gunakan repository resmi Docker untuk Ubuntu. Bagian ini untuk VPS baru yang belum mempunyai Docker/container lain:

```bash
apt-get update
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
source /etc/os-release
cat > /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: ${UBUNTU_CODENAME:-$VERSION_CODENAME}
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
docker version
```

Jika Docker sudah tersedia, periksa instalasinya dan gunakan yang ada bila sesuai; jangan menghapus data/container untuk menjalankan bagian ini.

Setelah backend dan LIVE terpasang, akhiri LIVE yang sedang berjalan sebelum mengaktifkan Egress:

```bash
cd "$KOCEHIND_PACKAGE"
/opt/kocehind/venv/bin/python AKTIFKAN_REKAMAN_LIVE.py --check
/opt/kocehind/venv/bin/python AKTIFKAN_REKAMAN_LIVE.py
systemctl is-active docker kocehind-egress kocehind-studio-worker
```

Wrapper memakai versi image Egress tetap dari paket lalu menyimpan digest image. Rekaman hanya berlaku untuk LIVE baru setelah aktivasi. Uji satu siaran sampai selesai dan periksa hasil privat pada Studio; status `active` saja tidak membuktikan rekaman sudah berhasil.

## 8. Firebase push/notifikasi HP

Unduh service-account JSON dari project Firebase milik Anda dan unggah secara privat ke `/root/firebase-service-account.json`. Project harus cocok dengan Firebase pada aplikasi Android.

```bash
install -o root -g kocehind -m 640 /root/firebase-service-account.json /etc/kocehind/firebase-service-account.json
nano /etc/kocehind/backend.env
```

Isi:

```dotenv
FCM_PROJECT_ID='PROJECT_ID_FIREBASE_ANDA'
FCM_CREDENTIALS_FILE=/etc/kocehind/firebase-service-account.json
```

Kemudian:

```bash
cd "$KOCEHIND_SRC"
bash scripts/vps2_aktifkan_push.sh
systemctl restart kocehind-api
systemctl is-active kocehind-push
```

Service-account tidak dimasukkan ke APK/web. Worker aktif belum membuktikan notifikasi sampai ke HP; uji token perangkat dan dua tombol diagnosis di Pengaturan aplikasi.

## 9. Kipay, Google Play dan MychaAI

Unit pembayaran dan Play sudah dipasang oleh installer dasar. Aktifkan setelah credential layanan tersedia.

Kipay, mulai dengan sandbox:

```bash
cd "$KOCEHIND_PACKAGE"
/opt/kocehind/venv/bin/python SETUP_KIPAY.py --mode sandbox
/opt/kocehind/venv/bin/python SETUP_KIPAY.py --check
systemctl is-active kocehind-payments
```

Setup meminta API key dan webhook secret secara tersembunyi. Daftarkan callback `https://api.kocehind.web.id/payments/kipay/callback` pada project yang sama. Setelah checkout dan webhook sandbox terbukti, konfigurasi production memakai:

```bash
cd "$KOCEHIND_PACKAGE"
/opt/kocehind/venv/bin/python SETUP_KIPAY.py --mode production
```

Google Play: siapkan service-account dengan akses Android Publisher/Play Console untuk aplikasi Anda, lalu unggah ke `/root/play-service-account.json`:

```bash
cd "$KOCEHIND_PACKAGE"
/opt/kocehind/venv/bin/python SETUP_GOOGLE_PLAY.py --credentials /root/play-service-account.json --check
/opt/kocehind/venv/bin/python SETUP_GOOGLE_PLAY.py --credentials /root/play-service-account.json
systemctl is-active kocehind-play
```

Produk koin dan izin Play Console harus disiapkan pada akun Anda. Uji dari internal testing/license tester; JSON valid saja belum membuktikan pembelian berjalan.

MychaAI:

```bash
cd "$KOCEHIND_PACKAGE"
/opt/kocehind/venv/bin/python SETUP_MYCHAAI.py
/opt/kocehind/venv/bin/python SETUP_MYCHAAI.py --check
/opt/kocehind/venv/bin/python SETUP_MYCHAAI.py --test
systemctl is-active kocehind-mycha
```

Isi API key, Base URL HTTPS dan ID model yang benar dari akun provider Anda secara eksplisit saat setup. Nilai default di source bukan verifikasi bahwa akun/provider Anda memakai endpoint tersebut. `--test` memanggil provider untuk uji kecil.

## 10. Moderasi OpenNSFW2 untuk video dan LIVE

Bagian ini memasang runtime berat pada virtualenv terpisah. Backend/API tidak memakai TensorFlow pada proses HTTP-nya. LIVE harus sudah dikonfigurasi sebelum pemeriksaan monitor LIVE.

```bash
apt-get install -y libglib2.0-0 libgl1
python3 -m venv /opt/kocehind/moderation-venv
/opt/kocehind/moderation-venv/bin/pip install --upgrade pip
/opt/kocehind/moderation-venv/bin/pip install -r "$KOCEHIND_SRC/backend/requirements-moderation.txt"
/opt/kocehind/moderation-venv/bin/pip check
install -d -m 755 /opt/kocehind/backend/deploy
install -m 644 "$KOCEHIND_SRC/backend/deploy/prepare_nsfw_model.py" /opt/kocehind/backend/deploy/
install -m 644 "$KOCEHIND_SRC/backend/check_moderation_runtime.py" "$KOCEHIND_SRC/backend/enable_moderation.py" /opt/kocehind/backend/
install -d -o kocehind -g kocehind -m 750 /var/lib/kocehind/nsfw /var/lib/kocehind/private-live/moderation
install -o kocehind -g kocehind -m 640 "$KOCEHIND_SRC/backend/models/open_nsfw_weights.h5" /var/lib/kocehind/nsfw/open_nsfw_weights.h5
nano /etc/kocehind/backend.env
```

Tambahkan:

```dotenv
NSFW_WEIGHTS_PATH=/var/lib/kocehind/nsfw/open_nsfw_weights.h5
NSFW_WEIGHTS_SHA256=14ca261f48bdd88c1eecba96a761bd1579b523adae1b749b0a4ffd8b7ed8babe
```

Uji model/decoder sebagai user service:

```bash
(
  set -a
  source /etc/kocehind/backend.env
  set +a
  export KERAS_BACKEND=tensorflow TF_CPP_MIN_LOG_LEVEL=2 OMP_NUM_THREADS=2 TF_NUM_INTRAOP_THREADS=2 TF_NUM_INTEROP_THREADS=1
  cd /opt/kocehind/backend
  runuser -u kocehind -- /opt/kocehind/moderation-venv/bin/python deploy/prepare_nsfw_model.py
  runuser -u kocehind -- /opt/kocehind/moderation-venv/bin/python check_moderation_runtime.py
)
```

Jika preflight lulus:

```bash
install -m 644 "$KOCEHIND_SRC/backend/deploy/kocehind-moderation.service" "$KOCEHIND_SRC/backend/deploy/kocehind-live-safety.service" /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now kocehind-moderation kocehind-live-safety
systemctl is-active kocehind-moderation kocehind-live-safety
```

Model perlu waktu memuat dan mengirim heartbeat readiness. Setelah worker siap, aktifkan kebijakan melalui checker bawaan:

```bash
(
  set -a
  source /etc/kocehind/backend.env
  set +a
  cd /opt/kocehind/backend
  /opt/kocehind/moderation-venv/bin/python enable_moderation.py
)
```

Jika checker menyatakan worker belum siap, lihat log dan ulangi setelah readiness tersedia. Checker tidak mengaktifkan kebijakan bila readiness belum ada. Kelulusan preflight bukan ukuran akurasi model terhadap konten pengguna nyata.

## 11. Model tambahan: kekerasan/gore dan ujaran kebencian

```bash
cd "$KOCEHIND_SRC"
bash scripts/vps2_pasang_content_safety.sh
```

Script membuat virtualenv terpisah, mengunduh snapshot model sesuai revisi yang dipin dan memeriksa runtime. Catat path yang ditampilkan pada baris **Model siap di ...**. Setelah command selesai sukses, edit:

```bash
nano /etc/kocehind/backend.env
```

Tambahkan/isi menggunakan path hasil instalasi sebenarnya:

```dotenv
SAFETY_MODEL_DIR=/var/lib/kocehind/content-models/GANTI_DENGAN_FOLDER_YANG_DINYATAKAN_SIAP
MULTI_MODERATION_ENABLED=1
SAFETY_VISION_ENABLED=1
SAFETY_HATE_ENABLED=1
```

Jangan mengisi path folder yang pembuatannya gagal atau sekadar memilih folder paling baru. Uji dengan environment service terbaru:

```bash
(
  set -a
  source /etc/kocehind/backend.env
  set +a
  cd /opt/kocehind/backend
  runuser -u kocehind -- /opt/kocehind/content-safety-venv/bin/python check_content_runtime.py --require-vision
)
```

Sebelum menerapkan penindakan pada trafik publik, evaluasi ambang/hasil terhadap sampel berlabel yang sesuai. Evaluator ada pada `backend/evaluate_content_safety.py`; skor model bukan persentase akurasi. Tidak ada dataset pengguna yang dipulihkan atau diciptakan oleh panduan ini.

Untuk memasang kedua worker setelah runtime dan evaluasi siap:

```bash
install -m 644 "$KOCEHIND_SRC/backend/deploy/kocehind-content-safety.service" "$KOCEHIND_SRC/backend/deploy/kocehind-live-content-safety.service" /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now kocehind-content-safety kocehind-live-content-safety
systemctl restart kocehind-api
systemctl is-active kocehind-content-safety kocehind-live-content-safety
```

Model tambahan bergantung pada kapasitas CPU/RAM dan unduhan model. Jika pemasangan/preflight gagal, perbaiki error sebelum menganggap deteksi aktif. Jangan mengubah ambang menjadi angka tebakan untuk menyembunyikan kegagalan runtime.

## 12. Akun admin dan pemeriksaan seluruh layanan

Daftar melalui web/aplikasi, selesaikan OTP Mailjet, kemudian promosikan akun tersebut:

```bash
read -rp 'Email akun yang sudah Anda verifikasi: ' KOCEHIND_ADMIN_EMAIL
bash "$KOCEHIND_SRC/scripts/vps2_buat_admin.sh" promote --email "$KOCEHIND_ADMIN_EMAIL"
```

Pasang dan jalankan laporan integrasi dari paket asli:

```bash
install -m 644 "$KOCEHIND_PACKAGE/DIAGNOSA_INTEGRASI.py" /opt/kocehind/backend/DIAGNOSA_INTEGRASI.py
cd /opt/kocehind/backend
/opt/kocehind/venv/bin/python DIAGNOSA_INTEGRASI.py
curl -fsS https://api.kocehind.web.id/health
curl -fsS https://web.kocehind.web.id/api/health
curl -fsS https://admin.kocehind.web.id/api/health
```

Laporan menandai layanan yang perlu aktif berdasarkan konfigurasi. Setelah seluruh bagian sudah dikonfigurasi, pemeriksaan langsung dapat memakai:

```bash
systemctl is-active postgresql redis-server nginx docker \
  kocehind-api kocehind-email kocehind-streams kocehind-mycha \
  kocehind-studio-worker kocehind-live kocehind-live-worker kocehind-egress \
  kocehind-push kocehind-payments kocehind-play \
  kocehind-moderation kocehind-live-safety \
  kocehind-content-safety kocehind-live-content-safety
```

Layanan integrasi yang belum mempunyai credential tidak otomatis berfungsi. Gunakan log per layanan yang bermasalah, misalnya:

```bash
journalctl -u kocehind-api -n 80 --no-pager
journalctl -u kocehind-email -n 80 --no-pager
journalctl -u kocehind-live -u kocehind-live-worker -n 80 --no-pager
journalctl -u kocehind-moderation -u kocehind-live-safety -n 80 --no-pager
```

Uji end-to-end: daftar/login, OTP, unggah foto/video/sampul, media private, HLS, komentar/pesan WebSocket, push ke HP, pembayaran sandbox, LIVE dua jaringan, rekaman Studio dan moderasi yang Anda aktifkan. `systemctl active` dan health HTTP saja belum membuktikan semua integrasi bekerja.

## 13. Buat backup instalasi yang baru

```bash
umask 077
bash "$KOCEHIND_SRC/backend/deploy/backup_backend.sh" /root/kocehind-backups
tar -czf /root/kocehind-konfigurasi-baru.tar.gz -C / etc/kocehind
tar -czf /root/kocehind-rekaman-privat.tar.gz -C /var/lib/kocehind private-live
chmod 600 /root/kocehind-konfigurasi-baru.tar.gz /root/kocehind-rekaman-privat.tar.gz
```

Unduh salinannya ke perangkat/penyimpanan lain secara privat. Backup database/media bawaan belum mencakup konfigurasi dan rekaman privat, sehingga dua arsip tambahan disediakan. Backup yang dibuat saat aplikasi terus menerima penulisan belum menjadi snapshot gabungan database/file untuk perpindahan produksi.

APK/AAB signed dan signing key Android bukan hasil instalasi backend ini. Bila ingin menyediakan unduhan APK, publikasikan APK yang sudah ditandatangani melalui `scripts/vps2_pasang_apk.sh` atau alat Play yang sesuai dari paket. Kembalinya server tidak mengembalikan signing key yang hilang.

## Referensi dan batas verifikasi

- Command project diperiksa pada source asli 2.18.0, termasuk installer fresh, unit systemd dan setup masing-masing integrasi.
- Resolver dependensi OpenNSFW2 diperiksa dengan `pip --dry-run` pada Python 3.12; pemeriksaan ini tidak memuat model atau membuktikan inferensi pada VPS Anda.
- Panduan tidak mengeksekusi instalasi, menerbitkan sertifikat, mengirim email, mengaktifkan pembayaran atau memasang model pada VPS Anda dari percakapan ini.
- [Docker Engine untuk Ubuntu: repository resmi](https://docs.docker.com/engine/install/ubuntu/)
- [LiveKit: Egress terpisah dan kebutuhan resource](https://docs.livekit.io/transport/self-hosting/egress/)
- [LiveKit: port dan firewall](https://docs.livekit.io/transport/self-hosting/ports-firewall/)
