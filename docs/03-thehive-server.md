[Back to README](../README.md)

# 03. TheHive 5 Server
**Server**: Ubuntu 24.04 LTS on DigitalOcean  
**Installed on**: `<THEHIVE_PUBLIC_IP>`  
**Last Updated**: April 2026

---

## What Was Installed

| Component | Version | Port |
|-----------|---------|------|
| Java (Amazon Corretto) | 11.0.30 LTS | — |
| Apache Cassandra | 4.1.11 | 9042 |
| Elasticsearch | 7.17.29 | 9200 |
| **TheHive** | **5.5.3-1** | **9000** |

> [!IMPORTANT]
> **Access TheHive**: `http://<THEHIVE_PUBLIC_IP>:9000`  
> **Default login**: `admin@thehive.local` / `secret`

---

## Step 1 — System Update

```bash
apt-get update && apt-get full-upgrade -y
```

---

## Step 2 — Install Dependencies

```bash
apt install -y wget gnupg apt-transport-https git ca-certificates \
  ca-certificates-java curl software-properties-common python3-pip lsb-release
```

---

## Step 3 — Install Java (Amazon Corretto 11)

```bash
# Add Corretto GPG key
wget -qO- https://apt.corretto.aws/corretto.key | gpg --dearmor -o /usr/share/keyrings/corretto.gpg

# Add Corretto repository
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" | \
  tee /etc/apt/sources.list.d/corretto.sources.list

# Install Java 11 Corretto
apt-get update -y
apt-get install -y java-common java-11-amazon-corretto-jdk

# Set JAVA_HOME
echo JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto" | tee -a /etc/environment
export JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto"

# Verify
java -version
```

**Expected output:**
```
openjdk version "11.0.30" 2026-01-20 LTS
OpenJDK Runtime Environment Corretto-11.0.30.7.1
```

---

## Step 4 — Install Apache Cassandra 4.1.x

> [!NOTE]
> The repo series is `41x` (not `40x` as in older guides). Use `41x` for Cassandra 4.1.x.

```bash
# Add Cassandra GPG key
wget -qO - https://downloads.apache.org/cassandra/KEYS | \
  gpg --dearmor -o /usr/share/keyrings/cassandra-archive.gpg

# Add Cassandra 4.1.x repository
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] https://debian.cassandra.apache.org 41x main" | \
  tee /etc/apt/sources.list.d/cassandra.sources.list

# Install Cassandra
apt-get update -y
apt-get install -y cassandra

# Enable and start
systemctl enable cassandra
systemctl start cassandra

# Verify
cassandra -v
```

**Expected output:** `4.1.11`

---

## Step 5 — Install Elasticsearch 7.x

```bash
# Add Elasticsearch GPG key
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Add Elasticsearch 7.x repository
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | \
  tee /etc/apt/sources.list.d/elastic-7.x.list

# Install Elasticsearch
apt-get update -y
apt-get install -y elasticsearch

# Enable and start
systemctl enable elasticsearch
systemctl start elasticsearch
```

**Verify it's running:**
```bash
systemctl status elasticsearch --no-pager
curl -s http://localhost:9200
```

---

## Step 5.5 — Configure Memory Limits & Cluster Name (Crucial Step)

> [!IMPORTANT]
> **Prevent Out-Of-Memory (OOM) Crashes & Connection Errors**
> By default, Elasticsearch allocates 4GB of RAM, which crashes typical 8GB cloud servers once Cassandra and TheHive run alongside it. Furthermore, TheHive's default `/etc/thehive/application.conf` expects the Cassandra cluster to be named `thp`, but Cassandra defaults to `Test Cluster`. You **must** apply these fixes before installing and starting TheHive.

**1. Limit Elasticsearch Heap to 1GB:**
```bash
mkdir -p /etc/elasticsearch/jvm.options.d/
echo "-Xms1024m" > /etc/elasticsearch/jvm.options.d/heap.options
echo "-Xmx1024m" >> /etc/elasticsearch/jvm.options.d/heap.options
systemctl restart elasticsearch
```

**2. Limit Cassandra Heap to 512MB & Fix Cluster Name:**
```bash
# Lower Cassandra Heap Size
sed -i 's/^-Xms.*/-Xms512m/' /etc/cassandra/jvm*.options
sed -i 's/^-Xmx.*/-Xmx512m/' /etc/cassandra/jvm*.options

# Change Cassandra Cluster Name to 'thp'
sed -i "s/^cluster_name:.*/cluster_name: 'thp'/" /etc/cassandra/cassandra.yaml

# When Cassandra installed in Step 4, it automatically started using the default 'Test Cluster' name.
# Because we just changed the name to 'thp' above, we MUST delete the old auto-generated database files.
# If we skip this rm command, Cassandra will crash on startup due to a cluster name mismatch.
systemctl stop cassandra
rm -rf /var/lib/cassandra/data/*
systemctl start cassandra
```

---

## Step 6 — Install TheHive 5.5.3

> [!WARNING]
> **IMPORTANT — OLD METHOD NO LONGER WORKS**: The old `archives.strangebee.com` APT repository has been discontinued. The domain `archives.strangebee.com` no longer exists in DNS. Do NOT use the old `deb [signed-by=...] https://deb.strangebee.com thehive-5.2 main` repository method.

> [!NOTE]
> **New Method (2025+)**: StrangeBee now distributes TheHive as a direct `.deb` download from `thehive.download.strangebee.com`. The current version is **5.5.3-1**.

```bash
# Download TheHive 5.5.3 .deb package directly (445MB)
wget -O /tmp/thehive.deb \
  "https://thehive.download.strangebee.com/5.5/deb/thehive_5.5.3-1_all.deb"

# Install using apt (handles dependencies automatically)
apt-get install -y /tmp/thehive.deb

# Create file storage directory
mkdir -p /opt/thp/thehive/files
chown -R thehive:thehive /opt/thp

# Enable and start TheHive
systemctl enable thehive
systemctl start thehive
```

---

## Step 7 — Verify All Services

```bash
# Check all services are running
systemctl status cassandra --no-pager
systemctl status elasticsearch --no-pager
systemctl status thehive --no-pager

# Check listening ports
ss -tlnp | grep -E '9042|9200|9000'
```

**Expected ports:**
```
127.0.0.1:9042   → Cassandra
127.0.0.1:9200   → Elasticsearch  
0.0.0.0:9000     → TheHive (web UI)
```

---

## TheHive Configuration

The default config is at `/etc/thehive/application.conf`. It is pre-configured for:

- **Cassandra** on `127.0.0.1:9042`, keyspace `thehive`
- **Elasticsearch** on `127.0.0.1:9200`, index `thehive`
- **File storage** at `/opt/thp/thehive/files`

No changes needed for a basic local setup.

---

## Accessing TheHive

1. Open browser: `http://<THEHIVE_PUBLIC_IP>:9000`
2. Login with:
   - Username: `admin@thehive.local`
   - Password: `secret`
3. **Change the default password immediately after first login!**

---

## Quick Reference — Service Management

```bash
# Start/Stop/Restart services
systemctl start|stop|restart cassandra
systemctl start|stop|restart elasticsearch
systemctl start|stop|restart thehive

# View logs
journalctl -u thehive -f
journalctl -u cassandra -f
journalctl -u elasticsearch -f

# TheHive logs directly
tail -f /var/log/thehive/application.log
```

---

## Troubleshooting

### TheHive not starting
```bash
journalctl -u thehive -n 50 --no-pager
```
Wait 60-90 seconds after starting — TheHive takes time to initialize the database schema on first boot.

### Cassandra not connecting
```bash
nodetool status   # should show UN (Up/Normal)
cqlsh 127.0.0.1 9042  # test connection
```

### Elasticsearch not starting
```bash
journalctl -u elasticsearch -n 30 --no-pager
# Check heap settings
cat /etc/elasticsearch/jvm.options.d/*.options
```

### DNS issues (for future package installs)
If the server has DNS issues, set Google DNS:
```bash
echo 'nameserver 8.8.8.8
nameserver 1.1.1.1' > /etc/resolv.conf
```

---

> [!NOTE]
> Shuffle (cloud) must be able to reach TheHive on port 9000 for the automation in [06](06-shuffle-soar-workflow.md#phase-3-make-thehive-reachable-from-shuffle) to work. The home-IP-only firewall rule from [01](01-cloud-infrastructure.md) blocks it by default.

## Notes on URL Changes

The original tutorial commands used **outdated URLs**:

| Original (OLD — BROKEN) | Updated (WORKING) |
|-------------------------|-------------------|
| `archives.strangebee.com/keys/strangebee.gpg` | Direct `.deb` download from `thehive.download.strangebee.com` |
| `deb.strangebee.com thehive-5.2` apt repo | No apt repo anymore — use direct download |
| `thehive-5.2` version | Current version: **5.5.3-1** |
| Cassandra `40x` | Current Cassandra: **4.1.x** → use `41x` |

Alternatively, StrangeBee provides an **automated install script**:
```bash
wget -q -O /tmp/install_script.sh https://scripts.download.strangebee.com/latest/sh/install_script.sh
# Then run as non-root user:
bash /tmp/install_script.sh
```
> Note: This script is interactive and requires a non-root user with sudo access.
