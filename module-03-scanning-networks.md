# 🔍 CEH v13 — Module 03: Scanning Networks

> 📌 Personal study notes for CEH v13 exam preparation. Covers key concepts, tools, and practical commands used in network scanning during ethical hacking engagements.
---

## 📚 Table of Contents

1. [Overview](#overview)
2. [Types of Scanning](#types-of-scanning)
3. [TCP/IP Concepts for Scanning](#tcpip-concepts-for-scanning)
4. [Nmap — The Core Tool](#nmap--the-core-tool)
5. [Scanning Techniques](#scanning-techniques)
6. [OS Fingerprinting](#os-fingerprinting)
7. [Banner Grabbing](#banner-grabbing)
8. [Vulnerability Scanning](#vulnerability-scanning)
9. [IDS/Firewall Evasion](#idsfirewall-evasion)
10. [Drawing Network Diagrams](#drawing-network-diagrams)
11. [CEH Exam Tips](#ceh-exam-tips)

---

## 🧭 Overview

**Scanning** is Phase 2 of the ethical hacking methodology (after Footprinting). It involves actively probing the target network to gather information about:

- Live hosts (which systems are online)
- Open ports (which services are running)
- Operating systems (what OS is in use)
- Services & versions (what software is running)
- Vulnerabilities (what can be exploited)

### Scanning vs Footprinting
| | Footprinting | Scanning |
|---|---|---|
| **Nature** | Passive / Active | Active |
| **Interaction** | No direct contact with target | Direct packets sent to target |
| **Goal** | Gather public info | Identify live hosts, ports, services |

---

## 🔢 Types of Scanning

| Type | Purpose |
|---|---|
| **Port Scanning** | Find open/closed/filtered ports |
| **Network Scanning** | Discover live hosts in a network |
| **Vulnerability Scanning** | Identify security weaknesses |

### Port States (Nmap)
| State | Meaning |
|---|---|
| `open` | Service actively accepting connections |
| `closed` | Port accessible but no service listening |
| `filtered` | Firewall/filter blocking — nmap can't determine state |
| `unfiltered` | Accessible but can't determine open/closed (ACK scan) |
| `open\|filtered` | Can't tell if open or filtered |
| `closed\|filtered` | Can't tell if closed or filtered |

---

## 🌐 TCP/IP Concepts for Scanning

### TCP 3-Way Handshake
```
Client          Server
  |--- SYN ------->|
  |<-- SYN-ACK ----|
  |--- ACK ------->|
  (Connection Established)
```

### TCP Flags — Must Know for Exam

| Flag | Full Name | Use |
|---|---|---|
| `SYN` | Synchronize | Initiate connection |
| `ACK` | Acknowledge | Confirm receipt |
| `FIN` | Finish | Gracefully close connection |
| `RST` | Reset | Abruptly terminate connection |
| `PSH` | Push | Send data immediately |
| `URG` | Urgent | Prioritize data |

> 💡 **Exam Tip:** Different scan types manipulate these flags — know which flag each scan sends and what response indicates open/closed.

---

## 🛠️ Nmap — The Core Tool

### Basic Syntax
```bash
nmap [Scan Type] [Options] [Target]
```

### Target Specification
```bash
nmap 192.168.1.1                  # Single IP
nmap 192.168.1.1-254              # IP range
nmap 192.168.1.0/24               # CIDR subnet
nmap scanme.nmap.org              # Domain
nmap -iL targets.txt              # From file
nmap --exclude 192.168.1.5        # Exclude host
```

### Output Formats
```bash
nmap -oN output.txt               # Normal output
nmap -oX output.xml               # XML output
nmap -oG output.gnmap             # Grepable output
nmap -oA output                   # All formats at once
```

---

## 🔬 Scanning Techniques

### 1. TCP Connect Scan (`-sT`)
- Completes full 3-way handshake
- **Loud** — easily detected and logged
- Does NOT require root/admin privileges
- Used when SYN scan is not possible

```bash
nmap -sT 192.168.1.1
```

**Logic:**
```
Open port:   SYN → SYN-ACK → ACK (connection complete) → RST
Closed port: SYN → RST
```

---

### 2. SYN Scan / Stealth Scan (`-sS`) ⭐ Most Common
- Also called **Half-Open Scan**
- Never completes the handshake → harder to detect
- Requires root/admin privileges
- **Default scan** when run as root

```bash
nmap -sS 192.168.1.1
sudo nmap -sS 192.168.1.0/24
```

**Logic:**
```
Open port:   SYN → SYN-ACK → RST (we reset before completing)
Closed port: SYN → RST
Filtered:    SYN → No response / ICMP unreachable
```

---

### 3. UDP Scan (`-sU`)
- Scans UDP ports (DNS-53, DHCP-67/68, SNMP-161)
- Slower than TCP scans
- No handshake — relies on ICMP "port unreachable" for closed ports

```bash
sudo nmap -sU 192.168.1.1
sudo nmap -sU -p 53,67,161 192.168.1.1
```

**Logic:**
```
Open port:    UDP probe → UDP response (or no response = open|filtered)
Closed port:  UDP probe → ICMP Port Unreachable
```

---

### 4. NULL Scan (`-sN`)
- Sends packet with **NO flags set**
- Used for **firewall evasion** (bypasses some stateless firewalls)
- Works only on Linux/Unix targets (Windows always sends RST)

```bash
sudo nmap -sN 192.168.1.1
```

**Logic:**
```
Open port:   No response
Closed port: RST
```

---

### 5. FIN Scan (`-sF`)
- Sends packet with only **FIN flag**
- Same evasion purpose as NULL scan

```bash
sudo nmap -sF 192.168.1.1
```

---

### 6. Xmas Scan (`-sX`)
- Sends packet with **FIN + PSH + URG** flags (lights up like a Christmas tree 🎄)
- Same response logic as NULL and FIN

```bash
sudo nmap -sX 192.168.1.1
```

> 💡 **NULL, FIN, Xmas — Same Logic:**
> - Open/Filtered → No response
> - Closed → RST
> - All three bypass some firewalls but **don't work on Windows**

---

### 7. ACK Scan (`-sA`)
- Used to **map firewall rules**, NOT to find open ports
- Determines if ports are **filtered or unfiltered**

```bash
sudo nmap -sA 192.168.1.1
```

**Logic:**
```
Unfiltered: RST (firewall lets it through)
Filtered:   No response or ICMP unreachable
```

---

### 8. IDLE / Zombie Scan (`-sI`)
- Most stealthy scan — uses a **zombie host** as intermediary
- Attacker's IP never appears in target logs
- Requires finding a host with predictable IPID sequence

```bash
sudo nmap -sI zombie_ip target_ip
```

---

### 9. ICMP Ping Sweep (Host Discovery)
```bash
nmap -sn 192.168.1.0/24           # Ping sweep (no port scan)
nmap -PE 192.168.1.0/24           # ICMP Echo ping
nmap -PP 192.168.1.0/24           # ICMP Timestamp ping
nmap -PM 192.168.1.0/24           # ICMP Address Mask ping
```

---

### Scan Comparison Table

| Scan | Flag Sent | Open Response | Closed Response | Privileges | Stealth |
|---|---|---|---|---|---|
| TCP Connect | SYN | SYN-ACK | RST | User | ❌ Low |
| SYN Stealth | SYN | SYN-ACK→RST | RST | Root | ✅ Medium |
| UDP | — | UDP/No reply | ICMP Unreach | Root | ✅ Medium |
| NULL | None | No reply | RST | Root | ✅ High |
| FIN | FIN | No reply | RST | Root | ✅ High |
| Xmas | FIN+PSH+URG | No reply | RST | Root | ✅ High |
| ACK | ACK | RST | RST | Root | ✅ High |
| IDLE | SYN (zombie) | — | — | Root | ✅✅ Highest |

---

## 💻 OS Fingerprinting

### Active OS Fingerprinting
Sends specially crafted packets and analyzes responses to guess OS.

```bash
nmap -O 192.168.1.1               # OS detection
nmap -O --osscan-guess 192.168.1.1  # Aggressive guess
```

### Passive OS Fingerprinting
Captures traffic without sending packets — tools: **p0f**, **Wireshark**

```bash
p0f -i eth0                       # Passive fingerprinting on interface
```

### TTL Values — Quick OS Identification

| OS | Default TTL |
|---|---|
| Windows | 128 |
| Linux / Android | 64 |
| Cisco / Network Devices | 255 |
| macOS | 64 |

> 💡 **Exam Tip:** TTL decrements by 1 at each hop. If you receive TTL=118, original was likely 128 → Windows.

---

## Banner Grabbing

Grabbing service banners reveals software name, version, and OS — useful for finding known CVEs.

### Using Telnet
```bash
telnet 192.168.1.1 80
# Then type:
GET / HTTP/1.0
```

### Using Netcat
```bash
nc -v 192.168.1.1 80
nc -v 192.168.1.1 22              # SSH banner grab
nc -v 192.168.1.1 21              # FTP banner grab
```

### Using Nmap Service Detection
```bash
nmap -sV 192.168.1.1              # Service/version detection
nmap -sV --version-intensity 9 192.168.1.1  # Aggressive version detection
nmap -A 192.168.1.1               # Aggressive (OS + version + scripts + traceroute)
```

### Using curl
```bash
curl -I http://192.168.1.1        # HTTP headers (reveals server info)
```

---

##  Vulnerability Scanning

### Nmap Scripting Engine (NSE)
```bash
nmap --script vuln 192.168.1.1              # Run all vuln scripts
nmap --script smb-vuln-ms17-010 192.168.1.1 # EternalBlue check
nmap --script http-shellshock 192.168.1.1   # Shellshock check
nmap --script ftp-anon 192.168.1.1          # Anonymous FTP check
nmap --script default 192.168.1.1           # Default safe scripts
```

### Nessus (GUI Tool)
- Most widely used vulnerability scanner
- Generates detailed reports with CVSS scores
- CEH exam often references Nessus for vuln scanning

### OpenVAS
- Open-source alternative to Nessus
```bash
openvas-start                     # Start OpenVAS service
```

---

##  IDS/Firewall Evasion Techniques

### 1. Fragmentation
Split packets into tiny fragments to confuse IDS reassembly:
```bash
nmap -f 192.168.1.1               # Fragment packets (8 bytes)
nmap -ff 192.168.1.1              # 16-byte fragments
nmap --mtu 16 192.168.1.1         # Custom MTU size
```

### 2. Decoy Scan
Make scan appear to come from multiple IPs:
```bash
nmap -D RND:10 192.168.1.1        # 10 random decoys
nmap -D 10.0.0.1,10.0.0.2,ME 192.168.1.1  # Specific decoys + your IP
```

### 3. Source Port Manipulation
Use a trusted source port (80, 443, 53) to bypass firewall rules:
```bash
nmap --source-port 53 192.168.1.1
nmap -g 80 192.168.1.1
```

### 4. Slow Scan (Timing)
Slow down scan to evade rate-based IDS detection:
```bash
nmap -T0 192.168.1.1              # Paranoid (slowest)
nmap -T1 192.168.1.1              # Sneaky
nmap -T2 192.168.1.1              # Polite
nmap -T3 192.168.1.1              # Normal (default)
nmap -T4 192.168.1.1              # Aggressive
nmap -T5 192.168.1.1              # Insane (fastest)
```

### 5. IP Spoofing
```bash
nmap -S spoofed_ip -e eth0 -Pn 192.168.1.1
```

### 6. MAC Spoofing
```bash
nmap --spoof-mac 0 192.168.1.1    # Random MAC
nmap --spoof-mac Apple 192.168.1.1 # Vendor-specific MAC
```

### 7. Proxy / Anonymizer
Route scans through proxies:
```bash
nmap --proxies socks4://proxy_ip:port 192.168.1.1
```

---

## 🗺️ Drawing Network Diagrams

### Traceroute
```bash
traceroute 192.168.1.1            # Linux
tracert 192.168.1.1               # Windows
nmap --traceroute 192.168.1.1     # Nmap traceroute
```

### Network Mapping Tools
| Tool | Use |
|---|---|
| **Network Topology Mapper** | GUI-based network diagramming |
| **SolarWinds** | Enterprise network mapping |
| **Zenmap** | Nmap GUI with topology view |
| **Maltego** | Visual link analysis |

---

## 📝 CEH Exam Tips

### ⭐ Must-Know for Exam

1. **SYN scan = Half-open scan = Stealth scan** — all same thing
2. **NULL, FIN, Xmas** don't work on Windows — open ports give no response
3. **ACK scan** is for firewall mapping, NOT port discovery
4. **IDLE scan** = Zombie scan — most stealthy, uses third-party host
5. **TTL 128 = Windows, TTL 64 = Linux**
6. **-sn** = Ping sweep (host discovery, no port scan)
7. **-A** = Aggressive scan (OS + version + scripts + traceroute)
8. **-T0 to T5** = Timing templates (T0 slowest, T5 fastest)

### Common Port Numbers for Exam

| Port | Protocol | Service |
|---|---|---|
| 21 | TCP | FTP |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 135 | TCP | RPC |
| 139/445 | TCP | SMB/NetBIOS |
| 143 | TCP | IMAP |
| 161/162 | UDP | SNMP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 3389 | TCP | RDP |

### Scanning Tools Summary

| Tool | Purpose |
|---|---|
| **Nmap** | Port scanning, OS detection, version detection |
| **Hping3** | Packet crafting, custom scans |
| **Zenmap** | Nmap GUI |
| **Nessus** | Vulnerability scanning |
| **OpenVAS** | Open-source vulnerability scanner |
| **p0f** | Passive OS fingerprinting |
| **Netcat** | Banner grabbing, port scanning |
| **Wireshark** | Packet capture & analysis |
| **Angry IP Scanner** | Fast ping + port scanner (Windows-friendly) |

---

## 🔗 References

- CEH v13 Official Courseware — Module 03
- [Nmap Official Documentation](https://nmap.org/book/man.html)
- [OWASP Testing Guide — Network Scanning](https://owasp.org/www-project-web-security-testing-guide/)
- [Nmap Cheat Sheet — StationX](https://www.stationx.net/nmap-cheat-sheet/)

---

*Notes prepared by Vaishnavi Trivedi | CEH v13 Exam Prep | github.com/Vaishnavi-07823*
