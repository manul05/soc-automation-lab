[Back to README](../README.md)

# 01. Cloud Infrastructure (DigitalOcean)

Wazuh and TheHive both run Java-based stacks (OpenSearch, Cassandra, Elasticsearch), so they need real memory.
This lab hosts them on two DigitalOcean droplets. Any cloud provider works; the sizing is what matters.

## Step 1: Create a project

1. Log in to DigitalOcean.
2. Click **New Project** and name it something like `SOC-Automation`.

## Step 2: Create the droplets

| Setting | Wazuh droplet | TheHive droplet |
| :--- | :--- | :--- |
| Hostname | `Wazuh` | `Thehive` |
| Image | Ubuntu 24.04 LTS | Ubuntu 24.04 LTS |
| Size | 8 GB RAM / 160 GB disk | 8 GB RAM / 160 GB disk |
| Region | Closest to you (this lab used BLR1) | Same region as Wazuh |
| Authentication | **SSH key** (preferred) or a strong root password | Same |

- **Do not skimp on RAM.** The Wazuh indexer is memory-hungry and the whole stack becomes unstable below 8 GB.
- TheHive runs Cassandra and Elasticsearch alongside the application, which is why it also needs 8 GB (and still
  needs the heap limits in [03](03-thehive-server.md)).

## Step 3: Firewall rules

> [!CAUTION]
> Droplets get a public IP the moment they boot. Internet scanners will start probing SSH within minutes, so create
> the firewall **before** you install anything.

Go to **Networking > Firewalls** and create one firewall per droplet.

**Wazuh firewall (inbound)**

| Port | Protocol | Source | Purpose |
| :-: | :-: | :--- | :--- |
| 22 | TCP | Your home IP only | SSH administration |
| 443 | TCP | Your home IP only | Wazuh dashboard |
| 1514 | TCP | Windows endpoint's public IP | Agent event channel |
| 1515 | TCP | Windows endpoint's public IP | Agent enrollment |

**TheHive firewall (inbound)**

| Port | Protocol | Source | Purpose |
| :-: | :-: | :--- | :--- |
| 22 | TCP | Your home IP only | SSH administration |
| 9000 | TCP | Your home IP **and** the Shuffle connection (see below) | TheHive web UI and API |

> [!IMPORTANT]
> **Home-IP-only on port 9000 blocks the automation.** The Shuffle workflow runs on Shuffle's cloud, so it connects
> to TheHive from outside your home network. The options and trade-offs are explained in
> [06, Phase 3](06-shuffle-soar-workflow.md#phase-3-make-thehive-reachable-from-shuffle). Decide this before you
> reach the Shuffle phase so you are not debugging a "connection refused" later.

Apply each firewall to its droplet, then verify from outside your allow-list that the blocked ports really are closed.

## Step 4: Record your addresses

Keep a private note (not in this repo) with the two public IPs and the endpoint's public IP. Every guide in this repo
uses `<WAZUH_PUBLIC_IP>`, `<THEHIVE_PUBLIC_IP>` and `<ENDPOINT_PUBLIC_IP>` as placeholders.

> [!WARNING]
> Never commit real IP addresses, passwords or API keys to this repository. Home IPs and cloud IPs are both
> sensitive: they tell an attacker exactly what to scan.
