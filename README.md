# Helmet Care — Dehumidifier Station (Web & IoT Platform)

Platform web monitoring dan kontrol berbasis **Laravel 13.x (Blade MVC)** untuk sistem embedded IoT **Helmet Care Dehumidifier Station** berbasis mikrokontroler **ESP32**, sensor kelembapan & suhu **DHT22**, sensor jarak ultrasonik **HC-SR04**, modul **Relay Kipas**, dan indikator **LED**.

---

## Fitur Utama

1. **Dashboard Station (`/`)**
   - Pemantauan metrik sensor real-time: **Kelembapan Saat Ini (% RH)**, **Suhu Dalam Helm (°C)**, **Deteksi Helm (cm)**, serta status **Kipas & LED**.
   - **Grafik Kelembapan Real-Time** dengan garis batas histeresis (`Batas nyala 65%` & `Batas kering 55%`) dan pembaruan otomatis (*live polling* setiap 2 detik).
   - Panel **Siklus Pengeringan** aktif beserta tombol override **Matikan / Nyalakan Kipas Manual** via MQTT.
   - Tabel ringkasan **Log Pengeringan Harian**.
2. **Riwayat Pengeringan (`/riwayat`)**
   - Filter berdasarkan rentang tanggal (`7 hari terakhir`, `30 hari terakhir`, `Semua waktu`) dan status siklus (`Semua`, `Berjalan`, `Selesai`).
   - 3 kartu ringkasan KPI: **Total Siklus**, **Rata-rata Durasi (menit)**, dan **Rata-rata Penurunan RH (poin)**.
   - Grafik batang **Durasi Pengeringan per Hari** (hijau untuk siklus selesai, oranye untuk siklus berjalan).
   - Fitur **Ekspor CSV** (`/riwayat/export`) dan tabel riwayat berhalaman (*pagination*).
3. **Perangkat (`/perangkat`)**
   - Status koneksi **ESP32 Station 01** dan diagnostik keempat komponen hardware (`HC-SR04`, `DHT22`, `Modul Relay`, `Lampu LED`).
   - Informasi jaringan Wi-Fi (`SSID`, `RSSI dBm`, `Endpoint Server`, `Topic MQTT`, `Versi Firmware`) dan log **Aktivitas Perangkat**.
   - Panel interaktif **Simulator Telemetry MQTT** untuk menguji respons website sebelum perangkat fisik ESP32 terhubung.
4. **Pengaturan Ambang (`/pengaturan`)**
   - Konfigurasi nilai ambang histeresis sensor (`Kipas menyala di atas % RH`, `Kipas mati di bawah % RH`, `Jarak deteksi helm cm`, `Interval kirim data detik`).
   - Pengaturan **Mode Operasi** (`Otomatis` & notifikasi selesai kering) serta tombol **Kembalikan Default** yang langsung menyinkronkan konfigurasi ke ESP32 melalui MQTT.

---

## Arsitektur MVC & Komunikasi IoT (MQTT + REST Bridge)

```text
app/
├── Console/Commands/
│   ├── MqttListenCommand.php        # Daemon subscriber MQTT (`php artisan mqtt:listen`)
│   └── MqttSimulateCommand.php      # Simulator CLI payload ESP32 (`php artisan mqtt:simulate`)
├── Http/Controllers/
│   ├── Api/TelemetryController.php  # Endpoint JSON IoT (`/api/iot/telemetry`, `/api/iot/state`, `/api/iot/config`)
│   ├── DashboardController.php      # Halaman utama Dashboard Station & manual fan toggle
│   ├── DryingHistoryController.php  # Halaman Riwayat Pengeringan & Ekspor CSV
│   ├── DeviceController.php         # Halaman status Perangkat & simulator web
│   └── ThresholdSettingController.php # Halaman Pengaturan Ambang histeresis
├── Models/
│   ├── Station.php                  # Konfigurasi station, snapshot sensor terbaru, & status aktuator
│   ├── DryingCycle.php              # Siklus pengeringan harian & perhitungan penurunan RH
│   ├── SensorReading.php            # Data time-series sensor DHT22 & HC-SR04
│   └── DeviceActivityLog.php        # Log kronologis aktivitas perangkat
└── Services/
    ├── MqttBrokerService.php        # Integrasi client MQTT (`php-mqtt/client`)
    └── TelemetryIngestionService.php # Mesin status histeresis & siklus pengeringan
```

### Kontrak Topik MQTT (`config/mqtt.php`)

| Arah | Topik MQTT | Deskripsi |
| :--- | :--- | :--- |
| **ESP32 &rarr; Server** | `helmet-care/stations/station-01/telemetry` | Mengirim pembacaan sensor DHT22 & HC-SR04 setiap `N` detik |
| **Server &rarr; ESP32** | `helmet-care/stations/station-01/commands` | Mengirim perintah manual relay kipas (`ON`/`OFF`) & status LED |
| **Server &rarr; ESP32** | `helmet-care/stations/station-01/config` | Menyinkronkan perubahan ambang batas (% RH, jarak cm, interval) |

### Contoh Payload JSON dari ESP32 (MQTT atau HTTP `POST /api/iot/telemetry`)

```json
{
  "station_code": "station-01",
  "humidity": 61.0,
  "temperature": 31.4,
  "distance_cm": 8.0,
  "wifi_ssid": "HelmetCare-IoT-Lab",
  "wifi_rssi": -54,
  "firmware_version": "v1.4.2-esp32"
}
```

---

## Panduan Instalasi & Menjalankan Proyek

1. **Setup satu perintah (otomatis install Composer, `.env`, key, migrasi SQLite, & build NPM):**
   ```bash
   composer run setup
   php artisan db:seed
   ```
   *(Atau jalankan manual langkah 1–3 di bawah ini)*

2. **Instalasi manual (opsional jika tidak memakai `composer run setup`):**
   ```bash
   composer install
   cp .env.example .env
   php artisan key:generate
   touch database/database.sqlite
   php artisan migrate:fresh --seed
   npm install
   npm run build
   ```

---

## Quick Commands to Run & Test

Salin dan jalankan perintah berikut di terminal untuk menjalankan website, menyimulasikan sensor ESP32 via MQTT, dan menjalankan pengujian BDD:

```bash
# Start the local web server (http://127.0.0.1:8000)
php artisan serve

# Simulate an ESP32 sending MQTT telemetry from CLI
php artisan mqtt:simulate --humidity=58.0 --temperature=31.2 --distance=8.0

# Start the background MQTT topic subscriber (when connecting real ESP32 hardware)
php artisan mqtt:listen

# Run the BDD feature test suite
php artisan test
```
