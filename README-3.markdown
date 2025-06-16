# Video Management System (VMS) untuk Backup CCTV

## Deskripsi
Sistem ini dirancang untuk mengelola backup otomatis rekaman CCTV dari kantor pusat (HO) dan cabang, dengan penyimpanan selama 6 bulan. Sistem mendukung backup ke cloud atau perangkat keras lokal, akses cepat untuk memutar/view/download rekaman, dan penghapusan otomatis setelah 6 bulan. Solusi ini mengatasi kendala lambatnya upload ke Telegram dengan mengoptimalkan proses backup menggunakan penyimpanan cloud (contoh: AWS S3) atau NAS lokal dengan kompresi video.

## Kebutuhan Sistem
1. **Backup Otomatis**: Rekaman CCTV dari HO dan cabang disimpan otomatis ke penyimpanan terpusat.
2. **Penyimpanan**: Mendukung cloud (AWS S3/Google Cloud Storage) atau perangkat keras (NAS/SAN).
3. **Akses Cepat**: Rekaman dapat diputar, dilihat, dan diunduh kapan saja melalui antarmuka web.
4. **Penghapusan Otomatis**: Rekaman dihapus otomatis setelah 6 bulan untuk mengelola ruang penyimpanan.
5. **Optimasi Upload**: Mengatasi kendala lambatnya upload ke Telegram dengan kompresi video dan penyimpanan terdistribusi.

## Prasyarat
- **Perangkat Keras**:
  - Server lokal atau NAS dengan kapasitas minimal 10TB (estimasi untuk 6 bulan, tergantung jumlah kamera dan resolusi).
  - Koneksi internet stabil (minimal 50 Mbps upload untuk cloud).
- **Perangkat Lunak**:
  - Python 3.8+ untuk script otomatisasi.
  - FFmpeg untuk kompresi video.
  - AWS CLI atau Google Cloud SDK untuk integrasi cloud.
  - Web server (contoh: Flask/Django) untuk antarmuka akses.
- **Layanan Cloud** (opsional):
  - AWS S3/Google Cloud Storage untuk penyimpanan.
  - AWS Lambda/Cloud Functions untuk otomatisasi penghapusan.

## Instalasi
1. **Setup Server Lokal**:
   - Pasang Python, FFmpeg, dan dependensi:
     ```bash
     pip install boto3 flask schedule
     sudo apt-get install ffmpeg
     ```
   - Konfigurasi NAS atau mount drive untuk penyimpanan lokal.
2. **Setup Cloud** (jika digunakan):
   - Buat bucket di AWS S3 atau Google Cloud Storage.
   - Konfigurasi kredensial di `~/.aws/credentials` atau Google Cloud SDK.
3. **Deploy Script**:
   - Salin script `backup_cctv.py` (lihat bagian Contoh Script) ke server.
   - Jalankan script:
     ```bash
     python backup_cctv.py
     ```
4. **Setup Antarmuka Web**:
   - Deploy aplikasi Flask untuk akses rekaman.
   - Konfigurasi firewall untuk port 5000 (atau port lain).

## Penggunaan
1. **Backup Otomatis**:
   - Script `backup_cctv.py` secara periodik memeriksa folder CCTV, mengompresi video menggunakan FFmpeg, dan mengunggahnya ke cloud/NAS.
2. **Akses Rekaman**:
   - Buka antarmuka web di `http://<server-ip>:5000`.
   - Cari rekaman berdasarkan tanggal, lokasi (HO/cabang), atau kamera.
   - Putar langsung atau unduh file.
3. **Penghapusan Otomatis**:
   - Script menghapus file yang lebih tua dari 6 bulan setiap minggu.

## Diagram Alur
Berikut adalah diagram alur sistem VMS:

```mermaid
graph TD
    A[Kamera CCTV HO/Cabang] -->|Rekam Video| B[Server Lokal]
    B -->|Kompresi FFmpeg| C[Penyimpanan Sementara]
    C -->|Upload Otomatis| D{Storage}
    D --> E[Cloud: AWS S3/Google Cloud]
    D --> F[Lokal: NAS/SAN]
    E -->|Akses API| G[Web Interface]
    F -->|Akses File| G
    G -->|View/Download| H[User]
    C -->|Cek Umur File| I[Script Penghapusan]
    I -->|Hapus > 6 Bulan| J[File Dihapus]
```

## Contoh Script
Berikut adalah contoh script Python untuk otomatisasi backup:

```python
import os
import boto3
import schedule
import time
from datetime import datetime, timedelta
import subprocess

# Konfigurasi
STORAGE_TYPE = "cloud"  # "cloud" atau "local"
AWS_BUCKET = "cctv-backup-bucket"
LOCAL_PATH = "/mnt/nas/cctv"
TEMP_PATH = "/tmp/cctv"

def compress_video(input_path, output_path):
    """Kompresi video menggunakan FFmpeg."""
    cmd = f"ffmpeg -i {input_path} -vcodec libx264 -crf 28 {output_path}"
    subprocess.run(cmd, shell=True)

def upload_to_s3(file_path, file_name):
    """Unggah file ke AWS S3."""
    s3 = boto3.client("s3")
    s3.upload_file(file_path, AWS_BUCKET, f"cctv/{file_name}")

def delete_old_files():
    """Hapus file yang lebih tua dari 6 bulan."""
    six_months_ago = datetime.now() - timedelta(days=180)
    for root, _, files in os.walk(TEMP_PATH):
        for file in files:
            file_path = os.path.join(root, file)
            file_time = datetime.fromtimestamp(os.path.getmtime(file_path))
            if file_time < six_months_ago:
                os.remove(file_path)

def process_cctv_files():
    """Proses backup CCTV."""
    for root, _, files in os.walk("/path/to/cctv/source"):
        for file in files:
            input_path = os.path.join(root, file)
            output_path = os.path.join(TEMP_PATH, f"compressed_{file}")
            compress_video(input_path, output_path)
            if STORAGE_TYPE == "cloud":
                upload_to_s3(output_path, file)
            else:
                os.rename(output_path, os.path.join(LOCAL_PATH, file))
            os.remove(input_path)

# Jadwal
schedule.every(1).hours.do(process_cctv_files)
schedule.every(1).weeks.do(delete_old_files)

while True:
    schedule.run_pending()
    time.sleep(60)
```

## Pemecahan Masalah
- **Upload Lambat**: Pastikan koneksi internet stabil. Gunakan FFmpeg dengan CRF lebih tinggi untuk kompresi lebih kecil.
- **Penyimpanan Penuh**: Tambah kapasitas NAS atau gunakan cloud dengan tier penyimpanan murah (contoh: S3 Glacier).
- **Akses Web Gagal**: Periksa firewall dan port yang digunakan.

## Kontribusi
Silakan ajukan pull request atau laporkan isu di repositori proyek.

## Lisensi
MIT License