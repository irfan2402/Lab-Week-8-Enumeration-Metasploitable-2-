# Lab-Week-8 & 9-Enumeration-Metasploitable-2

Author: Muhammad Irfan Danial Bin Mohd Nadzir (L01-B04)

---

##  Lab Environment Setup

### Victim Machine — Metasploitable 2

Confirmed victim IP via `ifconfig` on Metasploitable 2:

```
eth0  inet addr: 192.168.56.101   Bcast: 192.168.56.255   Mask: 255.255.255.0
      inet6 addr: fe80::a00:27ff:fe0:92a/64  (Oracle VirtualBox)
```

<img width="1912" height="918" alt="image" src="https://github.com/user-attachments/assets/31785072-95d2-4fd3-9dfe-119b73c2c722" />

### Attacker Machine — Windows 10

Confirmed attacker IP via `ipconfig`, then verified connectivity:

```
IPv4 Address: 192.168.56.107
Subnet Mask:  255.255.255.0

ping 192.168.56.101
Reply from 192.168.56.101: bytes=32 time=1ms TTL=64   
```

<img width="980" height="511" alt="image" src="https://github.com/user-attachments/assets/c976bdbb-2de6-49c5-b047-9acab464e949" />

| Component     | Detail                                  |
|-------------- |------------------------------------------|
| Attacker OS   | Windows 10 (Version 22H2)               |
| Attacker IP   | 192.168.56.107                         |
| Victim        | Metasploitable 2 (Ubuntu 8.04)          |
| Victim IP     | 192.168.56.101                          |
| Network       | VirtualBox Host-Only Adapter            |

---

## 🔴 Section A — Basic Enumeration

### Challenge 1 — NetBIOS Enumeration

**Tool:** `nbtstat` (Windows CMD)  
**Command:**
```
nbtstat -A 192.168.56.101
```

**Command Purpose:**  
`nbtstat -A` queries the NetBIOS name table of a remote machine using its IP address. This reveals the hostname, workgroup/domain, and active services (e.g., file sharing, messenger) without authentication.

**Output:** 
<img width="977" height="326" alt="image" src="https://github.com/user-attachments/assets/2a3add5e-bc75-4599-99e6-adf583962fdd" />

**Analysis:**
- `<00> UNIQUE` → Hostname is `METASPLOITABLE`
- `<20> UNIQUE` → File Server (SMB) service is **active**
- `<03> UNIQUE` → Messenger service running
- `WORKGROUP` → workgroup name (default)

**Security implication:**  
SMB presence indicates potential for share enumeration and relay attacks. The <03> entry could be used for internal phishing.

---
### Challenge 2 — Fast Nmap Scan

**Tool:** Zenmap (Windows)  
**Command:**
```
nmap -T4 -F 192.168.56.101
```

**Command Purpose:**  
`-T4` speeds up the scan with aggressive timing; `-F` scans only the top 100 most common ports (fast scan). This quickly identifies open doors before deeper enumeration.

**Output:**
<img width="1155" height="812" alt="image" src="https://github.com/user-attachments/assets/1f9aeb11-4fb7-4965-9583-c318e1bb3e32" />

**Analysis:**  
18 open ports found — extremely large attack surface. Notable services: FTP, Telnet (cleartext), NFS, MySQL, VNC, and X11 all exposed.

**Security implication:**  
Each open port is a potential entry point. Attackers can target outdated or misconfigured services for exploitation.

---

### Challenge 5 — TTL OS Fingerprinting

**Tool:** `ping` (Windows CMD)  
**Command:**
```
ping 192.168.56.101
```

**Command Purpose:**  
Sends ICMP echo requests; the `TTL` (Time To Live) value in the reply helps estimate the remote operating system based on initial TTL defaults (Linux=64, Windows=128, etc.).

**Output:**
<img width="972" height="275" alt="image" src="https://github.com/user-attachments/assets/01b4cac8-50e7-4c5e-85e8-13da55467b8e" />


**TTL Reference:**

| TTL | OS Guess |
|-----|----------|
| **64** | **Linux / Unix ✅** |
| 128 | Windows |
| 255 | Cisco / BSD |

**Analysis:**  
TTL=64 confirms victim is running **Linux**. Consistent with Metasploitable 2 (Ubuntu 8.04).

**Security implication:**  
Remote OS discovery enables targeted exploits specific to Linux kernel versions.

---

### Challenge 9 — FTP Banner Grabbing

**Tool:** Windows Telnet Client  
**Command:**
```
telnet 192.168.56.101 21
```

**Command Purpose:** 
Connects to the FTP service on port 21 and reads the welcome banner, which often discloses the software name and version. This is a non‑intrusive way to identify vulnerable FTP servers.

**Output:**
<img width="977" height="76" alt="image" src="https://github.com/user-attachments/assets/a0866561-f835-4d28-a063-207b08ee3a4c" />


**Analysis:**   
FTP service version is `vsFTPd 2.3.4`, which has a known backdoor vulnerability (CVE‑2011‑2523) allowing remote root access.

**Security implication:**  
This version should be upgraded immediately. The backdoor can be triggered by a username containing :) in some implementations.

---

### Challenge 10 — Anonymous FTP Login

**Tool:** `ftp` (Windows CMD)  
**Command:**
```
ftp 192.168.56.101
```

**Command Purpose:**  
Attempts to log into the FTP server with the username `anonymous` (no password). This checks whether the server allows unauthenticated file access – a common misconfiguration.

**Session:**
<img width="981" height="388" alt="image" src="https://github.com/user-attachments/assets/4fd741c5-cc7b-473c-9bfe-9789ddc10531" />


**Analysis:**  
Anonymous FTP login is **allowed** — no credentials required. Root directory is accessible and browsable.

**Security implication:**  
Even empty, anonymous access is a misconfiguration. It could be used to upload files if write permissions are accidentally enabled, or to fingerprint the system.

---

## 🟠 Section B — Intermediate Enumeration

### Challenge 11 — SMB NSE Enumeration

**Tool:** Zenmap (Windows)

#### Part 1: OS Discovery
**Command:**
```
nmap --script smb-os-discovery -p445 192.168.56.101
```

**Command Purpose:**  
Uses Nmap’s SMB script to query the SMB service on port 445 and retrieve the operating system, Samba version, and workgroup name.


**Output:**
<img width="981" height="360" alt="image" src="https://github.com/user-attachments/assets/0ede9258-e4d3-415b-9883-db1d81a5c16b" />

Part 2: User Enumeration

Command:

```
nmap --script smb-enum-users -p445 192.168.56.101
```

Command Purpose:

Enumerates local users on the remote system by querying the SMB user list (RID cycling). This can reveal valid usernames for other attacks.

<img width="979" height="440" alt="image" src="https://github.com/user-attachments/assets/cb517722-6748-48a0-8be3-870fa61ca2ba" />
<img width="978" height="512" alt="image" src="https://github.com/user-attachments/assets/c639d61d-9fa3-49b4-8031-1a810cd1e6a7" />
<img width="975" height="510" alt="image" src="https://github.com/user-attachments/assets/e74e5a98-d67b-41e9-a3ce-57498397e7d7" />

**Users discovered (partial list):**
```
METASPLOITABLE\backup    (RID: 1068)  — Account disabled
METASPLOITABLE\bin       (RID: 1004)  — Account disabled
METASPLOITABLE\daemon    (RID: 1002)  — Account disabled
METASPLOITABLE\ftp       (RID: 1214)  — Account disabled
METASPLOITABLE\msfadmin  (RID: 3000)  — Normal user account ⚠️
METASPLOITABLE\mysql     (RID: 1218)  — Account disabled
METASPLOITABLE\postgres  (RID: 1216)  — Account disabled
METASPLOITABLE\root      (RID: 1000)  — Account disabled
METASPLOITABLE\service   (RID: 3004)  — Account disabled
METASPLOITABLE\user      (RID: 3002)  — Normal user account ⚠️
METASPLOITABLE\www-data  (RID: 1066)  — Account disabled
... (30+ users total)
```

**Analysis:**  
Samba 3.0.20 is vulnerable to **CVE-2007-2447** (usermap_script command injection). Full user list is leaked including service accounts. `msfadmin` and `user` are active non-disabled accounts.

**Security implication:**  
Attackers can use the active usernames for brute‑force attacks against SMB, SSH, or FTP. The disabled accounts still leak system service information.

---

### Challenge 14 — SNMP NSE

**Tool:** Zenmap (Windows)  
**Command:**
```
nmap -sU -p161 --script snmp-sysdescr,snmp-processes 192.168.56.101
```

**Command Purpose:**  
Performs a UDP scan on port 161 (SNMP) and runs scripts that attempt to read system description and running processes using community string `public`. This tests for SNMP misconfigurations.

**Output:**
<img width="978" height="251" alt="image" src="https://github.com/user-attachments/assets/a0f4bd98-93d4-402c-bf23-f42a831c1126" />


**Analysis:**  
SNMP port 161/UDP returned `closed` on this scan — the service was not responding at the time of the test. This result shows the scanner correctly identified the port state. No information could be gathered because SNMP is simply not running.

**Security implication:**  
The attack surface is reduced, but we lose a potential source of enumeration data. This also indicates the system is not configured for SNMP monitoring.

---

### Challenge 15 — DNS Zone Transfer

**Tool:** `nslookup` (Windows CMD)  
**Commands:**
```
nslookup
> server 192.168.56.101
> ls -d example.com
```

**Command Purpose:**  
Attempts a DNS zone transfer (AXFR) – a legitimate DNS operation that, if misconfigured, can leak all DNS records of a domain. The `ls -d` command in `nslookup` requests a zone listing.

**Output:**
<img width="977" height="340" alt="image" src="https://github.com/user-attachments/assets/21986a5e-a34a-4903-b0c0-9900b9d7b755" />


**Analysis:**  
Zone transfer was `refused` — the DNS server (ISC BIND 9.4.2 as found in Challenge 16) has zone transfer security enabled for `example.com`. This is the **expected secure behaviour**.

**Security implication:**  
Proper zone transfer restrictions prevent DNS reconnaissance. No records were leaked.

---

### Challenge 16 — Version Detection

**Tool:** Zenmap (Windows)  
**Command:**
```
nmap -sV 192.168.56.101
```

**Command Purpose:**  
Probes open ports to determine the exact version of each service (e.g., vsFTPd 2.3.4). This information is critical for finding known vulnerabilities.

**Output:**
<img width="1154" height="816" alt="image" src="https://github.com/user-attachments/assets/1724d50e-084d-45ff-b86e-7ba7f28ede88" />


**Analysis:**  
Every service is severely outdated. Port 1524 `(bindshell)` is a Metasploitable root shell listening openly — direct root access with no exploit needed.

**Security implication:**  
This target is critically vulnerable. Immediate actions would include patching or replacing all services, removing backdoors, and implementing network segmentation.

---

### Challenge 17 — OS Detection

**Tool:** Zenmap (Windows)  
**Command:**
```
nmap -O 192.168.56.101
```

**Command Purpose:**  
Uses TCP/IP stack fingerprinting (e.g., TCP options, window size, TTL) to guess the remote operating system. This helps tailor further attacks.

**Output:**
<img width="1149" height="824" alt="image" src="https://github.com/user-attachments/assets/68473a28-dbd8-4297-bf01-fa9813d84e41" />


**Analysis:**  
Nmap fingerprinting confirms `Linux kernel 2.6.x`, consistent with Ubuntu 8.04 LTS (Metasploitable 2). Network distance of 1 hop confirms direct Layer 2 connectivity.

**Security implication:**  
Knowing the exact kernel version helps attackers tailor privilege escalation exploits.

---

### Challenge 20 — DNSSEC Enumeration

**Tool:** Zenmap (Windows)  
**Command:**
```
nmap -p 53 --script dns-nsec-enum 192.168.56.111
```

**Command Purpose:**  
Attempts to enumerate DNS NSEC records, which can expose additional domain names if DNSSEC is misconfigured. The script requires a domain argument for full functionality.

**Output:**
<img width="1152" height="495" alt="image" src="https://github.com/user-attachments/assets/2f768981-b369-4e83-82e3-cfe37f7d86a5" />

**Analysis:**  
Port 53 is open (DNS active), but NSEC enumeration requires a domain name argument to work. The script ran correctly but needs `--script-args dns-nsec-enum.domains=localdomain` for a full result. DNS is served by ISC BIND 9.4.2, which does not support DNSSEC by default, so NSEC enumeration is not applicable.

**Security implication:**  
Without a domain name, DNSSEC leakage is unlikely. This does not indicate vulnerability.

---


## 🔵 Section C — Advanced Enumeration

###  Challenge 22 — Correlation Table

**Purpose:**  
Compile data from several enumerators to create a consolidated user, service and possible vulnerability profile of the target. Data correlation can be used to discover patterns (such as a username that is employed on SMB may be employed on SSH) and prioritize attack paths.

**Sources used:**  
- SMB enumeration (Challenge 11)  
- FTP banner & anonymous login (Challenges 9 & 10)  
- NetBIOS enumeration (Challenge 1)  

**Correlation Table (minimum 3 sources):**

| Source                     | Username / Service          | Group / Role / Context               | Notes                                                                 |
|----------------------------|-----------------------------|--------------------------------------|-----------------------------------------------------------------------|
| SMB (Ch11)                 | `msfadmin`                  | Normal user account (RID 3000)       | Active SMB user – likely also valid for SSH, FTP, or VNC             |
| FTP (Ch9, Ch10)            | `anonymous` + `vsFTPd 2.3.4`| Anonymous access + vulnerable version | Backdoored FTP allows remote root without credentials                |
| NetBIOS (Ch1)              | `METASPLOITABLE`            | Hostname                             | Confirms target identity and workgroup                               |

**Analysis:**  
The correlation has revealed that the target is running a vulnerable version of vsFTPd (2.3.4) which not only has a backdoor but also allows anonymous login. `msfadmin` is a good target for other services to be brute‑forced (SSH, VNC). The hostname `METASPLOITABLE` is the NetBIOS name of the computer, for consistent system identification. In combination, these results show that the system is highly susceptible to public exploits.

---

### Challenge 29 — SMTP Enumeration via Nmap

**Tool:** Zenmap (Windows)

#### Part 1: smtp-enum-users
**Command:**
```
nmap -p 25 --script smtp-enum-users 192.168.56.101
```

**Command Purpose:**  
Attempts to enumerate valid email users using SMTP commands (VRFY, EXPN, RCPT). A vulnerable server will confirm which usernames exist.

**Output:**
<img width="1159" height="822" alt="image" src="https://github.com/user-attachments/assets/342262d8-5c9d-45a7-b27d-d4832d7af47a" />


**Analysis:**  
SMTP port 25 is open (Postfix). The `smtp-enum-users` script ran but RCPT method returned an unhandled code — meaning the server responded in a non-standard way. VRFY/EXPN manual testing via Netcat would yield better results.

#### Part 2: smtp-open-relay
**Command:**
```
nmap -p 25 --script smtp-open-relay 192.168.56.101
```

**Command Purpose:**  
Tests whether the SMTP server allows relaying mail from external domains (an open relay). Open relays are serious security issues as they can be used to send spam.

**Output:**
<img width="1151" height="818" alt="image" src="https://github.com/user-attachments/assets/b5a6af5c-da50-40e5-bfb2-837bb7096234" />

**Analysis:**  
SMTP relay is **not open** — the Postfix server correctly rejects unauthorized relaying. This is a properly configured aspect of the otherwise vulnerable machine.

**Security implication:**  
While other services on the target are severely vulnerable, the mail service follows best practices, preventing spam relay and user enumeration.

---

##  Key Findings Summary

| Service | Version | Risk | CVE / Note |
|---------|---------|------|------------|
| FTP | vsFTPd 2.3.4 | 🔴 Critical | CVE-2011-2523 — backdoor RCE |
| SSH | OpenSSH 4.7p1 | 🟠 High | Multiple known CVEs |
| SMB | Samba 3.0.20 | 🔴 Critical | CVE-2007-2447 — RCE via username |
| HTTP | Apache 2.2.8 | 🟠 High | CVE-2007-5000, CVE-2007-6388 |
| MySQL | 5.0.51a | 🟠 High | Default credentials, no bind restriction |
| PostgreSQL | 8.3.0–8.3.7 | 🟠 High | trust auth enabled |
| Bindshell | port 1524 | 🔴 Critical | Direct root shell — no exploit needed |
| NFS | 2-4 | 🟠 High | Service present (port 2049); actual exports not enumerated but may be misconfigured |
| VNC | protocol 3.3 | 🟠 High | Weak auth, old protocol |
| UnrealIRCd | — | 🔴 Critical | CVE-2010-2075 — backdoor |

---

##  Tools Used

| Tool | Platform | Used For |
|------|----------|----------|
| Zenmap (Nmap 7.99) | Windows | Port scans, NSE scripts (SMB, SNMP, SMTP, DNS, DNSSEC, IPv6), version/OS detection |
| Windows CMD | Windows | `nbtstat`, `ping`, `ftp`, `nslookup` |
| Windows Telnet Client | Windows | FTP banner grabbing (port 21) |

---





