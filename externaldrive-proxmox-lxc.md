# Menambahkan External Storage ke Container LXC Immich di Proxmox

Dokumentasi ini menjelaskan langkah-langkah menambahkan HDD eksternal ke container LXC yang menjalankan Immich di Proxmox, sehingga Immich dapat langsung menggunakan storage baru tanpa kendala.

---

## Prasyarat

1. Proxmox VE sudah terinstal
2. LXC container dengan Immich sudah berjalan
3. HDD eksternal sudah terpasang di host Proxmox
4. Anda memiliki akses root ke Proxmox host dan LXC container
5. **PENTING:** LXC container harus menggunakan **Privileged Access** agar dapat mengakses device eksternal

---

## 1. Mount HDD di Host Proxmox

1. **Periksa device name HDD eksternal:**
   ```bash
   lsblk
   ```

2. **Buat folder mount point di host:**
   ```bash
   mkdir -p /mnt/data
   ```

3. **Mount HDD ke folder tersebut:**
   ```bash
   mount /dev/sdb1 /mnt/data
   ```

4. **Pastikan filesystem HDD kompatibel dengan Linux (disarankan ext4). Cek dengan:**
   ```bash
   blkid /dev/sdb1
   ```

5. **Opsional: agar HDD otomatis ter-mount saat reboot, tambahkan di `/etc/fstab`:**
   ```bash
   /dev/sdb1   /mnt/data   ext4   defaults   0 0
   ```

## 2. Bind Mount HDD ke LXC Immich

Agar container bisa mengakses HDD, lakukan bind mount:

```bash
pct set <vmid> -mp0 /mnt/data,mp=/mnt/data
```

Ganti `<vmid>` dengan ID LXC Immich.

- `mp0` adalah opsi mount pertama; Proxmox mendukung beberapa mount point jika perlu
- `mp=/mnt/data` menunjukkan mount point di dalam container

**Catatan:** Tidak perlu mount ulang di dalam container, bind mount dari host sudah cukup.

## 3. Set Permissions Folder Upload

Pastikan folder upload di HDD bisa diakses oleh user Immich:

```bash
chown -R immich:immich /mnt/data/upload
chmod -R 755 /mnt/data/upload
```

Cek dengan:
```bash
ls -ld /mnt/data/upload
```

## 4. Update Path Storage Immich

1. **Masuk ke folder instalasi Immich:**
   ```bash
   cd /opt/immich
   ```

2. **Edit file `.env`:**
   ```bash
   nano .env
   ```

3. **Ubah variable `IMMICH_MEDIA_LOCATION` ke path HDD:**
   ```bash
   IMMICH_MEDIA_LOCATION=/mnt/data/upload
   ```

4. **Simpan dan keluar** (Ctrl+O, Enter, Ctrl+X)

## 5. Restart Container LXC

Restart container LXC dari host Proxmox untuk menerapkan perubahan:

```bash
pct restart <vmid>
```

Ganti `<vmid>` dengan ID container LXC Immich.

Tunggu beberapa saat hingga container selesai booting, lalu cek status:
```bash
pct status <vmid>
```

## 6. Verifikasi

1. **Periksa isi folder upload di container:**
   ```bash
   ls -l /mnt/data/upload
   ```

2. **Pastikan semua folder/foto/video muncul di Immich web UI**

3. **Pastikan Immich dapat menambahkan file baru tanpa error**

## 7. Tips & Catatan

- **WAJIB:** Container LXC harus dalam mode **Privileged** agar bisa mengakses external storage. Jika container unprivileged, ubah dulu di Proxmox GUI: Options > Privileged = Yes, lalu restart container
- Jangan mount HDD di LXC lagi jika sudah menggunakan bind mount dari host
- Pastikan permissions folder upload sesuai user immich
- Jika HDD dilepas atau diganti, bind mount dan .env tetap harus diarahkan ke path yang benar
- Backup .env sebelum melakukan perubahan besar

---

Dokumentasi ini memastikan Immich di LXC Proxmox bisa langsung memanfaatkan HDD eksternal dengan lancar dan aman.
