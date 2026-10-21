CAKRA AI — image installer
Repo ini adalah artefak publik untuk memasang CAKRA dari image private di Docker Hub. Source aplikasi tetap di repo private krmtwre/Cakra.

Installer tidak membangun source. Ia mengunduh bundle rilis, menarik image dari krmtwre/apps, menulis .env, lalu menjalankan Docker Compose.

Quick start (server baru)
Urutan singkat; detail tiap langkah ada di section setelah ini.

1. Cek host

docker --version && docker compose version
free -g && df -h /
Butuh Docker >= 24 + Compose plugin v2, bash >= 4, RAM >= 4 GB, disk bebas

= 20 GB, dan user dengan akses Docker. buildx tidak diperlukan.

2. Siapkan akses image

Buat Docker Hub PAT read-only untuk akun krmtwre (akses ke repo krmtwre/apps). PAT hanya dipakai saat docker pull; installer tidak menulisnya ke .env, argumen CLI, atau log.

3. Jalankan installer

curl -fsSL https://github.com/krmtwre/cakra-install/releases/download/vX/install.sh | bash
Butuh terminal interaktif (prompt dibaca dari /dev/tty). Untuk folder lain: bash -s -- --dir /opt/cakra. Ganti vX dengan tag rilis nyata, mis. v1.0.0-rc.1 — pakai versi spesifik, bukan latest.

4. Jawab prompt

Nama project → mode akses (1 HTTP langsung, 2 HTTPS di nginx, 3 HTTPS di atas nginx/Cloudflare/LB) → IP bind → port → domain → konfirmasi → PAT (tersembunyi).

5. Catat output

Installer mencetak URL publik, Swagger, dan password awal admin, operator, auditor. Password hanya ditampilkan sekali di sini.

Tanpa interaksi (CI/otomasi) — variabel harus sampai ke bash, bukan ke curl:

export DOCKER_PAT='<pat-read-only>'
export CAKRA_MODE=http CAKRA_DOMAIN=cakra.contoh.id CAKRA_HTTP_PORT=8080
curl -fsSL https://github.com/krmtwre/cakra-install/releases/download/vX/install.sh \
  | bash -s -- --yes --dir /opt/cakra
Operasional setelah terpasang:

cd /opt/cakra
./scripts/backup.sh                 # pg_dump + arsip S3 + .env terenkripsi
./scripts/restore.sh backups/<dir>  # restore
./scripts/upgrade.sh vY             # ganti versi; data & rahasia tetap
./scripts/uninstall.sh [--purge]    # stop, atau hapus container + volume
Jebakan yang sering kena:

Instalasi baru menolak jalan bila volume <project>_cakra_pgdata atau <project>_cakra_s3data sudah ada — pilih --dir/project lain, atau hapus volume itu setelah memastikan datanya tidak dibutuhkan.

--reconfigure hanya untuk versi yang sama; beda versi → --upgrade dulu.

Produksi: verifikasi checksum sebelum eksekusi.

curl -fsSLO <base>/install.sh
curl -fsSLO <base>/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing && bash install.sh
Login Docker Hub memakai config sementara dan dibersihkan saat exit; config Docker host tidak berubah kecuali memakai --keep-login.

Requirement
Linux x86_64 atau arm64, bash >= 4.
Docker >= 24 dengan Compose plugin v2 (tanpa buildx).
User dengan akses Docker (grup docker); hindari root.
curl, tar, sha256sum, openssl, awk, sed, grep.
RAM >= 4 GB (disarankan 8 GB), disk bebas >= 20 GB.
Port yang dipilih bebas.
Docker Hub PAT read-only untuk akun krmtwre.
Cek requirement tanpa memasang:

curl -fsSL https://github.com/krmtwre/cakra-install/releases/download/vX/install.sh \
  | bash -s -- --requirements
Instalasi
curl -fsSL https://github.com/krmtwre/cakra-install/releases/download/vX/install.sh \
  | bash
Default folder: $PWD/cakra. Ganti dengan --dir:

curl -fsSL .../install.sh | bash -s -- --dir /opt/cakra
Installer menanyakan:

Folder dan nama project Compose.
Mode akses:
HTTP langsung — nginx melayani HTTP di port pilihan.
HTTPS di nginx — cert + key sudah ada di server.
HTTPS di atas nginx/Cloudflare/LB — origin HTTP, URL publik HTTPS.
Domain, IP bind, dan port.
Docker Hub PAT read-only.
PAT tidak ditulis ke .env, tidak masuk argumen CLI, dan tidak dicetak. Login dilakukan memakai config Docker sementara: kredensial ditulis ke direktori temporer dan dihapus saat installer selesai, jadi config Docker host tidak tersentuh. --keep-login menyimpan login di config host untuk pemakaian ulang.

Mode non-interaktif
Gunakan --yes dan sediakan PAT lewat environment:

DOCKER_PAT='<read-only-pat>' \
  curl -fsSL .../install.sh | bash -s -- --yes --dir /opt/cakra
Nilai lain dapat dioverride lewat environment: CAKRA_MODE=http|https|above, CAKRA_DOMAIN, CAKRA_BIND_IP, CAKRA_HTTP_PORT, CAKRA_HTTPS_PORT, CAKRA_TLS_CERT, CAKRA_TLS_KEY. Untuk HTTPS, cert dan key harus ada.

Opsi installer
Opsi	Arti
--yes, -y	Pakai default/env, tanpa prompt. PAT tetap diperlukan.
--dir DIR	Folder instalasi (default $PWD/cakra).
--reconfigure	Ubah domain/port/mode. Image dan rahasia dipertahankan.
--upgrade	Ganti ke versi installer ini. Rahasia/data dipertahankan.
--keep-login	Simpan login di config Docker host (default: config sementara yang dihapus).
--bundle-dir DIR	Pakai bundle lokal (pengujian/offline).
--requirements	Tampilkan requirement.
--version	Tampilkan versi installer.
Setelah instalasi
<folder>/
├── .env                 # rahasia, mode 600 — jangan commit
├── docker-compose.yml   # image-only, tanpa build
├── install.sh           # installer versi ini
├── images.env           # ref image + digest
├── images.lock
├── nginx/
│   ├── http.conf.template
│   ├── default.conf.template
│   └── ssl/options-ssl-params.conf
└── scripts/
    ├── backup.sh
    ├── restore.sh
    ├── upgrade.sh
    └── uninstall.sh
Akun awal dibuat backend saat pertama start. Password ada di .env (DEFAULT_ADMIN_PASSWORD, DEFAULT_OPERATOR_PASSWORD, DEFAULT_AUDITOR_PASSWORD) dan dicetak sekali oleh installer. Ganti saat login pertama.

Upgrade
cd /opt/cakra
./scripts/upgrade.sh vX.Y.Z-rc.N
Atau langsung:

curl -fsSL https://github.com/krmtwre/cakra-install/releases/download/vX/install.sh \
  | bash -s -- --upgrade --dir /opt/cakra
Installer membackup .env, compose, dan nginx/ ke backups/config-*, menarik image baru, lalu menjalankan health check. Data volume tidak dihapus. upgrade.sh mengunduh install.sh rilis dan memverifikasi checksum-nya (SHA256SUMS) sebelum menjalankannya. Backup data penuh tetap disarankan:

./scripts/backup.sh
Rollback
Jika upgrade gagal, installer mencetak perintah rollback. Ringkasnya:

cp backups/config-<timestamp>/.env .env
cp backups/config-<timestamp>/docker-compose.yml docker-compose.yml
docker compose up -d --no-build --remove-orphans
Untuk rollback versi image, jalankan installer versi lama dengan --upgrade.

Backup dan restore
./scripts/backup.sh                  # backups/<timestamp>/
./scripts/backup.sh /path/backup     # folder tujuan
./scripts/backup.sh --skip-s3        # tanpa arsip objek S3
Backup berisi dump PostgreSQL, arsip volume SeaweedFS, .env terenkripsi AES-256-CBC/PBKDF2, manifest, dan checksum. Passphrase lewat BACKUP_PASSPHRASE atau prompt; tidak disimpan.

./scripts/restore.sh backups/<timestamp>
Restore menimpa database dan volume S3. .env aktif tidak ditimpa; hasil dekripsi ditulis ke .env.restored untuk dibandingkan manual.

Uninstall
./scripts/uninstall.sh           # stop container, volume tetap
./scripts/uninstall.sh --purge   # stop + hapus volume (DATA HILANG)
Mode HTTPS
HTTPS di nginx
Siapkan cert dan key di server, misalnya:

/etc/cakra/tls/fullchain.pem
/etc/cakra/tls/privkey.pem
Installer memount folder cert ke gateway dan memakai nginx/default.conf.template. Port 80 dipakai untuk redirect ke HTTPS.

HTTPS di atas nginx/Cloudflare/LB
Pilih mode 3. Origin tetap HTTP di port yang dipilih; SSL diterminasi di Cloudflare/tunnel/LB. Jangan publikasikan origin HTTP. S3_PUBLIC_ENDPOINT dan COOKIE_SECURE diset mengikuti URL publik HTTPS.
