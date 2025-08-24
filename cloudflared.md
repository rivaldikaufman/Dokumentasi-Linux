# Panduan Instalasi: Cloudflare Tunnel di Proxmox VE/Linux Distro

Dokumen ini merinci langkah-langkah untuk mengekspos layanan Proxmox Web UI ke internet secara aman menggunakan Cloudflare Tunnel (`cloudflared`).

-----

## 1\. Informasi Umum

  * **Software/Sistem**: Cloudflare Tunnel (`cloudflared`) versi stabil terbaru.
  * **Platform**: Proxmox VE (Debian Linux, arsitektur amd64/x64).
  * **Prasyarat**:
      * Server Proxmox VE yang aktif dan berjalan.
      * Akun Cloudflare yang aktif.
      * Sebuah nama domain yang sudah dikelola DNS-nya melalui Cloudflare.
      * Akses terminal (SSH/konsol) ke server Proxmox dengan hak akses `root` atau `sudo`.
  * **Estimasi Waktu**: 10-15 menit.

-----

## 2\. Persiapan Instalasi

  * **Download**: File instalasi akan diunduh langsung dari repositori resmi Cloudflare menggunakan `wget`.
  * **Tools yang Dibutuhkan**: `wget`, `dpkg`, `nano` (atau editor teks terminal lainnya). Semua tools ini umumnya sudah tersedia di Proxmox VE.
  * **Backup**: Sebelum memulai, sangat disarankan untuk melakukan backup semua Virtual Machine (VM) dan Container (CT) yang penting melalui fitur backup internal Proxmox.
  * **Permissions**: Seluruh proses instalasi dan konfigurasi memerlukan hak akses `root` atau pengguna dengan `sudo`.

-----

## 3\. Langkah-Langkah Instalasi

### **Step 1: Instalasi Paket `cloudflared`**

Langkah pertama adalah mengunduh dan menginstal daemon `cloudflared` di host Proxmox.

```bash
# Mengunduh paket instalasi terbaru
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb

# Menginstal paket menggunakan dpkg
sudo dpkg -i cloudflared-linux-amd64.deb
```

### **Step 2: Autentikasi ke Akun Cloudflare**

Hubungkan `cloudflared` di server Anda dengan akun Cloudflare Anda.

```bash
cloudflared tunnel login
```

Jalankan perintah di atas. Salin URL yang muncul di terminal, buka di browser, login ke akun Cloudflare Anda, dan pilih domain yang akan digunakan.

### **Step 3: Membuat Tunnel**

Buat sebuah tunnel persisten yang akan menjadi jembatan antara server Anda dan jaringan Cloudflare.

```bash
# Ganti <NAMA_TUNNEL> dengan nama yang deskriptif, contoh: proxmox-nucleus
cloudflared tunnel create <NAMA_TUNNEL>
```

**PENTING**: Catat **Tunnel UUID** dan lokasi file **credentials (`.json`)** yang muncul setelah perintah ini dijalankan. Anda akan membutuhkannya pada langkah berikutnya.

### **Step 4: Konfigurasi Tunnel**

Buat file konfigurasi untuk menentukan bagaimana tunnel akan mengarahkan traffic.

1.  Buat direktori konfigurasi:

    ```bash
    sudo mkdir -p /etc/cloudflared/
    ```

2.  Buat dan edit file `config.yml`:

    ```bash
    sudo nano /etc/cloudflared/config.yml
    ```

3.  Salin dan tempelkan konfigurasi berikut ke dalam file. **Sesuaikan nilai yang ada di dalam tanda kurung `<...>`**.

    ```yaml
    tunnel: <TUNNEL_UUID_DARI_STEP_3>
    credentials-file: /root/.cloudflared/<TUNNEL_UUID_DARI_STEP_3>.json

    ingress:
      # Aturan untuk mengarahkan traffic ke Proxmox Web UI
      - hostname: <SUBDOMAIN.DOMAIN_ANDA> # Contoh: proxmox.rivaldi.fun
        service: https://<IP_PROXMOX_LOKAL>:8006 # Contoh: https://192.168.1.16:8006
        originRequest:
          noTLSVerify: true # Wajib ada karena Proxmox menggunakan sertifikat self-signed

      # Aturan fallback untuk menolak semua traffic lain
      - service: http_status:404
    ```

4.  Simpan file dan keluar dari editor (di `nano`: `Ctrl+X`, lalu `Y`, lalu `Enter`).

### **Step 5: Membuat DNS Record**

Arahkan nama domain publik Anda ke tunnel yang baru dibuat.

```bash
# Gunakan NAMA_TUNNEL dan SUBDOMAIN.DOMAIN_ANDA yang sama dengan langkah sebelumnya
cloudflared tunnel route dns <NAMA_TUNNEL> <SUBDOMAIN.DOMAIN_ANDA>
```

### **Step 6: Menjalankan Tunnel sebagai Service**

Instal dan jalankan `cloudflared` sebagai layanan `systemd` agar selalu aktif secara otomatis.

```bash
# Menginstal service
sudo cloudflared service install

# Memulai service sekarang juga
sudo systemctl start cloudflared

# Mengaktifkan service agar berjalan otomatis saat boot
sudo systemctl enable cloudflared
```

### **Step 7: Verifikasi**

1.  **Akses Publik**: Buka browser dan kunjungi `https://<SUBDOMAIN.DOMAIN_ANDA>`. Halaman login Proxmox seharusnya muncul.
2.  **Status Service**: Periksa status layanan di server untuk memastikan semuanya berjalan normal.
    ```bash
    sudo systemctl status cloudflared
    ```
    Pastikan output menunjukkan status **active (running)**.

### **Step 8: Strategi Rollback (Pembatalan)**

Jika terjadi masalah atau Anda ingin menonaktifkan akses, ikuti langkah-langkah berikut:

1.  Hentikan dan nonaktifkan layanan `cloudflared`:
    ```bash
    sudo systemctl stop cloudflared
    sudo systemctl disable cloudflared
    ```
2.  Hapus DNS record CNAME yang dibuat dari dashboard Cloudflare Anda.
3.  Hapus tunnel dari dashboard Cloudflare Zero Trust atau melalui CLI:
    ```bash
    cloudflared tunnel delete <NAMA_TUNNEL>
    ```
4.  (Opsional) Hapus paket `cloudflared`:
    ```bash
    sudo dpkg -r cloudflared
    ```
