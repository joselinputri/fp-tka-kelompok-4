# Implementasi Database — MongoDB (vm4-mongodb)
> **VM:** `vm4-mongodb` | IP Internal: `10.184.0.6` | IP External: `34.101.207.8`
> **Spesifikasi:** 2 vCPU, 2 GB RAM, $18/bulan

---

## 1. Koneksi ke VM MongoDB

Akses VM melalui GCP Console:

1. Buka [https://console.cloud.google.com](https://console.cloud.google.com)
2. Login dengan akun Google yang sudah di-invite ke project
3. Navigasi ke **Compute Engine → VM Instances**
4. Cari baris `vm4-mongodb`, klik tombol **SSH**

Atau via terminal laptop:

```bash
ssh joselinesilitonga@34.101.207.8
```

---

## 2. Instalasi MongoDB 7.0

### 2.1 Import GPG Key

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
```

### 2.2 Tambahkan Repository MongoDB

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

### 2.3 Install MongoDB

```bash
sudo apt-get update && sudo apt-get install -y mongodb-org
```

### 2.4 Jalankan dan Enable MongoDB

```bash
sudo systemctl start mongod && sudo systemctl enable mongod
```

### 2.5 Verifikasi Status

```bash
sudo systemctl status mongod
```

**Output yang diharapkan:**

![MongoDB Active Running](./result/mongodb_active.jpeg)

> ✅ Pastikan statusnya `Active: active (running)` sebelum lanjut.

Tekan `q` untuk keluar dari tampilan status.

---

## 3. Restore Database dari Dump

### 3.1 Siapkan File Dump (di laptop lokal)

```bash
cd ~/Downloads
unzip fp-tka-26-main.zip
cd fp-tka-26-main/Resources/DB
zip -r dump.zip dump/
```

### 3.2 Upload ke VM

Di browser SSH, klik tombol **UPLOAD FILE** (pojok kanan atas), lalu pilih file `dump.zip` yang baru dibuat.

### 3.3 Install Unzip di VM

```bash
sudo apt install unzip -y
```

### 3.4 Ekstrak File Dump

```bash
unzip dump.zip
```

### 3.5 Restore ke MongoDB

```bash
mongorestore --db orderdb dump/orderdb/
```

**Output yang diharapkan:**

![Hasil Mongorestore](./result/mongodb_restore.jpeg)

> ✅ Pastikan ada baris `12701 document(s) restored successfully. 0 document(s) failed to restore.`

Data yang ter-restore:
| Collection | Jumlah Dokumen |
|---|---|
| users | 505 |
| products | 96 |
| orders | 10.000 |
| audit_logs | 2.000 |
| sessions | 100 |

---

## 4. Pembuatan Index

Index diperlukan untuk mempercepat query, terutama pada endpoint `/admin/stats` yang memiliki aggregation pipeline berat.

```bash
mongosh orderdb --eval "
db.orders.createIndex({ user_id: 1, created_at: -1 });
db.orders.createIndex({ status: 1 });
db.orders.createIndex({ order_id: 1 }, { unique: true });
db.products.createIndex({ is_active: 1, category: 1 });
db.users.createIndex({ email: 1 }, { unique: true });
"
```

> ✅ MongoDB akan menampilkan nama index yang berhasil dibuat untuk setiap perintah.

---

## 5. Konfigurasi mongod.conf

### 5.1 Buka File Konfigurasi

```bash
sudo nano /etc/mongod.conf
```

### 5.2 Ubah Konfigurasi

Edit file sehingga menjadi seperti berikut:

```yaml
# mongod.conf

storage:
  dbPath: /var/lib/mongodb
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1        # Tuning cache = setengah RAM vm4 (2GB)

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 0.0.0.0          # Agar bisa diakses dari app server lain

processManagement:
  timeZoneInfo: /usr/share/zoneinfo
```

**Screenshot konfigurasi:**

![mongod.conf](./result/mongodb_conf.jpeg)

Simpan dengan: `Ctrl+X` → `Y` → `Enter`

### 5.3 Restart MongoDB

```bash
sudo systemctl restart mongod && sudo systemctl status mongod
```

---

## 6. Verifikasi Koneksi dan Data

### 6.1 Cek via URI Internal

```bash
mongosh mongodb://10.184.0.6:27017/orderdb --eval "db.stats()"
```

### 6.2 Tampilkan Statistik Database

```bash
mongosh orderdb --eval "db.stats()"
```

**Output yang diharapkan:**

![db.stats() output](./result/mongodb_dbstats.jpeg)

```json
{
  db: 'orderdb',
  collections: Long('5'),
  views: Long('0'),
  objects: Long('12701'),
  indexes: Long('10'),
  ok: 1
}
```

> ✅ Pastikan `collections: 5`, `objects: 12701`, `indexes: 10`, dan `ok: 1`

---

## 7. Ringkasan Konfigurasi MongoDB

| Parameter | Nilai |
|---|---|
| Versi | MongoDB 7.0 |
| Port | 27017 |
| Database | orderdb |
| bindIp | 0.0.0.0 |
| WiredTiger Cache | 1 GB |
| Total Index | 10 |
| Total Dokumen | 12.701 |

**MONGO_URI untuk App Server:**
```
mongodb://10.184.0.6:27017/orderdb
```

## 8. Deployment Backend (App Server)
Layanan backend dideploy menggunakan Gunicorn dengan 4 worker pada 3 instance terpisah untuk mendukung load balancing.<br>
**Konfigurasi Instance:**
- Web Server: Gunicorn (WSGI HTTP Server)
- Virtual Environment: Python 3.10 venv
- Dependency: Flask, PyMongo, Bcrypt, PyJWT, Gunicorn
- Port: 5000 (Internal)
<br>

### Daftar Instance Backend

| Instance | IP Address |
| :--- | :--- |
| App Server 1 (Redis) | `34.101.207.217:5000` |
| App Server 2 | `34.128.83.168:5000` |
| App Server 3 | `34.50.119.137:5000` |


  Lalu jalankan perintah 
```
  # Setup Environment
source venv/bin/activate
export MONGO_URI="mongodb://10.184.0.6:27017/orderdb"

# Menjalankan Gunicorn sebagai daemon
gunicorn -w 4 -b 0.0.0.0:5000 app:app --daemon
```

## 9. Implementasi Load Balancer & Frontend (vm2-nginx-frontend)
> **VM:** `vm2-nginx-frontend` | IP Internal: `10.184.0.2` | IP External: `34.101.72.188`
> **Spesifikasi:** 1 vCPU, 1 GB RAM, $6/bulan

### 9.1 Instalasi dan Tuning Nginx
Nginx diinstal di server `vm2` sebagai Reverse Proxy, Load Balancer, sekaligus Web Server statis untuk menyajikan file frontend.
```bash
# Update package list dan install Nginx
sudo apt update && sudo apt install nginx -y

# Mengatur kepemilikan folder agar bisa dimodifikasi oleh user aktif
sudo chown -R $USER:$USER /var/www/html
sudo chmod -R 755 /var/www/html
```

Tuning dilakukan pada `/etc/nginx/sites-available/default` untuk menangani lonjakan traffic:
- `worker_processes auto`: Menyesuaikan thread worker secara dinamis dengan core CPU.
- `gzip on`: Mengurangi ukuran transfer file text (HTML/CSS/JS) sehingga menghemat bandwidth dan meningkatkan kecepatan muat halaman.
- `keepalive_timeout 65`: Mempertahankan koneksi TCP client tetap terbuka demi efisiensi handshake.

### 9.2 Konfigurasi Load Balancing (Upstream)
Kami mengonfigurasi upstream backend dengan strategi **Least Connections (`least_conn`)** agar request diarahkan ke server dengan koneksi aktif paling sedikit. Hal ini mencegah overload pada salah satu server.

Kami juga mengaktifkan **Passive Health Check** menggunakan parameter `max_fails=3` dan `fail_timeout=10s` dikombinasikan dengan `proxy_next_upstream`. Jika salah satu backend mati, Nginx secara otomatis melewatkan server tersebut dan mendistribusikan request ke backend yang aktif secara real-time tanpa mengembalikan error 502/504 ke pengguna.

File `/etc/nginx/sites-available/default` yang dikonfigurasi:
```nginx
upstream backend {
    least_conn;
    server 10.184.0.3:5000 max_fails=3 fail_timeout=10s;
    server 10.184.0.4:5000 max_fails=3 fail_timeout=10s;
    server 10.184.0.5:5000 max_fails=3 fail_timeout=10s;
    keepalive 64;
}

server {
    listen 80 default_server;
    server_name _;
    root /var/www/html;
    index index.html;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Serve static frontend
    location / {
        try_files $uri $uri/ =404;
    }

    # API Rewrite & Proxy (mengatasi mismatch singular/plural dari frontend)
    location /order {
        if ($request_method = POST) { rewrite ^/order$ /orders break; }
        if ($request_method = GET) { rewrite ^/order/(.*)$ /orders/$1 break; }
        if ($request_method = PUT) { rewrite ^/order/(.*)$ /orders/$1/status break; }

        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    }

    location /orders {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    }

    location /auth/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /products {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /admin/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 9.3 Konfigurasi Frontend (CORS-free)
File `index.html` dan `styles.css` disalin dari `Resources/FE/` ke `/var/www/html/` di `vm2`.
Kami mengubah variabel `API_BASE` pada file `index.html` menjadi:
```javascript
const API_BASE = "";
```
Dengan merujuk ke path kosong (`""`), semua API call dari browser diarahkan secara otomatis ke IP Load Balancer (`http://34.101.72.188`) port 80. Nginx bertindak sebagai gerbang tunggal yang menyajikan file statis sekaligus melakukan reverse proxy ke backend server, sehingga menghindari isu Cross-Origin Resource Sharing (CORS) tanpa membutuhkan konfigurasi tambahan pada sisi backend.

#### Screenshot Pendukung:
- **Status Nginx Active:**
  ![Nginx Status](./result/nginx_status.png)
- **Tampilan Frontend:**
  ![Frontend Active](./result/frontend_dashboard.png)

## 10. Ansible Deployment

Pada bagian ini dilakukan proses otomatisasi deployment menggunakan Ansible.
Tujuan dari konfigurasi ini adalah menyiapkan seluruh infrastruktur aplikasi:

- Backend Flask server
- MongoDB database server
- Nginx load balancer
- Systemd service untuk menjalankan aplikasi

| Komponen | Deskripsi |
|---|---|
| **Backend Flask** | Server aplikasi yang menangani request API |
| **MongoDB** | Database server untuk menyimpan data order |
| **Nginx** | Load balancer yang mendistribusikan traffic ke backend |
| **Systemd** | Service manager untuk menjalankan aplikasi secara otomatis |


### 10.1 Preparation

Sebelum menjalankan Ansible, pastikan:

- Semua VM sudah aktif
- SSH antar server dapat digunakan
- Python3 tersedia pada target server

### 10.2 Ansible Inventory Configuration

### Setup SSH Key

Langkah ini memungkinkan **vm4 (control node)** terhubung ke seluruh app server tanpa password.

**Di vm4 (control node)** — salin public key:

```bash
cat ~/.ssh/id_rsa.pub
```

**Di app1, app2, app3 (target server)** — tambahkan public key ke authorized_keys:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys   # paste public key di sini
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Target server dan IP-nya:

| Server | IP |
|---|---|
| app1 | 10.184.0.3 |
| app2 | 10.184.0.4 |
| app3 | 10.184.0.5 |

**Test koneksi SSH dari vm4:**

```bash
ssh -i ~/.ssh/id_rsa maritzaadelia076@10.184.0.3
```

Output yang diharapkan:

```maritzaadelia076@vm3-appserver1-redis:~$```

![tes ssh](./result/tes-ssh.jpeg)

Check koneksi:

```bash
ansible all -i inventory.ini -m ping
```
 ![ansible ping](./result/ansible-ping.jpeg)
 
File:
`inventory.ini`

### File: `inventory.ini`

> **Fungsi:** Mendefinisikan seluruh server yang dikelola Ansible beserta kredensial dan konfigurasi koneksi SSH-nya. File ini menjadi referensi utama untuk semua perintah dan playbook Ansible.

Isi konfigurasi:
```bash
# Grup server frontend (Nginx load balancer)
[frontend_servers]
frontend1 ansible_host=10.184.0.2

# Grup server backend (Flask app)
[appservers]
app1 ansible_host=10.184.0.3
app2 ansible_host=10.184.0.4
app3 ansible_host=10.184.0.5

# Grup server database (MongoDB)
[mongodb]
mongo ansible_host=10.184.0.6

# Variabel global yang berlaku untuk semua host
[all:vars]
ansible_user=maritzaadelia076                                    # Username SSH di semua server
ansible_ssh_private_key_file=/home/maritzaadelia076/.ssh/id_rsa # Path ke private key
ansible_python_interpreter=/usr/bin/python3                      # Interpreter Python yang digunakan Ansible
ansible_become=true                                              # Aktifkan privilege escalation (sudo)
ansible_become_method=sudo                                       # Metode escalation menggunakan sudo
```

### 10.3 Backend Deployment
File:
`
setup_appserver.yml
`
### File: `setup_appserver.yml`

> **Fungsi:** Mengotomatisasi deployment aplikasi Flask ke semua app server. Playbook ini menginstal dependensi, menyalin source code, dan mendaftarkan aplikasi sebagai systemd service agar berjalan otomatis saat server restart.

```bash
---
- name: Setup Flask App Server
  hosts: appservers    # Dijalankan di semua server dalam grup [appservers]
  become: yes          # Jalankan sebagai root (sudo)

  tasks:

    # Pastikan package list up-to-date sebelum instalasi
    - name: Update packages
      apt:
        update_cache: yes

    # Install runtime Python dan Git untuk dependency management
    - name: Install python dependencies
      apt:
        name:
          - python3
          - python3-pip
          - git
        state: present

    # Buat direktori kerja aplikasi dengan permission yang tepat
    - name: Create app directory
      file:
        path: /opt/order-service
        state: directory
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'

    # Salin source code Flask dari control node ke semua app server
    - name: Copy Flask app
      copy:
        src: app.py
        dest: /opt/order-service/app.py
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0644'

    # Salin file daftar library Python yang dibutuhkan
    - name: Copy requirements
      copy:
        src: requirements.txt
        dest: /opt/order-service/requirements.txt
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0644'

    # Install semua library dari requirements.txt menggunakan pip3
    - name: Install python libraries
      pip:
        requirements: /opt/order-service/requirements.txt
        executable: pip3

    # Install Gunicorn sebagai WSGI server untuk menjalankan Flask di production
    - name: Install gunicorn
      pip:
        name:
          - gunicorn
        executable: pip3

    # Buat unit file systemd agar aplikasi berjalan otomatis sebagai service
    - name: Create systemd service
      copy:
        dest: /etc/systemd/system/order.service
        content: |
          [Unit]
          Description=Order Processing Flask Service
          After=network.target    # Service dijalankan setelah network siap

          [Service]
          User={{ ansible_user }}
          WorkingDirectory=/opt/order-service

          # Inject URI koneksi MongoDB sebagai environment variable
          Environment="MONGO_URI=mongodb://10.184.0.6:27017/orderdb"

          # Jalankan Flask via Gunicorn dengan 4 worker process, bind ke port 5000
          ExecStart=/usr/local/bin/gunicorn \
          -w 4 \
          --timeout 60 \
          -b 0.0.0.0:5000 \
          app:app

          Restart=always      # Restart otomatis jika service crash
          RestartSec=5        # Tunggu 5 detik sebelum restart

          [Install]
          WantedBy=multi-user.target

    # Reload systemd agar mengenali unit file baru
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    # Aktifkan service (auto-start on boot) dan restart untuk apply perubahan
    - name: Enable and start order service
      systemd:
        name: order
        enabled: yes
        state: restarted
```

### File: `setup_mongodb.yml`

> **Fungsi:** Menginstal dan menjalankan MongoDB pada server database, serta mendefinisikan index yang dibutuhkan aplikasi untuk performa query yang optimal.

```yaml
---
- name: Setup MongoDB Server
  hosts: mongodb    # Hanya dijalankan di server dalam grup [mongodb]
  become: yes

  tasks:

    # Pastikan package list up-to-date sebelum instalasi
    - name: Update apt
      apt:
        update_cache: yes

    # Install MongoDB dari repository Ubuntu
    - name: Install MongoDB
      apt:
        name: mongodb
        state: present

    # Pastikan MongoDB berjalan dan aktif saat server boot
    - name: Start MongoDB
      systemd:
        name: mongodb
        state: started
        enabled: yes

    # Catat rencana index ke file dokumentasi
    # Index ini perlu dibuat secara manual atau via script migrasi
    - name: Create index info file
      copy:
        dest: /tmp/indexes.txt
        content: |
          orders:
          order_id unique     # Unique index untuk mencegah duplikasi order
          created_at index    # Index untuk query berdasarkan tanggal
          status index        # Index untuk filter berdasarkan status order
```

### File: `setup_nginx.yml`

> **Fungsi:** Menginstal Nginx dan mengkonfigurasinya sebagai load balancer yang mendistribusikan traffic ke ketiga backend server menggunakan algoritma **Least Connections**.

```bash
---
- name: Setup Nginx Load Balancer
  hosts: frontend_servers    # Dijalankan di server dalam grup [frontend_servers]
  become: yes

  tasks:

    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Configure nginx
      copy:
        dest: /etc/nginx/sites-available/default
        content: |

          # Definisi upstream: kelompok backend server yang menerima traffic
          upstream backend {
              least_conn;    # Kirim request ke server dengan koneksi paling sedikit

              server 10.184.0.3:5000;    # app1
              server 10.184.0.4:5000;    # app2
              server 10.184.0.5:5000;    # app3
          }

          server {
              listen 80;    # Nginx mendengarkan koneksi di port 80 (HTTP)

              # Sajikan file statis (frontend) dari direktori default
              location / {
                  root /var/www/html;
                  index index.html;
              }

              # Teruskan semua request ke path /api/ ke backend Flask
              location /api/ {
                  proxy_pass http://backend;
              }
          }

    # Restart Nginx agar konfigurasi baru aktif
    - name: Restart nginx
      systemd:
        name: nginx
        state: restarted
```

## 10.4 Menjalankan Playbook

Sebelum deploy, lakukan syntax check untuk memastikan tidak ada kesalahan:

```bash
ansible-playbook -i inventory.ini setup_appserver.yml --syntax-check
ansible-playbook -i inventory.ini setup_mongodb.yml --syntax-check
ansible-playbook -i inventory.ini setup_nginx.yml --syntax-check
```

![ansible syntax check](./result/ansible-syntaxcheck.jpeg)

Setelah syntax check berhasil, jalankan semua playbook **sesuai urutan**:

```bash
# 1. Deploy database terlebih dahulu
ansible-playbook -i inventory.ini setup_mongodb.yml

# 2. Deploy backend
ansible-playbook -i inventory.ini setup_appserver.yml

# 3. Deploy load balancer
ansible-playbook -i inventory.ini setup_nginx.yml
```

![ansible run](./result/ansible.jpeg)

---

## 10.5 Testing Deployment

### Health Check via Load Balancer

```bash
curl http://34.101.72.188
curl http://34.101.72.188/health
```

![ansible curl](./result/ansible-curl.jpeg)

### Test Koneksi Langsung ke Backend (Gunicorn Port 5000)

![ansible gunicorn](./result/ansible-5000.jpeg)
