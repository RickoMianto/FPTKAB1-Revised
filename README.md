# FPTKAB1-Revised

# Aplikasi Analisis Sentimen

## Daftar Isi
- [Pendahuluan](#pendahuluan)
- [Arsitektur dan Spesifikasi VM](#arsitektur-dan-spesifikasi-vm)
- [Implementasi dan Konfigurasi](#implementasi-dan-konfigurasi)
  - [Konfigurasi Worker](#konfigurasi-worker)
  - [Setup Frontend](#setup-frontend)
  - [Setup Database MongoDB](#setup-database-mongodb)
  - [Setup Backend](#setup-backend)
  - [Konfigurasi Load Balancer](#konfigurasi-load-balancer)
- [Pengujian Endpoint](#pengujian-endpoint)
- [Hasil Pengujian Load](#hasil-pengujian-load)
- [Revisi, Kesimpulan, dan Saran](#revisi-kesimpulan-dan-saran)

## Pendahuluan
Anda adalah seorang lulusan Teknologi Informasi dengan keahlian dalam merancang, membangun, dan mengelola aplikasi berbasis cloud. Dalam proyek ini, Anda ditugaskan untuk mendeploy aplikasi Analisis Sentimen menggunakan backend yang ditulis dalam Python (`sentiment-analysis.py`) dengan endpoint sebagai berikut:

### Endpoint
- **Analyze Text**
  - **Endpoint:** POST /analyze
  - **Deskripsi:** Endpoint ini menerima input teks dan mengembalikan skor sentimen dari teks tersebut.
  - **Request:**
    ```json
    {
      "text": "Teks Anda di sini"
    }
    ```
  - **Response:**
    ```json
    {
      "sentiment": <sentiment_score>
    }
    ```

- **Retrieve History**
  - **Endpoint:** GET /history
  - **Deskripsi:** Endpoint ini mengambil riwayat teks yang telah dianalisis sebelumnya beserta skor sentimennya.
  - **Response:**
    ```json
    [
      {
        "text": "Teks sebelumnya di sini",
        "sentiment": <sentiment_score>
      },
      ...
    ]
    ```

## Arsitektur dan Spesifikasi VM
Arsitektur menggunakan 3 droplet Digital Ocean dengan spesifikasi sebagai berikut:

- 3 VM x $21 = $63/bulan

![Pratinjau Droplet](link_gambar_di_sini)

## Implementasi dan Konfigurasi
Arsitektur terdiri dari 2 VM worker dan 1 VM load balancer.

### Konfigurasi Worker
Setiap VM worker dikonfigurasi dengan:

- Nginx untuk frontend
- Python3-pip, Python3-venv untuk setup backend
- Flask
- Flask-CORS
- TextBlob
- PyMongo
- Gunicorn
- Gevent
- MongoDB

### Setup Frontend
1. Update dan instal Nginx:
    ```bash
    sudo apt-get update
    sudo apt-get install nginx
    ```

2. Clone repository dan pindahkan file frontend:
    ```bash
    git clone https://github.com/fuaddary/fp-tka.git
    mv fp-tka/Resources/FE/index.html /var/www/html/index.html
    mv fp-tka/Resources/FE/styles.css /var/www/html/styles.css
    sudo systemctl restart nginx
    ```

3. Konfigurasi Nginx untuk frontend dan backend pada port yang sama:
    ```bash
    sudo nano /etc/nginx/sites-available/default
    ```

    Modifikasi konfigurasi sesuai kebutuhan dan restart Nginx:
    ```bash
    sudo systemctl restart nginx
    ```

### Setup Database MongoDB
1. Instal MongoDB:
    ```bash
    sudo apt-get install gnupg curl
    curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
    echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
    sudo apt-get update
    sudo apt-get install -y mongodb-org
    sudo systemctl enable mongod
    sudo ufw allow 27017
    ```

### Setup Backend
1. Instal dependensi Python:
    ```bash
    sudo apt-get update
    sudo apt-get install python3-pip python3-venv
    ```

2. Setup virtual environment dan instal paket:
    ```bash
    cd fp-tka/Resources/BE
    python3 -m venv venv
    source venv/bin/activate
    pip install flask flask-cors textblob pymongo gunicorn gevent
    ```

3. Ganti nama dan jalankan backend:
    ```bash
    mv sentiment-analysis.py sentiment_analysis.py
    gunicorn -b 0.0.0.0:80 -w 5 -k gevent --timeout 60 --graceful-timeout 60 sentiment_analysis:app --daemon
    ```

### Konfigurasi Load Balancer
1. Instal dan konfigurasi Nginx:
    ```bash
    sudo apt-get update
    sudo apt-get install nginx
    sudo unlink /etc/nginx/sites-enabled/default
    sudo nano /etc/nginx/sites-available/app
    ```

    Konfigurasi load balancer:
    ```nginx
    upstream backend_servers {
      server 152.42.251.221;
      server 128.199.103.250;
    }

    server {
      listen 80;
      server_name 157.230.246.133;

      location / {
        proxy_cache my_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;

        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;

        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header X-Cached $upstream_cache_status;
      }
    }
    ```

2. Buat symlink dan update konfigurasi Nginx:
    ```bash
    sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
    sudo nano /etc/nginx/nginx.conf
    ```

    Update konfigurasi Nginx:
    ```nginx
    user www-data;
    worker_processes auto;
    error_log /var/log/nginx/error.log warn;
    pid /var/run/nginx.pid;
    worker_rlimit_nofile 100000;

    events {
        worker_connections 4096;
    }

    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        log_format main '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" '
                        '"$http_x_forwarded_for"';

        access_log /var/log/nginx/access.log main;

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;

        proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m;
        proxy_temp_path /var/cache/nginx/temp;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
    }
    ```

3. Restart Nginx:
    ```bash
    sudo systemctl restart nginx
    ```

## Pengujian Endpoint
### Pengujian dengan Rest Client
- **Get All History**
  ![History](link_gambar_di_sini)

- **Create a New Text**
  ![New Text](link_gambar_di_sini)

### Pengujian dari Frontend
![Frontend](link_gambar_di_sini)

## Hasil Pengujian Load
### RPS Maksimum (60 detik)
RPS maksimum yang dicapai selama stress testing adalah ~900-1000 RPS.

![RPS](link_gambar_di_sini)

### Pengujian Peak Concurrency
- **1000 Concurrency**
  ![1000 Concurrency](link_gambar_di_sini)

- **2000 Concurrency**
  ![2000 Concurrency](link_gambar_di_sini)

- **3000 Concurrency**
  ![3000 Concurrency](link_gambar_di_sini)

- **4000 Concurrency**
  ![4000 Concurrency](link_gambar_di_sini)

## Revisi, Kesimpulan, dan Saran
### Revisi
Setelah melakukan re-konfigurasi, poin-poin penting adalah:
- Pengujian dengan Locust menunjukkan bahwa desain cloud dapat menangani hingga 1015 RPS tanpa kegagalan, tetapi waktu respons meningkat setelah 600 RPS.
- Rata-rata waktu respons stabil sekitar 200 ms pada beban rendah hingga menengah.
- Kapasitas maksimum efektif adalah 600-700 RPS.

### Kesimpulan
- Desain layanan cloud stabil dan dapat menangani beban tinggi hingga 1015 RPS.
- Waktu respons meningkat signifikan setelah 600 RPS, menunjukkan adanya bottleneck.
- Kapasitas maksimum efektif adalah 600-700 RPS.

### Saran
- **Jaringan dan Koneksi:** Pastikan koneksi internet stabil dan bandwidth mencukupi.
- **Load Balancer Alternatif:** Pertimbangkan menggunakan VM sebagai load balancer alternatif.
- **Skalabilitas:** Pertimbangkan kemampuan untuk menskalakan VM sesuai kebutuhan.
- **Optimasi Performa Backend:** Optimalkan backend atau database untuk menangani beban lebih tinggi.

---

README ini memberikan gambaran komprehensif tentang setup, implementasi, dan hasil pengujian proyek ini. Silakan sesuaikan dan kembangkan sesuai kebutuhan.
