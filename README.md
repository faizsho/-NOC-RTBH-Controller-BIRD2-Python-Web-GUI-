🛡️ NOC RTBH Controller (BIRD2 + Python Web GUI)
Enterprise-Grade Remote Triggered Black Hole (RTBH) Controller with Live Dashboard

Sebuah sistem terpusat untuk memanajemen routing BGP Blackhole (RTBH) pada level Internet Service Provider (ISP). Sistem ini menggunakan BIRD 2 sebagai routing engine utama dan Python Flask sebagai Web GUI Dashboard interaktif untuk memudahkan tim Network Operations Center (NOC) dalam memantau, menambah, dan menghapus daftar IP Blacklist maupun Whitelist.

Didesain dengan tema Slate Dark Mode yang responsif, ringan, dan aman.

✨ Fitur Utama
1. Automated BGP Peering: Terhubung dengan Edge Router (seperti MikroTik, Cisco, dll) via iBGP/eBGP untuk mendistribusikan jutaan rute blackhole secara real-time.

2. Multiple Source Aggregation: Mampu menarik ribuan IP jahat dari berbagai link pihak ketiga secara bersamaan, membersihkan duplikat, dan mengonversinya ke format BIRD.

3. Smart Whitelist Engine (Anti False-Positive): Melindungi IP krusial (DNS, Core Infra) agar tidak ikut terblokir, baik di-input manual maupun via remote URL.

4. Live BGP Monitoring: Memantau status peering secara real-time (Established/Down) langsung dari Web GUI.

5. IP/Subnet Lookup Engine: Fitur pencarian super cepat untuk mengecek apakah sebuah IP atau Subnet (misal /24) sedang berada di dalam daftar blokir BIRD.

6. Secure Auth Gateway: Dilengkapi halaman login terenkripsi untuk mencegah akses tidak sah.

🛠️ Prasyarat Sistem (Requirements)
-> OS: Ubuntu 22.04 LTS (atau varian Debian modern lainnya)
-> BIRD v2.x (sudo apt install bird2)
-> Python 3 & pip (sudo apt install python3-pip)
-> Flask (pip3 install flask)

🚀 Panduan Instalasi Lengkap
Langkah 1: Persiapan Struktur Direktori & File
Buat semua file text dan direktori yang dibutuhkan oleh Web GUI untuk menyimpan data sumber URL dan IP.

Bash
sudo mkdir -p /opt/static
sudo touch /opt/whitelist_manual.txt
sudo touch /opt/whitelist_sources.txt
sudo touch /opt/rtbh_sources.txt
sudo touch /etc/bird/rtbh-dynamic.conf

# Masukkan logo perusahaan (Opsional)
# Upload file logo.png ke dalam folder /opt/static/
Langkah 2: Konfigurasi BIRD 2 Utama
Buka konfigurasi utama BIRD 2:

Bash
sudo nano /etc/bird/bird.conf
Ganti isinya dengan konfigurasi standar RTBH ini:

Cuplikan kode
router id 10.10.10.100; # Ganti dengan IP Server Ubuntu ini

# Filter BGP untuk menambahkan community Blackhole
filter bgp_out_rtbh {
    bgp_community.add((65000, 666)); # Sesuaikan dengan community MikroTik/Upstream
    accept;
}

# Dummy Protocol
protocol device {}
protocol direct { ipv4; }
protocol kernel { ipv4; learn; persist; }

# Protocol statis untuk membaca file hasil generate IP Blacklist
protocol static rtbh_list {
    ipv4;
    include "/etc/bird/rtbh-dynamic.conf"; 
}

# File untuk membaca daftar Peer BGP yang bisa disinkronisasi dari Web GUI
include "/etc/bird/peers.conf";
Langkah 3: Script Updater Otomatis (Bash Engine)
Buat script bash yang bertugas mengambil semua URL dari Web GUI, menyaring IP, mengecualikan IP whitelist, dan memasukkannya ke BIRD.

Bash
sudo nano /opt/rtbh_updater.sh
Isi dengan script agregator ini:

Bash
#!/bin/bash

# Path Files
RTBH_SOURCES="/opt/rtbh_sources.txt"
WL_SOURCES="/opt/whitelist_sources.txt"
WL_MANUAL="/opt/whitelist_manual.txt"
BIRD_CONF="/etc/bird/rtbh-dynamic.conf"

TEMP_RTBH="/tmp/rtbh_raw.txt"
TEMP_WL="/tmp/wl_raw.txt"

> "$TEMP_RTBH"
> "$TEMP_WL"

# 1. Download & Ekstrak IP Blacklist
while IFS= read -r url; do
    [[ -z "$url" || "$url" == \#* ]] && continue
    curl -s -f --connect-timeout 10 "$url" | grep -oE '\b((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)(/[0-9]{1,2})?\b' >> "$TEMP_RTBH"
done < "$RTBH_SOURCES"

# 2. Download & Ekstrak IP Whitelist (Remote + Manual)
if [ -f "$WL_MANUAL" ]; then cat "$WL_MANUAL" >> "$TEMP_WL"; fi
while IFS= read -r url; do
    [[ -z "$url" || "$url" == \#* ]] && continue
    curl -s -f --connect-timeout 10 "$url" | grep -oE '\b((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)(/[0-9]{1,2})?\b' >> "$TEMP_WL"
done < "$WL_SOURCES"

# 3. Proses Filter: Hapus IP Whitelist dari daftar Blacklist, Format ke BIRD syntax
sort -u "$TEMP_WL" > /tmp/wl_sorted.txt
sort -u "$TEMP_RTBH" > /tmp/rtbh_sorted.txt

# Pengecualian dan konversi
grep -vFf /tmp/wl_sorted.txt /tmp/rtbh_sorted.txt | awk '/^[0-9]/ {
    ip = $1
    if (ip !~ /\//) { ip = ip "/32" }
    print "route " ip " blackhole;"
}' > "$BIRD_CONF"

# 4. Terapkan Rekonfigurasi BIRD
birdc configure > /dev/null

# 5. Bersihkan sampah
rm -f "$TEMP_RTBH" "$TEMP_WL" /tmp/wl_sorted.txt /tmp/rtbh_sorted.txt
Berikan hak akses eksekusi:

Bash
sudo chmod +x /opt/rtbh_updater.sh
Langkah 4: Web GUI Dashboard (Python Flask)
Buat file backend untuk tampilan dashboard visualnya.

Bash
sudo nano /opt/rtbh_gui.py
(Catatan: Paste seluruh kode Python Final Enterprise + Logo yang ada di percakapan sebelumnya ke dalam file ini).

Langkah 5: Menjalankan Sebagai Service Permanen (Systemd)
Agar Web GUI berjalan otomatis di background dan tahan terhadap reboot:

Bash
sudo nano /etc/systemd/system/rtbh-gui.service
Isi dengan:

Ini, TOML
[Unit]
Description=RTBH Monitor Web GUI
After=network.target bird.service

[Service]
Type=simple
User=root
WorkingDirectory=/opt
ExecStart=/usr/bin/python3 /opt/rtbh_gui.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
Aktifkan service-nya:

Bash
sudo systemctl daemon-reload
sudo systemctl enable rtbh-gui
sudo systemctl start rtbh-gui
Langkah 6: Automasi Sinkronisasi (Cronjob)
Jadwalkan agar server otomatis mengunduh dan memperbarui daftar IP setiap 1 jam sekali.

Bash
sudo crontab -e
Tambahkan baris ini di paling bawah:

Plaintext
0 * * * * /bin/bash /opt/rtbh_updater.sh > /dev/null 2>&1
💻 Cara Mengakses Dashboard
Buka browser dan masuk ke http://[IP-SERVER-UBUNTU]:8080

Login menggunakan kredensial default:

Username: admin

Password: sandianda (Bisa diubah di dalam file rtbh_gui.py)

Anda siap mengatur keamanan jaringan dalam satu layar!
