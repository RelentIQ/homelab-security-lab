# Purple Team Exercise: Active Directory Enumeration & Attack Detection

## Executive Summary

This document presents a purple team exercise conducted against a Windows Server 2022 Active Directory environment in an isolated homelab. The exercise demonstrates both offensive penetration testing methodology and defensive SIEM-based detection capabilities using Wazuh 4.14. All attacks were executed from a Kali Linux platform connecting remotely via Tailscale mesh VPN, simulating real-world remote engagement conditions.

**Key Findings:**
- Unauthenticated enumeration was largely blocked — anonymous LDAP, RPC RID cycling, and DNS zone transfers were denied
- Authenticated enumeration exposed all domain users, groups, and an overprivileged service account
- Kerberoasting successfully extracted a crackable TGS hash for a Domain Admin account
- Sensitive data (passwords, employee lists) was accessible via misconfigured SMB shares
- Wazuh SIEM detected 2,630+ events across all attack phases, including custom rule triggers for brute force and privilege escalation

---

## Environment

| Component | Details |
|-----------|---------|
| **Target** | DC01 — Windows Server 2022, Active Directory Domain Controller (homelab.local) |
| **Attacker** | Kali Linux 2025.2 on M2 Pro MacBook via Parallels |
| **SIEM** | Wazuh 4.14 (All-in-one: Manager, Indexer, Dashboard) |
| **Network** | Tailscale mesh VPN (remote access from separate physical network) |
| **Agents** | 3 Wazuh agents (DC01, vuln-web-apps, domain workstation) |

---

## Phase 1: Reconnaissance

### Port Scanning

**Tool:** Nmap 7.98
**Command:** `nmap -Pn -sT -sV -sC -O <target> -oN dc01_scan.txt`

**Results — 13 open ports identified:**

| Port | Service | Version |
|------|---------|---------|
| 53/tcp | DNS | Simple DNS Plus |
| 88/tcp | Kerberos | Microsoft Windows Kerberos |
| 135/tcp | MSRPC | Microsoft Windows RPC |
| 389/tcp | LDAP | AD LDAP (homelab.local) |
| 445/tcp | SMB | Microsoft-DS |
| 464/tcp | kpasswd5 | — |
| 593/tcp | RPC over HTTP | Microsoft Windows RPC over HTTP 1.0 |
| 636/tcp | LDAPS | tcpwrapped |
| 3268/tcp | Global Catalog | AD LDAP |
| 3269/tcp | Global Catalog SSL | tcpwrapped |
| 3389/tcp | RDP | Microsoft Terminal Services |
| 5357/tcp | HTTP | Microsoft HTTPAPI 2.0 |
| 5985/tcp | WinRM | Microsoft HTTPAPI 2.0 |

**Key intelligence gathered from nmap scripts:**
- Domain: homelab.local
- NetBIOS Domain: HOMELAB
- Computer Name: DC01
- OS: Windows Server 2022 (Build 10.0.20348)
- SMB Signing: Enabled and Required (blocks SMB relay attacks)
- RDP Certificate CN: DC01.homelab.local

**MITRE ATT&CK:** T1046 — Network Service Discovery

**Wazuh Detection:** 84 events (Rule 92110 — Sysmon detected WinRM/network activity from attacker IP)

---

### DNS Zone Transfer Attempt

**Command:** `dig axfr homelab.local @<target>`
**Result:** Transfer failed — zone transfers are restricted.

**Finding:** DNS is properly configured to deny zone transfers to unauthorized hosts.

---

### Unauthenticated Enumeration Attempts

| Technique | Tool | Command | Result |
|-----------|------|---------|--------|
| LDAP Anonymous Bind | ldapsearch | `ldapsearch -x -H ldap://<target> -b "DC=homelab,DC=local"` | **Denied** — Requires authentication |
| RID Brute Force | NetExec | `nxc smb <target> -u '' -p '' --rid-brute` | **Denied** — No output |
| Guest RID Brute Force | NetExec | `nxc smb <target> -u 'guest' -p '' --rid-brute` | **Denied** — No output |
| SMB Null Session | enum4linux | `enum4linux -a <target>` | **Denied** — Froze on policy enumeration |
| RPC Null Session | rpcclient | `rpcclient -U "" -N <target>` | **Denied** |

**Finding:** Anonymous enumeration is blocked across all services. This is a positive security configuration.

**Wazuh Detection:** 82 events (Rule 92657 — "ANONYMOUS LOGON via NTLM from KALI-LINUX-2025-2, possible pass-the-hash attack")

---

## Phase 2: Password Attack

### Password Spraying

Since unauthenticated enumeration failed, a password spray was attempted using common username/password combinations.

**Tool:** Hydra, NetExec
**Technique:** Spray common passwords against likely usernames

**Wazuh Detection:**
- 45 events — Rule 60122 (Logon Failure — Unknown user or bad password)
- 50 events — Rule 60104 (Windows audit failure)
- **5 events — Rule 100002 (Custom: Brute force attack detected)** ← Custom rule triggered!

**MITRE ATT&CK:** T1110.003 — Brute Force: Password Spraying

---

## Phase 3: Authenticated Enumeration

After obtaining valid credentials (simulating a successful password spray), authenticated enumeration was performed.

### Domain User Enumeration

**Command:** `nxc smb <target> -u 'jsmith' -p '<password>' --users`

**Discovered accounts:**
- Administrator (Domain Admin)
- jsmith (Domain User)
- sconnor (Domain User)
- admin.backup (Domain Admin — **overprivileged**)
- krbtgt (Kerberos service account)
- Guest (disabled)

**Critical Finding:** The `admin.backup` account is a member of Domain Admins — this is a common misconfiguration where backup service accounts are given excessive privileges.

### Group Enumeration

**Command:** `nxc smb <target> -u 'jsmith' -p '<password>' --groups`

Enumerated all domain security groups and their memberships, confirming admin.backup's presence in Domain Admins.

**MITRE ATT&CK:** T1087.002 — Account Discovery: Domain Account

### Share Enumeration

**Command:** `nxc smb <target> -u 'jsmith' -p '<password>' --shares`

| Share | Access |
|-------|--------|
| ADMIN$ | No access |
| C$ | No access |
| Company | **READ/WRITE** |
| IPC$ | Read only |
| NETLOGON | Read only |
| SYSVOL | Read only |

**Critical Finding:** The "Company" share is accessible with full read/write permissions for all authenticated domain users.

### Sensitive Data Exfiltration

**Command:** `smbclient //<target>/Company -U 'jsmith' -c "ls; get passwords.txt; get employee-list.txt"`

Successfully downloaded sensitive files from the open share:
- **passwords.txt** — Contained plaintext credentials
- **employee-list.txt** — Contained employee information

**MITRE ATT&CK:** T1135 — Network Share Discovery, T1039 — Data from Network Shared Drive

**Wazuh Detection:**
- 1,088 events — Rule 92106 (SMB admin share access activity)
- 9 events — Rule 100006 (Custom: LDAP enumeration detected)

---

## Phase 4: Kerberoasting

### Service Principal Name Discovery

After discovering the overprivileged `admin.backup` account, a Kerberoasting attack was attempted to extract its service ticket hash for offline cracking.

**Tool:** Impacket
**Command:** `impacket-GetUserSPNs homelab.local/jsmith:'<password>' -dc-ip <target> -request`

**Result:** Successfully extracted a Kerberos TGS-REP hash (RC4/Type 23) for the `admin.backup` account.

```
$krb5tgs$23$*admin.backup$HOMELAB.LOCAL$homelab.local/admin.backup*$[hash redacted]
```

### Hash Cracking

**Tool:** John the Ripper / Hashcat
**Command:** `hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt`

The hash was successfully cracked, revealing the service account password and providing a path to full Domain Admin access.

**MITRE ATT&CK:** T1558.003 — Steal or Forge Kerberos Tickets: Kerberoasting

**Wazuh Detection:**
- 4 events — Rule 100009 (Custom: User added to security-enabled global group — privilege escalation, Level 12 CRITICAL)

---

## Phase 5: Attack Summary & Kill Chain

```
Reconnaissance ──► Enumeration ──► Credential Access ──► Privilege Escalation
     │                   │                  │                      │
  Port Scan         Anonymous          Password             Kerberoasting
  (nmap)            Enum (blocked)     Spray                (admin.backup)
     │                   │                  │                      │
  13 ports          Auth Enum           Valid creds          Domain Admin
  identified        (users, shares)     (jsmith)             hash cracked
```

---

## SIEM Detection Summary

### Total Events Captured: 2,630+

| Rule ID | Description | Count | Severity | MITRE ATT&CK |
|---------|-------------|-------|----------|---------------|
| **100009** | User added to privileged group | 4 | **CRITICAL (12)** | T1098 |
| **100002** | Brute force: Multiple failed logins | 5 | **HIGH (10)** | T1110 |
| **100006** | LDAP enumeration detected | 9 | **MEDIUM (8)** | T1087.002 |
| 92657 | Anonymous NTLM logon (pass-the-hash warning) | 82 | **MEDIUM (6)** | T1550.002 |
| 60122 | Logon Failure — Unknown user or bad password | 45 | **MEDIUM (5)** | T1110 |
| 60104 | Windows audit failure | 50 | **MEDIUM (5)** | — |
| 92110 | WinRM/network activity from attacker IP | 84 | **LOW (4)** | T1046 |
| 92106 | SMB admin share access | 1,088 | **LOW (3)** | T1135 |
| 92652 | NTLM authenticated remote logon (jsmith) | 7 | **MEDIUM (6)** | T1078 |
| 60112 | Audit policy changed | 18 | **MEDIUM (8)** | T1562.002 |

### Custom Rules Performance

All 4 custom detection rules fired successfully:
- **100002** — Brute force detection (frequency-based, 8+ failures in 60 seconds)
- **100006** — LDAP enumeration detection
- **100008** — New user account creation
- **100009** — Privilege escalation via group membership change

### Detection Gaps Identified

| Attack Technique | Detected? | Notes |
|-----------------|-----------|-------|
| Port scanning (nmap) | ⚠️ Partial | Detected via Sysmon network connections, not as a specific "port scan" alert |
| DNS zone transfer | ❌ No | Failed attempt not logged by Wazuh |
| Anonymous enumeration | ✅ Yes | NTLM anonymous logon alerts triggered |
| Password spraying | ✅ Yes | Both individual failures and brute force aggregation detected |
| Authenticated LDAP queries | ✅ Yes | Custom rule triggered |
| SMB share access | ✅ Yes | Multiple SMB-related rules fired |
| Kerberoasting | ⚠️ Partial | Group change detected, but TGS-REQ with RC4 not specifically flagged |
| Data exfiltration | ⚠️ Partial | Share access logged but file download not specifically alerted |

---

## Remediation Recommendations

### Critical Priority

1. **Remove admin.backup from Domain Admins** — Service accounts should follow the principle of least privilege. Create a dedicated backup operator role with only necessary permissions.

2. **Remove SPN from user account** — Use managed service accounts (gMSA) instead of assigning SPNs to regular user accounts. This eliminates the Kerberoasting attack vector entirely.

3. **Enforce strong password policy** — Implement a minimum 14-character password length, complexity requirements, and consider a banned password list.

4. **Remove plaintext credentials from shares** — Delete passwords.txt immediately. Implement a password manager for credential storage.

### High Priority

5. **Restrict Company share permissions** — Replace Everyone:Full Control with specific security group access. Apply both share and NTFS permissions following least privilege.

6. **Enable Network Level Authentication for RDP** — Prevents unauthenticated RDP connection attempts.

7. **Implement account lockout policy** — Lock accounts after 5 failed attempts to mitigate brute force attacks.

### Medium Priority

8. **Deploy advanced Kerberos monitoring** — Create specific Wazuh rules for Event 4769 with RC4 encryption type (0x17) to detect Kerberoasting attempts.

9. **Enable command-line logging** — Configure Process Creation auditing (Event 4688) with command-line arguments for enhanced forensic visibility.

10. **Implement SMB file access auditing** — Configure detailed audit SACLs on sensitive shares to log specific file access events.

---

## Lessons Learned

1. **Anonymous enumeration blocking works** — Properly configured Windows defaults prevented unauthenticated reconnaissance across LDAP, RPC, and SMB.

2. **Port scans are noisy but hard to classify** — Wazuh detected 84+ network connection events from the scan, but without custom correlation rules, they appear as generic network activity rather than a classified "port scan."

3. **Password spraying generates clear SIEM signals** — The custom brute force rule (100002) successfully aggregated individual failures into a single high-severity alert.

4. **Kerberoasting is quiet from a SIEM perspective** — The actual TGS request that extracts the hash generates minimal alerts. Detection requires specific rules targeting Event 4769 with weak encryption types.

5. **SMB share misconfigurations compound risk** — An overprivileged share combined with sensitive data creates a direct path from basic domain user to critical data exposure.

6. **Custom rules add significant detection value** — The 4 custom Wazuh rules detected attack patterns that default rules would have missed or reported at lower severity levels.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap 7.98 | Port scanning, service enumeration, OS detection |
| NetExec (nxc) | SMB/LDAP enumeration, password spraying |
| enum4linux | SMB/RPC enumeration |
| ldapsearch | LDAP enumeration |
| smbclient | SMB share access and file retrieval |
| Impacket (GetUserSPNs) | Kerberoasting |
| Hydra | Password brute force |
| rpcclient | RPC enumeration |
| dig | DNS zone transfer testing |
| Wazuh 4.14 | SIEM monitoring, alerting, and detection |
| Sysmon | Enhanced endpoint telemetry |

---

*This exercise was conducted in an isolated homelab environment. All systems are owned by the author and configured specifically for authorized security testing.*
