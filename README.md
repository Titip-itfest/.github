# ❄️ TITIP IoT - Smart Cold Locker (Loker Dingin Pintar)

Sistem loker pendingin pintar berbasis IoT untuk pedagang pasar dan pengguna umum. Sistem ini mengintegrasikan aplikasi web **Laravel PWA** dengan modul relay solenoid pada **Raspberry Pi** untuk kontrol penguncian otomatis melalui scan kamera QR Code atau tombol kontrol langsung.

---

## 📌 Daftar Isi
1. [Arsitektur Sistem](#-arsitektur-sistem)
2. [Skema Hardware & Rangkaian Pin Raspberry Pi](#-skema-hardware--rangkaian-pin-raspberry-pi)
3. [Panduan Menjalankan Aplikasi Web (Laravel di Laptop/Server)](#-panduan-menjalankan-aplikasi-web-laravel-di-laptopserver)
4. [Panduan Menjalankan Script Raspberry Pi (`unlock.py`)](#-panduan-menjalankan-script-raspberry-pi-unlockpy)
5. [Alur Operasional Penggunaan & Pengujian](#-alur-operasional-penggunaan--pengujian)
6. [API Integrasi Raspberry Pi](#-api-integrasi-raspberry-pi)
7. [Troubleshooting & Tips](#-troubleshooting--tips)

---

## 🏗 Arsitektur Sistem

```
+-------------------------------------------------------+
|                TITIP PWA (Laravel 11)                 |
|   - Katalog & Pemilihan Slot Loker                    |
|   - Pembayaran QRIS Simulasi                          |
|   - Scan Kamera QR Code & Tombol Buka Langsung        |
|   - Dashboard Monitoring & Pengakhiran Sewa           |
+---------------------------+---------------------------+
                            |
           HTTP API Polling | (Port 8000 / WiFi LAN)
                            v
+-------------------------------------------------------+
|               Raspberry Pi (Python 3)                 |
|                   `pi/unlock.py`                      |
|   - Poller background ke `/api/pi/command` (tiap 1s)  |
|   - Local HTTP Server (Port 5000: `/unlock`)          |
|   - Kontrol Relay Solenoid via GPIO 17 (Pin 11)       |
+---------------------------+---------------------------+
                            |
                   Sinyal   | GPIO 17
                            v
+-------------------------------------------------------+
|                 Hardware Modul Relay                  |
|                 + Solenoid Lock 12V                   |
+-------------------------------------------------------+
```

---

## 🔌 Skema Hardware & Rangkaian Pin Raspberry Pi

Relay mengendalikan solenoid lock (12V) menggunakan catu daya eksternal. Sinyal kontrol dikirim dari pin GPIO Raspberry Pi.

### Diagram Koneksi Pin Header (40-Pin):
| Pin Relay Module | Pin Fisik Raspberry Pi | Keterangan |
| :--- | :--- | :--- |
| **IN / Signal** | **Pin 11 (GPIO 17)** | Sinyal kendali ON/OFF relay |
| **VCC** | **Pin 2 atau 4 (5V)** | Sumber daya modul relay |
| **GND** | **Pin 6, 9, 14, atau 20** | Ground bersama (*Common Ground*) |

> [!NOTE]
> Solenoid 12V wajib menggunakan adaptor eksternal 12V. Hubungkan kutub positif adaptor ke pin **COM** relay, dan pin **NO (Normally Open)** relay ke kabel positif solenoid. Hubungkan kabel negatif solenoid langsung ke kutub negatif adaptor 12V.

---

## 💻 Panduan Menjalankan Aplikasi Web (Laravel di Laptop/Server)

### 1. Prasyarat:
- PHP >= 8.2 dengan ekstensi `pdo_mysql`, `mbstring`, `openssl`
- Composer
- Node.js & NPM
- MariaDB / MySQL

### 2. Konfigurasi Lingkungan:
Pastikan database sudah dibuat dan file `.env` sudah siap:
```bash
cd ~/Projects/Titip/titip-pwa

# Salin .env jika belum ada
cp .env.example .env

# Generate application key
php artisan key:generate

### 3. Migrasi & Seeder Database:
Jalankan migrasi tabel dan data awal loker:
```bash
php artisan migrate --seed
```
*Data awal akan membuat 3 loker (A1, B1, C1) dengan slot **A1-01** berstatus tersedia untuk disewa.*

### 4. Build Aset Frontend (Tailwind & Alpine.js):
```bash
# Build untuk mode produksi
npm run build

# ATAU jalankan dev server jika sedang mengubah tampilan
npm run dev
```

### 5. Jalankan Web Server Laravel:
> [!IMPORTANT]
> Gunakan flag `--host=0.0.0.0` agar server Laravel dapat diakses oleh Raspberry Pi melalui jaringan WiFi/LAN!

```bash
php artisan serve --host=0.0.0.0 --port=8000
```

### 6. Cek IP Laptop Anda:
Cari tahu IP laptop Anda yang terhubung ke WiFi:
```bash
hostname -I
# Contoh hasil: 192.168.1.13
```

*(Jika menggunakan firewall di Linux Fedora/Ubuntu, izinkan port 8000)*:
```bash
sudo firewall-cmd --add-port=8000/tcp --permanent && sudo firewall-cmd --reload
```

---

## 🍓 Panduan Menjalankan Script Raspberry Pi (`unlock.py`)

File `pi/unlock.py` bertindak sebagai bridge cerdas untuk:
1. **Mengontrol Solenoid Lock** via Relay GPIO 17 saat QR discan atau tombol di website ditekan.
2. **Membaca Sensor Suhu & Kelembaban (DS18B20 / Simulasi)** dan mengirimkan telemetry real-time ke Dashboard website setiap 5 detik.

### 1. Wiring Hardware (Relay & Sensor DS18B20)

| Komponen | Pin Raspberry Pi | Keterangan |
|---|---|---|
| **Relay VCC** | Pin 2 / 4 (5V) | Daya modul relay |
| **Relay GND** | Pin 6 / 9 / 14 / 20 (GND) | Ground modul relay |
| **Relay IN** | **GPIO 17 (Pin 11)** | Kontrol pemicu solenoid |
| **DS18B20 VCC (Merah)** | Pin 1 / 17 (3.3V) | Daya sensor suhu |
| **DS18B20 GND (Hitam)** | Pin 6 / 9 / 14 / 20 (GND) | Ground sensor |
| **DS18B20 DATA (Kuning)**| **GPIO 4 (Pin 7)** | Data 1-Wire sensor |

> [!IMPORTANT]
> **Resistor Pull-up Sensor DS18B20:**
> Pasang resistor **4.7kΩ** antara kabel **DATA (Kuning)** dan **VCC 3.3V (Merah)** pada sensor DS18B20 agar pembacaan 1-Wire stabil.

### 2. Mengaktifkan 1-Wire di Raspberry Pi:
Jika menggunakan sensor fisik DS18B20, aktifkan modul 1-Wire:
```bash
sudo raspi-config
# Pilih: Interface Options -> 1-Wire -> Enable -> Yes -> Finish
```
Atau tambahkan baris berikut di `/boot/config.txt` (atau `/boot/firmware/config.txt` pada Pi OS terbaru):
```ini
dtoverlay=w1-gpio,gpiopin=4
```
Lalu reboot Raspberry Pi: `sudo reboot`. Setelah reboot, sensor fisik otomatis terbaca di direktori `/sys/bus/w1/devices/28-*`.

### 3. Pasang Dependensi di Raspberry Pi:
Pada Raspberry Pi OS, pasang library `gpiozero`:
```bash
sudo apt update
sudo apt install python3-gpiozero
```

### 4. Jalankan `unlock.py`:
Jalankan script dengan menyertakan URL server laptop Anda:
```bash
python3 unlock.py http://<IP_LAPTOP>:8000
```
*Contoh*:
```bash
python3 unlock.py http://192.168.1.13:8000
```

### Fitur Interaktif pada Terminal `unlock.py`:
- Ketik `b` lalu Enter: Membuka solenoid lock secara manual.
- Ketik `t -15.5` lalu Enter: Mengubah nilai suhu secara langsung untuk menguji respon live di dashboard.
- Ketik `t auto` lalu Enter: Mengembalikan mode suhu ke sensor fisik/otomatis.
- Ketik `s` lalu Enter: Mengecek status suhu, kelembaban, dan koneksi server saat ini.
- Ketik `q` lalu Enter: Keluar dari program dengan aman.

> [!TIP]
> **Mode Simulasi di Laptop:**
> Anda juga dapat menjalankan `python3 unlock.py` langsung di laptop tanpa Raspberry Pi. Skrip otomatis mendeteksi ketiadaan sensor fisik dan GPIO, lalu beralih ke mode **Simulasi Suhu Berfluktuasi & Simulasi Visual Solenoid**, sehingga Anda tetap bisa menguji seluruh alur kerja sistem.

---

## 🔄 Alur Operasional Penggunaan & Pengujian

1. **Login / Register**:
   - Buka browser di `http://localhost:8000` (atau `http://192.168.1.13:8000`).
   - Akun bawaan seeder: Email `test@example.com` atau buat akun baru via menu Daftar.
2. **Pilih Loker & Durasi**:
   - Masuk ke menu **Pilih Locker**.
   - Pilih loker **A1**, pilih slot **A1-01**.
   - Masukkan durasi sewa (bisa diketik manual 1 - 24 jam atau menggunakan tombol `+` / `-`).
   - Klik **Lanjut ke Pembayaran**.
3. **Pembayaran QRIS**:
   - Halaman menampilkan QRIS dinamis dan total biaya sewa.
   - Klik tombol **"Saya Sudah Bayar"** untuk konfirmasi pembayaran.
4. **Membuka Loker**:
   - Halaman **Scan QRIS / Buka Locker** akan terbuka.
   - Anda memiliki dua cara untuk membuka solenoid:
     1. **Scan Kamera:** Klik *"Mulai Scan Kamera"* lalu arahkan ke QR Code.
     2. **Buka Langsung (Tanpa Scan):** Klik tombol kuning *"Buka Locker Langsung (Tanpa Scan)"*.
   - Relay solenoid GPIO 17 akan langsung **AKTIF selama 3 detik**, lalu otomatis terkunci kembali.
5. **Proteksi Status Sewa & Slot**:
   - Membuka loker **tidak akan** membatalkan atau menyelesaikan masa sewa.
   - Slot loker tetap berstatus **`occupied`** (terisi) selama durasi jam belum habis.
   - Pengguna lain tidak dapat menyewa slot tersebut hingga masa sewa selesai.
6. **Dashboard Monitoring & Mengakhiri Sewa**:
   - Di halaman Dashboard, pengguna dapat memantau sisa waktu countdown dan grafik suhu.
   - Jika ingin mengambil barang dan menyelesaikan sesi sewa sebelum waktunya, klik **"Akhiri Sewa Lebih Awal"**. Solenoid akan terbuka kembali dan status slot berubah menjadi tersedia (**`available`**).

---

## 🌐 API Integrasi Raspberry Pi

Endpoints berikut tersedia secara publik tanpa batasan CSRF untuk komunikasi Raspberry Pi:

| Method | Endpoint | Deskripsi |
| :--- | :--- | :--- |
| `GET` | `/api/pi/command` | Polling antrean instruksi buka loker (dipanggil oleh Raspberry Pi tiap detik). |
| `POST` | `/api/pi/confirm` | Konfirmasi dari Raspberry Pi bahwa relay solenoid telah dibuka. |
| `GET` | `/api/pi/status` | Heartbeat pengecekan status server dan status antrean perintah. |
| `POST/GET`| `/api/pi/trigger-unlock?locker=A1&slot=A1-01` | Memicu buka solenoid secara manual (berguna untuk testing cURL). |

---

## 🛠 Troubleshooting & Tips

### 1. Kamera Tidak Bisa Dibuka di Browser HP
- **Penyebab:** Browser modern (Chrome/Safari) memblokir akses kamera (*getUserMedia*) jika menggunakan protokol `http://` melalui IP jaringan (bukan `localhost`).
- **Solusi:** 
  - Gunakan tombol kuning **"Buka Locker Langsung (Tanpa Scan)"** saat pengujian dari perangkat yang tidak mendukung HTTPS lokal.
  - Atau gunakan tunneling HTTPS seperti **ngrok** / **Cloudflare Tunnel** (`ngrok http 8000`).

### 2. Raspberry Pi Melaporkan Server Tidak Dapat Dijangkau
- Pastikan laptop dan Raspberry Pi berada pada satu WiFi/hotspot yang sama.
- Pastikan Laravel dijalankan dengan `--host=0.0.0.0` (bukan hanya `php artisan serve`).
- Pastikan firewall laptop tidak memblokir port 8000:
  ```bash
  sudo firewall-cmd --add-port=8000/tcp --permanent && sudo firewall-cmd --reload
  ```

### 3. Relay Menyala Terus Saat Idle (Terbalik)
- Jika modul relay Anda tipe *Active-High* dan menyala terus saat posisi standby, buka file `pi/unlock.py` lalu ubah baris 21:
  ```python
  ACTIVE_HIGH = True  # Ubah dari False ke True
  ```

---

*Dikembangkan untuk Proyek TITIP - Loker Dingin IoT Pasar.*
# .github
