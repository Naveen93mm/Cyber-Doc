# SIEM Dashboard & Threat Detection

## Project Summary

Designed and deployed a mini Security Operations Center (SOC) setup on AWS using the **default VPC**, with two EC2 instances — a **Monitored Server** (generating real security logs) and a **SIEM Server** running **Splunk**. Installed the Splunk Universal Forwarder on the Monitored Server to ship `auth.log` and system logs to Splunk. Configured **UFW** and **Fail2Ban** on the Monitored Server for real firewall/brute-force events, simulated an SSH brute-force attack against it, and built Splunk dashboards + alert rules to detect brute-force attempts, privilege escalation, and suspicious login activity in near real-time.

## AWS Services Used

1. IAM – Identity and Access Management
2. Default VPC (no custom VPC/subnet/IGW/route table setup needed)
3. Default Security Group (edited for lab access)
4. EC2 – Elastic Compute Cloud (Monitored Server + SIEM Server)
5. Key Pairs (SSH access)

## Security Tools Used

1. **Splunk** (Free/Trial) – SIEM, log ingestion, dashboards, alerting
2. **UFW** – Linux firewall (generates block/allow events)
3. **Fail2Ban** – Brute-force protection (bans IPs, logs to `auth.log`)
4. **journalctl** – System/service log inspection
5. **grep / tail** – Manual log searching, real-time monitoring
6. **auth.log** – Source of SSH login/auth events

> Alternative: **ELK Stack** (Elasticsearch + Logstash + Kibana) can replace Splunk if your student wants the open-source route — setup notes included below as an alternative path.

---

## Project Mind Map

```
├── IAM
│   ├── Create IAM User
│   ├── Attach Policies
│   └── Login to Console
│
├── Networking (Default VPC)
│   └── Use existing default VPC + default subnet
│       (no custom VPC/IGW/Route Table needed)
│
├── Security Group (Default SG - edited)
│   ├── SSH (22) → Your IP + Attacker's IP (to allow the simulated brute-force)
│   ├── TCP 8000 → Your IP only (Splunk Web UI)
│   └── TCP 9997 → Same SG (Forwarder → Indexer traffic)
│
├── EC2 Setup
│   ├── Instance 1: Monitored Server (Ubuntu 22.04)
│   │   ├── Install UFW → enable firewall
│   │   ├── Install Fail2Ban → protect SSH
│   │   └── Install Splunk Universal Forwarder → ship logs
│   │
│   └── Instance 2: SIEM Server (Ubuntu 22.04 / Amazon Linux)
│       └── Install Splunk Enterprise (Free trial)
│
├── Log Pipeline
│   ├── auth.log (SSH logins, failures) ──┐
│   ├── ufw.log (firewall allow/deny)      ├──> Forwarder (9997) ──> Splunk Indexer
│   └── fail2ban.log (banned IPs) ─────────┘
│
├── Attack Simulation
│   ├── Simulate SSH brute-force (Hydra, from a 3rd "Attacker" box or your laptop)
│   ├── Fail2Ban auto-bans the IP after threshold
│   └── All events land in Splunk in real time
│
├── Detection Phase (Splunk)
│   ├── Search: failed SSH logins by source IP
│   ├── Alert Rule: >5 failed logins in 1 min → Brute-force Alert
│   ├── Dashboard Panel: Top attacking IPs
│   ├── Dashboard Panel: Fail2Ban ban events over time
│   └── Dashboard Panel: Successful logins after failures (possible compromise)
│
└── Final Output
    ├── Splunk Dashboard (screenshots)
    ├── Alert configuration + triggered alert proof
    └── Incident Summary Report
```

---

## Project Setup Guide

### 1. IAM Setup

- Create IAM user
- Attach policy: `AmazonEC2FullAccess`
- Login using IAM credentials

### 2. Use Default VPC (no custom networking needed)

- Default VPC + default subnet la direct ah EC2 launch panniduvom
- Separate VPC/Subnet/IGW/Route table create pannanum nu illa

### 3. Security Group (edit the default one)

- SSH (22) → Your IP + (optionally) the IP you'll simulate the attack from
- **8000** (Splunk Web UI) → Your IP only
- **9997** (Splunk Forwarder → Indexer) → Same Security Group (self-reference)

### 4. Launch EC2 Instances

Launch **2 instances** in the **default VPC / default subnet**:

| Instance | OS | Purpose |
|---|---|---|
| SIEM Server | Ubuntu 22.04 LTS | Runs Splunk Enterprise (Indexer + Search Head) |
| Monitored Server | Ubuntu 22.04 LTS | Generates logs (UFW, Fail2Ban, SSH), runs Forwarder |

Enable public IP for both, attach the edited SG to both.

---

### 5. SIEM Server Setup — Install Splunk Enterprise (Ubuntu)

```bash
#!/bin/bash
sudo apt update -y

# Download Splunk (check splunk.com/download for latest .deb link — free trial, no card needed)
wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/9.2.1/linux/splunk-9.2.1-<build>-linux-2.6-amd64.deb"

sudo dpkg -i splunk.deb
sudo /opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd 'YourAdminPass123'

# Enable Splunk to receive data from Forwarders on port 9997
sudo /opt/splunk/bin/splunk enable listen 9997 -auth admin:YourAdminPass123

# Start Splunk on boot
sudo /opt/splunk/bin/splunk enable boot-start
```

Access Splunk Web UI: `http://<SIEM_SERVER_PUBLIC_IP>:8000` → login with `admin` / `YourAdminPass123`

---

### 6. Monitored Server Setup — UFW, Fail2Ban, Splunk Forwarder

```bash
#!/bin/bash
sudo apt update -y

# --- UFW Firewall ---
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw logging on
sudo ufw --force enable

# --- Fail2Ban ---
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Configure Fail2Ban for SSH (default jail is usually already enabled)
sudo bash -c 'cat > /etc/fail2ban/jail.local' << 'EOF'
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 600
findtime = 60
EOF
sudo systemctl restart fail2ban

# --- Splunk Universal Forwarder ---
wget -O splunkforwarder.deb "https://download.splunk.com/products/universalforwarder/releases/9.2.1/linux/splunkforwarder-9.2.1-<build>-linux-2.6-amd64.deb"
sudo dpkg -i splunkforwarder.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd 'YourFwdPass123'

# Point forwarder to the SIEM server's indexer
sudo /opt/splunkforwarder/bin/splunk add forward-server <SIEM_SERVER_PRIVATE_IP>:9997 -auth admin:YourFwdPass123

# Add log sources to monitor
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -auth admin:YourFwdPass123
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/ufw.log -auth admin:YourFwdPass123
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/fail2ban.log -auth admin:YourFwdPass123

sudo /opt/splunkforwarder/bin/splunk enable boot-start
```

---

### 7. Verify Logs Are Reaching Splunk

On SIEM Server web UI:

- **Settings → Forwarder Management** → confirm the Monitored Server appears as connected
- Search: `index=main sourcetype=linux_secure` (or `source="/var/log/auth.log"`) → should show live SSH log entries

### 8. Manual Log Checks (before jumping into Splunk)

On the Monitored Server, useful for cross-verification and quick troubleshooting:

```bash
# Real-time tail of auth log
sudo tail -f /var/log/auth.log

# Search failed SSH attempts
sudo grep "Failed password" /var/log/auth.log

# Check systemd service logs (e.g. fail2ban, ssh)
sudo journalctl -u fail2ban -f
sudo journalctl -u ssh -f

# Check currently banned IPs
sudo fail2ban-client status sshd
```

### 9. Simulate an Attack (SSH Brute-Force)

From a separate machine/EC2 instance (or your laptop, with your own IP allowed in SG):

```bash
# Example using Hydra against the Monitored Server (only against your own lab instance!)
hydra -l ubuntu -P /usr/share/wordlists/rockyou.txt ssh://<MONITORED_SERVER_PUBLIC_IP>
```

Fail2Ban will detect repeated failures and auto-ban the attacking IP after the configured threshold (5 attempts in this lab). All of this — failed attempts + the ban event — flows into Splunk via the Forwarder.

### 10. Build Splunk Detection Rules & Dashboard

**Search for failed logins:**
```
source="/var/log/auth.log" "Failed password"
| stats count by src_ip
| sort -count
```

**Create an Alert (Brute-Force Detection):**
- Splunk → **Search & Reporting** → run the search above with a time filter (Last 5 minutes)
- Save As → **Alert**
- Trigger condition: `count > 5`
- Action: Send email / log to Splunk notification

**Build a Dashboard** with panels:
- Top Attacking Source IPs (bar chart)
- Failed vs Successful Logins over Time (line chart)
- Fail2Ban Ban Events (table)
- UFW Blocked Connections (table)

### 11. (Alternative) ELK Stack Setup — if student prefers open-source

```bash
# On SIEM Server (Ubuntu)
sudo apt install elasticsearch logstash kibana -y

# Configure Logstash input to read auth.log, output to Elasticsearch
# /etc/logstash/conf.d/auth.conf
# input { file { path => "/var/log/auth.log" } }
# output { elasticsearch { hosts => ["localhost:9200"] } }

sudo systemctl enable elasticsearch logstash kibana
sudo systemctl start elasticsearch logstash kibana
```
Kibana Web UI: `http://<SIEM_SERVER_PUBLIC_IP>:5601` → build dashboards/visualizations same way as Splunk (Discover → Visualize → Dashboard).

### 12. Analysis & Report Generation

| Event Type | Detection Method | Response Taken |
|---|---|---|
| SSH Brute-force | Splunk alert (>5 failed logins/5 min) | Fail2Ban auto-ban confirmed in logs |
| Firewall blocks | UFW log → Splunk panel | Reviewed source IPs |
| Repeated failed → 1 success | Splunk correlation search | Flagged as possible compromise, investigate further |

### 13. Final Architecture Flow

```
Attacker → SSH Brute-force attempt → Monitored Server
Monitored Server → auth.log / ufw.log / fail2ban.log
Splunk Forwarder → ships logs (9997) → Splunk Indexer (SIEM Server)
Splunk → Search + Alert Rules → Dashboard
Fail2Ban → auto-bans attacker IP → logged event → visible in Splunk
```

### 14. Final Output

- Splunk dashboard with brute-force detection panels
- Triggered alert screenshot (proof the detection rule fired)
- `fail2ban-client status sshd` output showing the banned IP
- Final Incident Report: what happened, how it was detected, how it was mitigated

---

## Deliverable Checklist (for student submission)

- [ ] Screenshot: Splunk Forwarder Management showing connected Monitored Server
- [ ] Screenshot: Splunk search results for failed SSH logins
- [ ] Screenshot: Triggered brute-force alert
- [ ] Screenshot: Final Splunk Dashboard (all panels)
- [ ] `fail2ban-client status sshd` output
- [ ] Final Word/PDF Incident Report
