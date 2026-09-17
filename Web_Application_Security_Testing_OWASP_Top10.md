# Web Application Security Testing (OWASP Top 10)

## Project Summary

Designed and executed a web application penetration test on AWS using the **default VPC**, with two EC2 instances — an **Attacker instance** and a **Target instance** hosting an intentionally vulnerable web application (DVWA / OWASP Juice Shop). Performed directory & file enumeration with Gobuster, automated web server scanning with Nikto, manual request interception & testing with Burp Suite, and SQL Injection testing with SQLMap. Mapped every finding to the relevant **OWASP Top 10 (2021)** category and compiled a formal penetration test report with severity ratings and remediation steps.

## AWS Services Used

1. IAM – Identity and Access Management
2. Default VPC (no custom VPC/subnet/IGW/route table setup needed)
3. Default Security Group (edited for lab access)
4. EC2 – Elastic Compute Cloud (Attacker + Target instances)
5. Key Pairs (SSH access)

## Security Tools Used

1. **Gobuster** – Directory/file enumeration
2. **Nikto** – Web server security scanning
3. **Burp Suite** – Intercepting proxy, manual web app testing
4. **SQLMap** – SQL Injection testing
5. **DVWA / OWASP Juice Shop** – Vulnerable target application (Docker)

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
│   ├── SSH (22) → Your IP only
│   ├── HTTP (80) → Anywhere (to browse the vulnerable app)
│   └── All TCP → Attacker's SG (self-reference, for scanning)
│
├── EC2 Setup (both in default VPC)
│   ├── Instance 1: Attacker (Ubuntu 22.04)
│   │   ├── Install Gobuster
│   │   ├── Install Nikto
│   │   ├── Install Burp Suite (Community Edition)
│   │   └── Install SQLMap
│   │
│   └── Instance 2: Target (Amazon Linux / Ubuntu)
│       ├── Install Docker
│       ├── Deploy DVWA (SQLi, XSS, CSRF, weak auth)
│       └── Deploy OWASP Juice Shop (optional, broader coverage)
│
├── Testing Phase (mapped to OWASP Top 10)
│   ├── Gobuster → hidden dirs/files (A05: Security Misconfiguration)
│   ├── Nikto → outdated server, missing headers (A05, A06)
│   ├── Burp Suite → intercept requests, test auth/session (A01, A07)
│   ├── SQLMap → automate SQL Injection (A03: Injection)
│   └── Manual testing → XSS, CSRF, Broken Access Control
│
├── Analysis Phase
│   ├── Classify each finding under OWASP Top 10 category
│   ├── Assign severity (Critical/High/Medium/Low)
│   └── Capture proof-of-concept screenshots
│
└── Final Output
    ├── OWASP Top 10 mapped findings report
    ├── Remediation Recommendations
    └── PoC Screenshots (Burp requests, SQLMap dump, etc.)
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

- SSH (22) → Your IP only
- HTTP (80) → Anywhere (0.0.0.0/0) *(to access DVWA/Juice Shop in browser)*
- All TCP (0–65535) → Same Security Group *(self-reference — Attacker instance can freely scan Target)*

### 4. Launch EC2 Instances

Launch **2 instances** in the **default VPC / default subnet**:

| Instance | OS | Purpose |
|---|---|---|
| Attacker | Ubuntu 22.04 LTS | Runs Gobuster, Nikto, Burp Suite, SQLMap |
| Target | Amazon Linux 2023 **or** Ubuntu 22.04 | Hosts DVWA / Juice Shop |

Enable public IP for both, attach the edited SG to both instances.

---

### 5. Target Instance Setup (Amazon Linux OR Ubuntu)

**If Amazon Linux:**
```bash
#!/bin/bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user

# DVWA on port 80
sudo docker run -d -p 80:80 vulnerables/web-dvwa

# OWASP Juice Shop on port 3000 (optional, extra practice)
sudo docker run -d -p 3000:3000 bkimminich/juice-shop
```

**If Ubuntu:**
```bash
#!/bin/bash
sudo apt update -y
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu

# DVWA on port 80
sudo docker run -d -p 80:80 vulnerables/web-dvwa

# OWASP Juice Shop on port 3000 (optional, extra practice)
sudo docker run -d -p 3000:3000 bkimminich/juice-shop
```

> First time DVWA open pannumbodhu, "Create / Reset Database" button click pannanum — appo than login page varum (default login: `admin` / `password`). Security level ah **Low** ah vachu practice pannunga first.

---

### 6. Attacker Instance Setup (Ubuntu 22.04)

```bash
#!/bin/bash
sudo apt update -y
sudo apt upgrade -y

# Install Gobuster
sudo apt install gobuster -y

# Install Nikto
sudo apt install nikto -y

# Install SQLMap
sudo apt install sqlmap -y

# Install Burp Suite Community Edition
sudo apt install default-jre -y
wget "https://portswigger.net/burp/releases/download?product=community&type=Linux" -O burpsuite_community.sh
chmod +x burpsuite_community.sh
sudo ./burpsuite_community.sh -q
```

> Burp Suite GUI thevai — so Attacker instance ku **desktop GUI** venum (either launch Ubuntu with a Desktop AMI, or use X11 forwarding via `ssh -X`, or simplest: install Burp locally on your laptop and just proxy traffic to the Target's public IP — no need to install it on the EC2 instance at all).

---

### 7. Gobuster – Directory/File Enumeration

```bash
gobuster dir -u http://<TARGET_PUBLIC_IP> -w /usr/share/wordlists/dirb/common.txt -o gobuster_report.txt
```

Finds hidden admin panels, backup files, config files not linked anywhere → **A05: Security Misconfiguration**.

### 8. Nikto – Web Server Scan

```bash
nikto -h http://<TARGET_PUBLIC_IP> -o nikto_report.txt
```

Flags outdated server versions, missing security headers, default files → **A05, A06: Vulnerable & Outdated Components**.

### 9. Burp Suite – Manual Testing

- Set browser proxy to Burp (127.0.0.1:8080)
- Browse the DVWA app through Burp → capture requests in **Proxy → HTTP History**
- Send interesting requests to **Repeater** → manually tamper params to test:
  - Broken Access Control (A01) — change `user_id` in URL, see if you access another user's data
  - Session/Auth issues (A07) — check cookie flags (HttpOnly, Secure)
  - Stored/Reflected XSS — inject `<script>alert(1)</script>` in comment/search fields

### 10. SQLMap – SQL Injection Testing

```bash
# On DVWA's SQL Injection page, capture the vulnerable request in Burp, save as request.txt, then:
sqlmap -r request.txt --batch --dbs

# Or directly against a known vulnerable parameter:
sqlmap -u "http://<TARGET_PUBLIC_IP>/vulnerabilities/sqli/?id=1&Submit=Submit#" \
  --cookie="PHPSESSID=<your_session_id>; security=low" \
  --batch --dbs
```

Confirms SQL Injection → **A03: Injection**. Dump table names, then columns, then data to prove impact.

### 11. Analysis & Report Generation (OWASP Top 10 Mapping)

| Finding | OWASP Category | Severity | Tool Used |
|---|---|---|---|
| SQL Injection in login/search | A03: Injection | Critical | SQLMap, Burp |
| Reflected/Stored XSS | A03: Injection | High | Burp (manual) |
| Broken Access Control (IDOR) | A01: Broken Access Control | High | Burp |
| Outdated Apache/PHP version | A06: Vulnerable Components | Medium | Nikto |
| Exposed admin/backup files | A05: Security Misconfiguration | Medium | Gobuster |
| Missing security headers | A05: Security Misconfiguration | Low | Nikto |
| Weak session cookie flags | A07: Identification & Auth Failures | Medium | Burp |

### 12. Final Architecture Flow

```
Attacker (Burp/Gobuster/Nikto/SQLMap) --HTTP--> Target (DVWA/Juice Shop)
Attacker --SSH--> Your Laptop (report access)
Findings --> Mapped to OWASP Top 10 --> Severity rating --> Remediation Doc
```

### 13. Final Output

- Gobuster enumeration report
- Nikto scan report
- Burp Suite proxy history / repeater screenshots
- SQLMap injection & data dump output
- Final consolidated **Web App Penetration Test Report** mapped to OWASP Top 10 with remediation steps

---

## Deliverable Checklist (for student submission)

- [ ] `gobuster_report.txt`
- [ ] `nikto_report.txt`
- [ ] Burp Suite screenshots (Proxy history + Repeater tests)
- [ ] SQLMap output (databases/tables dumped)
- [ ] Final Word/PDF report: Findings mapped to OWASP Top 10 + Severity + Remediation
