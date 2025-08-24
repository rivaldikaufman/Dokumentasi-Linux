Mengatasi Error GPG Key apt update pada Proxmox 9 (Trixie)

Dokumen ini menyediakan panduan step-by-step untuk menyelesaikan error Missing key atau repository is not signed saat menjalankan apt update pada instalasi Proxmox VE 9 (berbasis Debian 13 "Trixie").
1. Ringkasan Masalah 🧐

    Masalah: Perintah apt update gagal memverifikasi repository Proxmox, menghasilkan error Sub-process /usr/bin/sqv returned an error code (1) dengan pesan Missing key atau The repository is not signed.

    Konteks: Masalah ini umum terjadi pada instalasi baru Proxmox VE 9 atau setelah upgrade, di mana konfigurasi repository atau GPG key tidak sesuai dengan standar Debian 13 "Trixie".

    Dampak: Sistem tidak dapat memperbarui paket dari repository Proxmox, menyebabkan server rentan terhadap bug dan celah keamanan.

2. Solusi yang Berhasil ✅

Solusi ini menggunakan metode modern dan yang direkomendasikan oleh Debian, yaitu menyimpan GPG key di direktori /etc/apt/keyrings dan menautkannya secara eksplisit pada file repository menggunakan parameter signed-by.
Langkah 1: Bersihkan Konfigurasi Lama 🧹

Hapus file konfigurasi GPG key dan repository Proxmox yang lama atau salah untuk menghindari konflik.

# Hapus file kunci lama jika ada
rm /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg

# Hapus file repository lama jika ada
rm /etc/apt/sources.list.d/proxmox.sources
rm /etc/apt/sources.list.d/pve-no-subscription.list

    Catatan: Tidak masalah jika perintah di atas menghasilkan error No such file or directory.

Langkah 2: Download GPG Key ke Lokasi yang Tepat 🔑

Download GPG key resmi Proxmox untuk versi "Trixie" dan simpan di direktori /etc/apt/keyrings/.

wget -qO /etc/apt/keyrings/proxmox-release-trixie.gpg https://enterprise.proxmox.com/debian/proxmox-release-trixie.gpg

Langkah 3: Buat File Repository Baru dengan signed-by 📝

Buat file konfigurasi repository yang baru dan secara eksplisit tunjuk lokasi GPG key yang sudah di-download.

# Perintah ini membuat file /etc/apt/sources.list.d/proxmox.list
echo "deb [signed-by=/etc/apt/keyrings/proxmox-release-trixie.gpg] http://download.proxmox.com/debian/pve trixie pve-no-subscription" | tee /etc/apt/sources.list.d/proxmox.list

3. Hasil Akhir 🎉

    Status: Masalah berhasil diselesaikan.

    Verifikasi: Jalankan kembali perintah apt update. Proses akan berjalan lancar tanpa ada Err atau Warning terkait repository Proxmox.

    apt update

    Selanjutnya, Anda dapat melakukan upgrade sistem dengan aman:

    apt full-upgrade -y

    Waktu Penyelesaian: Kurang dari 2 menit.

4. Tips Tambahan 💡

    Best Practice: Selalu gunakan metode signed-by untuk menambahkan repository pihak ketiga pada sistem berbasis Debian 12 (Bookworm) ke atas. Cara ini lebih aman dan mencegah konflik antar GPG key.

    Peringatan: Pastikan Anda menggunakan URL GPG key dan repository yang benar sesuai dengan versi Proxmox dan Debian yang Anda gunakan. Dokumentasi ini spesifik untuk Proxmox 9 (Trixie).
