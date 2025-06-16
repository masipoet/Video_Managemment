# Video Management System (VMS) untuk Backup dan Streaming CCTV

## Deskripsi
Sistem ini dirancang untuk mengelola **backup otomatis** dan **streaming langsung** rekaman CCTV dari kantor pusat (HO) dan cabang, dengan penyimpanan selama 6 bulan. Sistem mendukung backup ke cloud (AWS S3/Google Cloud Storage) atau perangkat keras lokal (NAS/SAN), akses cepat untuk memutar/melihat/mengunduh rekaman, streaming langsung melalui antarmuka web, dan penghapusan otomatis setelah 6 bulan. Solusi ini mengatasi kendala lambatnya upload ke Telegram dengan mengoptimalkan proses backup menggunakan kompresi video dan penyimpanan terdistribusi.

## Kebutuhan Sistem
1. **Backup Otomatis**: Rekaman CCTV dari HO dan cabang disimpan otomatis ke penyimpanan terpusat.
2. **Streaming Langsung**: CCTV dapat dilihat secara langsung melalui antarmuka web menggunakan protokol RTSP/HLS.
3. **Penyimpanan**: Mendukung cloud (AWS S3/Google Cloud Storage) atau perangkat keras (NAS/SAN).
4. **Akses Cepat**: Rekaman dapat diputar, dilihat, dan diunduh kapan saja; streaming langsung tersedia on-demand.
5. **Penghapusan Otomatis**: Rekaman dihapus otomatis setelah 6 bulan untuk mengelola ruang penyimpanan.
6. **Optimasi Upload**: Mengatasi kendala lambatnya upload dengan kompresi video (H.264/H.265) dan prioritas jaringan via Mikrotik QoS.

## Prasyarat
- **Perangkat Keras**:
  - Server lokal atau NAS dengan kapasitas minimal 10TB (estimasi untuk 6 bulan, tergantung jumlah kamera dan resolusi).
  - Koneksi internet stabil (minimal 50 Mbps upload untuk cloud, 2 Mbps per kamera untuk streaming).
  - Router Mikrotik untuk pengaturan QoS (opsional, untuk optimasi jaringan).
- **Perangkat Lunak**:
  - Python 3.8+ untuk script otomatisasi.
  - FFmpeg untuk kompresi dan transcoding video.
  - Nginx dengan modul RTMP untuk streaming.
  - AWS CLI atau Google Cloud SDK untuk integrasi cloud.
  - Web server (Flask/Django) untuk antarmuka akses.
- **Layanan Cloud** (opsional):
  - AWS S3/Google Cloud Storage untuk penyimpanan.
  - AWS Lambda/Cloud Functions untuk otomatisasi penghapusan.

## Instalasi
1. **Setup Server Lokal**:
   - Pasang Python, FFmpeg, dan Nginx:
     ```bash
     pip install boto3 flask schedule
     sudo apt-get install ffmpeg nginx libnginx-mod-rtmp
     ```
   - Konfigurasi NAS atau mount drive untuk penyimpanan lokal.
2. **Setup Cloud** (jika digunakan):
   - Buat bucket di AWS S3 atau Google Cloud Storage.
   - Konfigurasi kredensial di `~/.aws/credentials` atau Google Cloud SDK.
3. **Setup Streaming Server**:
   - Konfigurasi Nginx untuk RTMP/HLS di `/etc/nginx/nginx.conf`:
     ```nginx
     rtmp {
         server {
             listen 1935;
             application live {
                 live on;
                 hls on;
                 hls_path /tmp/hls;
             }
         }
     }
     ```
   - Jalankan FFmpeg untuk menarik RTSP dari kamera:
     ```bash
     ffmpeg -i rtsp://<camera-ip>/stream -c:v copy -c:a aac -f flv rtmp://<server-ip>/live/<camera-id>
     ```
4. **Deploy Script**:
   - Salin script `backup_cctv.py` (lihat bagian Contoh Script) ke server.
   - Jalankan script:
     ```bash
     python backup_cctv.py
     ```
5. **Setup Antarmuka Web**:
   - Deploy aplikasi Flask untuk akses rekaman dan streaming.
   - Tambahkan HLS.js untuk streaming langsung:
     ```html
     <video id="player" controls></video>
     <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
     <script>
         if (Hls.isSupported()) {
             var video = document.getElementById('player');
             var hls = new Hls();
             hls.loadSource('http://<server-ip>/hls/<camera-id>.m3u8');
             hls.attachMedia(video);
         }
     </script>
     ```
   - Konfigurasi firewall untuk port 5000 (web) dan 1935 (RTMP).

## Penggunaan
1. **Backup Otomatis**:
   - Script `backup_cctv.py` secara periodik memeriksa folder CCTV, mengompresi video menggunakan FFmpeg, dan mengunggahnya ke cloud/NAS.
2. **Streaming Langsung**:
   - Buka antarmuka web di `http://<server-ip>:5000`.
   - Pilih kamera (HO/cabang) dan klik "Live" untuk streaming langsung.
3. **Akses Rekaman**:
   - Cari rekaman berdasarkan tanggal, lokasi, atau kamera.
   - Putar langsung atau unduh file.
4. **Penghapusan Otomatis**:
   - Script menghapus file yang lebih tua dari 6 bulan setiap minggu.

## Diagram Alur
Berikut adalah diagram alur sistem VMS dengan dukungan streaming langsung:

```mermaid
graph TD
    A[Kamera CCTV HO/Cabang] -->|RTSP Stream| B[Streaming Server]
    A -->|Rekam Video| C[Server Lokal]
    B -->|HLS/WebRTC| D[Web Interface]
    C -->|Kompresi FFmpeg| E[Penyimpanan Sementara]
    E -->|Upload Otomatis| F{Storage}
    F --> G[Cloud: AWS S3/Google Cloud]
    F --> H[Lokal: NAS/SAN]
    G -->|Akses API| D
    H -->|Akses File| D
    D -->|Live/View/Download| I[User]
    E -->|Cek Umur File| J[Script Penghapusan]
    J -->|Hapus > 6 Bulan| K[File Dihapus]
```

## Contoh Script
Berikut adalah contoh script Python untuk otomatisasi backup (streaming ditangani oleh FFmpeg dan Nginx):

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

## Optimasi Jaringan (Mikrotik)
Untuk memastikan streaming lancar:
- Konfigurasi QoS di Mikrotik untuk memprioritaskan port RTMP (1935) dan HLS (80/443):
  ```mikrotik
  /queue simple
  add name=streaming target=<server-ip> dst-port=1935,80,443 protocol=tcp priority=1
  ```
- Batasi bandwidth upload backup agar tidak mengganggu streaming:
  ```mikrotik
  /queue simple
  add name=backup target=<server-ip> max-limit=10M/10M
  ```

## Pemecahan Masalah
- **Upload Lambat**: Gunakan FFmpeg dengan CRF lebih tinggi atau tingkatkan bandwidth.
- **Streaming Terputus**: Periksa latensi jaringan dan pastikan kamera mendukung RTSP stabil.
- **Penyimpanan Penuh**: Gunakan S3 Glacier untuk penyimpanan jangka panjang atau tambah kapasitas NAS.
- **Akses Web Gagal**: Periksa firewall untuk port 5000, 1935, 80, dan 443.

## Kontribusi
Silakan ajukan pull request atau laporkan isu di repositori proyek.

## Lisensi
MIT License