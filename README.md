# CAKRA AI — image installer

Artefak publik untuk memasang **CAKRA AI** dari image private di Docker Hub.
Source aplikasi tetap di repo private [`krmtwre/Cakra`](https://github.com/krmtwre/Cakra).

Installer **tidak** membangun source. Ia mengunduh bundle rilis, menarik image
dari `krmtwre/apps`, menulis `.env`, lalu menjalankan Docker Compose.

- Image: `krmtwre/apps` — private, di-pin per tag **dan** digest
- Rilis & checksum: halaman [Releases](https://github.com/krmtwre/cakra-install/releases)
- Tidak perlu python / node / postgres / redis / seaweedfs / nginx di host

## Quick start

### 1. Cek host

```bash
docker --version && docker compose version
free -g && df -h /
```

Butuh Docker ≥ 24 + Compose plugin v2 (**buildx tidak diperlukan**), bash ≥ 4,
RAM ≥ 4 GB, disk bebas ≥ 20 GB, dan user dengan akses Docker (grup `docker`).

### 2. Siapkan akses image

Buat Docker Hub PAT **read-only** untuk akun `krmtwre` dengan akses ke repo
`krmtwre/apps`. PAT hanya dipakai saat `docker pull`; installer tidak
menulisnya ke `.env`, argumen CLI, atau log.

### 3. Jalankan installer

```bash
curl -fsSL https://github.com/krmtwre/cakra-install/releases/download/v1.0.0-rc.1/install.sh | bash
```

Ganti `v1.0.0-rc.1` dengan tag rilis yang ingin dipakai — selalu versi
spesifik. Prompt dibaca dari `/dev/tty`, jadi jalankan di terminal interaktif.

Instal ke folder lain (default `$PWD/cakra`):

```bash
curl -fsSL <installer-url> | bash -s -- --dir /opt/cakra
```

### 4. Jawab prompt

1. Nama project Compose
2. Mode akses — `1` HTTP langsung, `2` HTTPS di nginx, `3` HTTPS di atas
   nginx/Cloudflare/LB
3. IP bind (default `0.0.0.0`)
4. Port HTTP (dan HTTPS untuk mode 2)
5. Domain
6. Konfirmasi
7. Docker Hub PAT (input tersembunyi, tidak tampil di layar)

### 5. Catat output

Installer mencetak URL publik, Swagger, dan password awal `admin`, `operator`,
`auditor`. **Password hanya ditampilkan sekali** di sini; salin sebelum menutup
terminal.

## Requirement

| Komponen | Kebutuhan |
| :--- | :--- |
| OS | Linux x86_64 atau arm64, `bash` ≥ 4 |
| Docker | Engine ≥ 24 + Compose plugin v2 (tanpa buildx) |
| User | Akses Docker; hindari menjalankan sebagai root |
| Tools | `curl`, `tar`, `sha256sum`, `openssl`, `awk`, `sed`, `grep` |
| Resource | RAM ≥ 4 GB (disarankan 8 GB), disk bebas ≥ 20 GB |
| Jaringan | Port yang dipilih bebas |
| Registry | Docker Hub PAT read-only untuk `krmtwre` |

Cek requirement tanpa memasang apa pun:

```bash
curl -fsSL <installer-url> | bash -s -- --requirements
```

## Opsi installer

| Opsi | Arti |
| :--- | :--- |
| `--yes`, `-y` | Non-interaktif: pakai default/env. PAT tetap diperlukan. |
| `--dir DIR` | Folder instalasi (default `$PWD/cakra`). |
| `--reconfigure` | Ubah domain/port/mode. Image dan rahasia dipertahankan. |
| `--upgrade` | Ganti ke versi installer ini. Rahasia dan data dipertahankan. |
| `--keep-login` | Simpan login Docker di config host (default: config sementara, dihapus saat selesai). |
| `--bundle-dir DIR` | Pakai bundle lokal (pengujian/offline). |
| `--requirements` | Tampilkan requirement lalu keluar. |
| `--version` | Tampilkan versi installer lalu keluar. |

## Instalasi non-interaktif

Variabel environment harus sampai ke `bash`, bukan hanya ke `curl`:

```bash
export DOCKER_PAT='<pat-read-only>'
export CAKRA_MODE=http CAKRA_DOMAIN=cakra.contoh.id CAKRA_HTTP_PORT=8080
curl -fsSL <installer-url> | bash -s -- --yes --dir /opt/cakra
```

Variabel yang dikenali: `DOCKER_PAT`, `CAKRA_MODE` (`http` | `https` | `above`),
`CAKRA_DOMAIN`, `CAKRA_BIND_IP`, `CAKRA_HTTP_PORT`, `CAKRA_HTTPS_PORT`,
`CAKRA_CERT_DIR`, `CAKRA_TLS_CERT`, `CAKRA_TLS_KEY`.

## Mode akses

### HTTP langsung

nginx melayani HTTP di port yang dipilih. Tidak ada persiapan tambahan.

### HTTPS di nginx

Sertifikat harus sudah ada di server sebelum installer dijalankan, misalnya:

```
/etc/cakra/tls/fullchain.pem
/etc/cakra/tls/privkey.pem
```

Installer memount folder sertifikat ke gateway dan memakai
`nginx/default.conf.template`. Port 80 dipakai untuk redirect ke HTTPS.
Installer memvalidasi path sertifikat; ia tidak membuat sertifikat.

### HTTPS di atas nginx / Cloudflare / LB

Pilih mode `3`. Origin tetap HTTP di port yang dipilih, TLS diterminasi di
Cloudflare/tunnel/LB. **Jangan publikasikan origin HTTP.**
`S3_PUBLIC_ENDPOINT` dan `COOKIE_SECURE` mengikuti URL publik HTTPS yang
dipilih.

## Setelah instalasi

```
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
```

Akun awal dibuat backend saat start pertama. Password-nya ada di `.env`
(`DEFAULT_ADMIN_PASSWORD`, `DEFAULT_OPERATOR_PASSWORD`,
`DEFAULT_AUDITOR_PASSWORD`) dan dicetak sekali oleh installer. Ganti saat login
pertama.

## Operasional

```bash
cd /opt/cakra

./scripts/backup.sh                 # pg_dump + arsip S3 + .env terenkripsi
./scripts/restore.sh backups/<dir>  # restore database + volume S3
./scripts/upgrade.sh vX.Y.Z-rc.N    # ganti versi; data & rahasia tetap
./scripts/uninstall.sh              # stop container, volume tetap
./scripts/uninstall.sh --purge      # stop + hapus volume (DATA HILANG)
```

### Upgrade

```bash
./scripts/upgrade.sh vX.Y.Z-rc.N
```

Atau langsung dengan installer versi baru:

```bash
curl -fsSL <installer-url-baru> | bash -s -- --upgrade --dir /opt/cakra
```

Installer membackup `.env`, `docker-compose.yml`, dan `nginx/` ke
`backups/config-*`, menarik image baru, lalu menjalankan health check. Volume
data tidak dihapus. `upgrade.sh` memverifikasi `SHA256SUMS` sebelum menjalankan
installer. Backup data penuh tetap disarankan sebelum upgrade:

```bash
./scripts/backup.sh
```

### Rollback

Jika upgrade gagal, installer mencetak perintah rollback. Ringkasnya:

```bash
cp backups/config-<timestamp>/.env .env
cp backups/config-<timestamp>/docker-compose.yml docker-compose.yml
docker compose up -d --no-build --remove-orphans
```

Untuk rollback versi image, jalankan installer versi lama dengan `--upgrade`.

### Backup dan restore

```bash
./scripts/backup.sh                  # ke backups/<timestamp>/
./scripts/backup.sh /path/backup     # folder tujuan
./scripts/backup.sh --skip-s3        # tanpa arsip objek S3
```

Backup berisi dump PostgreSQL, arsip volume SeaweedFS, `.env` terenkripsi
AES-256-CBC/PBKDF2, manifest, dan checksum. Passphrase lewat
`BACKUP_PASSPHRASE` atau prompt; tidak disimpan.

```bash
./scripts/restore.sh backups/<timestamp>
```

Restore menimpa database dan volume S3. `.env` aktif **tidak** ditimpa; hasil
dekripsi ditulis ke `.env.restored` untuk dibandingkan manual.

## Verifikasi rilis

Setiap rilis berisi `install.sh`, `cakra-install-<versi>.tar.gz`, dan
`SHA256SUMS`. Untuk produksi, verifikasi sebelum eksekusi:

```bash
BASE=https://github.com/krmtwre/cakra-install/releases/download/v1.0.0-rc.1
curl -fsSLO "$BASE/install.sh"
curl -fsSLO "$BASE/SHA256SUMS"
sha256sum -c SHA256SUMS --ignore-missing && bash install.sh
```

Isi bundle: `docker-compose.yml` (tanpa `build:`), `images.env` (tag + digest),
`images.lock`, `nginx/`, `scripts/`, `env.release.template`, `VERSION`, serta
`README.md`, `TESTING.md`, dan `THIRD-PARTY.md`.

## Keamanan

- PAT Docker Hub **tidak** ditulis ke `.env`, tidak masuk argumen CLI, dan tidak
  dicetak. Login memakai `DOCKER_CONFIG` sementara yang dihapus saat installer
  selesai, sehingga config Docker host tidak tersentuh. `--keep-login` memilih
  menyimpannya.
- `.env` berisi semua rahasia (password DB, JWT, AES, kredensial S3) dengan mode
  `600`. Jangan commit dan jangan bagikan.
- Semua image di-pin per digest, sehingga tag yang bergeser tidak mengubah
  artefak yang dipasang.
- Image memuat komponen pihak ketiga (PostgreSQL/pgvector, Redis, SeaweedFS,
  nginx). Daftar lisensi ada di `THIRD-PARTY.md` dalam bundle rilis.

## Jebakan yang sering kena

- Instalasi baru **menolak jalan** bila volume `<project>_cakra_pgdata` atau
  `<project>_cakra_s3data` sudah ada. Pilih `--dir`/project lain, atau hapus
  volume tersebut setelah memastikan datanya tidak dibutuhkan.
- `--reconfigure` hanya untuk versi yang sama dengan yang terpasang; beda versi
  → jalankan `--upgrade` dulu.
- `curl ... | bash` memerlukan TTY. Untuk otomasi pakai `--yes` + `DOCKER_PAT`.
- Port yang sedang dipakai proses lain ditolak saat validasi, bukan saat
  container dinyalakan.
