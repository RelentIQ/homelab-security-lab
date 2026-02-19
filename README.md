# Homelab Security Lab

A purple team security testing environment built on Proxmox VE, combining production automation services, intentionally vulnerable targets, a Windows Active Directory domain, and centralized SIEM monitoring. This project demonstrates the complete attack → detect → respond lifecycle used in enterprise security operations.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Proxmox VE 9.1 Host                                 │
│                    (Intel iMac — i5, 32GB RAM, 500GB SSD)                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐           │
│  │  VM 100           │  │  VM 102           │  │  VM 103           │          │
│  │  n8n-prod         │  │  vuln-web-apps    │  │  DC01              │          │
│  │                   │  │                   │  │                   │          │
│  │  • n8n (Docker)   │  │  • DVWA (Docker)  │  │  • Windows Server │          │
│  │  • SFTP Server    │  │  • Juice Shop     │  │    2022            │          │
│  │  • 4GB RAM        │  │    (Docker)       │  │  • Active Directory│          │
│  │                   │  │  • Wazuh Agent    │  │  • DNS Server      │          │
│  │                   │  │  • 2GB RAM        │  │  • Wazuh Agent     │          │
│  │                   │  │                   │  │  • 4GB RAM         │          │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘          │
│           │                      │                      │                    │
│  ┌────────┴──────────────────────┴──────────────────────┘                    │
│  │                                                                           │
│  │  ┌──────────────────┐                                                     │
│  │  │  VM 105           │                                                    │
│  │  │  wazuh-siem       │                                                    │
│  │  │                   │                                                    │
│  │  │  • Wazuh Manager  │                                                    │
│  │  │  • Wazuh Indexer  │                                                    │
│  │  │  • Wazuh Dashboard│                                                    │
│  │  │  • 6GB RAM        │                                                    │
│  │  └────────┬─────────┘                                                    │
│  │           │                                                               │
│  └───────────┴──── vmbr0 (Bridge Network) ──────────────────────────────────┘
│                         │                                                    │
└─────────────────────────┼────────────────────────────────────────────────────┘
                          │
                 ┌────────▼────────┐
                 │   Tailscale     │
                 │   Mesh VPN      │
                 └────────┬────────┘
                          │
          ┌───────────────┴───────────────────┐
          │                                   │
    ┌─────▼──────┐               ┌────────────▼────────────┐
    │  Kali       │               │  Windows 10 Pro          │
    │  Laptop     │               │  Domain-Joined Workstation│
    │  (M2 Pro    │               │                          │
    │  MacBook)   │               │  • HOMELAB\domain user   │
    │             │               │  • Wazuh Agent           │
    │  Attack     │               │  • Realistic target      │
    │  Platform   │               │                          │
    └─────────────┘               └──────────────────────────┘
```

## Infrastructure Components

### Hypervisor

| Component | Details |
|-----------|---------|
| **Platform** | Proxmox VE 9.1 |
| **Hardware** | Intel iMac (i5 processor, 32GB RAM, 500GB SSD) |
| **Networking** | Bridged networking (vmbr0) for inter-VM communication |
| **Storage** | LVM-thin provisioning on local storage |

### Remote Access

| Component | Details |
|-----------|---------|
| **Solution** | Tailscale mesh VPN |
| **Architecture** | Zero-trust — no port forwarding required for management |
| **Encryption** | WireGuard-based encrypted tunnels |
| **Use Case** | Secure remote management and penetration testing from any network |

All VMs, the attack platform, and the domain-joined workstation connect through the Tailscale mesh, enabling penetration testing from a separate physical network — simulating real-world remote engagement conditions.

### Virtual Machines

| VM ID | Name | OS | Purpose | RAM | Key Services |
|-------|------|----|---------|-----|-------------|
| 100 | n8n-prod | Debian Linux | Production automation | 4GB | n8n (Docker), SFTP server |
| 102 | vuln-web-apps | Debian Linux | Web application targets | 2GB | DVWA, Juice Shop, Wazuh Agent |
| 103 | DC01 | Windows Server 2022 | Domain Controller | 4GB | Active Directory, DNS, Wazuh Agent |
| 105 | wazuh-siem | Debian 12 | SIEM & Monitoring | 6GB | Wazuh Manager, Indexer, Dashboard |

### External Machines

| Machine | OS | Role | Wazuh Agent |
|---------|-----|------|-------------|
| MacBook Pro (M2) | Kali Linux (Parallels VM) | Attack platform | — |
| Desktop PC | Windows 10 Pro | Domain-joined workstation target | ✅ |

## Purple Team Environment

This lab is designed around the **purple team methodology** — attacking and defending simultaneously to demonstrate the complete security lifecycle.

### Red Team (Offensive)

The Kali Linux attack platform connects remotely via Tailscale and targets:

- **DVWA / Juice Shop** — OWASP Top 10 web application attacks
- **DC01** — Active Directory enumeration, Kerberoasting, password spraying
- **Domain workstation** — Lateral movement, privilege escalation
- **Exposed services** — SMB (445), RDP (3389), LDAP (389), Kerberos (88), WinRM (5985)

### Blue Team (Defensive)

Wazuh SIEM provides centralized monitoring across the entire environment:

- **Real-time alert generation** for attack detection
- **Vulnerability scanning** with CVE identification
- **File integrity monitoring** across all endpoints
- **MITRE ATT&CK framework mapping** for detected threats
- **Log correlation** across Windows event logs, Linux syslogs, and application logs

### The Workflow

1. **Attack** — Execute techniques from Kali against lab targets
2. **Detect** — Monitor Wazuh dashboard for generated alerts
3. **Investigate** — Triage alerts, correlate logs, identify attack patterns
4. **Document** — Write professional reports covering methodology, detection, and remediation

## Active Directory Environment

### Domain Configuration

| Setting | Value |
|---------|-------|
| **Domain** | homelab.local |
| **Domain Controller** | DC01 (Windows Server 2022) |
| **Forest Functional Level** | Windows Server 2016 |
| **DNS** | Integrated with AD DS |

### Intentional Misconfigurations (for attack practice)

These weaknesses are deliberately configured to practice common Active Directory attack techniques:

- **Weak user passwords** — Domain accounts with easily guessable credentials
- **Overprivileged service account** — A backup admin account added to Domain Admins
- **Open file shares** — "Company" share with Everyone:Full Control
- **RDP enabled** — Without Network Level Authentication
- **WinRM enabled** — For remote management attack practice
- **Selective firewall rules** — Key attack surface ports intentionally exposed

### Attack Scenarios Supported

- **AD Enumeration** — LDAP queries, BloodHound, enum4linux
- **Kerberoasting** — Extracting service ticket hashes for offline cracking
- **AS-REP Roasting** — Targeting accounts without pre-authentication
- **Password Spraying** — Testing common passwords against domain accounts
- **SMB Enumeration** — Share discovery, file exfiltration
- **RDP Brute Force** — Credential testing against remote desktop
- **Lateral Movement** — Pass-the-Hash, WinRM pivoting
- **Privilege Escalation** — Exploiting overprivileged accounts

## SIEM — Wazuh 4.14

### Deployment

All-in-one deployment on VM 105 (Debian 12) with Wazuh Manager, Indexer, and Dashboard running on a single node. Agents deployed across the infrastructure report to the manager via Tailscale.

### Agent Coverage

| Endpoint | Agent Status | Monitored Data |
|----------|-------------|----------------|
| vuln-web-apps (VM 102) | ✅ Active | System logs, Docker events, vulnerability scans |
| DC01 (VM 103) | ✅ Active | Windows Event Logs, AD authentication, security events |
| Windows 10 Workstation | ✅ Active | Endpoint security events, logon activity |

### Detection Capabilities

- **Vulnerability Detection** — Automated CVE scanning across all agents
- **Security Configuration Assessment** — CIS benchmark compliance checks
- **Log Analysis** — Real-time parsing and alerting on security events
- **Integrity Monitoring** — File and registry change detection
- **Active Response** — Automated blocking of detected threats

## Production Services (VM 100)

### n8n Workflow Automation

Production automation platform running in Docker, handling business-critical workflows including data processing and file transfers.

**Docker Compose configuration:**

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_SECURE_COOKIE=false
      - TZ=Europe/London
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false
      - N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES=false
      - N8N_RESTRICT_FILE_ACCESS_TO=/home/ftp-user
      - N8N_RUNNERS_TASK_TIMEOUT=36000
    volumes:
      - ./n8n-data:/home/node/.n8n
      - /home/ftp-user:/home/ftp-user
```

### SFTP Server

Secure file transfer service with chroot jail isolation:

```
Match User ftp-user
    ForceCommand internal-sftp
    PasswordAuthentication yes
    ChrootDirectory /home/ftp-user
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
```

## Vulnerable Web Applications (VM 102)

### DVWA — Damn Vulnerable Web Application

| Detail | Value |
|--------|-------|
| **Access** | Port 8080 |
| **Vulnerabilities** | SQL Injection, XSS (Reflected/Stored/DOM), Command Injection, File Upload, CSRF, Brute Force, File Inclusion |
| **Security Levels** | Low, Medium, High, Impossible |

### OWASP Juice Shop

| Detail | Value |
|--------|-------|
| **Access** | Port 3000 |
| **Challenges** | 100+ CTF-style challenges |
| **Coverage** | Full OWASP Top 10 (2021) |

```yaml
services:
  dvwa:
    image: vulnerables/web-dvwa
    restart: always
    ports:
      - "8080:80"

  juice-shop:
    image: bkimminich/juice-shop
    restart: always
    ports:
      - "3000:3000"
```

## Network Design Decisions

**Production isolation:** VM 100 (n8n-prod) is hardened — only necessary ports are exposed, and the SFTP server uses chroot jails to prevent lateral movement.

**Active Directory realism:** The Windows domain with intentional misconfigurations mirrors common enterprise security gaps, providing practice with real-world AD attack techniques.

**Centralized monitoring:** Wazuh agents on all target machines enable simultaneous offensive testing and defensive detection — the foundation of purple team operations.

**Zero-trust remote access:** Tailscale mesh VPN connects the entire lab without exposing any management interfaces to the public internet.

## Tools & Technologies

**Infrastructure:** Proxmox VE, Docker, Docker Compose, Tailscale, Debian Linux, Windows Server 2022

**SIEM & Detection:** Wazuh 4.14 (Manager, Indexer, Dashboard), Filebeat

**Offensive Security:** Kali Linux, Burp Suite, Nmap, Metasploit Framework, sqlmap, Gobuster, Hydra, John the Ripper, BloodHound, Impacket

**Active Directory:** AD DS, DNS, Group Policy, Kerberos

**Automation:** n8n workflow automation, Bash scripting, PowerShell

**Networking:** Bridged networking, WireGuard VPN tunneling, SFTP with chroot isolation

## Resource Allocation

Total host RAM: 32GB

| VM | Allocated | Status |
|----|-----------|--------|
| VM 100 — n8n-prod | 4GB | Stopped (started on demand) |
| VM 102 — vuln-web-apps | 2GB | Running |
| VM 103 — DC01 | 4GB | Running |
| VM 105 — wazuh-siem | 6GB | Running |
| **Active total** | **12GB** | Leaves 20GB headroom for Proxmox host |

## Related Repositories

- [`penetration-testing-writeups`](https://github.com/RelentIQ/penetration-testing-writeups) — Documented attack methodologies with OWASP/CWE mapping and remediation guidance
- [`security-automation-scripts`](https://github.com/RelentIQ/security-automation-scripts) — Recon, analysis, and automation tools

## Author

**Nikolay** — Aspiring Cybersecurity Professional

- Currently pursuing eJPT certification
- Completing TryHackMe offensive security and SOC analyst paths
- Practicing purple team operations through homelab exercises
- Documenting professional-grade writeups covering offensive methodology, SIEM detection, and remediation

## Disclaimer

This project is for **educational purposes only**. All testing is performed against systems I own or have explicit permission to test. The intentional misconfigurations in the Active Directory environment exist solely for authorized security testing practice.
