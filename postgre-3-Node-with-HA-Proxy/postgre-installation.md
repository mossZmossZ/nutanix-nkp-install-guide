# PostgreSQL 16 HA Cluster (Ubuntu 24.04 LTS)
## 3 PostgreSQL Nodes + Patroni + etcd + 1 HAProxy

---

# Architecture

| Host | Role |
|------|------|
| postgre-1 | PostgreSQL + Patroni + etcd |
| postgre-2 | PostgreSQL + Patroni + etcd |
| postgre-3 | PostgreSQL + Patroni + etcd |
| haproxy-1 | HAProxy |

Replace placeholders:

- `<IP1>`
- `<IP2>`
- `<IP3>`
- `<HAPROXY_IP>`
- `<FQDN>`

---

# 1. OS Preparation (All PostgreSQL Nodes)

## Update System

```
sudo apt update && sudo apt upgrade -y
```

---

## Disable Firewall (Lab Only)

Ubuntu uses UFW:

```
sudo systemctl disable ufw --now
```

Production: open ports:

- 2379 (etcd client)
- 2380 (etcd peer)
- 5432 (PostgreSQL)
- 8008 (Patroni REST)

---

## Configure /etc/hosts (All Nodes Including HAProxy)

```
<IP1> postgre-1 postgre-1.example.com
<IP2> postgre-2 postgre-2.example.com
<IP3> postgre-3 postgre-3.example.com
<HAPROXY_IP> haproxy-1 haproxy-1.example.com
```

---

# 2. Install PostgreSQL 16 (Official PGDG)

Install required tools:

```
sudo apt install -y curl gnupg2 lsb-release build-essential git wget
```

Add PostgreSQL repository:

```
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
sudo gpg --dearmor -o /usr/share/keyrings/postgresql.gpg

echo "deb [signed-by=/usr/share/keyrings/postgresql.gpg] \
http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | \
sudo tee /etc/apt/sources.list.d/pgdg.list
```

Install PostgreSQL:

```
sudo apt update
sudo apt install -y postgresql-16 postgresql-client-16
```

Install development tools for Patroni:

```
sudo apt install -y python3 python3-venv python3-pip python3-dev
```

Disable default PostgreSQL service (Patroni will manage it):

```
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

Do NOT run initdb manually.

---

## Install Locale (ALL NODES)

Verify:

```
locale -a | grep en_US
```

If missing:

```
sudo locale-gen en_US.UTF-8
sudo update-locale
```

---

# 3. Install etcd (ALL PostgreSQL Nodes)

## Download

```
cd /tmp
ETCD_RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest | grep tag_name | cut -d '"' -f 4)
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_RELEASE}/etcd-${ETCD_RELEASE}-linux-amd64.tar.gz
tar xvf etcd-${ETCD_RELEASE}-linux-amd64.tar.gz
cd etcd-${ETCD_RELEASE}-linux-amd64

sudo mv etcd etcdctl etcdutl /usr/local/bin/
sudo chown root:root /usr/local/bin/etcd*
sudo chmod 755 /usr/local/bin/etcd*
```

---

## Create etcd user

```
sudo useradd -r -s /usr/sbin/nologin etcd
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```

---

## Create `/etc/etcd.conf`

Example (ALL NODE):

```
ETCD_NAME=postgre-1.example.com
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_PEER_URLS=http://<IP1>:2380
ETCD_LISTEN_CLIENT_URLS=http://<IP1>:2379,http://127.0.0.1:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://<IP1>:2380
ETCD_ADVERTISE_CLIENT_URLS=http://<IP1>:2379
ETCD_INITIAL_CLUSTER=postgre-1.example.com=http://<IP1>:2380,postgre-2.example.com=http://<IP2>:2380,postgre-3.example.com=http://<IP3>:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
```

---

## Create systemd service

`/etc/systemd/system/etcd.service`

```
[Unit]
Description=etcd
After=network.target

[Service]
User=etcd
Type=simple
EnvironmentFile=/etc/etcd.conf
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## CLEAN START (If Recreating)

Run on each node before first start:

```
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd/*
sudo chown -R etcd:etcd /var/lib/etcd
```

---

## Start Order

### 1️⃣ Start pg1 etcd ONLY

```
sudo systemctl daemon-reload
sudo systemctl enable etcd --now
```

Verify:

```
ss -lntp | grep 2379
```

Then check:

```
etcdctl member list -w table
```

You should see pg1 only.

---

### 2️⃣ Start pg2 etcd

After pg1 is running:

```
sudo systemctl start etcd
```

Verify cluster.

---

### 3️⃣ Start pg3 etcd

```
sudo systemctl start etcd
```

Final verification:

```
etcdctl member list -w table
```

You must see 3 members in `started` state.

---

# 4. Install Patroni (All PostgreSQL Nodes)

```
sudo python3 -m venv /opt/patroni
sudo /opt/patroni/bin/pip install --upgrade pip
sudo /opt/patroni/bin/pip install patroni[etcd] psycopg2-binary
sudo chown -R postgres:postgres /opt/patroni
```

---

# 5. Create `/etc/patroni.yml`

Example for postgre-1:

```
scope: pg_cluster
name: postgre-1

restapi:
  listen: '<IP1>:8008'
  connect_address: '<IP1>:8008'

etcd3:
  hosts:
    - postgre-1.example.com:2379
    - postgre-2.example.com:2379
    - postgre-3.example.com:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true

  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

postgresql:
  listen: '<IP1>:5432'
  connect_address: '<IP1>:5432'
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin

  authentication:
    superuser:
      username: postgres
      password: StrongPostgres@123
    replication:
      username: replicator
      password: StrongRep@123

  parameters:
    wal_level: replica
    hot_standby: "on"
    max_wal_senders: 10
    max_replication_slots: 10
```

Set permissions:

```
sudo chown postgres:postgres /etc/patroni.yml
sudo chmod 640 /etc/patroni.yml
```

---

# 6. Create Patroni Service

`/etc/systemd/system/patroni.service`

```
[Unit]
Description=Patroni PostgreSQL HA
After=network.target

[Service]
User=postgres
Group=postgres
ExecStart=/opt/patroni/bin/patroni /etc/patroni.yml
Restart=always

[Install]
WantedBy=multi-user.target
```

---

# 7. Clean Bootstrap Procedure

On ALL nodes:

```
sudo systemctl stop patroni
sudo pkill -9 postgres
sudo rm -rf /var/lib/postgresql/16/main
sudo mkdir -p /var/lib/postgresql/16/main
sudo chown -R postgres:postgres /var/lib/postgresql
```

Clear etcd (run once only):

```
etcdctl del --prefix /service/pg_cluster
```

---

# 8. Correct Start Order

1. Ensure etcd running on ALL nodes.
2. Start Patroni on postgre-1 ONLY.

```
sudo systemctl start patroni
```

3. Wait until Leader is running.
4. Start Patroni on postgre-2.
5. Start Patroni on postgre-3.

Verify:

```
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni.yml list
```

Expected:

- 1 Leader
- 2 Replicas

---

Client connects only to:

```
<VIP_IP>:5000  (WRITE)
<VIP_IP>:5001  (READ)
```

---

# 1. Install HAProxy (Both HAProxy Nodes)

```bash
sudo apt update
sudo apt install -y haproxy
```

---

# 2. Configure HAProxy

Edit on BOTH haproxy-1 and haproxy-2:

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

---

## HAProxy Configuration

```
global
    daemon
    maxconn 5000

defaults
    mode tcp
    timeout connect 5s
    timeout client  30m
    timeout server  30m
    option tcplog

# WRITE traffic (Leader only)
listen postgres_write
    bind *:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2

    server pg1 <IP1>:5432 check port 8008
    server pg2 <IP2>:5432 check port 8008
    server pg3 <IP3>:5432 check port 8008

# READ traffic (Replicas)
listen postgres_read
    bind *:5001
    balance leastconn
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2

    server pg1 <IP1>:5432 check port 8008
    server pg2 <IP2>:5432 check port 8008
    server pg3 <IP3>:5432 check port 8008
```

Enable HAProxy:

```bash
sudo systemctl enable haproxy --now
```

---

# 3. Install Keepalived (Floating VIP)

Install on BOTH HAProxy nodes:

```bash
sudo apt install -y keepalived
```

---

# 4. Keepalived Configuration

## On haproxy-1 (MASTER)

```
sudo nano /etc/keepalived/keepalived.conf
```

```
vrrp_script chk_haproxy {
    script "/bin/systemctl is-active --quiet haproxy"
    interval 5
    fall 3
    rise 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 200
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass StrongPass123
    }

    virtual_ipaddress {
        <VIP_IP>/24
    }

    track_script {
        chk_haproxy
    }
}
```

---

## On haproxy-2 (BACKUP)

```
sudo nano /etc/keepalived/keepalived.conf
```

```
vrrp_script chk_haproxy {
    script "/bin/systemctl is-active --quiet haproxy"
    interval 5
    fall 3
    rise 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass StrongPass123
    }

    virtual_ipaddress {
        <VIP_IP>/24
    }

    track_script {
        chk_haproxy
    }
}
```

---

# 5. Enable Keepalived

On BOTH nodes:

```bash
sudo systemctl enable keepalived --now
```

Verify VIP:

```bash
ip a | grep <VIP_IP>
```

VIP should appear ONLY on haproxy-1.

---

# 6. Failover Test

## Test HAProxy failover

On haproxy-1:

```bash
sudo systemctl stop haproxy
```

Expected:

- VIP automatically moves to haproxy-2
- Clients continue working

Check:

```bash
ip a | grep <VIP_IP>
```

---

## Test Node Failover

Stop PostgreSQL leader:

```bash
sudo systemctl stop patroni
```

Expected:

- Patroni elects new Leader
- HAProxy detects change via REST API
- No manual action required

---

# 7. Firewall Ports (Production)

Open these ports:

### PostgreSQL nodes:
- 2379 (etcd client)
- 2380 (etcd peer)
- 5432 (Postgres)
- 8008 (Patroni REST)

### HAProxy nodes:
- 5000 (write)
- 5001 (read)
- VRRP protocol (112)

---

# Final Result

✔ 3 PostgreSQL nodes  
✔ etcd quorum  
✔ Patroni automatic failover  
✔ 2 HAProxy nodes  
✔ Floating VIP  
✔ Automatic HAProxy failover  
✔ Automatic DB leader failover  
✔ Zero manual intervention  

---

# Production Recommendation

- Use dedicated interface for VIP
- Enable firewall properly
- Consider HAProxy stats socket
- Enable TLS between components
- Use strong authentication passwords
- Monitor with Prometheus

---

# Client Connection

Write:

```
psql -h <VIP_IP> -p 5000 -U postgres
```

Read:

```
psql -h <VIP_IP> -p 5001 -U postgres
```

---

Cluster is now fully Highly Available end-to-end.