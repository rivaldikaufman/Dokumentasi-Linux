Dokumentasi Ringkas & Lengkap: Setup Shairport Sync (AirPlay 2) + NQPTP di Linux

1) Ringkasan Eksekutif
	•	Topik utama: Men-setup Shairport Sync sebagai AirPlay 2 receiver dengan output ALSA (jack 3.5 mm), serta sinkronisasi waktu AirPlay 2 via NQPTP.
	•	Masalah inti:
	1.	Shairport awalnya tidak built dengan SoXR → muncul peringatan.
	2.	Shairport sering gagal dengan pesan “can not find the nqptp service” meskipun NQPTP aktif → isu start-order dan hak akses ke /dev/shm/nqptp.
	•	Hasil akhir:
	•	Shairport berhasil dibuild ulang dengan SoXR.
	•	NQPTP aktif (UDP 319/320 + kontrol 127.0.0.1:9000).
	•	Unit service Shairport di-adjust: Requires/After=nqptp.service dan SupplementaryGroups=nqptp → Shairport jalan stabil.

⸻

2) Lingkungan & Target
	•	OS/Tools: Linux (systemd), gcc, autotools, make.
	•	Komponen:
	•	Shairport Sync (AirPlay 2, ALSA, Avahi, OpenSSL, SoXR)
	•	NQPTP (Not Quite PTP) untuk sinkronisasi AirPlay 2
	•	ALSA (analog out: plughw:0,0; mixer Master)
	•	Avahi/mDNS
	•	Output audio: Jack 3.5 mm (ALC294 Analog / card 0, device 0).
	•	Nama device: Harman Kardon Aura.

⸻

3) Kronologi Teknis (High-level)

3.1. Validasi awal Shairport & port AirPlay 2
	•	shairport-sync -V menampilkan AirPlay2 & sysconfdir:/usr/local/etc.
	•	Listener TCP 7000 aktif (ss -lntp | grep 7000).

3.2. ALSA & Mixer
	•	Mixer tersedia: Master, Headphone, Speaker, PCM, …
	•	Set volume & unmute: Master, PCM, Headphone → 80%.
	•	Auto-Mute Mode di-Disabled.
	•	alsactl store untuk persist pengaturan.

3.3. SoXR tidak terdeteksi → build ulang
	•	Log: “soxr option not available because this version … built without libsoxr”.
	•	Aksi: apt-get install -y libsoxr-dev → reconfigure & rebuild Shairport:
./configure --with-alsa --with-avahi --with-ssl=openssl --with-airplay-2 --with-soxr → make → make install.
	•	Verifikasi: shairport-sync -V kini mencantumkan …-ALSA-soxr-….
	•	Konfigurasi: interpolation = "soxr"; (sempat di-switch ke basic sementara untuk hilangkan warning).

3.4. NQPTP status & konflik port
	•	nqptp -v manual sempat error port 319 “Address already in use” → wajar karena service NQPTP sudah running dan memegang port 319/320.
	•	Verifikasi soket:
	•	UDP 319/320 bound oleh nqptp
	•	UDP 127.0.0.1:9000 (kontrol) aktif
	•	File /dev/shm/nqptp kadang tidak ada di momen tertentu (race/start-order).

3.5. Shairport gagal menemukan NQPTP
	•	Log berulang: fatal error: “can not find the nqptp service”.
	•	Penyebab utama (gabungan):
	•	Start-order: Shairport start sebelum NQPTP siap (file shm & port kontrol).
	•	Hak akses: Proses Shairport perlu membership grup nqptp untuk membaca /dev/shm/nqptp.

3.6. Perbaikan systemd unit
	•	nqptp.service (ringkas):
	•	User=nqptp, Group=nqptp, AmbientCapabilities=CAP_NET_BIND_SERVICE.
	•	shairport-sync.service dibenahi:
	•	[Unit] → Requires=nqptp.service dan After=nqptp.service avahi-daemon.service.
	•	[Service] → SupplementaryGroups=audio nqptp, ExecStart=/usr/local/bin/shairport-sync --log-to-syslog, Restart=on-failure.
	•	Opsional (drop-in): tambahkan ExecStartPre untuk menunggu /dev/shm/nqptp & 127.0.0.1:9000 siap (lihat contoh benar di § 6.3).
	•	Setelah stop/start berurutan (NQPTP dulu, lalu Shairport), kondisi stabil:
	•	/dev/shm/nqptp ada & readable
	•	UDP 319/320 & 9000 aktif
	•	Proses Shairport memiliki grup nqptp (contoh: Groups: 29 113 988 → 988 = nqptp)
	•	Shairport Started tanpa error.

3.7. Pesan D-Bus
	•	Peringatan: WARNING: Unhandled message … ActivatableServicesChanged → benign (tidak mengganggu fungsi audio).

⸻

4) Hasil & Kesimpulan
	•	SoXR aktif dan digunakan (interpolation = "soxr"; versi binary “…-soxr-…”).
	•	NQPTP berjalan baik: port 319/320/9000 aktif, file /dev/shm/nqptp tersedia.
	•	Shairport terhubung ke NQPTP setelah:
	•	menambahkan Requires/After ke nqptp.service
	•	memberi SupplementaryGroups=nqptp pada service Shairport
	•	memastikan start order: NQPTP sebelum Shairport.
	•	Output audio ALSA ke jack 3.5 mm OK (mixer & volume terset).

⸻

5) Rangkuman Poin Penting per Bagian

5.1. Validasi & Port
	•	✅ AirPlay 2 terdeteksi di versi string.
	•	✅ ss -lntp menunjukkan listener TCP 7000.

5.2. Audio (ALSA)
	•	✅ Device: plughw:0,0.
	•	✅ Mixer: Master/PCM/Headphone 80% (unmute).
	•	✅ Auto-Mute Mode = Disabled.
	•	✅ alsactl store untuk persist.

5.3. SoXR
	•	❌ Awalnya tidak built → warning.
	•	✅ libsoxr-dev + rebuild dengan --with-soxr → fixed.

5.4. NQPTP
	•	✅ Service aktif (user/group nqptp), bind UDP 319/320.
	•	✅ Kontrol 127.0.0.1:9000 aktif.
	•	⚠️ Jalankan sekali; nqptp -v manual saat service hidup memang akan bentrok port.

5.5. Integrasi Shairport ↔ NQPTP
	•	❌ Error “can not find the nqptp service” saat:
	•	start-order belum tepat, atau
	•	Shairport tidak punya akses grup nqptp.
	•	✅ Solusi: Requires/After=nqptp.service + SupplementaryGroups=nqptp + start NQPTP dulu.

5.6. D-Bus Warning
	•	⚠️ Pesan ActivatableServicesChanged dapat diabaikan.

⸻

6) Konfigurasi & Unit (Contoh Akhir)

6.1. shairport-sync.conf (cuplikan relevan)

general = {
  name = "Harman Kardon Aura";
  interpolation = "soxr";          // kualitas tinggi
  // optional: default_airplay_volume / volume_* jika diperlukan
};

alsa = {
  output_backend = "alsa";
  output_device  = "plughw:0,0";   // ALC294 Analog (jack 3.5mm)
  mixer_control_name = "Master";   // jika "Master" tidak ada, gunakan "PCM"
  mixer_control_index = 0;
};

// Contoh optimasi buffer (opsional, kamu sempat set):
// audio_backend_buffer_interpolation_threshold_in_seconds = 0.12;

6.2. nqptp.service (ringkas)

[Unit]
Description=NQPTP -- Not Quite PTP
Wants=network-online.target
After=network.target network-online.target
Before=shairport-sync.service

[Service]
ExecStart=/usr/local/bin/nqptp
User=nqptp
Group=nqptp
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target

6.3. shairport-sync.service (disarankan via drop-in override.conf)

Lebih aman menyimpan perubahan di drop-in agar file utama tetap standar.

/etc/systemd/system/shairport-sync.service.d/override.conf

[Unit]
Requires=nqptp.service avahi-daemon.service
After=nqptp.service avahi-daemon.service network-online.target sound.target

[Service]
User=shairport-sync
Group=shairport-sync
SupplementaryGroups=audio nqptp

# pastikan pakai binary /usr/local:
ExecStart=
ExecStart=/usr/local/bin/shairport-sync --log-to-syslog

# (Opsional) tunggu hingga NQPTP siap: /dev/shm/nqptp + UDP 9000
ExecStartPre=/bin/sh -c 'for i in $(seq 1 30); do \
  [ -r /dev/shm/nqptp ] && ss -lunp | grep -q "127.0.0.1:9000" && exit 0; \
  sleep 0.2; \
done; echo "NQPTP belum siap"; exit 1'

Restart=on-failure
RestartSec=3

Setelah membuat/mengubah drop-in:
systemctl daemon-reload && systemctl restart nqptp && systemctl restart shairport-sync

⸻

7) Checklist Verifikasi Cepat
	1.	Versi
	•	shairport-sync -V → mengandung AirPlay2 & soxr.
	2.	NQPTP siap
	•	ls -l /dev/shm/nqptp → ada, owner nqptp:nqptp.
	•	ss -lunp | grep -E ':(319|320|9000)' → 319/320/9000 aktif oleh nqptp.
	3.	Hak akses Shairport
	•	pid=$(pgrep -x shairport-sync); grep ^Groups: /proc/$pid/status → berisi grup nqptp.
	4.	Log Shairport
	•	journalctl -u shairport-sync -n 30 --no-pager → Started tanpa error NQPTP.
	5.	Audio keluar
	•	Cek suara, mixer Master/PCM/Headphone tidak mute & level wajar.

⸻

8) Troubleshooting Cepat
	•	Masih “can not find the nqptp service”:
	•	Pastikan /dev/shm/nqptp ada & readable.
	•	Pastikan Shairport bergabung di grup nqptp (via SupplementaryGroups= dan/atau /etc/group).
	•	Pastikan start-order: NQPTP lebih dulu; pakai Requires/After= atau ExecStartPre wait-loop.
	•	Port 319 “Address already in use” saat nqptp -v:
	•	Abaikan jika service NQPTP sudah berjalan — jangan jalankan binary kedua kali.
	•	Peringatan D-Bus ActivatableServicesChanged:
	•	Boleh diabaikan (tidak mempengaruhi audio).
	•	SoXR warning muncul lagi:
	•	Cek -V harus ada soxr; kalau hilang, ulangi build dengan --with-soxr dan libsoxr-dev.

⸻

9) Ringkasan Command yang Dipakai

# Cek versi & port
/usr/local/bin/shairport-sync -V
ss -lntp | grep 7000

# Mixer & volume
amixer -c 0 scontrols
amixer -c 0 set Master 80% unmute
amixer -c 0 set PCM 80% unmute
amixer -c 0 set Headphone 80% unmute
amixer -c 0 set 'Auto-Mute Mode' Disabled
alsactl store

# Build Shairport dengan SoXR
apt-get install -y libsoxr-dev
cd ~/shairport-sync
make clean || true
./configure --with-alsa --with-avahi --with-ssl=openssl --with-airplay-2 --with-soxr
make -j"$(nproc)"
make install

# Build NQPTP
apt-get install -y build-essential git autoconf automake libtool
cd /usr/local/src
git clone https://github.com/mikebrady/nqptp.git
cd nqptp
autoreconf -fi
./configure
make -j"$(nproc)"
make install

# Service management
systemctl daemon-reload
systemctl restart nqptp
systemctl restart shairport-sync

# Verifikasi NQPTP
ls -l /dev/shm/nqptp
ss -lunp | grep -E ':(319|320|9000)'

# Verifikasi grup proses shairport
pid=$(pgrep -x shairport-sync); grep ^Groups: /proc/$pid/status

# Log
journalctl -u shairport-sync -n 60 --no-pager


⸻

10) Penutup

Seluruh tahapan memastikan audio analog bekerja mulus via ALSA, SoXR aktif untuk kualitas interpolasi yang lebih baik, dan sinkronisasi AirPlay 2 beres lewat NQPTP. Kunci stabilitas ada pada: build opsi yang tepat, hak akses grup nqptp, dan urutan start service yang benar. Jika ingin lebih tahan-gangguan, gunakan drop-in override.conf dengan ExecStartPre kecil yang menunggu resource NQPTP benar-benar siap sebelum Shairport dijalankan.
