# PostgreSQL 16 HA Cluster (Rocky Linux 9)
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

## Disable SELinux

Edit:

```
/etc/selinux/config
```

Set:

```
SELINUX=disabled
```

Reboot system.

---

## Disable Firewall (Lab Only)

```
sudo systemctl disable firewalld --now
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

```
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16 postgresql16-server
sudo dnf install -y epel-release
sudo dnf install -y python3 python3-devel gcc git wget curl
```

Install Locale (ALL NODES)

```
sudo dnf install -y glibc-langpack-en
sudo localedef -i en_US -f UTF-8 en_US.UTF-8
locale -a | grep en_US
```

Disable default service:

```
sudo systemctl disable postgresql-16 --now
```

Do NOT run initdb manually.

---

# 3. Install etcd (All PostgreSQL Nodes)

## Download Latest Version

```
ETCD_RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest | grep tag_name | cut -d '"' -f 4)
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_RELEASE}/etcd-${ETCD_RELEASE}-linux-amd64.tar.gz
tar xvf etcd-${ETCD_RELEASE}-linux-amd64.tar.gz
cd etcd-${ETCD_RELEASE}-linux-amd64
sudo mv etcd* /usr/local/bin/
```

---

## Create etcd User and Directories

```
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```

---

## Create `/etc/etcd.conf` (Adjust Per Node)

Example for postgre-1:

```
ETCD_NAME=postgre-1.example.com
ETCD_LISTEN_PEER_URLS=http://<IP1>:2380
ETCD_LISTEN_CLIENT_URLS=http://<IP1>:2379,http://127.0.0.1:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://<IP1>:2380
ETCD_ADVERTISE_CLIENT_URLS=http://<IP1>:2379
ETCD_INITIAL_CLUSTER=postgre-1.example.com=http://<IP1>:2380,postgre-2.example.com=http://<IP2>:2380,postgre-3.example.com=http://<IP3>:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
```

---

## Create systemd Service

`/etc/systemd/system/etcd.service`

```
[Unit]
Description=etcd
After=network.target

[Service]
User=etcd
EnvironmentFile=/etc/etcd.conf
ExecStart=/usr/local/bin/etcd
Restart=always

[Install]
WantedBy=multi-user.target
```

Start etcd on ALL nodes:

```
sudo systemctl daemon-reload
sudo systemctl enable etcd --now
```

Verify:

```
etcdctl member list
```

---

# 4. Install Patroni (All PostgreSQL Nodes)

```
sudo dnf install -y python3 python3-devel gcc
sudo python3 -m venv /opt/patroni
sudo /opt/patroni/bin/pip install --upgrade pip
sudo /opt/patroni/bin/pip install patroni[etcd] psycopg2-binary
sudo chown -R postgres:postgres /opt/patroni
```

---

## Fix Locale Issue (Important)

```
sudo localedef -i en_US -f UTF-8 en_US.UTF-8
locale -a | grep en_US
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
  data_dir: /var/lib/pgsql/16/data
  bin_dir: /usr/pgsql-16/bin

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
sudo rm -rf /var/lib/pgsql/16/data
sudo mkdir -p /var/lib/pgsql/16/data
sudo chown -R postgres:postgres /var/lib/pgsql
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
```
sudo systemctl start patroni
```

Verify:
```
sudo -u postgres /opt/patroni/bin/patronictl -c /etc/patroni.yml list
```

Expected:

- 1 Leader
- 2 Replicas

---

# 9. Install HAProxy (haproxy-1)

```
sudo dnf install -y haproxy
```

Edit `/etc/haproxy/haproxy.cfg`:

```
global
    daemon

defaults
    mode tcp
    timeout connect 5s
    timeout client 30m
    timeout server 30m

listen postgres_write
    bind *:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    server pg1 <IP1>:5432 check port 8008
    server pg2 <IP2>:5432 check port 8008
    server pg3 <IP3>:5432 check port 8008

listen postgres_read
    bind *:5001
    balance leastconn
    option httpchk OPTIONS /replica
    http-check expect status 200
    server pg1 <IP1>:5432 check port 8008
    server pg2 <IP2>:5432 check port 8008
    server pg3 <IP3>:5432 check port 8008
```

Start HAProxy:

```
sudo systemctl enable haproxy --now
```

---

# 10. Testing

Write:

```
psql -h <HAPROXY_IP> -p 5000 -U postgres
```

Read:

```
psql -h <HAPROXY_IP> -p 5001 -U postgres
```

---

# 11. Failover Test

Stop leader:

```
sudo systemctl stop patroni
```

Cluster will elect new leader automatically.
HAProxy redirects traffic automatically.

---

# Final Result

- 3-node PostgreSQL HA
- etcd quorum
- Patroni-managed failover
- HAProxy read/write split
- Clean bootstrap process
- Official PGDG packages
- Locale issue fixed