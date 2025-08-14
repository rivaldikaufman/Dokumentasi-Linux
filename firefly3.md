# Panduan Lengkap & Anti-Gagal: Install Firefly III di Manjaro dengan Docker

Dokumentasi ini dirancang untuk melakukan instalasi Firefly III dari awal hingga akhir, dengan menyertakan solusi untuk berbagai masalah umum yang sering terjadi.

---

## Langkah 0: Persiapan Awal (Prerequisites)

Pastikan semua *alat tempur* sudah terpasang dan berjalan dengan benar di sistem Manjaro lu.

### 1. Install Docker & Docker Compose

```bash
sudo pacman -Syu docker docker-compose
```

### 2. Jalankan dan Aktifkan Service Docker

```bash
sudo systemctl start docker.service
sudo systemctl enable docker.service
```

### 3. Tambahkan User ke Grup Docker (Sangat Penting!)

Ini agar lu tidak perlu mengetik `sudo` setiap kali menjalankan command docker.

```bash
sudo usermod -aG docker $USER
```

> **Ingat:** Setelah menjalankan command ini, lu **wajib** logout dan login kembali, atau restart PC agar perubahan diterapkan.

---

## Langkah 1: Struktur Proyek

Buat folder khusus untuk menyimpan semua file konfigurasi Firefly III.

```bash
# Buat folder di home directory
mkdir ~/firefly-iii

# Masuk ke dalam folder tersebut
cd ~/firefly-iii
```

> Semua perintah selanjutnya akan dijalankan dari dalam direktori ini.

---

## Langkah 2: Konfigurasi `docker-compose.yml`

Buat file `docker-compose.yml`:

```bash
nano docker-compose.yml
```

Isi dengan:

```yaml
# version: '3.8'

services:
  app:
    image: fireflyiii/core:latest
    container_name: firefly_iii_app
    restart: unless-stopped
    volumes:
      - firefly_iii_upload:/var/www/html/storage/upload
    env_file: .env
    ports:
      - 8081:8080
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    container_name: firefly_iii_db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USERNAME}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_DATABASE}
    volumes:
      - firefly_iii_db:/var/lib/postgresql/data

volumes:
  firefly_iii_upload:
  firefly_iii_db:
```

Simpan (`Ctrl+X`, lalu `Y`, lalu `Enter`).

---

## Langkah 3: Konfigurasi `.env` (Paling Krusial!)

Buat file `.env`:

```bash
nano .env
```

Isi dengan:

```env
# --- Kredensial Database ---
DB_CONNECTION=pgsql
DB_HOST=firefly_iii_db
DB_PORT=5432
DB_DATABASE=fireflyiii

# Gunakan DB_USERNAME, bukan DB_USER!
DB_USERNAME=firefly

# Ganti dengan password yang kuat dan acak!
DB_PASSWORD=GANTI_DENGAN_PASSWORD_ACAK_YANG_AMAN

# --- Variabel Spesifik Firefly III ---
APP_URL=http://localhost:8081

# Kosongkan, akan diisi di langkah selanjutnya
APP_KEY=
```

Simpan file ini.

---

## Langkah 4: Generate `APP_KEY` (Jurus Anti-Gagal)

Jalankan perintah:

```bash
docker run --rm -e APP_KEY=base64:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA= fireflyiii/core:latest php artisan key:generate --show
```

Copy hasilnya (termasuk `base64:...`) ke baris `APP_KEY=` di `.env`.

---

## Langkah 5: Menjalankan Aplikasi

```bash
docker-compose up -d --remove-orphans
```

* `-d` → container berjalan di background
* `--remove-orphans` → membersihkan container sisa

---

## Langkah 6: Akses Aplikasi

Tunggu ±1 menit, lalu buka:

```
http://localhost:8081
```

---

## Troubleshooting & FAQ

### 1. **Error:** `port is already allocated`

* **Penyebab:** Port sudah dipakai aplikasi lain.
* **Solusi:** Ganti port di `docker-compose.yml` dan `APP_URL` di `.env` ke nomor lain (misal: `8082`).

---

### 2. **Error:** Database `Restarting` terus

* **Penyebab:** Volume database korup.
* **Solusi:**

```bash
docker-compose down
docker volume rm namafolderproyek_firefly_iii_db
docker-compose up -d
```

---

### 3. **Error:** `password authentication failed for user "www-data"`

* **Penyebab:** Salah ketik `DB_USERNAME` di `.env`.
* **Solusi:** Perbaiki `.env`, lalu:

```bash
docker-compose down
docker-compose up -d
```

---

### 4. **Update Firefly III**

```bash
cd ~/firefly-iii
docker-compose pull
docker-compose up -d
```

---

