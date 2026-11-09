# Assessment: Setup Cluster PostgreSQL 16 HA

**Patroni + etcd + HAProxy + Keepalived — RHEL 9.6**

---

## 1. Informasi Cluster

| Item | Detail |
|------|--------|
| OS | Red Hat Enterprise Linux 9.6 |
| PostgreSQL | 16.x (PGDG repo) |
| Patroni | 4.x (Python virtualenv) |
| etcd | 3.5.x (RHEL AppStream / binary release) |
| HAProxy | 2.4.x (RHEL AppStream) |
| Keepalived | 2.2.x (RHEL AppStream) |
| Jumlah Node | 3 |

### Daftar IP

| Hostname | IP Address | Role |
|----------|-----------|------|
| testing-migrasi-hasura-1 | 10.100.13.94 | PostgreSQL + Patroni + etcd + HAProxy + Keepalived |
| testing-migrasi-hasura-2 | 10.100.13.95 | PostgreSQL + Patroni + etcd + HAProxy + Keepalived |
| testing-migrasi-hasura-3 | 10.100.13.96 | PostgreSQL + Patroni + etcd + HAProxy + Keepalived |
| VIP (Virtual IP) | 10.100.13.97 | Floating IP HAProxy via Keepalived |


### SSH

```bash
ssh hasura@10.100.13.94
ssh hasura@10.100.13.95
ssh hasura@10.100.13.96
```
pw: password

### Arsitektur

```
                        ┌──────────────┐
                        │   VIP (.97)  │
                        │  Keepalived  │
                        └──────┬───────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
       ┌──────┴──────┐ ┌──────┴──────┐ ┌───────┴─────┐
       │  HAProxy    │ │  HAProxy    │ │  HAProxy    │
       │  :5000/5001 │ │  :5000/5001 │ │  :5000/5001 │
       └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
              │                │                │
       ┌──────┴──────┐ ┌──────┴──────┐ ┌───────┴─────┐
       │  Patroni    │ │  Patroni    │ │  Patroni    │
       │  PostgreSQL │ │  PostgreSQL │ │  PostgreSQL  │
       │  .94        │ │  .95        │ │  .96        │
       └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
              │                │                │
       ┌──────┴──────┐ ┌──────┴──────┐ ┌───────┴─────┐
       │  etcd       │ │  etcd       │ │  etcd       │
       │  :2379/2380 │ │  :2379/2380 │ │  :2379/2380 │
       └─────────────┘ └─────────────┘ └─────────────┘
```

---

## 2. Prerequisite (Semua Node)

### 2.1 Set user & Hostname

```bash
sudo useradd hasura
sudo passwd hasura
usermod -aG wheel hasura
id hasura
exit
```

```bash
# Node 1
hostnamectl set-hostname db-hasura-1

# Node 2
hostnamectl set-hostname db-hasura-2
# Node 3
hostnamectl set-hostname db-hasura-3
```

### 2.2 Update /etc/hosts (Semua Node)

```bash
sudo tee -a /etc/hosts <<EOF
10.100.13.94    db-hasura-1
10.100.13.95    db-hasura-2
10.100.13.96    db-hasura-3
EOF
```

### 2.3 Disable SELinux (Opsional / Sesuai Kebijakan)

> **Catatan:** Jika SELinux harus tetap enforcing, pastikan policy untuk PostgreSQL, etcd, HAProxy, dan Keepalived sudah dikonfigurasi.

```bash
# opsi 1 Jika ingin disable (permissive)
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
setenforce 0

# opsi 2 izin sealinux untuk haproxy yang tepat
# Install policy tools jika belum ada
sudo dnf install -y policycoreutils-python-utils

# 1. Labeli port HAProxy agar diizinkan SELinux
sudo semanage port -a -t haproxy_port_t -p tcp 5000
sudo semanage port -a -t haproxy_port_t -p tcp 5001
sudo semanage port -a -t haproxy_port_t -p tcp 7000

# Verifikasi label sudah terdaftar
sudo semanage port -l | grep -E "5000|5001|7000"

# 2. Izinkan HAProxy connect ke port PostgreSQL (5432)
sudo setsebool -P haproxy_connect_any 1

# Verifikasi
getsebool haproxy_connect_any
# Output: haproxy_connect_any --> on

# sealinux untuk haproxy
sudo setsebool -P haproxy_connect_any 1
```

### 2.4 Firewall Rules

```bash
# PostgreSQL
sudo firewall-cmd --permanent --add-port=15432/tcp --add-port=5432/tcp --add-port=5433/tcp

# Patroni REST API
sudo firewall-cmd --permanent --add-port=8008/tcp

# etcd client & peer
sudo firewall-cmd --permanent --add-port=2379/tcp
sudo firewall-cmd --permanent --add-port=2380/tcp

# HAProxy (read-write & read-only)
sudo firewall-cmd --permanent --add-port=5000/tcp
sudo firewall-cmd --permanent --add-port=5001/tcp

# HAProxy Stats
sudo firewall-cmd --permanent --add-port=7000/tcp

# Keepalived (VRRP)
sudo firewall-cmd --permanent --add-protocol=vrrp

sudo firewall-cmd --reload
```

### 2.5 Disable dan Stop Default PostgreSQL Module (jika ada)

```bash
sudo dnf module disable postgresql -y
```

---

## 3. Instalasi PostgreSQL 16 (Semua Node)

### 3.1 Tambah PGDG Repository

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

### 3.2 Install PostgreSQL 16 Server

```bash
sudo dnf install -y postgresql16-server postgresql16-contrib
```

### Fix masalah url download di repo file PGDG:

```bash
sudo sed -i 's/\$releasever_major\.\$releasever_minor/9/g' /etc/yum.repos.d/pgdg-redhat-all.repo
sudo dnf clean all
sudo dnf makecache
sudo dnf install -y postgresql16-server postgresql16-contrib
```

> **Jangan jalankan `postgresql-16-setup --initdb`** — Patroni akan handle inisialisasi cluster.

### 3.3 Pastikan Binary Tersedia

```bash
/usr/pgsql-16/bin/psql --version
/usr/pgsql-16/bin/pg_isready --version
```

---

## 4. Instalasi & Konfigurasi etcd (Semua Node)

### 4.1 Install etcd

```bash
sudo dnf install -y etcd
```

> Jika paket tidak tersedia di AppStream, download binary dari [GitHub etcd releases](https://github.com/etcd-io/etcd/releases).

```bash
# install tar dan nano
sudo dnf install -y tar
sudo dnf install -y nano

# Set versi etcd
ETCD_VER=v3.7.3

# Download binary
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd.tar.gz

# Extract
tar xzvf /tmp/etcd.tar.gz -C /tmp/

# Copy binary ke /usr/local/bin
sudo cp /tmp/etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/
sudo cp /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
sudo cp /tmp/etcd-${ETCD_VER}-linux-amd64/etcdutl /usr/local/bin/

# Set permission
sudo chmod +x /usr/local/bin/etcd
sudo chmod +x /usr/local/bin/etcdctl
sudo chmod +x /usr/local/bin/etcdutl

# export path
echo 'export PATH=$PATH:/usr/local/bin' | sudo tee /etc/profile.d/local-bin.sh
source /etc/profile.d/local-bin.sh
etcd --version

# Verifikasi
etcd --version
etcdctl version
```

### 4.2 Konfigurasi etcd

buat direktori baru untuk etcd
```bash
# Buat direktori data dan config
sudo mkdir -p /data/etcd
sudo mkdir -p /etc/etcd
sudo chown -R postgres:postgres /data/etcd
sudo chmod 700 /data/etcd
```

buat service untuk etcd
```bash
sudo nano /etc/systemd/system/etcd.service
```

```bash
[Unit]
Description=etcd distributed key-value store
Documentation=https://etcd.io/docs
After=network.target

[Service]
Type=notify
User=postgres
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=5s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```


Edit `/etc/etcd/etcd.conf` di masing-masing node. Contoh untuk **Node 1** (`testing-migrasi-hasura-1`):

```bash
sudo mkdir -p /etc/etcd
sudo nano /etc/etcd/etcd.conf
```

```ini
ETCD_NAME="db-hasura-1"
ETCD_DATA_DIR="/data/etcd"

ETCD_LISTEN_PEER_URLS="http://10.100.13.94:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.100.13.94:2379,http://127.0.0.1:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.100.13.94:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.100.13.94:2379"

ETCD_INITIAL_CLUSTER="db-hasura-1=http://10.100.13.94:2380,db-hasura-2=http://10.100.13.95:2380,db-hasura-3=http://10.100.13.96:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="cluster123"
```

**Node 2** — ganti `ETCD_NAME="db-hasura-2"`, semua IP `10.100.13.94` jadi `10.100.13.95`.
**Node 3** — ganti `ETCD_NAME="db-hasura-3"`, semua IP `10.100.13.94` jadi `10.100.13.96`.

### 4.3 Enable & Start etcd

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd
```

### 4.4 Verifikasi Cluster etcd

```bash
etcdctl member list
etcdctl endpoint health --endpoints=http://10.100.13.94:2379,http://10.100.13.95:2379,http://10.100.13.96:2379
```

### 4.5 Jika cluster sudah terbantuk, ubah intial cluster jadi existing disemua node

config ini akan berlaku di restart berikutnya.
```bash
sudo sed -i 's/ETCD_INITIAL_CLUSTER_STATE="new"/ETCD_INITIAL_CLUSTER_STATE="existing"/' /etc/etcd/etcd.conf
```

---

## 5. Instalasi & Konfigurasi Patroni (Semua Node)
 
> Semua command dijalankan sebagai user biasa dengan `sudo` (bukan login root).
> Patroni dijalankan sebagai user `postgres`, sama dengan user etcd di setup ini.
 
### 5.1 Install Python & Dependency
 
```bash
sudo dnf install -y python3 python3-pip
python3 --version
```
 
RHEL 9.6 bawaan Python 3.9, sudah cukup untuk Patroni 4.x. `gcc`, `python3-devel`, dan `libpq-devel` tidak diperlukan karena driver PostgreSQL dipasang dalam bentuk binary wheel.
 
### 5.2 Buat Virtualenv & Install Patroni
 
```bash
sudo python3 -m venv /opt/patroni/venv
sudo /opt/patroni/venv/bin/pip install --upgrade pip
sudo /opt/patroni/venv/bin/pip install "patroni[etcd3,psycopg3]==4.1.5"
```
 
Catatan:
 
- Pakai extra **`etcd3`**, bukan `etcd`. Extra `etcd` memakai API v2 yang sudah tidak tersedia di etcd 3.7.
- Tidak perlu `source activate`, langsung panggil pip dari dalam venv supaya aman dipakai bersama `sudo`.
Verifikasi dan buat symlink supaya `patronictl` bisa dipanggil tanpa full path:
 
```bash
/opt/patroni/venv/bin/patroni --version
sudo ln -sf /opt/patroni/venv/bin/patronictl /usr/bin/patronictl
sudo ln -sf /opt/patroni/venv/bin/patroni /usr/bin/patroni
```
 
### 5.3 Buat Directory
 
```bash
sudo mkdir -p /etc/patroni
sudo mkdir -p /data/postgresql/16/main
sudo mkdir -p /data/log/postgresql
sudo mkdir -p /data/log/patroni
sudo chown -R postgres:postgres /data/postgresql
sudo chown -R postgres:postgres /data/log/postgresql
sudo chown -R postgres:postgres /data/log/patroni
sudo chmod 700 /data/postgresql/16/main
 
# Buat file pgpass
sudo touch /etc/patroni/16-main.pgpass
sudo chown postgres:postgres /etc/patroni/16-main.pgpass
sudo chmod 600 /etc/patroni/16-main.pgpass

# kasih akses ke user postgres untuk file patroni
sudo chown -R postgres:postgres /etc/patroni
sudo chmod 755 /etc/patroni
sudo chmod 644 /etc/patroni/16-main.yml

# Verifikasi
ls -la /etc/ | grep patroni
# Output: drwxr-xr-x  2 postgres postgres ... patroni
```
 
| Path | Owner | Permission | Fungsi |
|---|---|---|---|
| `/etc/patroni` | `root` | `755` | Config Patroni |
| `/etc/patroni/16-main.pgpass` | `postgres` | `600` | Ditulis ulang Patroni setiap start |
| `/data/postgresql/16/main` | `postgres` | `700` | Data directory PostgreSQL |
| `/data/log/postgresql` | `postgres` | `755` | Log PostgreSQL |
| `/data/log/patroni` | `postgres` | `755` | Log Patroni |
 
Catatan:
 
- `chmod 700` di data directory wajib. PostgreSQL menolak start kalau permission data directory lebih longgar dari `700`/`750`.
- `/data` sendiri harus bisa ditembus user `postgres` (`755`, owner root sudah cukup).
- Log Patroni ditulis ke `/data/log/patroni/patroni.log`, dirotasi otomatis oleh Patroni (maks 10 file x 25 MB).
Pastikan data directory masih kosong dan service bawaan RPM tidak aktif, karena inisialisasi dilakukan oleh Patroni:
 
```bash
ls -ld /data /data/postgresql/16/main
sudo ls -la /data/postgresql/16/main/
sudo systemctl disable --now postgresql-16 2>/dev/null
```
 
Kalau `/data/postgresql/16/main/` sudah ada isinya (pernah `initdb`), kosongkan dulu di **semua node**:
 
```bash
sudo rm -rf /data/postgresql/16/main/*
```
 
### 5.4 Konfigurasi Patroni
 
Buat file `/etc/patroni/16-main.yml`. Contoh untuk **Node 1**:
 
```bash
sudo nano /etc/patroni/16-main.yml
```
 
```yaml
scope: pg-cluster
namespace: /service/
name: db-hasura-1
 
log:
  level: INFO
  dir: /data/log/patroni
  file_num: 10
  file_size: 26214400
 
restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.100.13.94:8008
 
etcd3:
  hosts:
    - 10.100.13.94:2379
    - 10.100.13.95:2379
    - 10.100.13.96:2379
 
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 67108864   # 64 MB
    check_timeline: true
    postgresql:
      use_pg_rewind: true
      remove_data_directory_on_rewind_failure: false
      remove_data_directory_on_diverged_timelines: true
      use_slots: true
      parameters:
        max_connections: 150
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 512MB
        max_wal_senders: 20
        max_replication_slots: 20
        wal_log_hints: "on"
        track_commit_timestamp: "off"
 
  initdb:
    - encoding: UTF8
    - data-checksums
 
  pg_hba:
    - local   all          all                          peer
    - local   replication  all                          peer
    - host    all          all         127.0.0.1/32     scram-sha-256
    - host    all          all         ::1/128          scram-sha-256
    - host    all          all         10.100.13.0/24   scram-sha-256
    - host    replication  all         10.100.13.0/24   scram-sha-256
 
postgresql:
  listen: "*:15432"
  connect_address: 10.100.13.94:15432
  use_unix_socket: true
  data_dir: /data/postgresql/16/main
  bin_dir: /usr/pgsql-16/bin
  pgpass: /etc/patroni/16-main.pgpass
 
  authentication:
    superuser:
      username: postgres
      password: "3dmP@ssw0rd"
    replication:
      username: replicator
      password: "rep-pass"
    # rewind:
    #   username: postgres
    #   password: "<GANTI_PASSWORD_POSTGRES>"
 
  parameters:
    unix_socket_directories: "/var/run/postgresql"
    logging_collector: "on"
    log_directory: "/data/log/postgresql"
    log_filename: "postgresql-%Y-%m-%d.log"
    log_rotation_age: "1d"
    log_rotation_size: "100MB"
 
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```
 
Yang diganti per node hanya 3 baris: `name`, `restapi.connect_address`, dan `postgresql.connect_address`.
 
| | Node 1 | Node 2 | Node 3 |
|---|---|---|---|
| `name` | `db-hasura-1` | `db-hasura-2` | `db-hasura-3` |
| `connect_address` (restapi) | `10.100.13.94:8008` | `10.100.13.95:8008` | `10.100.13.96:8008` |
| `connect_address` (postgresql) | `10.100.13.94:5432` | `10.100.13.95:5432` | `10.100.13.96:5432` |
 
Catatan:
 
- Section `etcd3:` (bukan `etcd:`), menyesuaikan extra yang di-install di 5.2.
- Blok `bootstrap.users` sudah dihapus karena tidak didukung lagi di Patroni 4.x. User `replicator` dibuat otomatis oleh Patroni dari `authentication.replication`. User aplikasi/admin dibuat manual setelah cluster jalan (lihat 5.9).
- Password `postgres` dan `replicator` harus sama di ketiga node.
### 5.5 Set Ownership
 
```bash
sudo chown postgres:postgres /etc/patroni/16-main.yml
sudo chmod 600 /etc/patroni/16-main.yml
```
 
File ini dibaca langsung oleh proses Patroni (user `postgres`), jadi owner-nya harus `postgres`, berbeda dengan `/etc/etcd/etcd.conf` yang dibaca systemd sebagai root.
 
Validasi syntax sebelum start:
 
```bash
sudo -u postgres /opt/patroni/venv/bin/patroni --validate-config /etc/patroni/16-main.yml
```
 
Kalau tidak ada output, config valid.
 
### 5.6 Buat Systemd Service
 
```bash
sudo tee /etc/systemd/system/patroni@.service <<'EOF'
[Unit]
Description=Patroni PostgreSQL HA - %i
After=syslog.target network.target etcd.service
 
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni/%i.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no
 
[Install]
WantedBy=multi-user.target
EOF
```
 
### 5.7 Buka Firewall
 
```bash
sudo firewall-cmd --permanent --add-port=5432/tcp --add-port=8008/tcp
sudo firewall-cmd --reload
```
 
### 5.8 Enable & Start Patroni
 
> **Jalankan Node 1 terlebih dahulu** (sebagai leader/initdb). Tunggu sampai Node 1 berstatus `Leader` / `running`, baru start Node 2 & 3.
 
```bash
sudo systemctl daemon-reload
sudo systemctl enable patroni@16-main

# Start di confluent-1 dulu (akan jadi primary)
sudo systemctl start patroni@16-main
sudo systemctl status patroni@16-main

# Tunggu 30 detik, lalu start di confluent-2 dan confluent-3
sudo systemctl start patroni@16-main
```
 
Kalau Patroni gagal start sebelum sempat menulis log (misalnya config error atau permission), cek lewat `sudo journalctl -u patroni@16-main -n 50 --no-pager`.
 
### 5.9 Verifikasi Patroni
 
```bash
patronictl -c /etc/patroni/16-main.yml list
```
 
Output yang diharapkan:
 
```
+ Cluster: pg-cluster ----+---------+-----------+----+-----------+
| Member      | Host         | Role    | State     | TL | Lag in MB |
+-------------+--------------+---------+-----------+----+-----------+
| db-hasura-1 | 10.100.13.94 | Leader  | running   |  1 |           |
| db-hasura-2 | 10.100.13.95 | Replica | streaming |  1 |         0 |
| db-hasura-3 | 10.100.13.96 | Replica | streaming |  1 |         0 |
+-------------+--------------+---------+-----------+----+-----------+
```
 
Setelah cluster jalan, buat user admin di node Leader:
 
```bash
sudo -u postgres psql -c "CREATE ROLE admin WITH LOGIN PASSWORD '<GANTI_PASSWORD_ADMIN>' CREATEROLE CREATEDB;"
```
 
### 5.10 Kalau Bootstrap Gagal
 
Kalau Node 1 gagal initdb dan perlu diulang dari awal, hapus data directory **dan** state cluster di etcd, karena Patroni menyimpan status "sudah di-initialize" di etcd:
 
```bash
sudo systemctl stop patroni          # di semua node
sudo rm -rf /data/postgresql/16/main/* # di semua node
patronictl -c /etc/patroni/patroni.yml remove pg-cluster   # sekali saja, ikuti konfirmasinya
```
 
Lalu start ulang dari Node 1.

---
## 6. Instalasi & Konfigurasi HAProxy (Semua Node)

### 6.1 Install HAProxy

```bash
sudo dnf install -y haproxy
```

### 6.2 Konfigurasi HAProxy

Edit `sudo nano /etc/haproxy/haproxy.cfg`:

```cfg
global
        log 127.0.0.1 local2
        chroot /var/lib/haproxy
        pidfile /var/run/haproxy.pid
        stats socket /var/lib/haproxy/stats mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        maxconn 2000
        nbthread 2

defaults
        log     global
        mode    tcp
        option  tcplog
        option  dontlognull
        option  tcpka
        retries 3
        timeout connect 10s
        timeout client  1h
        timeout server  1h
        timeout check   10s
        timeout queue   30s

listen stats
        mode http
        bind *:7000
        stats enable
        stats uri /
        stats refresh 5s

listen primary
        bind *:5432
        maxconn 300
        option httpchk GET /primary
        http-check expect status 200
        default-server inter 2s fall 2 rise 3 on-marked-down shutdown-sessions
        server db-hasura-1 10.100.13.94:15432 maxconn 140 check port 8008
        server db-hasura-2 10.100.13.95:15432 maxconn 140 check port 8008
        server db-hasura-3 10.100.13.96:15432 maxconn 140 check port 8008

listen standbys
        bind *:5433
        maxconn 300
        balance leastconn
        option httpchk GET /replica
        http-check expect status 200
        default-server inter 2s fall 2 rise 3 on-marked-down shutdown-sessions
        server db-hasura-1 10.100.13.94:15432 maxconn 140 check port 8008
        server db-hasura-2 10.100.13.95:15432 maxconn 140 check port 8008
        server db-hasura-3 10.100.13.96:15432 maxconn 140 check port 8008
```

### 6.3 Enable & Start HAProxy

```bash
# Izinkan HAProxy bind ke non-local address
sudo setsebool -P haproxy_connect_any 1

sudo systemctl enable --now haproxy
```

---

## 7. Instalasi & Konfigurasi Keepalived (Semua Node)
 
Keepalived memegang VIP di node yang HAProxy-nya sehat. Aplikasi (Hasura, dll) cukup connect ke `VIP:5432` (read-write) dan `VIP:5433` (read-only). Keepalived **hanya** mengecek HAProxy; urusan PostgreSQL mana yang jadi primary sudah ditangani Patroni dan HAProxy.
 
### 7.1 Install Keepalived
 
```bash
sudo dnf install -y keepalived nmap-ncat
keepalived --version
nc --version
```
 
Buka protokol VRRP di firewall (tetap dibutuhkan walaupun pakai unicast):
 
```bash
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload
sudo firewall-cmd --list-protocols
```
 
Cek nama interface yang dipakai IP `10.100.13.x`, nanti dipakai di config (di RHEL biasanya `ens192` atau sejenisnya, bukan `eth0`):
 
```bash
ip -br a
```
 
### 7.2 Script Health Check
 
Buat file `/etc/keepalived/chk_haproxy.sh`:
 
```bash
sudo nano /etc/keepalived/chk_haproxy.sh
```
 
```bash
#!/bin/bash
PGPORT=5432
/usr/bin/nc -z -w 1 127.0.0.1 $PGPORT >/dev/null 2>&1
if [ $? -eq 0 ]; then
 exit 0
else
 exit 1
fi
```
 
```bash
sudo chown root:root /etc/keepalived/chk_haproxy.sh
sudo chmod 755 /etc/keepalived/chk_haproxy.sh
```
 
Tes manual, hasil `0` artinya sehat:
 
```bash
sudo /etc/keepalived/chk_haproxy.sh; echo $?
```
 
Tes juga terhadap port yang tertutup, hasilnya harus `1`:
 
```bash
/usr/bin/nc -z -w 1 127.0.0.1 5999; echo $?
```
 
Catatan:
 
- Script ini sama dengan yang dipakai di cluster existing. Bedanya hanya `nc` ditulis full path `/usr/bin/nc` (Keepalived menjalankan script dengan `PATH` minimal) dan ditambah `-w 1` supaya pengecekan selesai dalam 1 detik, lebih kecil dari `interval 2`.
- Cukup cek port 5432. Kalau proses HAProxy mati, port otomatis tertutup dan script `exit 1`.
- Owner harus `root` dan tidak boleh writable oleh user lain.

### 7.3 Pindahkan Script agar Tidak Terkendala SELinux

Keepalived berjalan di domain SELinux `keepalived_t` yang aksesnya terbatas. Script yang ditaruh di `/etc/keepalived/` ikut berjalan dengan batasan itu, sehingga saat SELinux enforcing:
 
- koneksi `nc` ke port 5432 (label `postgresql_port_t`) bisa ditolak
Akibatnya script selalu `exit 1` di semua node, priority semua node turun bersamaan, dan VIP **tidak pernah pindah** walaupun HAProxy benar-benar mati. Tes manual dengan `sudo` tetap terlihat sukses, jadi masalah ini tidak kelihatan sampai terjadi insiden.
 
Di RHEL, path `/usr/libexec/keepalived/` sudah punya label SELinux `keepalived_unconfined_script_exec_t`. Script di path ini dijalankan Keepalived tanpa batasan di atas.
 
Pindahkan script:
 
```bash
sudo mkdir -p /usr/libexec/keepalived
sudo mv /etc/keepalived/chk_haproxy.sh /usr/libexec/keepalived/chk_haproxy.sh
sudo chown root:root /usr/libexec/keepalived/chk_haproxy.sh
sudo chmod 755 /usr/libexec/keepalived/chk_haproxy.sh
sudo restorecon -Rv /usr/libexec/keepalived
```
 
Cek label-nya:
 
```bash
ls -Z /usr/libexec/keepalived/chk_haproxy.sh
```
 
Hasil yang benar mengandung `keepalived_unconfined_script_exec_t`, contoh:
 
```
system_u:object_r:keepalived_unconfined_script_exec_t:s0 /usr/libexec/keepalived/chk_haproxy.sh
```
 
Kalau label-nya masih `bin_t` atau `etc_t`, pasang aturan label lalu jalankan `restorecon` lagi:
 
```bash
sudo semanage fcontext -a -t keepalived_unconfined_script_exec_t "/usr/libexec/keepalived(/.*)?"
sudo restorecon -Rv /usr/libexec/keepalived
```
 
(`semanage` ada di paket `policycoreutils-python-utils`.)
 
Tes manual sekali lagi:
 
```bash
sudo /usr/libexec/keepalived/chk_haproxy.sh; echo $?
```
 
Pastikan di config Keepalived (7.4) path script sudah mengarah ke lokasi baru:
 
```
script "/usr/libexec/keepalived/chk_haproxy.sh"
```
 
Lakukan langkah ini di ketiga node. Verifikasi bahwa script benar-benar berjalan dari Keepalived ada di 7.6.

### 7.4 Konfigurasi Keepalived
 
Backup config bawaan, lalu buat baru:
 
```bash
sudo mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.orig
sudo nano /etc/keepalived/keepalived.conf
```
 
**Node 1 (MASTER):**
 
```conf
global_defs {
 router_id db-hasura-1
 enable_script_security
 script_user root
}
vrrp_script chk_haproxy {
 script "/usr/libexec/keepalived/chk_haproxy.sh"
 interval 2
 weight -50
 fall 2
 rise 1
}
vrrp_instance VI_1 {
 state BACKUP
 interface ens33
 virtual_router_id 51
 priority 130
 advert_int 1
 unicast_src_ip 10.100.13.94
 unicast_peer {
  10.100.13.95
  10.100.13.96
 }
 virtual_ipaddress {
  10.100.13.97/32 dev ens33
 }
 track_script {
  chk_haproxy
 }
}
```
 
Yang diganti per node:
 
| | Node 1 | Node 2 | Node 3 |
|---|---|---|---|
| `router_id` | `db-hasura-1` | `db-hasura-2` | `db-hasura-3` |
| `state` | `MASTER` | `BACKUP` | `BACKUP` |
| `priority` | `100` | `90` | `80` |
| `unicast_src_ip` | `10.100.13.94` | `10.100.13.95` | `10.100.13.96` |
| `unicast_peer` | `.95`, `.96` | `.94`, `.96` | `.94`, `.95` |
 
Yang **harus sama** di ketiga node: `virtual_router_id`, `auth_pass`, `virtual_ipaddress`, dan isi `vrrp_script`.
 
Catatan:
 
- **`weight -50`**: saat script gagal, priority turun 20. Node 1 jadi 100 − 50 = 50, di bawah node 2 (90), sehingga VIP pindah. Jangan pakai weight positif kecil seperti `weight 2`, karena priority node yang HAProxy-nya mati tetap paling tinggi dan VIP tidak pindah. Nilai weight harus lebih besar dari selisih priority antar node.
- **`unicast_peer`**: dipakai karena multicast VRRP sering diblokir switch/firewall di network internal. Dengan unicast, ketiga node saling kirim langsung ke IP masing-masing.
- **`virtual_router_id 51`**: harus unik di segmen `10.100.13.0/24`. Konfirmasi ke tim network kalau ada cluster Keepalived lain di VLAN yang sama.
- **`auth_pass`**: maksimal 8 karakter, lebih dari itu dipotong.
- Ganti `10.100.13.97` dan `<NAMA_INTERFACE>` setelah VIP di-approve.
Validasi config:
 
```bash
sudo keepalived -t -f /etc/keepalived/keepalived.conf
```
 
### 7.5 Enable & Start Keepalived
 
Start node 1 dulu, lalu node 2 dan node 3:
 
```bash
sudo systemctl enable --now keepalived
sudo systemctl status keepalived
sudo journalctl -u keepalived -n 30 --no-pager
```
 
Di log node 1 harus ada `Entering MASTER STATE`, di node 2 dan 3 `Entering BACKUP STATE`.
 
### 7.6 Verifikasi VIP
 
```bash
ip -br a | grep 10.100.13.97
```
 
VIP hanya boleh muncul di **satu** node (node 1). Kalau muncul di lebih dari satu node, berarti paket VRRP antar node tidak sampai (cek firewall `vrrp` dan `unicast_peer`).
 
Tes koneksi lewat VIP:
 
```bash
psql -h 10.100.13.97 -p 5432 -U postgres -c "select inet_server_addr(), pg_is_in_recovery();"
```
 
Port 5432 harus ke leader Patroni dengan `f`, port 5433 ke replica dengan `t`.
 
### 7.7 Tes Perpindahan VIP
 
```bash
# di node pemegang VIP (node 1)
sudo systemctl stop haproxy
 
# dalam beberapa detik VIP harus pindah ke node 2
ip -br a | grep 10.100.13.97          # jalankan di node 2
 
# kembalikan
sudo systemctl start haproxy            # di node 1
```
 
Karena priority node 1 paling tinggi, VIP kembali ke node 1 setelah HAProxy-nya hidup lagi. Kalau tidak ingin VIP balik otomatis (supaya tidak terjadi dua kali perpindahan), set ketiga node `state BACKUP` dan tambahkan `nopreempt` di `vrrp_instance`.
 
---
 
## 8. Testing & Validasi
 
### 8.1 Koneksi via VIP
 
```bash
psql -h 10.100.13.97 -p 5432 -U postgres -c "SELECT pg_is_in_recovery();"
# Harus return 'f' (false) — terhubung ke primary
 
psql -h 10.100.13.97 -p 5433 -U postgres -c "SELECT pg_is_in_recovery();"
# Harus return 't' (true) — terhubung ke replica
```
 
### 8.2 Cek Status Patroni
 
```bash
/opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
 
### 8.3 Cek HAProxy Stats
 
Akses `http://<NODE_IP>:7000` di browser untuk melihat status backend.
 
### 8.4 Cek etcd Cluster
 
```bash
etcdctl member list
etcdctl endpoint status --endpoints=http://10.100.13.94:2379,http://10.100.13.95:2379,http://10.100.13.96:2379 -w table
```
 
---
 
## 9. Failover Test
 
### 9.1 Switchover (Manual / Planned)
 
```bash
/opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml switchover --leader testing-migrasi-hasura-1 --candidate testing-migrasi-hasura-2 --force
```
 
### 9.2 Failover (Simulasi Down)
 
```bash
# Di node leader, stop Patroni
systemctl stop patroni
 
# Verifikasi leader berpindah
/opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
 
### 9.3 Keepalived Failover
 
```bash
# Stop keepalived di node MASTER
systemctl stop keepalived
 
# Cek VIP berpindah ke node BACKUP
# (jalankan di node BACKUP)
ip addr show eth0 | grep 10.100.13.97
```
 
---
 
## 10. Urutan Start & Stop Service
 
### Start (Urutan)
 
```
1. etcd        → systemctl start etcd
2. Patroni     → systemctl start patroni
3. HAProxy     → systemctl start haproxy
4. Keepalived  → systemctl start keepalived
```
 
### Stop (Urutan Terbalik)
 
```
1. Keepalived  → systemctl stop keepalived
2. HAProxy     → systemctl stop haproxy
3. Patroni     → systemctl stop patroni
4. etcd        → systemctl stop etcd
```
 
---
 
## 11. Catatan Tambahan
 
- **Password:** Semua placeholder `<GANTI_PASSWORD_*>` harus diganti dengan credential yang sesuai sebelum deployment.
- **VIP:** Ganti `10.100.13.97` setelah IP virtual di-approve.
- **Network Interface:** Pastikan nama interface di Keepalived (`eth0`) sesuai dengan interface yang dipakai di masing-masing node.
- **SELinux:** Jika enforcing, perlu buat policy khusus agar Patroni (virtualenv), etcd, dan Keepalived berjalan tanpa kendala.
- **Offline Install:** Jika environment tidak punya akses internet, semua RPM dan Python wheel harus di-download terlebih dahulu dan ditransfer ke node via media offline.
- **Backup:** Pertimbangkan setup `pgBackRest` atau `pg_basebackup` cron sebagai strategi backup setelah cluster berjalan.

---

## 11. Buat db untuk metadata hasura

untuk db metadata
```bash
psql -U postgres -h 10.100.13.97 -p 5432

CREATE ROLE hasura_core WITH LOGIN PASSWORD 'Pwd3dmhasura';

CREATE DATABASE hasura_chm
  OWNER hasura_core
  ENCODING 'UTF8'
  TEMPLATE template0;

CREATE DATABASE hasura_chm_schema
  OWNER hasura_core
  ENCODING 'UTF8'
  TEMPLATE template0;

REVOKE ALL ON DATABASE hasura_chm FROM PUBLIC;
REVOKE ALL ON DATABASE hasura_chm_schema FROM PUBLIC;
```

untuk db source
```bash
CREATE ROLE hasura_source WITH LOGIN PASSWORD 'password';

CREATE DATABASE hasura_source
  OWNER hasura_source
  ENCODING 'UTF8'
  TEMPLATE template0;

```