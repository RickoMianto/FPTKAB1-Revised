# Final Project

## Teknologi Komputasi Awan

### Kelas B

#### Kelompok B1

Nama | NRP
---- | ----
Daffa Rajendra Priyatama | 5027231009
Muhammad Faqih Husain | 5027231023
Ricko Mianto Jaya Saputra | 5027231031
Nicholas Arya Krisnugroho Rerangin | 5027231058
Benjamin Khawarismi Habibi | 5027231078

## Daftar Isi
- [Final Project Overview](#final-project-overview)
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

## Final Project Overview
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
Setelah melakukan analisis secara menyeluruh dan mempertimbangkan aspek harga dan spesifikasi, kami memutuskan untuk mengadopsi Digital Ocean sebagai platform cloud provider pilihan kami. Berikut adalah beberapa pertimbangan yang membuat kami memutuskan untuk menggunakan Digital Ocean:
1. Efisiensi Biaya: Digital Ocean menghadirkan struktur harga yang lebih ekonomis dibandingkan dengan kompetitor seperti Azure, memungkinkan pengoptimalan pengeluaran infrastruktur cloud tanpa mengorbankan performa.
2. Performa Tinggi: Spesifikasi hardware yang ditawarkan Digital Ocean tergolong tangguh dan mampu memenuhi kebutuhan komputasi yang kompleks. Hal ini memungkinkan kami untuk menjalankan aplikasi dan beban kerja dengan lancar tanpa hambatan.
3. Optimalisasi Performa Device: Berbeda dengan VM yang dapat membebani perangkat, Digital Ocean dirancang dengan mempertimbangkan efisiensi penggunaan sumber daya, sehingga meminimalisir dampak negatif terhadap performa device.

* Berikut adalah rancangan arsitektur yang telah kami buat untuk final project kami
![fptka2 (4)](https://github.com/RickoMianto/FPTKAB1-Revised/assets/150517828/a0c49c4a-ae0d-4bfb-9f25-88186fe36941)

* Berikut adalah tabel harga dan spesifikasi VM yang kami gunakan
![fptka (1)](https://github.com/RickoMianto/FPTKAB1-Revised/assets/150517828/9d35bdf7-c4cd-4df2-994c-80fd0b5d7de9)

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
  ![Test End Point](https://github.com/RickoMianto/FPTKAB1-Revised/assets/149749135/7aeaf3bb-e6fa-4038-a070-390adab4d483)

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
  ![WhatsApp Image 2024-06-29 at 17 07 41_a551a50c](https://github.com/RickoMianto/FPTKAB1-Revised/assets/88548292/0f3acdd7-68e6-43f2-a80e-f45348f1f8d0)

- **2000 Concurrency**
  ![WhatsApp Image 2024-06-29 at 17 11 06_b712b553](https://github.com/RickoMianto/FPTKAB1-Revised/assets/88548292/26ea91b8-00c4-4eac-8317-1ef089f19604)

- **3000 Concurrency**
  ![WhatsApp Image 2024-06-29 at 17 16 10_95a14e26](https://github.com/RickoMianto/FPTKAB1-Revised/assets/88548292/fca47923-29d2-4f20-ad94-9adc5be5f5da)

- **4000 Concurrency**
  ![WhatsApp Image 2024-06-29 at 17 20 00_990094d8](https://github.com/RickoMianto/FPTKAB1-Revised/assets/88548292/c0e7466d-5c6f-4660-98d2-b310c4f74134)

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
