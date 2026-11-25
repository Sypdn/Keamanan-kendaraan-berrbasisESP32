# Sistem Keamanan Kendaraan Berbasis ESP32

Sistem keamanan kendaraan berbasis ESP32 dengan deteksi getaran, kontrol relay, pelacakan GPS, dan dashboard web. Firmware dapat memutus relay serta menyalakan buzzer secara lokal saat getaran terdeteksi tanpa menunggu respon server.

## Fitur

- Deteksi getaran melalui sensor SW-420 atau sejenis pada pin GPIO 27.
- Pemutusan relay otomatis pada GPIO 26 dan aktivasi buzzer pada GPIO 25.
- Pelacakan posisi melalui GPS NEO-6M pada GPIO 16 (RX) dan GPIO 17 (TX).
- Kontrol relay jarak jauh dari dashboard melalui PHP dan MySQL.
- Kontrol relay tambahan menggunakan penerima RF RX500 pada GPIO 32.
- Monitoring getaran, relay, status Wi-Fi/ESP32, status GPS, dan koordinat pada dashboard.
- Peta lokasi menggunakan Leaflet dengan OpenStreetMap.
- Penyimpanan URL server dan URL lokal secara permanen pada ESP32 dengan `Preferences`.
- Migrasi tabel dan kolom database otomatis saat endpoint PHP dijalankan.

## Cara Kerja

1. ESP32 membaca sensor getaran dan menerima data GPS.
2. Setelah startup 3 detik, sensor getaran diaktifkan dan dipantau melalui interrupt serta pembacaan berkala.
3. Saat getaran terdeteksi, relay diputus dan buzzer menyala langsung oleh ESP32.
4. ESP32 mengirim status getaran, relay, dan GPS ke server setiap sekitar 5 detik.
5. Dashboard mengambil status setiap 2 detik dan menampilkan status serta lokasi pada peta.
6. Perintah relay dari dashboard disimpan ke database, kemudian diambil ESP32 setiap sekitar 1 detik.
7. ESP32 mengirim ACK beserta hasil `applied` atau `rejected` agar dashboard mengetahui perintah telah dieksekusi.

Catatan: saat alarm getaran aktif, sambungan relay dari dashboard atau remote diperlakukan sebagai override sementara selama 5 detik. Relay yang diputus karena getaran tidak tersambung kembali otomatis oleh sistem.

## Struktur Proyek

```text
Keamanan-kendaraan-berbasisESP32/
├── ESP32/
│   └── sistem_keamanan/
│       └── sistem_keamanan.ino
├── api/
│   ├── update_status.php
│   ├── get_status.php
│   ├── send_command.php
│   ├── get_command.php
│   ├── ack_command.php
│   └── command_status.php
├── app.js
├── db.php
├── index.php
├── schema.sql
├── start_all.bat
├── stop_all.bat
└── style.css
```

## Kebutuhan

### Hardware

| Komponen | Pin ESP32 | Keterangan |
|---|---:|---|
| Sensor getaran SW-420 | GPIO 27 | Input dengan `INPUT_PULLDOWN` |
| Relay | GPIO 26 | HIGH = relay diputus |
| Buzzer | GPIO 25 | HIGH = buzzer menyala |
| GPS NEO-6M TX | GPIO 16 | RX ESP32 |
| GPS NEO-6M RX | GPIO 17 | TX ESP32 |
| Penerima RF RX500 | GPIO 32 | Opsional |

Sesuaikan level tegangan dan ground semua modul. Gunakan modul relay yang aman untuk beban kendaraan.

### Software

- Windows dengan XAMPP dan MySQL
- Arduino IDE atau PlatformIO
- Library Arduino:
  - `TinyGPS++`
  - `WiFi`
  - `WiFiClientSecure`
  - `HTTPClient`
  - `ArduinoJson`
  - `Preferences`
  - `RCSwitch`
- PHP dengan ekstensi `mysqli`
- Browser modern dengan akses internet untuk Leaflet/OpenStreetMap

## Instalasi

### 1. Konfigurasi database

Buat database MySQL terlebih dahulu, misalnya `keamanan_kendaraan`. Kemudian sesuaikan kredensial pada `db.php`:

```php
$DB_HOST = "localhost";
$DB_USER = "root";
$DB_PASS = "";
$DB_NAME = "keamanan_kendaraan";
```

Tabel `status`, `commands`, dan `logs` dibuat serta kolom yang diperlukan ditambahkan secara otomatis saat salah satu endpoint PHP diakses. `schema.sql` tersedia sebagai dokumentasi struktur, bukan sebagai syarat instalasi.

### 2. Konfigurasi dan upload firmware

Buka `ESP32/sistem_keamanan/sistem_keamanan.ino` lalu ubah kredensial Wi-Fi:

```cpp
const char* ssid     = "NAMA_WIFI";
const char* password = "PASSWORD_WIFI";
```

Pilih board ESP32 yang sesuai pada Arduino IDE, instal seluruh library yang diperlukan, lalu upload firmware.

### 3. Atur URL server ESP32

Setelah firmware berjalan, buka Serial Monitor dengan baud `115200`. Atur URL publik/tunnel dari Serial Monitor:

```text
link="https://domain.example/api"
```

Jika server dapat diakses langsung dalam jaringan lokal, URL lokal juga dapat disimpan:

```text
linklocal="http://192.168.1.10/Keamanan-kendaraan-berbasisESP32/api"
```

Nilai URL disimpan pada `Preferences` ESP32 sehingga tidak perlu dikirim ulang setelah restart. Awalan `/api` hanya perlu disesuaikan dengan struktur folder deployment.

Jangan mengakhiri URL dengan `/`.

### 4. Jalankan dashboard

Jika proyek berada langsung di folder XAMPP:

```text
http://localhost/Keamanan-kendaraan-berbasisESP32/
```

Untuk akses dari jaringan lain, jalankan Apache dan MySQL melalui XAMPP. Untuk akses melalui internet, gunakan tunnel yang meneruskan port 80 ke server lokal.

Pada Windows, `start_all.bat` dapat menjalankan Apache, MySQL, serta membuka SSH tunnel `localhost.run`. Sebelum dijalankan, pastikan:

- Lokasi XAMPP sesuai nilai `XAMPP_DIR` pada script.
- SSH tersedia pada `PATH`.
- Database sudah dibuat.

Gunakan `stop_all.bat` untuk menghentikan Apache, MySQL, dan SSH tunnel.

## Ringkasan API

| Endpoint | Method | Pemanggil | Fungsi |
|---|---|---|---|
| `/api/update_status.php` | `POST` | ESP32 | Mengirim getaran, relay, dan data GPS |
| `/api/get_status.php` | `GET` | Dashboard | Mengambil status terakhir dan umur heartbeat |
| `/api/send_command.php` | `POST` | Dashboard | Membuat perintah relay baru |
| `/api/get_command.php` | `GET` | ESP32 | Mengambil satu perintah relay yang belum dieksekusi |
| `/api/ack_command.php` | `POST` | ESP32 | Mengirim status hasil eksekusi perintah |
| `/api/command_status.php` | `GET` | Dashboard | Memeriksa apakah perintah sudah dieksekusi |

ESP32 dianggap terputus dari dashboard apabila tidak menerima heartbeat lebih dari 20 detik.

## Remote RF

Kode firmware mendukung remote dengan data RF `2451281` pada penerima RX500. Nilai ini dapat diubah melalui konstanta:

```cpp
const unsigned long KODE_REMOTE = 2451281;
```

Tombol RF pertama membalik kondisi relay: dari terhubung menjadi putus, atau sebaliknya. Ada jeda minimum 500 ms antara dua penerimaan.

## Pengujian Cepat

1. Buka dashboard dan pastikan status berubah menjadi `ESP32 TERHUBUNG`.
2. Aktifkan sensor getaran.
3. Pastikan indikator getaran menjadi `TERDETEKSI`, relay menjadi `TERPUTUS`, dan buzzer menyala.
4. Lihat koordinat GPS pada panel lokasi jika sinyal GPS tersedia.
5. Uji sakelar relay pada dashboard.
6. Pastikan dashboard menampilkan konfirmasi `ESP32 mengonfirmasi relay ...`.

Jika relay tidak merespons dashboard, periksa Serial Monitor, koneksi Wi-Fi, URL yang disimpan, serta hasil request pada endpoint API.

## Catatan Keamanan

- Jangan commit SSID, password Wi-Fi, atau kredensial database produksi.
- Dashboard saat ini tidak memiliki autentikasi. Batasi akses ke jaringan pribadi atau tambahkan autentikasi sebelum deployment publik.
- Endpoint relay dapat mengubah daya kendaraan, sehingga jangan mengekspos server tanpa kontrol akses dan HTTPS.
- Gunakan URL publik HTTPS yang dapat diakses ESP32 serta sertifikat yang valid atau layanan tunnel yang mendukung konfigurasi TLS.
- Pastikan sambungan relay terhadap sistem kelistrikan kendaraan aman dan sesuai spesifikasi komponen.

##  Hak Cipta & Lisensi

© 2026 Sypdn. Hak Cipta Dilindungi Undang-Undang.
