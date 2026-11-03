# Self-Host Web App di Proxmox + Docker + Portainer

Panduan dari nol untuk menjalankan **web app apa pun** (Laravel, PHP native, CodeIgniter, Node.js, Python, frontend statis) beserta **database apa pun** (MySQL/MariaDB, PostgreSQL, SQLite, MongoDB) di server lokal berbasis **Proxmox → LXC → Docker → Portainer**.

Ditulis dari pengalaman deploy aplikasi client sungguhan, termasuk semua jebakan yang ditemui di lapangan, jadi kamu nggak perlu trial-and-error lagi.

> Target panduan ini adalah **jaringan lokal** (akses lewat `http://IP-SERVER:PORT`). Untuk akses dari internet (domain, HTTPS, Cloudflare Tunnel) perlu langkah tambahan yang tidak dibahas di sini.

---

## Daftar Isi

- [Gambaran Arsitektur](#gambaran-arsitektur)
- [Prasyarat](#prasyarat)
- [Aturan Main](#aturan-main)
- [Variabel yang Dipakai](#variabel-yang-dipakai)
- [Langkah 0 — Masuk ke Terminal Server](#langkah-0--masuk-ke-terminal-server)
- [Langkah 1 — Kenali Aplikasinya](#langkah-1--kenali-aplikasinya)
- [Langkah 2 — Pindahkan Source Code ke Server](#langkah-2--pindahkan-source-code-ke-server)
- [Langkah 3 — Siapkan Database](#langkah-3--siapkan-database)
- [Langkah 4 — Pilih Resep Runtime & Buat Stack](#langkah-4--pilih-resep-runtime--buat-stack)
- [Langkah 5 — Import / Restore Database](#langkah-5--import--restore-database)
- [Langkah 6 — Finishing per Framework](#langkah-6--finishing-per-framework)
- [Langkah 7 — Tes & Akun Login](#langkah-7--tes--akun-login)
- [Langkah 8 — Bersih-bersih](#langkah-8--bersih-bersih)
- [Troubleshooting](#troubleshooting)
- [Pelajaran dari Lapangan](#pelajaran-dari-lapangan)
- [Checklist Singkat](#checklist-singkat)

---

## Gambaran Arsitektur

```
Proxmox host (web UI :8006)
└── LXC "docker" (IP-SERVER)
    └── Docker Engine + Portainer (:9443)
        ├── Stack: database bersama (opsional, mis. MySQL + phpMyAdmin)
        │     └── network: <db_network>
        └── Stack: <APP>
              ├── <APP>-app  ← source code di-bind dari /opt/<APP>/src
              └── <APP>-db   ← (opsional) database khusus aplikasi ini
```

Prinsipnya:

- **Satu aplikasi = satu stack** di Portainer, satu folder di `/opt/<APP>`, satu port.
- **Source code ada di host** (`/opt/<APP>/src`) lalu di-*bind mount* ke container. Kode bisa diedit tanpa build ulang image.
- **Database bisa menumpang** ke server database yang sudah ada (buat database + user terpisah), atau **punya container sendiri** di dalam stack.

---

## Prasyarat

- Server Proxmox dengan LXC/VM yang sudah terpasang **Docker** dan **Portainer**.
- Akses ke **web UI Proxmox** (untuk console) dan **Portainer**.
- Source code aplikasi: folder di PC, file ZIP, atau repo Git.
- (Opsional) Dump database: `.sql`, `.dump`, atau file SQLite.

---

## Aturan Main

1. **Jangan ubah apa pun milik orang lain di server**: password root, stack lain, network lain. Buat user/database/stack baru yang terpisah.
2. **Jangan klik `Prune`** di halaman Images. Prune menghapus *semua* image yang berstatus *Unused*, termasuk milik aplikasi lain.
3. **Jangan `Join/Leave network`** di container milik orang lain.
4. Kalau kerja lewat remote desktop (UltraViewer/AnyDesk/RDP): **copy-paste perintah, jangan diketik**, dan paste **satu perintah per sekali paste**. Lihat [Pelajaran dari Lapangan](#pelajaran-dari-lapangan).
5. Kalau sebuah langkah butuh password, **buat password acak lewat terminal lalu copy-paste**, jangan diketik manual.

---

## Variabel yang Dipakai

Ganti nilai berikut di seluruh panduan:

| Placeholder | Contoh | Keterangan |
|---|---|---|
| `IP-SERVER` | `192.168.1.240` | IP LXC/VM tempat Docker berjalan |
| `<APP>` | `inventaris` | Nama aplikasi: huruf kecil, tanpa spasi |
| `<PORT>` | `8083` | Port di server yang **belum dipakai** |
| `<CTID>` | `103` | ID container LXC di Proxmox |

Setiap kali membuka sesi terminal baru, set dulu variabelnya:

```bash
APP=inventaris; PORT=8083
```

Cek port yang sudah terpakai:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

---

## Langkah 0 — Masuk ke Terminal Server

Cara paling andal, **tanpa SSH dan tanpa password**:

1. Buka web UI Proxmox → klik **node** (mis. `pve`) → **>_ Shell**.
2. Masuk ke LXC Docker:
   ```bash
   pct enter <CTID>
   ```
3. Prompt berubah menjadi `root@<hostname>`. Semua perintah selanjutnya dijalankan di sini.

Cara alternatif: klik LXC-nya langsung → **Console**, atau SSH kalau memang sudah punya akses.

> **Kenapa bukan SSH?** Di LXC buatan *community-scripts*, root sering tidak punya password, dan Debian **melarang root login SSH pakai password** (`PermitRootLogin prohibit-password`). Daripada mengubah konfigurasi server orang, lebih baik lewat console Proxmox.

---

## Langkah 1 — Kenali Aplikasinya

Lihat isi folder project-nya. Ciri-ciri tiap jenis aplikasi:

| Yang ditemukan | Jenis aplikasi | Resep di Langkah 4 |
|---|---|---|
| `artisan`, `composer.json` berisi `laravel/framework` | Laravel | **A** |
| `spark`, `app/Config/`, `public/index.php` | CodeIgniter 4 | **A** |
| `application/`, `system/`, `index.php` di root | CodeIgniter 3 | **B** |
| File `.php` biasa, `koneksi.php` / `config.php` | PHP native | **B** |
| `package.json` dengan `express` / `fastify` / `next start` | Node.js (server) | **C** |
| `requirements.txt` / `pyproject.toml`, `manage.py` / `app.py` | Python (Django/Flask/FastAPI) | **D** |
| `package.json` dengan `vite` / `react-scripts` / `vue`, tanpa server | Frontend statis (SPA) | **E** |

Lalu tentukan **database**-nya:

| Petunjuk | Database |
|---|---|
| Laravel `.env`: `DB_CONNECTION=mysql`; PHP: `mysqli_connect`, `PDO('mysql:...')`; Node: `mysql2`; Python: `pymysql`, `mysqlclient` | MySQL / MariaDB |
| Node: `pg`; Python: `psycopg2`, `asyncpg`; env `PGHOST`, `DATABASE_URL=postgres://` | PostgreSQL |
| Node: `mongoose`, `mongodb` | MongoDB |
| File `*.sqlite`, `*.db`; `DB_CONNECTION=sqlite` | SQLite (cuma file, tanpa server) |

Juga cari:

- **Versi runtime**: `"php": "^8.2"` di `composer.json`, `"engines"` di `package.json`, `python_requires`.
- **Asset frontend**: apakah sudah ada `public/build` atau `dist/`. Kalau belum ada, perlu build dulu.
- **File konfigurasi**: `.env`, `.env.example`, `config.php`. Catat nama-nama variabelnya.
- **Dump database**: file `.sql` / `.dump`, sering berada di folder `database/` atau `db/`.
- **README**: instruksi instalasi, migrasi, dan akun default.

Perintah cepat (di server, setelah Langkah 2):

```bash
cd /opt/$APP/src
ls -a
cat composer.json package.json requirements.txt 2>/dev/null | head -80
cat .env.example 2>/dev/null
find . -maxdepth 3 \( -name '*.sql' -o -name '*.dump' -o -name '*.sqlite' \) -not -path './node_modules/*' -not -path './vendor/*'
```

---

## Langkah 2 — Pindahkan Source Code ke Server

Semua opsi berakhir di struktur yang sama:

```
/opt/<APP>/
├── src/          ← source code aplikasi
└── *.sql         ← (opsional) dump database
```

### Opsi A — Dari repo Git (paling bersih)

```bash
mkdir -p /opt/$APP && cd /opt/$APP
git clone https://github.com/USER/REPO.git src
```

Repo private: pakai *Personal Access Token*, atau clone di PC lalu pakai Opsi B.
Kalau `git` belum terpasang: `apt install -y git`.

### Opsi B — Upload lewat browser dengan filebrowser (tanpa SSH, drag-and-drop)

**1. Di PC Windows**, bungkus folder project. Windows 10/11 sudah punya `tar` bawaan:

```bat
cd "C:\path\ke\folder-induk"
tar -czf app.tar.gz NAMA_FOLDER_PROJECT
```

Tambahkan `--exclude=node_modules` sebelum nama folder kalau `node_modules` ikut dan ukurannya besar.

> Kenapa di-tar? Project bisa berisi ribuan file kecil. Satu file besar jauh lebih cepat dan lebih kecil kemungkinan gagal di tengah upload.

**2. Di server**, nyalakan file manager web sementara:

```bash
mkdir -p /opt/$APP && chown 1000:1000 /opt/$APP
docker run -d --name upload -p 8090:80 -v /opt/$APP:/srv filebrowser/filebrowser --noauth
```

- `chown 1000:1000` diperlukan karena filebrowser versi baru berjalan sebagai uid 1000. Tanpa ini, upload ditolak dengan **403 Forbidden**.
- `--noauth` menghindari masalah login (password acak yang sering salah ketik lewat remote). **Hanya untuk sementara.**

**3. Di browser PC**, buka `http://IP-SERVER:8090`, lalu drag `app.tar.gz` dan file dump database. Tunggu sampai 100%.

**4. Kembali ke server:**

```bash
docker rm -f upload                       # WAJIB: tanpa login, siapa pun bisa mengakses folder ini
cd /opt/$APP
tar -tzf app.tar.gz > /dev/null && echo ARSIP UTUH
tar --warning=no-unknown-keyword -xzf app.tar.gz && rm app.tar.gz
mv NAMA_FOLDER_PROJECT src
chown root:root /opt/$APP
ls /opt/$APP /opt/$APP/src
```

> Warning `Ignoring unknown extended header keyword 'SCHILY.fflags'` itu **aman**. Itu atribut file Windows yang tidak dikenal `tar` Linux. Opsi `--warning=no-unknown-keyword` menyembunyikannya.

### Opsi C — SCP (kalau SSH memang bisa dipakai)

```bat
scp -r NAMA_FOLDER_PROJECT USER@IP-SERVER:/opt/<APP>/
```

Format `scp` adalah `USER@IP`. `USER` adalah **nama user Linux**, bukan hostname mesin.

---

## Langkah 3 — Siapkan Database

Pilih salah satu.

### 3A. Menumpang di MySQL/MariaDB yang sudah ada

Cocok kalau server sudah punya MySQL dan aplikasinya memakai MySQL. Kita buat **database + user khusus** supaya aplikasi lain tetap aman.

**Cari container MySQL dan network-nya:**

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}'
docker inspect NAMA_CONTAINER_MYSQL -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'
```

Atau lewat Portainer: **Containers → (container MySQL) → Connected networks**. Catat nama network-nya, misalnya `mysql-stack_db_network`.

> Kalau MySQL-nya bagian dari stack, nama network = `<nama-stack>_<nama-network-di-yaml>`, atau `<nama-stack>_default` kalau YAML-nya tidak mendefinisikan network.

**Buat database + user (satu baris, password acak otomatis):**

```bash
DB_PASS=$(tr -dc a-hjkmnp-z2-9 </dev/urandom 2>/dev/null | head -c 20); echo "CREATE DATABASE IF NOT EXISTS $APP CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci; CREATE USER IF NOT EXISTS '$APP'@'%' IDENTIFIED BY '$DB_PASS'; ALTER USER '$APP'@'%' IDENTIFIED BY '$DB_PASS'; GRANT ALL PRIVILEGES ON $APP.* TO '$APP'@'%'; FLUSH PRIVILEGES;" | docker exec -i NAMA_CONTAINER_MYSQL sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' && echo DB OK
echo $DB_PASS
```

- Perintah ini memakai `MYSQL_ROOT_PASSWORD` yang sudah tersimpan **di dalam container**, jadi kamu tidak perlu tahu password root-nya.
- Untuk MariaDB image `linuxserver/mariadb`, variabelnya juga `MYSQL_ROOT_PASSWORD`. Untuk image `mariadb` resmi, bisa `MARIADB_ROOT_PASSWORD`. Ganti `mysql` dengan `mariadb` kalau client `mysql` tidak ada.
- Karakter password sengaja dibatasi (tanpa `0/o`, `1/l/i`) supaya tidak salah baca.
- Perintah ini aman diulang. `ALTER USER` memastikan password selalu sinkron.
- **Jangan tutup terminal**: `$DB_PASS` dipakai lagi di langkah berikutnya.

Nilai untuk konfigurasi aplikasi:

```
host     = NAMA_CONTAINER_MYSQL   (bukan IP, bukan localhost)
port     = 3306                    (port internal, bukan port yang di-publish)
database = <APP>
user     = <APP>
password = $DB_PASS
```

### 3B. Database baru di dalam stack aplikasi

Cocok kalau databasenya beda jenis (mis. PostgreSQL padahal server cuma punya MySQL), atau kalau aplikasinya mau dibuat terisolasi. Service database ditulis langsung di YAML stack (lihat Langkah 4). Cukup siapkan password:

```bash
tr -dc a-hjkmnp-z2-9 </dev/urandom 2>/dev/null | head -c 20; echo
```

Copy hasilnya, lalu tempel di YAML. **Semua tempat yang memakai password ini harus identik.**

Potongan service database untuk ditempel di YAML stack:

```yaml
  # PostgreSQL
  db:
    image: postgres:17
    container_name: <APP>-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: PASSWORD_DI_SINI
      POSTGRES_DB: <NAMA_DB>
    volumes:
      - <APP>_dbdata:/var/lib/postgresql/data
```

```yaml
  # MySQL
  db:
    image: mysql:8.0
    container_name: <APP>-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: PASSWORD_DI_SINI
      MYSQL_DATABASE: <NAMA_DB>
      MYSQL_USER: <APP>
      MYSQL_PASSWORD: PASSWORD_DI_SINI
    volumes:
      - <APP>_dbdata:/var/lib/mysql
```

```yaml
  # MongoDB
  db:
    image: mongo:7
    container_name: <APP>-db
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: PASSWORD_DI_SINI
    volumes:
      - <APP>_dbdata:/data/db
```

Jangan lupa deklarasi volume di bagian bawah YAML:

```yaml
volumes:
  <APP>_dbdata:
```

> **Jangan publish port database** (`ports:`) kecuali memang perlu diakses dari luar. Aplikasi di stack yang sama bisa mengaksesnya lewat nama service (`db`).

### 3C. SQLite

Tidak perlu server database. Pastikan file `.sqlite` berada di dalam `/opt/<APP>/src` (ikut ter-bind mount) dan folder tempat file itu **bisa ditulis** oleh user yang menjalankan aplikasi (lihat Troubleshooting).

---

## Langkah 4 — Pilih Resep Runtime & Buat Stack

Semua stack dibuat lewat **Portainer → Stacks → + Add stack → Web editor**, lalu **Deploy the stack**.

Ketentuan umum:

- Ganti `<APP>`, `<PORT>`, dan nama network di YAML secara manual. Variabel shell **tidak berlaku** di editor Portainer.
- Kalau memakai database bersama (3A), tambahkan network eksternal:
  ```yaml
  networks:
    dbnet:
      external: true
      name: NAMA_NETWORK_DB
  ```
  lalu di service aplikasi tambahkan `networks: [dbnet]`.
- Kalau memakai database di dalam stack (3B), tidak perlu bagian `networks`. Service di stack yang sama otomatis saling terhubung.
- Aplikasi **harus listen di `0.0.0.0`**, bukan `localhost`/`127.0.0.1`. Kalau tidak, aplikasi tidak bisa diakses dari luar container.

---

### Resep A — Laravel / CodeIgniter 4 (PHP, docroot `public/`)

**1. Build image sekali** (bisa dipakai ulang untuk semua aplikasi PHP dengan versi yang sama).

Portainer → **Images** → tombol **+ Build a new image** (di atas tabel Images, **bukan** form *Pull image*):

- **Name**: `php82-apache:local` (sesuaikan versi)
- **Build method**: Web editor

```dockerfile
FROM php:8.2-apache

RUN apt-get update && apt-get install -y --no-install-recommends \
      git unzip libzip-dev libpng-dev libjpeg62-turbo-dev libfreetype6-dev libicu-dev libpq-dev \
 && docker-php-ext-configure gd --with-freetype --with-jpeg \
 && docker-php-ext-install -j"$(nproc)" pdo_mysql mysqli pdo_pgsql pgsql zip gd bcmath intl exif \
 && a2enmod rewrite \
 && rm -rf /var/lib/apt/lists/*

ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
 && printf 'upload_max_filesize=20M\npost_max_size=20M\nmemory_limit=512M\n' > /usr/local/etc/php/conf.d/app.ini

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
WORKDIR /var/www/html
```

- Ganti `8.2` sesuai `"php"` di `composer.json`.
- Klik **Build the image**, lalu tunggu sampai tab Output selesai.
- Verifikasi: di halaman detail image harus ada `APACHE_DOCUMENT_ROOT=/var/www/html/public`.
- Label **Unused** itu normal sampai ada container yang memakainya. **Jangan Prune.**

**2. Stack:**

```yaml
services:
  app:
    image: php82-apache:local
    container_name: <APP>-app
    restart: unless-stopped
    ports:
      - "<PORT>:80"
    volumes:
      - /opt/<APP>/src:/var/www/html
    networks:
      - dbnet

networks:
  dbnet:
    external: true
    name: NAMA_NETWORK_DB
```

**3. Izin folder** (Laravel):

```bash
chown -R 33:33 /opt/$APP/src/storage /opt/$APP/src/bootstrap/cache
```

(CodeIgniter 4: `chown -R 33:33 /opt/$APP/src/writable`)

---

### Resep B — PHP Native / CodeIgniter 3 (docroot di root project)

Pakai image yang sama dengan Resep A. Docroot cukup di-override lewat environment, karena Apache membaca `${APACHE_DOCUMENT_ROOT}` saat runtime:

```yaml
services:
  app:
    image: php82-apache:local
    container_name: <APP>-app
    restart: unless-stopped
    environment:
      APACHE_DOCUMENT_ROOT: /var/www/html
    ports:
      - "<PORT>:80"
    volumes:
      - /opt/<APP>/src:/var/www/html
    networks:
      - dbnet

networks:
  dbnet:
    external: true
    name: NAMA_NETWORK_DB
```

Lalu edit file koneksi (`koneksi.php`, `config.php`, `application/config/database.php`):

```php
$host = 'NAMA_CONTAINER_MYSQL';  // bukan 'localhost'
$user = '<APP>';
$pass = 'PASSWORD';
$db   = '<APP>';
```

Edit lewat `nano /opt/$APP/src/koneksi.php`, atau upload versi yang sudah diedit lewat filebrowser.

Folder upload yang perlu ditulis aplikasi: `chown -R 33:33 /opt/$APP/src/uploads` (sesuaikan nama foldernya).

---

### Resep C — Node.js (Express, Fastify, NestJS, Next.js, dll.)

Tidak perlu build image. Pakai image `node` resmi. `npm ci` otomatis jalan saat pertama kali container menyala.

```yaml
services:
  app:
    image: node:20
    container_name: <APP>-app
    restart: unless-stopped
    working_dir: /app
    command: sh -c "[ -d node_modules ] || npm ci --omit=dev; npm start"
    environment:
      NODE_ENV: production
      HOST: 0.0.0.0
      HOSTNAME: 0.0.0.0
      PORT: 3000
      # variabel database sesuai .env.example aplikasi, contoh PostgreSQL:
      PGHOST: db
      PGPORT: 5432
      PGUSER: postgres
      PGPASSWORD: PASSWORD_DI_SINI
      PGDATABASE: <NAMA_DB>
    ports:
      - "<PORT>:3000"
    volumes:
      - /opt/<APP>/src:/app
    depends_on:
      - db

  # tempel service db dari Langkah 3B di sini

volumes:
  <APP>_dbdata:
```

Catatan:

- Sesuaikan `PORT` internal (`3000`) dan nama variabel dengan `.env.example`. Variabel di `environment:` **mengalahkan** file `.env` (dotenv tidak menimpa variabel yang sudah ada).
- `HOSTNAME`/`HOST` = `0.0.0.0` penting. Banyak app memakai `app.listen(PORT, HOSTNAME)`, dan `localhost` membuat app tidak bisa diakses dari luar container.
- Pakai `node:20` (Debian), **bukan** `node:20-alpine`, kalau ada dependency native seperti `canvas`, `bcrypt`, atau `sharp`.
- Kalau tidak ada `package-lock.json`, ganti `npm ci` dengan `npm install`.
- Kalau app perlu build dulu (Next.js, NestJS, TypeScript):
  ```yaml
  command: sh -c "[ -d node_modules ] || npm ci; [ -d .next ] || [ -d dist ] || npm run build; npm start"
  ```
- Kalau `npm start` menjalankan `nodemon`, ganti dengan `node path/ke/server.js`.

---

### Resep D — Python (Django / Flask / FastAPI)

```yaml
services:
  app:
    image: python:3.12-slim
    container_name: <APP>-app
    restart: unless-stopped
    working_dir: /app
    command: sh -c "pip install --no-cache-dir -r requirements.txt gunicorn && gunicorn -b 0.0.0.0:8000 MODUL_WSGI"
    environment:
      DATABASE_URL: postgres://postgres:PASSWORD_DI_SINI@db:5432/<NAMA_DB>
    ports:
      - "<PORT>:8000"
    volumes:
      - /opt/<APP>/src:/app
    depends_on:
      - db
```

Ganti `MODUL_WSGI`:

| Framework | Nilai |
|---|---|
| Django | `namaproject.wsgi` (folder yang berisi `wsgi.py`) |
| Flask | `app:app` (`file.py` : variabel Flask) |
| FastAPI | ganti gunicorn dengan `uvicorn main:app --host 0.0.0.0 --port 8000`, dan tambahkan `uvicorn` ke `pip install` |

Django setelah stack jalan:

```bash
docker exec $APP-app python manage.py migrate
docker exec $APP-app python manage.py collectstatic --noinput
docker exec -it $APP-app python manage.py createsuperuser
```

Kalau ada library yang butuh compiler (`psycopg2`, `mysqlclient`), pakai `python:3.12` (bukan `-slim`) atau ganti ke `psycopg2-binary`.

---

### Resep E — Frontend Statis (React / Vue / Vite / Angular)

**1. Build sekali di server:**

```bash
docker run --rm -v /opt/$APP/src:/app -w /app node:20 sh -c "npm ci && npm run build"
ls /opt/$APP/src/dist /opt/$APP/src/build 2>/dev/null
```

**2. Konfigurasi nginx untuk SPA** (supaya refresh di halaman `/dashboard` tidak 404):

```bash
mkdir -p /opt/$APP/nginx
printf 'server {\n  listen 80;\n  root /usr/share/nginx/html;\n  location / { try_files $uri $uri/ /index.html; }\n}\n' > /opt/$APP/nginx/default.conf
```

**3. Stack:**

```yaml
services:
  web:
    image: nginx:alpine
    container_name: <APP>-web
    restart: unless-stopped
    ports:
      - "<PORT>:80"
    volumes:
      - /opt/<APP>/src/dist:/usr/share/nginx/html:ro
      - /opt/<APP>/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
```

Ganti `dist` dengan `build` untuk Create React App. Kalau frontend memanggil API, URL API biasanya ditanam saat build (`VITE_API_URL`, `REACT_APP_API_URL`). Set variabel itu **sebelum** `npm run build`.

---

## Langkah 5 — Import / Restore Database

### Cek isi dump dulu

```bash
head -c 5 /opt/$APP/FILE_DUMP; echo
grep -n -i -E 'create database|^use ' /opt/$APP/FILE_DUMP | head
```

| Hasil | Artinya |
|---|---|
| `PGDMP` | Dump PostgreSQL format *custom*. Pakai `pg_restore` |
| Teks (`--`, `SET`, `CREATE TABLE`) | Dump SQL biasa. Pakai `mysql` / `psql` |
| Ada `CREATE DATABASE` / `USE nama_db` | Dump akan membuat/memakai database **dengan nama lain**. Hapus baris itu dulu, atau sesuaikan nama database |

Hapus baris `CREATE DATABASE` / `USE` (simpan cadangan dulu):

```bash
cp FILE_DUMP FILE_DUMP.bak
sed -i -E '/^(CREATE DATABASE|USE )/Id' FILE_DUMP
```

### MySQL / MariaDB

```bash
docker exec -i NAMA_CONTAINER_MYSQL sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" '"$APP" < /opt/$APP/FILE_DUMP.sql && echo IMPORT OK
```

### PostgreSQL

```bash
# format custom (PGDMP)
docker exec -i <APP>-db pg_restore -U postgres -d <NAMA_DB> --no-owner --no-privileges < /opt/<APP>/FILE_DUMP && echo RESTORE OK

# format SQL biasa
docker exec -i <APP>-db psql -U postgres -d <NAMA_DB> < /opt/<APP>/FILE_DUMP && echo RESTORE OK

# cek tabel
docker exec <APP>-db psql -U postgres -d <NAMA_DB> -c '\dt'
```

Kalau muncul `unsupported version`, dump dibuat dengan PostgreSQL yang lebih baru. Naikkan tag image (mis. `postgres:18`), lalu ulangi.

### MongoDB

```bash
docker cp /opt/$APP/dump <APP>-db:/dump
docker exec <APP>-db mongorestore -u root -p PASSWORD --authenticationDatabase admin /dump
```

> Saat import, terminal **diam tanpa output**. Itu normal. **Jangan Ctrl+C**: import yang terputus meninggalkan tabel setengah jadi. Dump ratusan MB bisa makan 5–15 menit.

> **Script `setup-db` bawaan aplikasi sering tidak jalan di Docker** (biasanya memanggil `pg_restore`/`mysql` yang tidak ada di image aplikasi, atau mengasumsikan `localhost`). Restore langsung di container database seperti di atas, karena hasilnya sama.

---

## Langkah 6 — Finishing per Framework

### Laravel

**Atur `.env`** (satu baris, jalankan di sesi yang sama dengan pembuatan `$DB_PASS`):

```bash
cd /opt/$APP/src && [ -f .env ] || cp .env.example .env; cp .env .env.bak && sed -i -e 's/^APP_ENV=.*/APP_ENV=local/' -e 's/^APP_DEBUG=.*/APP_DEBUG=true/' -e "s#^APP_URL=.*#APP_URL=http://IP-SERVER:$PORT#" -e 's/^#\? *DB_CONNECTION=.*/DB_CONNECTION=mysql/' -e 's/^#\? *DB_HOST=.*/DB_HOST=NAMA_CONTAINER_MYSQL/' -e 's/^#\? *DB_PORT=.*/DB_PORT=3306/' -e "s/^#\? *DB_DATABASE=.*/DB_DATABASE=$APP/" -e "s/^#\? *DB_USERNAME=.*/DB_USERNAME=$APP/" -e "s/^#\? *DB_PASSWORD=.*/DB_PASSWORD=$DB_PASS/" .env && rm -f bootstrap/cache/*.php && grep -E '^(APP_URL|DB_)' .env | grep -v PASSWORD
```

Cek password masuk (hasil harus 20):

```bash
grep '^DB_PASSWORD=' .env | cut -d= -f2 | tr -d '\n' | wc -c
```

`rm bootstrap/cache/*.php` menghapus cache konfigurasi dari Windows yang bisa berisi path `C:\...`.

**Install & setup:**

```bash
docker exec $APP-app composer install --no-dev --optimize-autoloader
docker exec $APP-app php artisan storage:link --force
docker exec $APP-app php artisan optimize:clear
```

- `APP_KEY` kosong? Jalankan `docker exec $APP-app php artisan key:generate`. Kalau data lama ikut diimport, **pakai APP_KEY lama** dari `.env` asli.
- Database dari dump: **jangan** `migrate`.
- Database kosong: `docker exec $APP-app php artisan migrate --seed`
- Tidak ada `public/build` tapi ada `vite.config.js`:
  ```bash
  docker run --rm -v /opt/$APP/src:/app -w /app node:20 sh -c "npm ci && npm run build"
  ```

### CodeIgniter 4

Edit `.env` (`cp env .env` kalau belum ada): set `app.baseURL`, `database.default.hostname`, `database.default.database`, `database.default.username`, `database.default.password`. Lalu:

```bash
docker exec $APP-app composer install --no-dev
```

### Node.js

```bash
docker logs --tail 40 $APP-app
```

Ulangi sampai `npm ci` selesai dan muncul tanda server jalan (*listening on 3000*). Setelah restore database: `docker restart $APP-app`.

> Pakai `docker logs --tail`, bukan `docker logs -f`. Lewat remote desktop, Ctrl+C untuk keluar dari `-f` sering tidak terkirim.

---

## Langkah 7 — Tes & Akun Login

Buka `http://IP-SERVER:<PORT>`.

> Email yang muncul di dropdown kolom login itu **autofill browser**, bukan isi database.

### Lihat akun yang ada

```bash
# MySQL
docker exec NAMA_CONTAINER_MYSQL sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" '"$APP"' -e "SELECT id,name,email FROM users LIMIT 10"'

# PostgreSQL
docker exec <APP>-db psql -U postgres -d <NAMA_DB> -c '\dt'
docker exec <APP>-db psql -U postgres -d <NAMA_DB> -c 'SELECT id, email FROM users LIMIT 10'
```

Nama tabel/kolom bisa berbeda (`user`, `pengguna`, `username`). Cek dengan `\dt` dan `\d nama_tabel` (PostgreSQL) atau `SHOW TABLES; DESCRIBE nama_tabel;` (MySQL).

Coba dulu password **`password`**. Banyak seeder memakai password default itu.

### Reset password admin

**Laravel:**

```bash
docker exec $APP-app php artisan tinker --execute="\App\Models\User::where('email','EMAIL')->update(['password'=>bcrypt('admin12345')]); echo 'RESET OK';"
```

**Node.js (bcrypt):**

```bash
HASH=$(docker exec $APP-app node -e "console.log(require('bcrypt').hashSync('admin12345',10))")
docker exec <APP>-db psql -U postgres -d <NAMA_DB> -c "UPDATE users SET password='$HASH' WHERE email='EMAIL'"
```

(Kalau app memakai `bcryptjs`, ganti `require('bcrypt')` dengan `require('bcryptjs')`.)

**PHP native** (cek dulu hash yang dipakai di kode login: `password_verify` = bcrypt, `md5(` = MD5):

```bash
HASH=$(docker exec $APP-app php -r "echo password_hash('admin12345', PASSWORD_DEFAULT);")
# atau untuk MD5:  HASH=$(echo -n admin12345 | md5sum | cut -d' ' -f1)
```

lalu `UPDATE` seperti di atas.

**Django:**

```bash
docker exec -it $APP-app python manage.py changepassword USERNAME
```

### phpMyAdmin

- Pakai phpMyAdmin yang sudah ada di server. Login dengan user `<APP>` dan password database-nya. Yang terlihat hanya database `<APP>`.
- Kalau phpMyAdmin yang ada auto-login sebagai user lain (`PMA_USER` diset di stack), nyalakan phpMyAdmin sementara yang menampilkan form login:
  ```bash
  docker run -d --name pma-tmp --network NAMA_NETWORK_DB -e PMA_HOST=NAMA_CONTAINER_MYSQL -p 8099:80 phpmyadmin:5.2-apache
  # selesai → docker rm -f pma-tmp
  ```

---

## Langkah 8 — Bersih-bersih

```bash
docker ps -a --format '{{.Names}}' | grep -x upload && docker rm -f upload   # filebrowser
docker rmi filebrowser/filebrowser:latest                                     # image-nya (opsional)
docker ps -a --format '{{.Names}}' | grep -x pma-tmp && docker rm -f pma-tmp
```

- Kalau sempat mengubah konfigurasi SSH (mis. `PermitRootLogin yes`), kembalikan:
  ```bash
  rm -f /etc/ssh/sshd_config.d/00-root.conf && systemctl restart ssh
  ```
- Simpan image dasar (`composer:2`, `php*-apache:local`, `node:20`) karena dipakai ulang untuk aplikasi berikutnya.
- Untuk produksi: `APP_DEBUG=false` (Laravel), `NODE_ENV=production`, `DEBUG=False` (Django).

### Mengubah stack nanti

- **Re-pull image and redeploy**: **matikan** kalau stack memakai image lokal (`*:local`). Image itu tidak ada di Docker Hub, jadi pull akan gagal.
- **Service dihapus dari YAML tapi container masih jalan?** Itu *orphan*. Nyalakan **Prune services** sebelum *Update the stack*, atau hapus container-nya manual. Jangan pakai `docker rm -f` untuk container yang **masih** ada di YAML, karena container itu akan dibuat ulang setiap update.

---

## Troubleshooting

| Gejala | Penyebab | Solusi |
|---|---|---|
| `Permission denied (publickey,password)` terus | Root dilarang SSH pakai password, atau password berubah karena diketik lewat remote | Jangan pakai SSH. Pakai console Proxmox + filebrowser |
| Upload filebrowser `403 Forbidden` | Folder tujuan milik root, filebrowser jalan sebagai uid 1000 | `chown 1000:1000 /opt/$APP` lalu refresh |
| Filebrowser `Wrong credentials` | Password acak salah ketik/baca | Jalankan ulang dengan `--noauth` |
| Prompt `>` dan tidak bisa keluar | Perintah terpotong saat paste (kutip/heredoc tidak tertutup) | Ketik `EOF` + Enter, Ctrl+D, atau tutup dan buka lagi tab console |
| `docker run requires at least 1 argument` | Ujung perintah (nama image) terpotong | Paste ulang, pastikan baris lengkap |
| Ctrl+O / F3 di nano tidak jalan, `Unknown sequence` | Shift nyangkut di remote desktop | Buka `osk` di PC remote dan lepas Shift, reconnect remote, atau hindari nano (buat file lewat Portainer) |
| `SQLSTATE[HY000] [2002]` / `ECONNREFUSED` | Host DB salah atau beda network | Host = nama container/service, cek `networks` di stack |
| `Access denied for user` / `password authentication failed` | Password di config tidak sama dengan di DB | Ulangi pembuatan user (aman diulang), lalu update config di sesi yang sama |
| Laravel `Permission denied` di `storage/logs` | Izin folder | `chown -R 33:33 storage bootstrap/cache` |
| Halaman 500 / putih | Bermacam-macam | `docker logs --tail 50 $APP-app`, `storage/logs/laravel.log` |
| CSS/JS tidak tampil | Asset belum di-build atau `APP_URL` salah | Build asset (node), cek `APP_URL` |
| Gambar upload rusak (Laravel) | Symlink storage belum ada | `php artisan storage:link --force` |
| Halaman SPA 404 saat refresh | Nginx tidak fallback ke `index.html` | Pakai `try_files` di Resep E |
| App Node jalan tapi tidak bisa diakses | Listen di `localhost` | Set `HOST`/`HOSTNAME=0.0.0.0` |
| `npm ci` gagal: lock file out of sync | `package-lock.json` tidak sinkron | Pakai `npm install --omit=dev` |
| Install `canvas` / `sharp` gagal | Image alpine atau versi Node tidak cocok | Pakai `node:20` (bukan alpine), coba `node:18`, hapus `node_modules` lalu ulang |
| PostgreSQL restore `unsupported version` | Dump dari versi Postgres lebih baru | Naikkan tag image Postgres |
| Restore `role "xxx" does not exist` | Dump membawa owner lama | Tambah `--no-owner --no-privileges` |
| `tr: write error: Broken pipe` | `head` berhenti membaca duluan | Aman diabaikan |
| Image berlabel `Unused` | Belum ada container yang memakai | Normal. **Jangan Prune** |
| Stack *Deployment in progress* lama | Sedang mengunduh image besar | Tunggu. Cek dengan `docker images` |
| SQLite `readonly database` | Folder DB tidak bisa ditulis | `chown -R 33:33 folder_db` (PHP) atau sesuaikan uid container |

---

## Pelajaran dari Lapangan

1. **Remote desktop (UltraViewer/AnyDesk) bisa membuat tombol modifier nyangkut.** Gejalanya: `F` membuka Feedback Hub (karena Win+F), huruf jadi kapital/simbol, password selalu ditolak, nano tidak merespons. Solusi: buka **On-Screen Keyboard** (`osk`) di PC remote dan klik tombol yang menyala, atau putus-sambung remote.
2. **Password yang diketik lewat remote hampir pasti salah.** Hitung jumlah titik di kolom password. Kalau berbeda dari panjang password, berarti ada ketikan dobel. Selalu copy-paste.
3. **Paste perintah satu-satu.** Blok panjang sering terpotong di jalan.
4. **Perintah satu baris lebih aman daripada heredoc (`<<EOF`).** Kalau heredoc terpotong, shell terjebak di prompt `>`.
5. **Variabel shell hilang kalau terminal ditutup.** Jalankan pembuatan password dan penulisan config **di sesi yang sama**.
6. **Bedakan hostname dan username.** Di `root@docker`, `root` = user dan `docker` = nama mesin.
7. **`docker inspect` / Portainer lebih jujur daripada tebakan.** Nama network stack bisa `<stack>_default` atau `<stack>_<nama>`, jadi cek dulu.
8. **Setiap kali membuka detail image, cek versinya.** Server bisa punya banyak image mirip (mis. PHP 8.3 milik app lain vs PHP 8.2 milikmu).
9. **Aplikasi bawaan GitHub (`-main`) biasanya tanpa `.env`, `vendor/`, `node_modules/`.** Siapkan dari `.env.example`.
10. **Database bersama: satu aplikasi = satu database + satu user.** Aplikasi lain tetap aman, dan phpMyAdmin hanya menampilkan database milik user yang login.

---

## Checklist Singkat

```
[ ] Masuk terminal: Proxmox → node → Shell → pct enter <CTID>
[ ] Tentukan APP dan PORT kosong
[ ] Kenali jenis aplikasi + database + versi runtime
[ ] Source code → /opt/<APP>/src (git clone / filebrowser / scp)
[ ] Hapus container filebrowser
[ ] Database: user+db di server bersama, atau service db di stack
[ ] (PHP) Build image php*-apache:local sekali lewat Portainer
[ ] Stack di Portainer: image, port, bind mount, network/env
[ ] Import/restore dump (cek CREATE DATABASE/USE dan format PGDMP dulu)
[ ] Config app: host DB = nama container, port internal, listen 0.0.0.0
[ ] Install dependency (composer / npm / pip) + migrate/storage link bila perlu
[ ] Buka http://IP-SERVER:<PORT>, login (reset admin bila perlu)
[ ] Bersih-bersih: container sementara, setelan SSH, APP_DEBUG
```

---

## Lisensi

Bebas dipakai, dimodifikasi, dan dibagikan. Kalau membantu, kasih ⭐ di repo ini.
