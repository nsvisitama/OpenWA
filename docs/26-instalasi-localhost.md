# 26 - Panduan Instalasi OpenWA di Localhost

Panduan ini merangkum langkah-langkah menjalankan **OpenWA** (WhatsApp API Gateway) di
komputer lokal (`localhost`) untuk kebutuhan development, uji coba, atau penggunaan
personal single-session. Disusun berdasarkan tinjauan langsung terhadap `Dockerfile`,
`docker-compose.yml`, `docker-compose.dev.yml`, `.env.example`/`.env.minimal`, dan
`docker-entrypoint.sh` di repositori ini.

Ada dua cara menjalankan OpenWA secara lokal:

| Opsi | Cocok untuk | Kebutuhan |
| --- | --- | --- |
| **A. Docker (disarankan)** | Cara tercepat, environment terisolasi, tidak perlu install Chromium/Node manual | Docker + Docker Compose |
| **B. Manual (Node.js langsung)** | Development aktif (hot-reload dashboard), debugging kode | Node.js 22 LTS, npm |

---

## 1. Prasyarat

### Opsi A — Docker

- **Docker** ≥ 20.10
- **Docker Compose** ≥ 2.0
- Untuk pengguna **Podman** (bukan Docker Engine), rootless socket harus aktif:

  ```bash
  systemctl --user start podman.socket
  systemctl --user enable podman.socket
  export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock
  ```

  Tambahkan baris `export` di atas ke `~/.bashrc` agar permanen.

### Opsi B — Manual (Node.js)

- **Node.js 22 LTS** (direkomendasikan; dashboard butuh minimal Node 20+)
- **npm**
- **Git**
- Karena engine default (`whatsapp-web.js`) berbasis **Chromium/Puppeteer**, di Linux
  pastikan library berikut tersedia (biasanya sudah ada di distro desktop, tapi sering
  hilang di server/WSL2 minimal):

  ```bash
  # Debian/Ubuntu
  sudo apt-get update && sudo apt-get install -y \
    chromium fonts-liberation libappindicator3-1 libasound2 \
    libatk-bridge2.0-0 libatk1.0-0 libcups2 libdbus-1-3 libdrm2 \
    libgbm1 libgtk-3-0 libnspr4 libnss3 libx11-xcb1 \
    libxcomposite1 libxdamage1 libxrandr2 xdg-utils
  ```

  Ini adalah paket yang sama seperti yang diinstal di image Docker produksi
  (lihat `Dockerfile`). Jika Anda memakai `ENGINE_TYPE=baileys`, dependency Chromium ini
  **tidak diperlukan** karena Baileys berbasis WebSocket tanpa browser.

---

## 2. Opsi A: Instalasi via Docker (Tercepat)

`docker-compose.dev.yml` di root repo dirancang khusus untuk smoke test/quick start lokal:
satu container, SQLite, storage lokal, dan `DATABASE_SYNCHRONIZE=true` (skema otomatis
tanpa migrasi manual).

```bash
# 1. Clone repository
git clone https://github.com/rmyndharis/OpenWA.git
cd OpenWA

# 2. Jalankan compose khusus development
docker compose -f docker-compose.dev.yml up -d

# 3. Lihat log (opsional, untuk melihat API key awal)
docker compose -f docker-compose.dev.yml logs -f openwa
```

Akses aplikasi (Dashboard sudah menyatu ke API pada port yang sama):

- **Dashboard**: http://localhost:2785
- **API**: http://localhost:2785/api
- **Swagger**: http://localhost:2785/api/docs
- **Health check**: http://localhost:2785/api/health

> Port dibind ke `127.0.0.1` secara default (hanya bisa diakses dari komputer sendiri).
> Untuk mengaksesnya dari perangkat lain di jaringan yang sama, set `BIND_HOST=0.0.0.0`
> di `.env` sebelum `up -d`.

### Menghentikan / reset

```bash
docker compose -f docker-compose.dev.yml down          # stop
docker compose -f docker-compose.dev.yml down --volumes # stop + hapus data (./data)
```

---

## 3. Opsi B: Instalasi Manual (Node.js langsung)

Cocok jika Anda ingin melakukan perubahan kode dan butuh hot-reload di dashboard (Vite
dev server) sekaligus API (`--watch`).

```bash
# 1. Clone repository
git clone https://github.com/rmyndharis/OpenWA.git
cd OpenWA

# 2. Install dependency (postinstall otomatis menjalankan `npm install` di folder dashboard/)
npm install

# 3. Salin konfigurasi minimal untuk localhost (SQLite, tanpa Redis/S3)
cp .env.minimal .env

# 4. Buat folder data yang dibutuhkan
mkdir -p data/sessions data/media

# 5. Jalankan API + Dashboard sekaligus (concurrently)
npm run dev
```

`npm run dev` menjalankan dua proses paralel:

- `start:dev` → NestJS API dengan `--watch` di port **2785**
- `dashboard:dev` → Vite dev server dashboard di port **2886** (hot reload, proxy
  otomatis ke `/api` dan `/socket.io` pada API port)

Akses:

- **Dashboard (dev, hot-reload)**: http://localhost:2886
- **API**: http://localhost:2785/api
- **Swagger**: http://localhost:2785/api/docs
- **Health check**: http://localhost:2785/api/health

Jika hanya ingin menjalankan API saja (tanpa dashboard dev server):

```bash
npm run start:dev
```

---

## 4. Konfigurasi `.env` untuk Localhost

`.env.minimal` sudah cukup untuk kebutuhan single-session personal berbasis SQLite. Poin
penting yang perlu diketahui:

| Variabel | Default (minimal) | Keterangan |
| --- | --- | --- |
| `PORT` | `2785` | Port API + Dashboard (bundled) |
| `DATABASE_TYPE` | `sqlite` | Tidak perlu service database eksternal |
| `DATABASE_SYNCHRONIZE` | `true` | Skema tabel dibuat otomatis (khusus dev; jangan dipakai di produksi) |
| `ENGINE_TYPE` | `whatsapp-web.js` | Ganti ke `baileys` bila tidak ingin bergantung pada Chromium |
| `AUTO_START_SESSIONS` | `false` | Set `true` agar sesi yang sudah login otomatis start saat server restart |
| `API_MASTER_KEY` | *(kosong)* | Kosongkan untuk dev; sistem akan membuat API key admin acak otomatis saat boot pertama |

Jika ingin memakai kunci dev bawaan (`dev-admin-key`) alih-alih kunci acak, tambahkan:

```env
ALLOW_DEV_API_KEY=true
```

> ⚠️ Hanya untuk localhost/dev. Jangan gunakan `ALLOW_DEV_API_KEY=true` pada environment
> yang dapat diakses publik.

Untuk konfigurasi lengkap (Redis, S3/MinIO, MCP server, rate limiting, dll), lihat
`.env.example` sebagai referensi Single Source of Truth semua variabel.

---

## 5. Mendapatkan API Key Pertama

Saat pertama kali boot, OpenWA otomatis membuat API key admin dan menuliskannya ke:

- `data/.api-key` (instalasi manual)
- volume `/app/data/.api-key` di dalam container (Docker)

Key ini juga tercetak di log startup. Lihat dengan:

```bash
cat data/.api-key
# atau untuk Docker:
docker compose -f docker-compose.dev.yml logs openwa | grep -i "api key"
```

---

## 6. Verifikasi Instalasi

```bash
# 1. Health check dasar
curl http://localhost:2785/api/health

# 2. Buat session WhatsApp
curl -X POST http://localhost:2785/api/sessions \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <API_KEY_ANDA>" \
  -d '{"name": "my-bot"}'

# 3. Start session
curl -X POST http://localhost:2785/api/sessions/{sessionId}/start \
  -H "X-API-Key: <API_KEY_ANDA>"

# 4. Ambil QR code dan scan dengan WhatsApp di HP
curl http://localhost:2785/api/sessions/{sessionId}/qr \
  -H "X-API-Key: <API_KEY_ANDA>"
```

Setelah QR discan, status session akan berubah menjadi `ready` dan Anda bisa mulai
mengirim pesan lewat endpoint `messages/send-text` (lihat `docs/06-api-specification.md`).

---

## 7. Troubleshooting Khusus Localhost

| Gejala | Kemungkinan Penyebab | Solusi |
| --- | --- | --- |
| `Port already in use` saat start | Port 2785/2886 sudah dipakai proses lain | `lsof -i :2785` lalu `kill -9 $(lsof -t -i:2785)`, atau ubah `PORT`/`API_PORT` di `.env` |
| Build `sqlite3`/`node-gyp` gagal saat `npm install` | Compiler native (python3/make/g++) belum terinstal | Debian/Ubuntu: `sudo apt-get install -y python3 make g++` |
| Session macet di `authenticating`, tidak pernah `ready` | Versi WhatsApp Web yang otomatis dipilih tidak kompatibel | Set `WWEBJS_WEB_VERSION` ke versi yang diketahui stabil (lihat `docs/12-troubleshooting-faq.md`) |
| QR timeout di percobaan pertama (WSL2 / resource terbatas) | Waktu tunggu boot 30 detik default terlampaui | Naikkan `WWEBJS_AUTH_TIMEOUT_MS` (mis. `120000`) |
| `chrome_crashpad_handler: --database is required` | Chromium tidak menemukan folder home/cache yang bisa ditulis | Hanya relevan pada container hardened; untuk instalasi manual biasanya tidak muncul karena berjalan sebagai user biasa |
| Podman: `FileNotFoundError` / socket tidak ditemukan | Podman socket belum aktif | Jalankan `systemctl --user start podman.socket` dan set `DOCKER_HOST` (lihat bagian Prasyarat) |
| Podman: `short-name did not resolve to an alias` | Podman rootless tidak fallback ke Docker Hub | Gunakan Docker Desktop/Engine, atau konfigurasikan unqualified-search registries Podman |

Untuk daftar troubleshooting lebih lengkap (performa, database, webhook, dsb.), lihat
[12 - Troubleshooting FAQ](./12-troubleshooting-faq.md).

---

## 8. Langkah Selanjutnya

- [Panduan Docker Lengkap (Bahasa Indonesia)](./DOCKER_ID.md) — untuk deployment produksi dengan profile Postgres/Redis/MinIO
- [API Specification](./06-api-specification.md) — referensi lengkap REST API
- [Development Guidelines](./08-development-guidelines.md) — standar kode untuk kontribusi
- [MCP Integration](./24-mcp-integration.md) — menghubungkan OpenWA ke AI agent (Claude, Cursor, dll)

---

<div align="center">

[← 25 - Integration Fabric](./25-integration-fabric.md) · [Documentation Index](./README.md)

</div>
