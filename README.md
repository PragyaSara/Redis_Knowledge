# Redis_Knowledge
A practical guide to understanding Redis concepts, persistence (RDB &amp; AOF), architecture, and production best practices.
# 🔴 Redis Production Deployment Guide

> Complete setup for all three production architectures — **Standalone**, **Replication + Sentinel**, and **Redis Cluster**.

---

## 📋 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Pre-Requisites & OS Hardening](#️-pre-requisites--os-hardening)
- [Architecture 1 — Standalone](#-architecture-1--standalone-redis)
- [Architecture 2 — Replication + Sentinel](#-architecture-2--replication--sentinel-high-availability)
- [Architecture 3 — Redis Cluster](#-architecture-3--redis-cluster-ha--horizontal-scaling)
- [Persistence — RDB vs AOF](#-persistence--rdb-vs-aof)
- [Security Hardening](#-security-hardening)
- [Monitoring](#-monitoring)
- [Troubleshooting](#-troubleshooting)
- [Quick Reference & Production Checklist](#-quick-reference--production-checklist)

---

## 🏗 Architecture Overview

| Architecture | Best For | High Availability | Scaling |
|---|---|---|---|
| Standalone | Dev / Small cache | ❌ | ❌ |
| Replication + Sentinel | HA without scaling | ✅ | ❌ |
| Redis Cluster | HA + Horizontal scaling | ✅ | ✅ |

---

## ⚙️ Pre-Requisites & OS Hardening

### 1. System Requirements

| Component | Minimum Recommendation |
|---|---|
| OS | Ubuntu 20.04 / 22.04 LTS (recommended) |
| RAM | 4 GB minimum; 8 GB+ for production |
| Disk | SSD strongly recommended for AOF persistence |
| CPU | 2+ cores |
| Redis Version | 7.x |

### 2. Install Redis

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install redis-server -y

# Verify
redis-server --version
redis-cli ping   # should return PONG
```

### 3. Kernel / OS Tuning (REQUIRED for production)

> ⚠️ **Skipping these causes Redis to log warnings and can cause fork failures during RDB snapshots (BGSAVE), which will block your server.**

```bash
# Disable Transparent Huge Pages (THP) — Redis hates this
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

# Make THP permanent on reboot
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' | \
  sudo tee -a /etc/rc.local
sudo chmod +x /etc/rc.local

# Set overcommit memory (required for background save / RDB / AOF)
sudo sysctl vm.overcommit_memory=1
echo 'vm.overcommit_memory = 1' | sudo tee -a /etc/sysctl.conf

# Increase system open file limit
echo 'fs.file-max = 100000' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Set Redis user limits
echo 'redis soft nofile 65536' | sudo tee -a /etc/security/limits.conf
echo 'redis hard nofile 65536' | sudo tee -a /etc/security/limits.conf
```

### 4. Firewall Rules

> Only open Redis ports to trusted IPs — **never expose Redis directly to the internet.**

```bash
# Standalone — allow only from your app server
sudo ufw allow from <APP_SERVER_IP> to any port 6379

# Sentinel — allow Redis + Sentinel ports
sudo ufw allow from <APP_SERVER_IP> to any port 26379
sudo ufw allow from <REDIS_NODE_IP> to any port 26379

# Cluster — allow data ports + cluster bus ports (port + 10000)
sudo ufw allow from <NODE_IP> to any port 7000:7005
sudo ufw allow from <NODE_IP> to any port 17000:17005

sudo ufw enable
```

### 5. Redis User & Directory Setup

```bash
# Verify redis user exists
id redis

# Set correct permissions
sudo chown redis:redis /etc/redis/redis.conf
sudo chown -R redis:redis /var/lib/redis
sudo chmod 640 /etc/redis/redis.conf
```

---

## 🟢 Architecture 1 — Standalone Redis

**Use when:** application caching, dev environments, small workloads, no HA requirement.

```
Application → Redis (single node)
If Redis crashes → service unavailable until Redis restarts.
```

### Step 1 — Configure `/etc/redis/redis.conf`

```bash
sudo nano /etc/redis/redis.conf
```

```conf
# ── NETWORK ──────────────────────────────────────────────
bind 127.0.0.1 <SERVER_IP>    # loopback + server IP
port 6379
protected-mode yes
tcp-backlog 511
timeout 300                    # close idle connections after 5 min
tcp-keepalive 60

# ── SECURITY ─────────────────────────────────────────────
requirepass YOUR_STRONG_PASSWORD_HERE
rename-command FLUSHALL ""     # disable dangerous commands
rename-command FLUSHDB  ""
rename-command DEBUG    ""
rename-command CONFIG   REDIS_CONFIG_COMMAND

# ── PERSISTENCE (AOF) ────────────────────────────────────
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec           # balance between perf and durability
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# ── PERSISTENCE (RDB Snapshots) ──────────────────────────
save 900 1                     # save if 1 key changed in 900 sec
save 300 10                    # save if 10 keys changed in 300 sec
save 60 10000                  # save if 10000 keys changed in 60 sec
dbfilename dump.rdb
dir /var/lib/redis

# ── MEMORY ───────────────────────────────────────────────
maxmemory 2gb                  # set to ~75% of available RAM
maxmemory-policy allkeys-lru

# ── LOGGING ──────────────────────────────────────────────
loglevel notice
logfile /var/log/redis/redis-server.log

# ── PERFORMANCE ──────────────────────────────────────────
hz 10
maxclients 10000
lua-time-limit 5000
```

### Step 2 — systemctl Commands

```bash
# Start
sudo systemctl start redis-server

# Enable on boot (IMPORTANT for production)
sudo systemctl enable redis-server

# Restart
sudo systemctl restart redis-server

# Reload config without full restart
sudo systemctl reload redis-server

# Stop
sudo systemctl stop redis-server

# Status
sudo systemctl status redis-server

# View logs
sudo journalctl -u redis-server -f
sudo tail -f /var/log/redis/redis-server.log
```

### Step 3 — Verify & Test

```bash
# Connect
redis-cli -a YOUR_STRONG_PASSWORD_HERE

# Test basic operations
PING                          # should return PONG
SET testkey 'hello'
GET testkey                   # should return 'hello'
DEL testkey

# Check memory
INFO memory

# Check connected clients
CLIENT LIST

# Check persistence status
INFO persistence

# Check stats
INFO stats
```

### Step 4 — Validate Persistence

```bash
# Test that data survives a restart
redis-cli -a YOUR_PASSWORD SET persist_test 'alive'
sudo systemctl restart redis-server
redis-cli -a YOUR_PASSWORD GET persist_test   # should return 'alive'

# Check RDB last save time
redis-cli -a YOUR_PASSWORD LASTSAVE

# Manually trigger RDB snapshot
redis-cli -a YOUR_PASSWORD BGSAVE

# Check AOF status
redis-cli -a YOUR_PASSWORD INFO persistence | grep aof
```

---

## 🟡 Architecture 2 — Replication + Sentinel (High Availability)

**Use when:** you need automatic failover and HA without horizontal scaling.

```
        Sentinel1 (port 26379)
        Sentinel2 (port 26380)
        Sentinel3 (port 26381)
               │
           Master (6379)
          /             \
  Replica1 (6380)   Replica2 (6381)
```

> When master fails, Sentinels vote and promote a replica automatically. Minimum 3 Sentinel instances required to form quorum.

### Server Layout

| Server | Role | Redis Port | Sentinel Port |
|---|---|---|---|
| server1 | Master | 6379 | 26379 |
| server2 | Replica 1 | 6380 | 26380 |
| server3 | Replica 2 | 6381 | 26381 |

### Step 1 — Master Configuration

File: `/etc/redis/redis.conf` on **server1**

```conf
# ── NETWORK ──────────────────────────────────────────────
bind 0.0.0.0
port 6379
protected-mode no              # Sentinels need to connect freely

# ── SECURITY ─────────────────────────────────────────────
requirepass YOUR_STRONG_PASSWORD
masterauth  YOUR_STRONG_PASSWORD  # needed when replica becomes master

# ── PERSISTENCE ──────────────────────────────────────────
appendonly yes
appendfsync everysec
save 900 1
save 300 10
save 60 10000
dir /var/lib/redis
dbfilename dump.rdb

# ── MEMORY ───────────────────────────────────────────────
maxmemory 4gb
maxmemory-policy allkeys-lru

# ── LOGGING ──────────────────────────────────────────────
loglevel notice
logfile /var/log/redis/redis-master.log
```

### Step 2 — Replica Configuration

File: `/etc/redis/redis.conf` on **server2** (Replica 1)

```conf
bind 0.0.0.0
port 6380
protected-mode no

# ── REPLICATION ───────────────────────────────────────────
replicaof <MASTER_IP> 6379
masterauth YOUR_STRONG_PASSWORD   # password to authenticate with master

# ── SECURITY ─────────────────────────────────────────────
requirepass YOUR_STRONG_PASSWORD

# Make replica read-only (recommended)
replica-read-only yes

# ── PERSISTENCE ──────────────────────────────────────────
appendonly yes
appendfsync everysec
dir /var/lib/redis

# ── MEMORY ───────────────────────────────────────────────
maxmemory 4gb
maxmemory-policy allkeys-lru

loglevel notice
logfile /var/log/redis/redis-replica1.log
```

> For **Replica 2** (server3): same config, change `port` to `6381` and `logfile` name.

### Step 3 — Start & Enable Redis on All Nodes

```bash
# Run on ALL three servers
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Verify replication on master
redis-cli -p 6379 -a YOUR_PASSWORD INFO replication
# Expected:
#   role:master
#   connected_slaves:2
#   slave0:ip=<replica1_ip>,port=6380,state=online
#   slave1:ip=<replica2_ip>,port=6381,state=online

# Verify from replica
redis-cli -p 6380 -a YOUR_PASSWORD INFO replication
# Expected:
#   role:slave
#   master_host:<master_ip>
#   master_link_status:up
```

### Step 4 — Sentinel Configuration

Create `/etc/redis/sentinel.conf` on **all three servers** (change port per server):

```conf
# Sentinel 1 (server1) — port 26379
# Sentinel 2 (server2) — port 26380
# Sentinel 3 (server3) — port 26381

port 26379                     # change to 26380 / 26381 on other servers
bind 0.0.0.0
protected-mode no

# Monitor master: name  ip  port  quorum
# quorum=2 means 2 sentinels must agree before failover
sentinel monitor mymaster <MASTER_IP> 6379 2

# Password for Redis nodes
sentinel auth-pass mymaster YOUR_STRONG_PASSWORD

# How long (ms) a node must be unreachable before marking it down
sentinel down-after-milliseconds mymaster 5000

# Max time (ms) allowed for entire failover process
sentinel failover-timeout mymaster 60000

# How many replicas to sync in parallel after failover (1 = safest)
sentinel parallel-syncs mymaster 1

logfile /var/log/redis/sentinel.log
loglevel notice
```

### Step 5 — Sentinel as systemd Service

Create `/etc/systemd/system/redis-sentinel.service` on each server:

```ini
[Unit]
Description=Redis Sentinel
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/sentinel.conf --sentinel
ExecStop=/bin/kill -s TERM $MAINPID
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start on all three servers
sudo systemctl daemon-reload
sudo systemctl start redis-sentinel
sudo systemctl enable redis-sentinel

# Status and logs
sudo systemctl status redis-sentinel
sudo journalctl -u redis-sentinel -f
sudo tail -f /var/log/redis/sentinel.log
```

### Step 6 — Verify Sentinel

```bash
# Check master
redis-cli -p 26379 SENTINEL masters

# Check replicas
redis-cli -p 26379 SENTINEL replicas mymaster

# Check all sentinels
redis-cli -p 26379 SENTINEL sentinels mymaster

# Get current master IP:PORT
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

### Step 7 — Test Failover

```bash
# Simulate hang on master
redis-cli -p 6379 -a YOUR_PASSWORD DEBUG SLEEP 30

# OR kill master process
sudo kill -9 <MASTER_PID>

# Watch sentinel promote a replica (within ~5 seconds)
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# Output IP will change to one of the replicas

# Watch failover logs
sudo tail -f /var/log/redis/sentinel.log
```

> ⚠️ After failover, the old master comes back as a **replica**. Sentinel reconfigures it automatically. Your application must use a **Sentinel-aware client** (not a direct IP) to auto-discover the new master.

### Sentinel Settings Reference

| Setting | Meaning |
|---|---|
| `monitor mymaster IP PORT QUORUM` | Master to watch; quorum = votes needed to trigger failover |
| `auth-pass` | Password to authenticate with Redis nodes |
| `down-after-milliseconds` | Node marked DOWN after this many ms of no response |
| `failover-timeout` | Max ms allowed for entire failover; doubles on retry |
| `parallel-syncs` | Replicas syncing simultaneously after failover (1 = safest) |

---

## 🔵 Architecture 3 — Redis Cluster (HA + Horizontal Scaling)

**Use when:** you need both high availability AND horizontal scaling.

```
Master1 (7000) <--> Replica1 (7003)
Master2 (7001) <--> Replica2 (7004)
Master3 (7002) <--> Replica3 (7005)
```

> **No Sentinel needed** — cluster handles failover internally.
> **IMPORTANT:** Use cluster-aware clients (ioredis, redis-py cluster mode, Jedis cluster).

### Server Layout

| Server | Role | Port | Bus Port | Hash Slots |
|---|---|---|---|---|
| server1 | Master 1 | 7000 | 17000 | 0 – 5460 |
| server2 | Master 2 | 7001 | 17001 | 5461 – 10922 |
| server3 | Master 3 | 7002 | 17002 | 10923 – 16383 |
| server4 | Replica of Master 1 | 7003 | 17003 | mirrors 7000 |
| server5 | Replica of Master 2 | 7004 | 17004 | mirrors 7001 |
| server6 | Replica of Master 3 | 7005 | 17005 | mirrors 7002 |

> **Bus Port = Redis Port + 10000.** Redis uses this for node-to-node communication. Open **both** ports in your firewall.

### Step 1 — Create Directory Structure

```bash
sudo mkdir -p /etc/redis/cluster/{7000,7001,7002,7003,7004,7005}
sudo chown -R redis:redis /etc/redis/cluster

sudo mkdir -p /var/lib/redis/cluster/{7000,7001,7002,7003,7004,7005}
sudo chown -R redis:redis /var/lib/redis/cluster
```

### Step 2 — Cluster Node Configuration

Generate configs for all 6 nodes at once:

```bash
for port in 7000 7001 7002 7003 7004 7005; do
  sudo tee /etc/redis/cluster/${port}/redis.conf > /dev/null <<EOF
bind 0.0.0.0
port ${port}
protected-mode no
cluster-enabled yes
cluster-config-file /etc/redis/cluster/${port}/nodes.conf
cluster-node-timeout 5000
cluster-require-full-coverage no
requirepass YOUR_STRONG_PASSWORD
masterauth YOUR_STRONG_PASSWORD
appendonly yes
appendfsync everysec
dir /var/lib/redis/cluster/${port}
dbfilename dump.rdb
loglevel notice
logfile /var/log/redis/cluster-${port}.log
EOF
done
```

### Step 3 — Systemd Service for Each Node

```bash
for port in 7000 7001 7002 7003 7004 7005; do
  sudo tee /etc/systemd/system/redis-cluster-${port}.service > /dev/null <<EOF
[Unit]
Description=Redis Cluster Node ${port}
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/cluster/${port}/redis.conf
ExecStop=/bin/kill -s TERM \$MAINPID
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
done

sudo systemctl daemon-reload

for port in 7000 7001 7002 7003 7004 7005; do
  sudo systemctl start redis-cluster-${port}
  sudo systemctl enable redis-cluster-${port}
done

# Verify all nodes are up
for port in 7000 7001 7002 7003 7004 7005; do
  echo -n "Port ${port}: "
  redis-cli -p ${port} -a YOUR_PASSWORD PING
done
```

### Step 4 — Create the Cluster

> ⚠️ **Run this ONLY ONCE after all 6 nodes are running. Running it again on an existing cluster will cause data loss.**

```bash
redis-cli -a YOUR_STRONG_PASSWORD --cluster create \
  <SERVER1_IP>:7000 \
  <SERVER2_IP>:7001 \
  <SERVER3_IP>:7002 \
  <SERVER4_IP>:7003 \
  <SERVER5_IP>:7004 \
  <SERVER6_IP>:7005 \
  --cluster-replicas 1

# Type 'yes' when prompted to confirm slot assignment
```

### Step 5 — Verify Cluster

```bash
# Overall cluster health
redis-cli -p 7000 -a YOUR_PASSWORD CLUSTER INFO
# cluster_state:ok              <- must be ok
# cluster_slots_assigned:16384  <- all slots assigned
# cluster_known_nodes:6

# List all nodes with roles
redis-cli -p 7000 -a YOUR_PASSWORD CLUSTER NODES

# Slot to node mapping
redis-cli -p 7000 -a YOUR_PASSWORD CLUSTER SLOTS

# Which slot does a key belong to?
redis-cli -p 7000 -a YOUR_PASSWORD CLUSTER KEYSLOT mykey
```

### Step 6 — Connect and Use Cluster

```bash
# ALWAYS use -c flag (cluster mode) — handles automatic key redirection
redis-cli -c -p 7000 -a YOUR_PASSWORD

# Test SET/GET — Redis redirects to correct node automatically
SET user 'pragya'
GET user
# -> Redirected to slot [5474] located at 127.0.0.1:7001
# 'pragya'
```

### Step 7 — systemctl Commands for Cluster

```bash
# Start / Stop / Restart a specific node
sudo systemctl start   redis-cluster-7000
sudo systemctl stop    redis-cluster-7000
sudo systemctl restart redis-cluster-7000

# View node logs
sudo journalctl -u redis-cluster-7000 -f
sudo tail -f /var/log/redis/cluster-7000.log

# Check status of all nodes at once
for port in 7000 7001 7002 7003 7004 7005; do
  echo "=== Port $port ==="
  sudo systemctl status redis-cluster-${port} --no-pager | grep Active
done
```

### Step 8 — Test Cluster Failover

```bash
# Check which nodes are masters
redis-cli -p 7000 -a YOUR_PASSWORD CLUSTER NODES | grep master

# Simulate crash of master on port 7000
sudo systemctl stop redis-cluster-7000

# Check cluster — replica promotes to master within ~5 seconds
redis-cli -p 7001 -a YOUR_PASSWORD CLUSTER NODES
# The old master (7000) shows as 'fail'
# Its replica now shows as 'master'

# Restart old master — comes back as replica automatically
sudo systemctl start redis-cluster-7000
```

### Hash Slots Explained

```bash
# Redis uses 16384 hash slots
# Key -> CRC16(key) % 16384 -> slot -> master node

# Check which slot a key maps to
CLUSTER KEYSLOT mykey

# Use hash tags to force related keys to the same slot
# Required for MGET and transactions in cluster mode
SET {user}.name  'pragya'
SET {user}.email 'pragya@example.com'
# Both keys map to slot of 'user' — same node
```

### Cluster Settings Reference

| Setting | Meaning |
|---|---|
| `cluster-enabled yes` | Enable cluster mode for this node |
| `cluster-config-file nodes.conf` | Auto-managed file tracking cluster state |
| `cluster-node-timeout 5000` | ms before unreachable node is considered failed |
| `cluster-require-full-coverage no` | Allow reads/writes even if some slots are unavailable |
| `--cluster-replicas 1` | One replica per master when creating cluster |

---

## 💾 Persistence — RDB vs AOF

| | RDB (Snapshots) | AOF (Append-Only File) |
|---|---|---|
| What it does | Point-in-time snapshot | Logs every write operation |
| File | `dump.rdb` | `appendonly.aof` |
| Durability | May lose data since last snapshot | Up to 1 second loss (everysec) |
| Restart speed | Fast | Slower (replays all writes) |
| File size | Compact binary | Larger, grows over time |
| Production recommendation | ✅ Keep enabled | ✅ Keep enabled |

### `appendfsync` Options

| Value | Behavior |
|---|---|
| `always` | Flush to disk on every write. Slowest, safest. |
| `everysec` ✅ | Flush once per second. At most 1 second of data loss. **Recommended.** |
| `no` | OS decides. Fastest but risky. Not recommended for production. |

### AOF Rewrite (compaction)

```bash
# Manually trigger AOF rewrite to compact the file
redis-cli -a YOUR_PASSWORD BGREWRITEAOF

# Check rewrite status
redis-cli -a YOUR_PASSWORD INFO persistence | grep aof_rewrite
```

### Backup Strategy

```bash
# Copy RDB snapshot (safe while Redis is running)
cp /var/lib/redis/dump.rdb /backup/redis-$(date +%Y%m%d-%H%M%S).rdb

# Automate with cron — backup every day at 2 AM
sudo crontab -e -u redis
# Add this line:
0 2 * * * cp /var/lib/redis/dump.rdb /backup/redis-$(date +\%Y\%m\%d).rdb

# Restore from RDB backup
sudo systemctl stop redis-server
sudo cp /backup/redis-20250101.rdb /var/lib/redis/dump.rdb
sudo chown redis:redis /var/lib/redis/dump.rdb
sudo systemctl start redis-server
```

---

## 🔒 Security Hardening

| Security Item | How to Apply |
|---|---|
| Set a strong password | `requirepass <password>` in redis.conf |
| Set masterauth | `masterauth <password>` — required on replicas |
| Disable dangerous commands | `rename-command FLUSHALL ""` in redis.conf |
| Bind to specific IPs | `bind 127.0.0.1 <APP_IP>` — never `0.0.0.0` on public servers |
| Enable protected-mode | `protected-mode yes` (standalone/sentinel) |
| Run as non-root | Redis runs as `redis` user by default — never run as root |
| Firewall all Redis ports | UFW/iptables — only allow trusted IPs |

### Disable Dangerous Commands

```conf
# Add to redis.conf
rename-command FLUSHALL  ""
rename-command FLUSHDB   ""
rename-command DEBUG     ""
rename-command SHUTDOWN  ""
rename-command CONFIG    "ADMIN_CONFIG_SECRET_HERE"
```

---

## 📊 Monitoring

### Essential INFO Commands

```bash
# All stats in one go
redis-cli -a YOUR_PASSWORD INFO all

# Memory usage
redis-cli -a YOUR_PASSWORD INFO memory

# Replication status
redis-cli -a YOUR_PASSWORD INFO replication

# Persistence status
redis-cli -a YOUR_PASSWORD INFO persistence

# Connected clients
redis-cli -a YOUR_PASSWORD INFO clients
redis-cli -a YOUR_PASSWORD CLIENT LIST

# Slow query log
redis-cli -a YOUR_PASSWORD SLOWLOG GET 10

# Real-time command monitoring (use carefully in production)
redis-cli -a YOUR_PASSWORD MONITOR

# Stats — hits, misses, evictions
redis-cli -a YOUR_PASSWORD INFO stats | grep -E 'keyspace|evicted|expired'
```

### Key Metrics to Watch

| Metric | What to Watch For |
|---|---|
| `used_memory_human` | Compare to `maxmemory`; alert at 80% |
| `evicted_keys` | High number = memory pressure; increase `maxmemory` |
| `rejected_connections` | `maxclients` hit; investigate client leaks |
| `rdb_last_bgsave_status` | Should be `ok`; `err` = snapshot failing |
| `aof_last_write_status` | Should be `ok` |
| `master_link_status` | On replica: should be `up`; `down` = replication broken |
| `cluster_state` | On cluster: should be `ok`; `fail` = cluster needs attention |
| `instantaneous_ops_per_sec` | Baseline + alert on sudden spikes |

### Prometheus + Grafana + Redis Exporter

```bash
# Install Redis Exporter
wget https://github.com/oliver006/redis_exporter/releases/latest/download/redis_exporter-linux-amd64.tar.gz
tar xf redis_exporter-linux-amd64.tar.gz
sudo mv redis_exporter /usr/local/bin/

# Create systemd service
sudo tee /etc/systemd/system/redis-exporter.service > /dev/null <<EOF
[Unit]
Description=Redis Exporter
After=network.target

[Service]
User=redis
ExecStart=/usr/local/bin/redis_exporter \
  --redis.addr=redis://127.0.0.1:6379 \
  --redis.password=YOUR_STRONG_PASSWORD
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start redis-exporter
sudo systemctl enable redis-exporter

# Metrics available at:
# http://localhost:9121/metrics
```

---

## 🔧 Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Redis won't start | Port in use / config error | `journalctl -u redis-server`; check port with `ss -tlnp` |
| `MISCONF: Redis configured to save RDB...` | `vm.overcommit_memory` not set | `sudo sysctl vm.overcommit_memory=1` |
| Replica not syncing / `master_link_status: down` | Network or password mismatch | Check `masterauth` equals master `requirepass`; check firewall |
| Sentinel not triggering failover | Quorum not reached | Check all 3 sentinels are running; verify `quorum 2` in config |
| `CLUSTERDOWN — Hash slot not served` | Master node down, no replica | `CLUSTER NODES` to find failed node; restart or replace |
| OOM — `maxmemory` exceeded | Memory limit hit | Increase `maxmemory` or tune `maxmemory-policy` |
| `WRONGTYPE Operation` | Key type mismatch in app | `TYPE <key>` to inspect; fix app logic |
| THP warning in logs | Transparent Huge Pages not disabled | `echo never > /sys/kernel/mm/transparent_hugepage/enabled` |
| Can't connect to cluster node | Bus port blocked | Open ports `17000–17005` in firewall |

---

## 📌 Quick Reference & Production Checklist

### systemctl Commands — All Architectures

| Action | Standalone | Replication + Sentinel | Cluster |
|---|---|---|---|
| Start | `systemctl start redis-server` | `systemctl start redis-server` | `systemctl start redis-cluster-7000` |
| Stop | `systemctl stop redis-server` | `systemctl stop redis-server` | `systemctl stop redis-cluster-7000` |
| Restart | `systemctl restart redis-server` | `systemctl restart redis-server` | `systemctl restart redis-cluster-7000` |
| Enable on boot | `systemctl enable redis-server` | `systemctl enable redis-server` | `systemctl enable redis-cluster-7000` |
| Sentinel start | N/A | `systemctl start redis-sentinel` | N/A |
| Sentinel enable | N/A | `systemctl enable redis-sentinel` | N/A |

> For cluster: repeat the command for each node port (7000–7005).

### Architecture Decision Guide

| Scenario | Recommended Architecture |
|---|---|
| Dev / testing | Standalone |
| Production HA, single region | Replication + Sentinel |
| Production HA + high write throughput | Redis Cluster |
| Session store for one app server | Standalone |
| Session store across multiple app servers | Redis Cluster |

### ✅ Production Checklist

- [ ] OS tuning done — THP disabled, `vm.overcommit_memory=1`
- [ ] `requirepass` set on all nodes
- [ ] `masterauth` set on all replicas
- [ ] Dangerous commands renamed/disabled
- [ ] Firewall rules — only trusted IPs on Redis ports
- [ ] Both RDB + AOF persistence enabled
- [ ] `systemctl enable` on all Redis/Sentinel services
- [ ] Log file path configured and logrotate set up
- [ ] Monitoring — Redis Exporter + Prometheus + Grafana
- [ ] Backups — cron job copying RDB snapshots
- [ ] Tested failover (Sentinel or Cluster)
