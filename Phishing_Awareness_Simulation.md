# Phishing Awareness Simulation

## Project Summary

Designed and executed an authorized phishing awareness simulation on AWS using the **default VPC**, with a single EC2 instance hosting **GoPhish** — an open-source phishing simulation framework. Created a realistic phishing campaign (landing page + email template), configured an SMTP sending profile, launched the campaign against a test group of "employee" email addresses, and tracked opens, clicks, and submitted-credential events in real time. Compiled a report analyzing employee susceptibility and recommending security awareness training.

> ⚠️ **Important**: Phishing simulations must only ever be run against people who have given consent (e.g. test/dummy email accounts you own, or an organization's employees with prior written authorization). Never target real people without authorization — that would be an actual phishing attack.

## AWS Services Used

1. IAM – Identity and Access Management
2. Default VPC (no custom VPC/subnet/IGW/route table setup needed)
3. Default Security Group (edited for lab access)
4. EC2 – Elastic Compute Cloud (GoPhish server)
5. Key Pairs (SSH access)

## Security Tools Used

1. **GoPhish** – Phishing campaign creation, landing pages, tracking dashboard
2. **SMTP relay** (e.g. Gmail App Password / Mailtrap / SES sandbox) – for sending simulated phishing emails
3. Test email accounts (your own, not real employees) as simulated targets

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
│   ├── TCP 3333 → Your IP only (GoPhish Admin UI)
│   └── TCP 80/8080 → Anywhere (Phishing landing page, so test users can click it)
│
├── EC2 Setup
│   └── Instance: GoPhish Server (Ubuntu 22.04)
│       ├── Install Go
│       ├── Download & Build GoPhish
│       └── Run GoPhish (admin UI on :3333, phishing server on :8080)
│
├── GoPhish Configuration
│   ├── Sending Profile (SMTP credentials)
│   ├── Landing Page (clone a realistic login page)
│   ├── Email Template (urgency-based phishing email)
│   ├── Target Group (test email accounts)
│   └── Launch Campaign
│
├── Simulation Phase
│   ├── Emails sent to target group
│   ├── Track: Email Opened
│   ├── Track: Link Clicked
│   └── Track: Credentials Submitted (on fake landing page)
│
├── Analysis Phase
│   ├── Click-through rate
│   ├── Credential submission rate
│   └── Time-to-click (how fast people fell for it)
│
└── Final Output
    ├── Campaign Results Dashboard (screenshots)
    ├── Susceptibility Report
    └── Awareness Training Recommendations
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
- **3333** (GoPhish Admin Dashboard) → **Your IP only** *(never expose this publicly — it controls the whole campaign)*
- **8080** (GoPhish phishing landing page) → Anywhere (0.0.0.0/0) *(test users need to reach this when they click the link)*

### 4. Launch EC2 Instance

Launch **1 instance** in the **default VPC / default subnet**:

| Instance | OS | Purpose |
|---|---|---|
| GoPhish Server | Ubuntu 22.04 LTS | Runs GoPhish campaign + admin dashboard |

Enable public IP, attach the edited SG.

---

### 5. GoPhish Server Setup (Ubuntu 22.04)

```bash
#!/bin/bash
sudo apt update -y
sudo apt upgrade -y

# Install unzip and dependencies
sudo apt install unzip -y

# Download latest GoPhish release (check github.com/gophish/gophish/releases for latest version)
wget https://github.com/gophish/gophish/releases/download/v0.12.1/gophish-v0.12.1-linux-64bit.zip

unzip gophish-v0.12.1-linux-64bit.zip -d gophish
cd gophish
chmod +x gophish

# Run GoPhish (leave running in background)
sudo ./gophish &
```

> First run panna, terminal la **admin default password** print aagum (one-time, save it!). Admin UI: `https://<EC2_PUBLIC_IP>:3333` — self-signed cert warning varum, "Proceed anyway" click pannunga.

---

### 6. Configure Sending Profile (SMTP)

GoPhish Admin UI → **Sending Profiles** → New Profile:

- Name: `Test SMTP`
- From: `IT Support <support@yourcompany-test.com>`
- Host: your SMTP relay (options for students):
  - **Gmail App Password**: `smtp.gmail.com:587` (create an App Password, not your real Gmail password)
  - **Mailtrap.io** (free, sandbox inbox — recommended for a training lab, emails never actually leave the sandbox)
- Username / Password: your SMTP credentials
- Click **Send Test Email** to confirm it works

### 7. Create Landing Page

GoPhish Admin UI → **Landing Pages** → New Page:

- Import a real login page by URL (e.g. clone your company's own test login page) or use **Import Site** feature
- Enable **Capture Submitted Data** and **Capture Passwords** (for the simulation to measure credential-entry rate)
- Add a **Redirect URL** to an internal "you just got phished" awareness page after submission — this is the teaching moment

### 8. Create Email Template

GoPhish Admin UI → **Email Templates** → New Template:

- Subject: e.g. `Urgent: Verify Your Account Access`
- Body: realistic-looking urgency message with a link → `{{.URL}}` (GoPhish auto-inserts the tracked link)
- Enable **Add Tracking Image** (invisible pixel to track email opens)

### 9. Create Target Group

GoPhish Admin UI → **Users & Groups** → New Group:

- Add **only test/dummy email addresses** you own or have explicit permission to target
- Format: First Name, Last Name, Email, Position

### 10. Launch Campaign

GoPhish Admin UI → **Campaigns** → New Campaign:

- Select Name, Email Template, Landing Page, URL (`http://<EC2_PUBLIC_IP>:8080`), Sending Profile, Target Group
- Set **Launch Date**
- Click **Launch Campaign**

### 11. Monitor Results (Real-Time Dashboard)

GoPhish Admin UI → **Campaign Results** shows a live timeline:

- Email Sent
- Email Opened
- Clicked Link
- Submitted Data (credentials entered)

### 12. Analysis & Report Generation

| Metric | Description |
|---|---|
| Open Rate | % of targets who opened the email |
| Click Rate | % who clicked the phishing link |
| Submission Rate | % who entered credentials on the fake page |
| Time to First Click | How fast the first person fell for it |
| Riskiest Segment | Which target group was most susceptible (if segmented) |

### 13. Final Architecture Flow

```
GoPhish Server → sends email via SMTP relay → Target's inbox
Target opens email → tracking pixel fires → GoPhish logs "Opened"
Target clicks link → redirected to Landing Page (:8080) → GoPhish logs "Clicked"
Target submits fake login → GoPhish logs "Submitted Data" → redirected to awareness page
GoPhish Dashboard → aggregates all events → Final Report
```

### 14. Final Output

- Campaign results dashboard (screenshots: open/click/submit stats)
- Sample phishing email + landing page used
- Susceptibility analysis report
- Recommendations: security awareness training topics based on what people fell for

---

## Deliverable Checklist (for student submission)

- [ ] Screenshot of GoPhish campaign creation (template, landing page, sending profile)
- [ ] Screenshot of live Campaign Results dashboard
- [ ] Exported campaign results (CSV, from GoPhish)
- [ ] Final report: Open/Click/Submit rates + Awareness Training Recommendations
