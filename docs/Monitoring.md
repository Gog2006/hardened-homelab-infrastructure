# Monitoring Stack Setup Guide
### Prometheus + Grafana + Node Exporter on Ubuntu Server 22.04 LTS

---

## Overview

This guide documents the full setup of a production-style monitoring stack on a dedicated Ubuntu Server VM. It covers infrastructure provisioning, OS hardening, and deployment of Prometheus, Node Exporter, and Grafana — a standard open-source observability stack used in professional DevOps environments.

### Stack

| Tool | Role | Port |
|------|------|------|
| Prometheus | Metrics collection & storage | 9090 |
| Node Exporter | Hardware & OS metrics exporter | 9100 |
| Grafana | Metrics visualization & dashboards | 3000 |

### Architecture

```
[Target VMs] → Node Exporter (9100)
                      ↓
              Prometheus (9090) ← scrapes metrics
                      ↓
               Grafana (3000) ← queries & visualizes
```

---

## Variables — Adapt Before Use

```bash
MONITORING_IP=<monitoring-vm-ip>      # e.g. 192.168.x.x
GATEWAY=<your-gateway-ip>             # e.g. 192.168.x.1
SUBNET=<your-subnet>                  # e.g. 192.168.x.0/24
SSH_KEY_PATH=<path-to-your-key>       # e.g. ~/.ssh/monitoring-vm
USER=<your-username>
```

---

## Table of Contents

1. [VM Provisioning](#1-vm-provisioning)
2. [Ubuntu Server Installation](#2-ubuntu-server-installation)
3. [SSH Key Authentication](#3-ssh-key-authentication)
4. [SSH Hardening](#4-ssh-hardening)
5. [Firewall Configuration](#5-firewall-configuration)
6. [System Updates](#6-system-updates)
7. [Prometheus Installation](#7-prometheus-installation)
8. [Node Exporter Installation](#8-node-exporter-installation)
9. [Prometheus Configuration](#9-prometheus-configuration)
10. [Grafana Installation](#10-grafana-installation)
11. [Grafana Configuration](#11-grafana-configuration)
12. [Verification & Access](#12-verification--access)

---

## 1. VM Provisioning

### Recommended Specifications

| Parameter | Recommended Value | Notes |
|-----------|-------------------|-------|
| CPU | 1–2 cores | 1 core sufficient for small labs |
| RAM | 2–4 GB | 2GB minimum for Prometheus + Grafana |
| Disk | 20–50 GB | More if storing long-term metrics |
| Network | Static IP | Required for stable scraping targets |
| OS | Ubuntu Server 22.04 LTS | LTS recommended for stability |

### Proxmox-Specific Notes

- Set CPU type to **`host`** for best performance on single-node setups
- Use **VirtIO** network driver for best throughput
- Enable **iothread** on SCSI disk for better I/O performance
- Disable **High Availability** on single-node Proxmox

---

## 2. Ubuntu Server Installation

### Network Configuration (Static IP)

During installation, configure the network manually:

| Field | Value |
|-------|-------|
| IPv4 Method | Manual |
| Subnet | `$SUBNET` |
| Address | `$MONITORING_IP` |
| Gateway | `$GATEWAY` |
| DNS | `1.1.1.1, 8.8.8.8` |

### Storage Layout

The default LVM layout is recommended:

| Mount Point | Suggested Size | Type |
|-------------|----------------|------|
| `/boot` | 2 GB | ext4 |
| `/` | 14–30 GB | ext4 (LVM) |

### SSH Configuration

During installation:
-  **Install OpenSSH server** — enable
-  **Allow password authentication** — enable temporarily *(disabled after key setup)*

### Featured Snaps

**Skip all snaps.** Install Prometheus and Grafana manually via `apt` for version control and reliability.

---

## 3. SSH Key Authentication

### Generate SSH Key Pair

**Linux/macOS:**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/monitoring-vm -C "monitoring-vm"
```

**Windows PowerShell:**
```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\monitoring-vm" -C "monitoring-vm"
```

### Copy Public Key to VM

**Linux/macOS:**
```bash
ssh-copy-id -i ~/.ssh/monitoring-vm.pub $USER@$MONITORING_IP
```

**Windows PowerShell:**
```powershell
type $env:USERPROFILE\.ssh\monitoring-vm.pub | ssh $USER@$MONITORING_IP "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Verify Key Installation

```bash
cat ~/.ssh/authorized_keys
```

### Test Key-Based Login

**Linux/macOS:**
```bash
ssh -i ~/.ssh/monitoring-vm $USER@$MONITORING_IP
```

**Windows PowerShell:**
```powershell
ssh -i "$env:USERPROFILE\.ssh\monitoring-vm" $USER@$MONITORING_IP
```

---

## 4. SSH Hardening

> Run on the VM after confirming key login works.

### Apply Hardening

```bash
sudo sed -i 's/#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sudo sed -i 's/#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
```

### Verify Configuration

```bash
grep -E "PasswordAuthentication|PubkeyAuthentication|PermitRootLogin" /etc/ssh/sshd_config
```

Expected output:
```
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

### Restart SSH Service

```bash
sudo systemctl restart sshd
```


---

## 5. Firewall Configuration

### Set Default Policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### Allow Required Ports

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 9090/tcp    # Prometheus
sudo ufw allow 3000/tcp    # Grafana
```

>  In production, restrict Prometheus and Grafana access to trusted IPs only:
> ```bash
> sudo ufw allow from <trusted-ip> to any port 9090
> sudo ufw allow from <trusted-ip> to any port 3000
> ```

### Enable & Verify

```bash
sudo ufw enable
sudo ufw status
```

---

## 6. System Updates

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

---

## 7. Prometheus Installation

### Install

```bash
sudo apt install prometheus -y
```

### Enable & Start

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

### Verify

```bash
sudo systemctl status prometheus
```

Access UI: `http://$MONITORING_IP:9090`

---

## 8. Node Exporter Installation

Node Exporter exposes system-level metrics (CPU, RAM, disk, network) to Prometheus.

### Install

```bash
sudo apt install prometheus-node-exporter -y
```

### Enable & Start

```bash
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus-node-exporter
```

### Verify

```bash
sudo systemctl status prometheus-node-exporter
```

Metrics available at: `http://$MONITORING_IP:9100/metrics`

---

## 9. Prometheus Configuration

### Edit Prometheus Config

```bash
sudo nano /etc/prometheus/prometheus.yml
```

### Add Scrape Targets

```yaml
scrape_configs:

  # Prometheus self-monitoring (default)
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Local Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  # Example: scrape a remote target
  # - job_name: 'app-vm'
  #   static_configs:
  #     - targets: ['<app-vm-ip>:9100']
```

### Apply Changes

```bash
sudo systemctl restart prometheus
```

### Verify Targets

1. Go to `http://$MONITORING_IP:9090`
2. Navigate to **Status** → **Targets**
3. All jobs should show status **UP** ✅

---

## 10. Grafana Installation

### Add Repository (Modern Method)

```bash
wget -q -O /tmp/grafana.gpg https://packages.grafana.com/gpg.key
sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg /tmp/grafana.gpg
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list
```

### Install

```bash
sudo apt update && sudo apt install grafana -y
```


### Enable & Start

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

### Verify

```bash
sudo systemctl status grafana-server
```

> On first start, Grafana runs database migrations. Wait ~60 seconds before accessing the UI.

Access UI: `http://$MONITORING_IP:3000`

---

## 11. Grafana Configuration

### First Login

- URL: `http://$MONITORING_IP:3000`
- Default credentials: `admin` / `admin`
- **Change password immediately on first login**

### Add Prometheus Data Source

1. **Connections** → **Data sources** → **Add data source**
2. Select **Prometheus**
3. URL: `http://$MONITORING_IP:9090`
4. Authentication: None (internal network)
5. Click **Save & Test** → expect green 

### Import Pre-Built Dashboard

1. **Dashboards** → **New** → **Import**
2. Enter ID: `1860` *(Node Exporter Full — community dashboard)*
3. Click **Load**
4. Select **Prometheus** as data source
5. Click **Import**

>  Other useful dashboard IDs:
> - `3662` — Prometheus 2.0 Stats
> - `7362` — Node Exporter with Full Metrics
> - `11074` — Node Exporter for Prometheus

---

## 12. Verification & Access

### Services Checklist

| Service | Port | Check Command |
|---------|------|---------------|
| SSH | 22 | `sudo systemctl status sshd` |
| Prometheus | 9090 | `sudo systemctl status prometheus` |
| Node Exporter | 9100 | `sudo systemctl status prometheus-node-exporter` |
| Grafana | 3000 | `sudo systemctl status grafana-server` |

### All Services at Once

```bash
for svc in sshd prometheus prometheus-node-exporter grafana-server; do
  echo "=== $svc ===" && sudo systemctl is-active $svc
done
```

### Optional: SSH Config Alias

Add to `~/.ssh/config` on your local machine:

```
Host monitoring-vm
    HostName <MONITORING_IP>
    User <USER>
    IdentityFile <SSH_KEY_PATH>
    StrictHostKeyChecking no
```

Usage:
```bash
ssh monitoring-vm
```

---

## Security Checklist

- [ ] SSH password authentication disabled
- [ ] Root SSH login disabled
- [ ] SSH key authentication enabled and tested
- [ ] UFW firewall active with only required ports open
- [ ] System packages fully updated
- [ ] Grafana default password changed
- [ ] Prometheus not exposed to the internet
- [ ] Grafana not exposed to the internet
- [ ] Node Exporter port (9100) restricted to internal network only

---

## Troubleshooting

### Grafana Won't Load

```bash
sudo systemctl status grafana-server
sudo journalctl -u grafana-server -n 50
```

Wait 60 seconds after start — Grafana runs DB migrations on first boot.

### Prometheus Targets Show DOWN

```bash
# Check Node Exporter is running
sudo systemctl status prometheus-node-exporter

# Confirm port is open
ss -tlnp | grep 9100

# Check firewall
sudo ufw status
```

### SSH Access Denied After Hardening

```bash
# From Proxmox console (direct VM access)
sudo nano /etc/ssh/sshd_config
# Re-enable PasswordAuthentication temporarily
sudo systemctl restart sshd
```

### Permission Issues on `.ssh` Directory

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```
