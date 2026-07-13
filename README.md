# ESP32-S3 ST7789 LVGL Example

Proyek ini adalah contoh aplikasi untuk ESP32-S3 yang menggunakan antarmuka SPI untuk mengontrol layar LCD dengan driver ST7789 dan menggunakan **LVGL** (Light and Versatile Graphics Library) untuk merender elemen antarmuka pengguna (UI). 

Aplikasi ini menampilkan sebuah angka (*counter*) di tengah layar dengan ukuran yang sangat besar (menggunakan font `Montserrat 48`), dan nilainya dapat ditambah atau dikurangi menggunakan dua buah tombol fisik.

## Fitur
* **Layar ST7789 via SPI**: Resolusi layar diatur menjadi **170 x 320** piksel.
* **LVGL Porting**: Menggunakan komponen `esp_lvgl_port` untuk mempermudah integrasi LVGL dengan driver layar dari ESP-IDF (`esp_lcd`).
* **Interaksi Tombol**: Dua buah tombol input (pada GPIO 0 dan GPIO 14) yang di-*polling* pada *main loop* untuk mengubah nilai *counter* yang ditampilkan di layar.
* **Font Besar**: Menggunakan *font* bawaan LVGL yang besar (`lv_font_montserrat_48`) untuk menonjolkan angka pada layar.

## Kebutuhan Perangkat Keras (Hardware)

* Sebuah *development board* ESP32-S3.
* Sebuah layar LCD dengan driver **ST7789** (resolusi 170x320 atau sejenis).
* Dua buah tombol (*push button*) yang dihubungkan ke *ground* (karena program menggunakan internal *pull-up* sehingga aktif LOW).

### Konfigurasi Pin (Pinout)

Berikut adalah konfigurasi pin default yang digunakan dalam program ini (bisa dilihat dan diubah pada file `main/main.c`):

| Fungsi Layar / Tombol       | Label Perangkat Keras | Pin GPIO (ESP32-S3) |
|-----------------------------|-----------------------|---------------------|
| Reset Layar                 | TFT_RST               | GPIO 5              |
| Chip Select (CS)            | TFT_CS                | GPIO 6              |
| Data / Command (DC)         | TFT_DC                | GPIO 7              |
| SPI Clock (SCLK / SCL)      | TFT_SCLK              | GPIO 8              |
| SPI Data (MOSI / SDA)       | TFT_MOSI              | GPIO 9              |
| Backlight Control           | TFT_BL                | GPIO 38             |
| **Tombol Penambah (Up)**    | BTN_UP                | **GPIO 0**          |
| **Tombol Pengurang (Down)** | BTN_DOWN              | **GPIO 14**         |

*Catatan: Konfigurasi ini mendefinisikan backlight aktif HIGH (`gpio_set_level(PIN_NUM_BK_LIGHT, 1)`).*

## Kebutuhan Perangkat Lunak (Software)

* **ESP-IDF** (Contoh ini dibuat dan diuji menggunakan ESP-IDF v6.0.1, tetapi juga mendukung v5.x).
* **Komponen**: `lvgl/lvgl` (v8.3.x) dan `espressif/esp_lvgl_port` via *ESP Component Registry* (sudah dikonfigurasi pada `main/idf_component.yml`).

## Cara Penggunaan

### 1. Konfigurasi Proyek

Pastikan *target* chip Anda sudah diatur ke `esp32s3`:
```bash
idf.py set-target esp32s3
```

*(Opsional)* Jika Anda ingin memastikan *font* sudah diaktifkan secara manual melalui Menuconfig:
```bash
idf.py menuconfig
```
Arahkan ke `Component config` > `LVGL configuration` > `Font usage` dan pastikan `Montserrat 48` sudah diaktifkan. (Sebagai catatan, *font* ini sudah diaktifkan secara otomatis melalui penambahan di file `sdkconfig.defaults`).

### 2. Build dan Flash

Jalankan perintah berikut untuk mengkompilasi program, mengunggahnya (*flash*) ke ESP32-S3, dan membuka serial monitor:

```bash
idf.py build
idf.py -p (PORT_ANDA) flash monitor
```
*(Contoh: `idf.py -p COM3 flash monitor` di Windows)*

Untuk keluar dari serial monitor, tekan kombinasi `Ctrl+]`.

## Cara Kerja Program (`main.c`)

1. **Inisialisasi GPIO**: 
   Mengaktifkan pin *backlight* layar sebagai *output* (menyala) dan kedua pin tombol sebagai *input* dengan *pull-up* internal.
2. **Inisialisasi SPI Bus & Panel IO**:
   Menggunakan `spi_bus_initialize` dan `esp_lcd_new_panel_io_spi` untuk menyiapkan komunikasi SPI ke driver ST7789.
3. **Inisialisasi Panel Driver (ST7789)**:
   Mendaftarkan driver layar dengan `esp_lcd_new_panel_st7789`.
   Terdapat perintah `esp_lcd_panel_set_gap(panel_handle, 35, 0)` yang menambahkan *offset* sebesar 35 piksel di sumbu X (umum diperlukan pada layar 170x320 yang tidak memanfaatkan penuh VRAM 240x320 ST7789).
4. **LVGL Porting**:
   Menjalankan inisialisasi awal LVGL melalui `lvgl_port_init()` dan mendaftarkan *display* dengan `lvgl_port_add_disp()`. Secara default, LVGL berjalan dalam *task* khusus di belakang layar.
5. **Pembuatan UI (Label LVGL)**:
   Mengunci akses LVGL terlebih dahulu menggunakan `lvgl_port_lock(0)`. Kemudian, program membuat label, mengatur fontnya menjadi `lv_font_montserrat_48`, menempatkannya di tengah layar, dan membukanya kembali `lvgl_port_unlock()`.
6. **Main Loop (Polling)**:
   Program akan terus mengecek status logika tombol setiap 50 milidetik.
   - Jika `GPIO 0` ditekan (*LOW*), nilai *counter* bertambah.
   - Jika `GPIO 14` ditekan (*LOW*), nilai *counter* berkurang.
   Setiap kali nilai berubah, program melakukan pembaruan teks (menggunakan format `lv_label_set_text_fmt`) dengan mengunci port terlebih dahulu dan menyesuaikan posisi label agar selalu berada di tengah.
