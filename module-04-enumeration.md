# 📡 CEH v13 — Module 04: Enumeration

> 📌 Personal study notes for CEH v13 exam preparation. Covers enumeration techniques, protocols, tools, and practical commands used to extract detailed information from target systems.

---

## 📚 Table of Contents

1. [Overview](#overview)
2. [What is Enumeration?](#what-is-enumeration)
3. [NetBIOS Enumeration](#netbios-enumeration)
4. [SNMP Enumeration](#snmp-enumeration)
5. [LDAP Enumeration](#ldap-enumeration)
6. [NTP Enumeration](#ntp-enumeration)
7. [SMTP Enumeration](#smtp-enumeration)
8. [DNS Enumeration](#dns-enumeration)
9. [SMB Enumeration](#smb-enumeration)
10. [FTP Enumeration](#ftp-enumeration)
11. [RPC Enumeration](#rpc-enumeration)
12. [Linux/Unix Enumeration](#linuxunix-enumeration)
13. [Enumeration Tools Summary](#enumeration-tools-summary)
14. [CEH Exam Tips](#ceh-exam-tips)

---

## 🧭 Overview

### Hacking Phases Recap
```
Phase 1 → Footprinting & Reconnaissance  (passive info gathering)
Phase 2 → Scanning Networks              (active — find live hosts, open ports)
Phase 3 → Enumeration                    (active — extract detailed system info)
Phase 4 → Vulnerability Analysis
Phase 5 → System Hacking
```

---

## ❓ What is Enumeration?

**Enumeration** is the process of extracting detailed information from a target system using **authenticated or unauthenticated connections** to services.

Unlike scanning (which finds what's open), enumeration **digs deeper** to extract:

| What We Extract | Example |
|---|---|
| **Usernames & Groups** | Administrator, Guest, Domain Users |
| **Network Shares** | \\server\C$, \\server\IPC$ |
| **Routing Tables** | Internal network structure |
| **Service Information** | Running services, versions |
| **Machine Names** | Hostnames, NetBIOS names |
| **Application Info** | Installed software, configs |
| **Audit & Policy Info** | Password policy, lockout threshold |

> 💡 **Key Difference:** Scanning = knock on doors. Enumeration = peek inside the open ones.

---

## 🖥️ NetBIOS Enumeration

**NetBIOS** (Network Basic Input/Output System) allows computers on a LAN to share files and printers. Runs on **port 137, 138 (UDP) and 139 (TCP)**.

### NetBIOS Name Table Codes

| Code | Type | Meaning |
|---|---|---|
| `<00>` | UNIQUE | Workstation / Computer name |
| `<00>` | GROUP | Domain name |
| `<03>` | UNIQUE | Messenger service (username) |
| `<20>` | UNIQUE | File & Printer Sharing service |
| `<1D>` | GROUP | Master Browser |
| `<1E>` | GROUP | Browser Service Elections |

> 💡 **Exam Tip:** `<20>` means File & Printer Sharing is enabled — high value target!

### Commands

```bash
# Windows — NetBIOS name table of remote host
nbtstat -A 192.168.1.1

# Windows — NetBIOS name table of local machine
nbtstat -n

# Windows — NetBIOS name cache
nbtstat -c

# Linux — enumerate NetBIOS names
nmblookup -A 192.168.1.1

# Net view — list shares on remote system
net view \\192.168.1.1

# Net use — map a network share
net use Z: \\192.168.1.1\C$

# List users via NetBIOS
net user /domain
net localgroup administrators
```

### Tool: **Hyena** / **SoftPerfect Network Scanner**
- GUI tools for NetBIOS enumeration on Windows networks
- Can enumerate users, groups, shares, policies from NetBIOS

---

## 📊 SNMP Enumeration

**SNMP** (Simple Network Management Protocol) is used to manage network devices. Runs on **UDP port 161 (queries) and 162 (traps)**.

### Key Concepts

| Term | Meaning |
|---|---|
| **MIB** | Management Information Base — database of network object info |
| **OID** | Object Identifier — unique ID for each MIB object |
| **Community String** | Password for SNMP access (`public` = read, `private` = write) |
| **SNMPv1/v2c** | Sends community strings in **plaintext** — vulnerable! |
| **SNMPv3** | Adds encryption + authentication — secure |

### Default Community Strings (Exam Favorite!)
- `public` — read-only access
- `private` — read-write access

> 💡 **Exam Tip:** If community string is `public`, attacker can read device info. If `private`, they can modify config!

### Commands

```bash
# snmpwalk — enumerate entire MIB tree
snmpwalk -v2c -c public 192.168.1.1

# snmpwalk — get specific OID
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.1

# snmpget — get specific value
snmpget -v2c -c public 192.168.1.1 sysDescr.0

# snmp-check — detailed SNMP enumeration
snmp-check 192.168.1.1 -c public

# Nmap SNMP scripts
nmap -sU -p 161 --script snmp-info 192.168.1.1
nmap -sU -p 161 --script snmp-sysdescr 192.168.1.1
nmap -sU -p 161 --script snmp-win32-users 192.168.1.1

# onesixtyone — brute force community strings
onesixtyone -c community_strings.txt 192.168.1.1
```

### Useful SNMP OIDs

| OID | Information |
|---|---|
| `1.3.6.1.2.1.1.1.0` | System description |
| `1.3.6.1.2.1.1.5.0` | Hostname |
| `1.3.6.1.2.1.25.4.2.1.2` | Running processes |
| `1.3.6.1.2.1.25.6.3.1.2` | Installed software |
| `1.3.6.1.4.1.77.1.2.25` | Windows user accounts |

---

## 📂 LDAP Enumeration

**LDAP** (Lightweight Directory Access Protocol) is used to access Active Directory. Runs on **port 389 (LDAP) and 636 (LDAPS/secure)**.

### Key Concepts

| Term | Meaning |
|---|---|
| **DN** | Distinguished Name — unique identifier for directory entry |
| **Base DN** | Starting point for directory search |
| **DC** | Domain Component (e.g., `dc=company,dc=com`) |
| **OU** | Organizational Unit |
| **CN** | Common Name (username, group name) |

### Commands

```bash
# ldapsearch — enumerate LDAP (Linux)
ldapsearch -h 192.168.1.1 -x -b "dc=company,dc=com"

# Anonymous LDAP enumeration
ldapsearch -h 192.168.1.1 -x -b "" -s base

# Enumerate users
ldapsearch -h 192.168.1.1 -x -b "dc=company,dc=com" "(objectClass=user)"

# Enumerate groups
ldapsearch -h 192.168.1.1 -x -b "dc=company,dc=com" "(objectClass=group)"

# Nmap LDAP scripts
nmap -p 389 --script ldap-search 192.168.1.1
nmap -p 389 --script ldap-rootdse 192.168.1.1
```

### Tool: **JXplorer** / **Softerra LDAP Administrator**
- GUI tools to browse and query LDAP/Active Directory

---

## ⏰ NTP Enumeration

**NTP** (Network Time Protocol) synchronizes clocks across systems. Runs on **UDP port 123**.

Attackers enumerate NTP to:
- Find hosts connected to the NTP server (internal network map)
- Identify OS and network device info

### Commands

```bash
# ntpdate — sync time / query NTP server
ntpdate 192.168.1.1

# ntptrace — trace NTP server hierarchy
ntptrace 192.168.1.1

# ntpq — query NTP server details
ntpq -p 192.168.1.1

# monlist — list recent clients (deprecated but exam favorite!)
ntpdc -c monlist 192.168.1.1

# Nmap NTP scripts
nmap -sU -p 123 --script ntp-info 192.168.1.1
nmap -sU -p 123 --script ntp-monlist 192.168.1.1
```

> 💡 **Exam Tip:** `monlist` command reveals all clients that have synced with the NTP server — huge info leak and also used in **NTP amplification DDoS attacks**.

---

## 📧 SMTP Enumeration

**SMTP** (Simple Mail Transfer Protocol) handles email sending. Runs on **TCP port 25**.

SMTP enumeration reveals **valid email addresses / usernames** using three commands:

| Command | Use | Response |
|---|---|---|
| `VRFY username` | Verify if user exists | 250 = exists, 550 = doesn't exist |
| `EXPN username` | Expand mailing list | Shows members of a mailing list |
| `RCPT TO` | Check if recipient exists | 250 = valid, 550 = invalid |

### Commands

```bash
# Connect to SMTP server
telnet 192.168.1.1 25
nc -v 192.168.1.1 25

# After connecting:
HELO attacker.com
VRFY admin
VRFY root
EXPN sales-team
RCPT TO: admin@target.com

# Nmap SMTP scripts
nmap -p 25 --script smtp-enum-users 192.168.1.1
nmap -p 25 --script smtp-commands 192.168.1.1
nmap -p 25 --script smtp-open-relay 192.168.1.1

# smtp-user-enum tool
smtp-user-enum -M VRFY -U users.txt -t 192.168.1.1
smtp-user-enum -M RCPT -U users.txt -D target.com -t 192.168.1.1
```

---

## 🌐 DNS Enumeration

**DNS** (Domain Name System) translates domain names to IPs. Runs on **UDP/TCP port 53**.

### DNS Record Types

| Record | Meaning |
|---|---|
| `A` | IPv4 address |
| `AAAA` | IPv6 address |
| `MX` | Mail server |
| `NS` | Name server |
| `CNAME` | Alias / canonical name |
| `TXT` | Text records (SPF, DKIM) |
| `SOA` | Start of Authority — primary DNS info |
| `PTR` | Reverse DNS lookup |

### Commands

```bash
# Basic DNS lookup
nslookup target.com
nslookup -type=MX target.com
nslookup -type=NS target.com

# dig — detailed DNS queries
dig target.com
dig target.com MX
dig target.com NS
dig target.com ANY          # All records

# Zone Transfer (AXFR) — dumps entire DNS zone
dig axfr @ns1.target.com target.com
nslookup
> server ns1.target.com
> set type=any
> ls -d target.com

# host command
host -t mx target.com
host -t ns target.com
host -a target.com

# fierce — DNS brute force
fierce --domain target.com

# dnsenum — comprehensive DNS enumeration
dnsenum target.com

# dnsrecon — advanced DNS recon
dnsrecon -d target.com
dnsrecon -d target.com -t axfr    # Zone transfer attempt
dnsrecon -d target.com -t brt     # Brute force subdomains
```

> 💡 **Exam Tip:** **DNS Zone Transfer** (AXFR) is a critical misconfiguration — if allowed, attacker gets ALL DNS records (every hostname and IP in the domain). Should be restricted to authorized secondary DNS servers only.

---

## 🔗 SMB Enumeration

**SMB** (Server Message Block) handles file/printer sharing. Runs on **TCP port 445** and **139 (NetBIOS over TCP)**.

### Commands

```bash
# smbclient — list shares (Linux)
smbclient -L //192.168.1.1 -N           # No password
smbclient -L //192.168.1.1 -U username

# Connect to a share
smbclient //192.168.1.1/C$ -U username

# enum4linux — comprehensive SMB/NetBIOS enumeration
enum4linux 192.168.1.1
enum4linux -a 192.168.1.1              # All enumeration
enum4linux -U 192.168.1.1             # Users only
enum4linux -S 192.168.1.1             # Shares only
enum4linux -P 192.168.1.1             # Password policy

# Nmap SMB scripts
nmap -p 445 --script smb-enum-shares 192.168.1.1
nmap -p 445 --script smb-enum-users 192.168.1.1
nmap -p 445 --script smb-security-mode 192.168.1.1
nmap -p 445 --script smb-vuln-ms17-010 192.168.1.1   # EternalBlue

# crackmapexec — SMB enumeration & attack
crackmapexec smb 192.168.1.1
crackmapexec smb 192.168.1.0/24 -u user -p pass --shares
```

---

## 📁 FTP Enumeration

**FTP** (File Transfer Protocol) runs on **TCP port 21** (control) and **port 20** (data).

### Commands

```bash
# Connect to FTP
ftp 192.168.1.1
nc -v 192.168.1.1 21              # Banner grab

# Anonymous FTP login (common misconfiguration)
ftp 192.168.1.1
> Username: anonymous
> Password: anything@email.com

# Nmap FTP scripts
nmap -p 21 --script ftp-anon 192.168.1.1        # Check anonymous login
nmap -p 21 --script ftp-syst 192.168.1.1        # System info
nmap -p 21 --script ftp-bounce 192.168.1.1      # FTP bounce attack check
```

---

## 🔧 RPC Enumeration

**RPC** (Remote Procedure Call) enables programs to execute code on remote systems. Runs on **TCP port 135**.

```bash
# rpcinfo — enumerate RPC services (Linux)
rpcinfo -p 192.168.1.1

# Windows — query RPC endpoint mapper
rpcdump.py 192.168.1.1            # Impacket tool

# Nmap RPC scripts
nmap -p 135 --script msrpc-enum 192.168.1.1
```

---

## 🐧 Linux/Unix Enumeration

```bash
# Finger — enumerate users (port 79)
finger @192.168.1.1
finger username@192.168.1.1

# rwho — list logged-in users (port 513)
rwho

# rusers — list users on network (RPC)
rusers -l 192.168.1.1

# rpcinfo — list RPC services
rpcinfo -p 192.168.1.1

# showmount — list NFS exports (port 2049)
showmount -e 192.168.1.1

# Mount NFS share
mount -t nfs 192.168.1.1:/share /mnt/target
```

---

## 🧰 Enumeration Tools Summary

| Tool | Protocol | What It Enumerates |
|---|---|---|
| **nbtstat** | NetBIOS | Names, MAC, shares |
| **enum4linux** | SMB/NetBIOS | Users, shares, groups, password policy |
| **snmpwalk** | SNMP | MIB tree, users, processes, software |
| **snmp-check** | SNMP | Detailed SNMP info |
| **onesixtyone** | SNMP | Community string brute force |
| **ldapsearch** | LDAP | Directory users, groups, OUs |
| **dnsenum** | DNS | Subdomains, zone transfer |
| **dnsrecon** | DNS | DNS records, zone transfer, brute force |
| **smtp-user-enum** | SMTP | Valid email usernames |
| **smbclient** | SMB | Shares, files |
| **crackmapexec** | SMB | Users, shares, creds |
| **rpcdump** | RPC | RPC endpoints |
| **Nmap NSE** | All | Scripts for every protocol |
| **NetScanTools Pro** | Multiple | GUI-based multi-protocol enum |
| **Hyena** | Windows | AD users, groups, policies |

---

## 📝 CEH Exam Tips

### ⭐ Must-Know for Exam

1. **NetBIOS port = 137, 138, 139** | SMB port = **445**
2. **SNMP port = UDP 161** | Traps = **UDP 162**
3. **LDAP port = 389** | Secure LDAP = **636**
4. **NTP port = UDP 123** | `monlist` = reveals all clients
5. **SMTP VRFY** = verify user | **EXPN** = expand mailing list | **RCPT TO** = check recipient
6. **DNS Zone Transfer = AXFR** — critical misconfiguration, dumps all DNS records
7. **Community strings:** `public` = read | `private` = read-write
8. **NetBIOS `<20>`** = File & Printer Sharing enabled
9. **enum4linux** = best tool for SMB/NetBIOS enumeration on Linux
10. **Anonymous FTP** = login with username `anonymous`, any email as password

### Enumeration vs Scanning — Quick Diff

| | Scanning | Enumeration |
|---|---|---|
| **Goal** | Find open ports & live hosts | Extract usernames, shares, services |
| **Depth** | Surface level | Deep, detailed |
| **Connection** | Often incomplete (SYN) | Full connection to services |
| **Tools** | Nmap, hping3 | enum4linux, snmpwalk, ldapsearch |

### Protocol → Port Quick Reference

| Protocol | Port | Transport |
|---|---|---|
| FTP | 21 | TCP |
| SSH | 22 | TCP |
| SMTP | 25 | TCP |
| DNS | 53 | TCP/UDP |
| DHCP | 67/68 | UDP |
| HTTP | 80 | TCP |
| NTP | 123 | UDP |
| NetBIOS | 137/138/139 | UDP/TCP |
| SNMP | 161/162 | UDP |
| LDAP | 389 | TCP |
| SMB | 445 | TCP |
| LDAPS | 636 | TCP |
| RDP | 3389 | TCP |

---

## 🔗 References

- CEH v13 Official Courseware — Module 04
- [Nmap NSE Scripts](https://nmap.org/nsedoc/)
- [enum4linux GitHub](https://github.com/CiscoCXSecurity/enum4linux)
- [OWASP Testing Guide — Enumeration](https://owasp.org/www-project-web-security-testing-guide/)

---

*Notes prepared by Vaishnavi Trivedi | CEH v13 Exam Prep | github.com/Vaishnavi-07823*
