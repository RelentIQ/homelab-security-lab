# Homelab Security Lab

A virtualized security testing environment built on Proxmox VE, combining production automation services with isolated penetration testing targets. This project demonstrates practical skills in virtualization, network security, containerization, and offensive security methodology.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Proxmox VE 9.1 Host                              │
│                     (Intel iMac — i5, 32GB RAM, 500GB SSD)               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌───────────────┐  │
│  │  VM 100              │  │  VM 101              │  │  VM 102       │  │
│  │  n8n-prod            │  │  metasploitable2     │  │  vuln-web-apps│  │
│  │                      │  │                      │  │               │  │
│  │  • n8n (Docker)      │  │  • Metasploitable 2  │  │  • DVWA       │  │
│  │  • SFTP Server       │  │  • Classic network   │  │    (Docker)   │  │
│  │  • Shopify file      │  │    exploitation      │  │  • Juice Shop │  │
│  │    transfers         │  │    targets           │  │    (Docker)   │  │
│  │  • 4GB RAM, 64GB     │  │  • 512MB RAM         │  │  • 2GB RAM    │  │
│  └──────────┬───────────┘  └──────────┬───────────┘  └──────┬────────┘  │
│             │                         │                      │           │
│             └────────────┬────────────┴──────────────────────┘           │
│                          │                                               │
│                 ┌────────▼────────┐                                      │
│                 │     vmbr0       │                                      │
│                 │  (Bridge Net)   │                                      │
│                 └────────┬────────┘                                      │
│                          │                                               │
└──────────────────────────┼───────────────────────────────────────────────┘
                           │
                  ┌────────▼────────┐
                  │   Tailscale     │
                  │   Mesh VPN      │
                  └────────┬────────┘
                           │
             ┌─────────────┴──────────────┐
             │                            │
       ┌─────▼─────┐              ┌───────▼───────┐
       │  Kali      │              │  Remote       │
       │  Laptop    │              │  Management   │
       │  (M2 Pro   │              │  Access       │
       │  MacBook)  │              │               │
       └────────────┘              └───────────────┘
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
| **Use Case** | Secure remote management and penetration testing from work network |

The attack laptop (Kali on MacBook Pro) connects to the lab over the Tailscale mesh from a separate physical network, simulating real-world remote penetration testing engagement conditions.

### Virtual Machines

| VM ID | Name | Purpose | RAM | Disk | Key Services |
|-------|------|---------|-----|------|-------------|
| 100 | n8n-prod | Production automation | 4GB | 64GB | n8n (Docker), SFTP server |
| 101 | metasploitable2 | Network exploitation target | 512MB | — | Intentionally vulnerable Linux services |
| 102 | vuln-web-apps | Web application testing | 2GB | 32GB | DVWA (Docker), Juice Shop (Docker) |

## Production Services (VM 100)

### n8n Workflow Automation

Production automation platform running in Docker, handling business-critical workflows including Shopify data processing and file transfers.

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
      - N8N_RESTRICT_FILE_ACCESS_TO=/home/ftp
      - N8N_RUNNERS_TASK_TIMEOUT=36000
    volumes:
      - ./n8n-data:/home/node/.n8n
      - /home/ftp:/home/ftp
```

**Key configuration notes:**
- `N8N_RESTRICT_FILE_ACCESS_TO` — Required in newer n8n versions to whitelist file paths for read/write operations
- `N8N_RUNNERS_TASK_TIMEOUT=36000` — Extended timeout (10 hours) for long-running data processing workflows; default 60s is insufficient for large dataset operations

### SFTP Server

Secure file transfer service with chroot jail isolation for the Shopify integration user.

**SSH configuration (`/etc/ssh/sshd_config`):**

```
Match User ftp
    ForceCommand internal-sftp
    PasswordAuthentication yes
    ChrootDirectory /home/ftp
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
```

This configuration restricts the `ftp` user to SFTP-only access within a chroot jail — they cannot execute shell commands, forward ports, or escape the designated directory. External access is provided via port forwarding on the network router.

## Vulnerable Targets

### DVWA — Damn Vulnerable Web Application (VM 102)

- **Access:** Port 8080
- **Default credentials:** admin / password
- **Vulnerabilities covered:**
  - SQL Injection (Union-based, Blind)
  - Cross-Site Scripting (Reflected, Stored, DOM)
  - Command Injection
  - File Upload vulnerabilities
  - CSRF (Cross-Site Request Forgery)
  - Brute Force authentication attacks
  - File Inclusion (Local/Remote)
- **Security levels:** Low, Medium, High, Impossible (for studying secure code)

### OWASP Juice Shop (VM 102)

- **Access:** Port 3000
- **Challenge-based:** 100+ CTF-style challenges
- **Covers:** Full OWASP Top 10 (2021)
  - Broken Access Control
  - Cryptographic Failures
  - Injection
  - Insecure Design
  - Security Misconfiguration
  - Vulnerable and Outdated Components
  - Identification and Authentication Failures
  - Software and Data Integrity Failures
  - Security Logging and Monitoring Failures
  - Server-Side Request Forgery (SSRF)

### Metasploitable 2 (VM 101)

- **Default credentials:** msfadmin / msfadmin
- **Vulnerabilities covered:**
  - Vulnerable network services (FTP, SSH, Telnet, Samba)
  - Web application vulnerabilities (TWiki, phpMyAdmin)
  - Misconfigured services
  - Privilege escalation vectors

## Network Design Decisions

**Production isolation:** VM 100 (n8n-prod) runs on the same bridge network but is hardened — only necessary ports are exposed, and the SFTP server uses chroot jails to prevent lateral movement.

**Attack surface management:** Vulnerable VMs (101, 102) are intentionally exposed on the internal network only. Tailscale provides encrypted access without exposing management interfaces to the public internet.

**Dual-purpose architecture:** The lab maintains production uptime for business-critical automation while providing a realistic attack environment. This mirrors enterprise environments where security teams must test without disrupting operations.

## Tools & Technologies

**Infrastructure:** Proxmox VE, Docker, Docker Compose, Tailscale, Debian Linux

**Offensive Security:** Kali Linux, Burp Suite, Nmap, Metasploit Framework, sqlmap, Gobuster, Hydra, John the Ripper

**Automation:** n8n workflow automation, Bash scripting

**Networking:** Bridged networking, VPN tunneling, SFTP with chroot isolation, port forwarding

## Setup Guide

### Prerequisites
- Intel-based machine with 16GB+ RAM (32GB recommended)
- Ethernet connection (WiFi drivers may not be supported in Proxmox)
- USB drive for Proxmox installer

### Installation Steps
1. Download Proxmox VE ISO from [proxmox.com](https://www.proxmox.com/en/downloads)
2. Flash to USB and install on bare metal (replaces existing OS)
3. Access web UI at `https://<host-ip>:8006`
4. Create VMs and allocate resources per the table above
5. Install Docker on Debian VMs and deploy containers
6. Install and configure Tailscale on each VM for remote access
7. Configure SFTP with chroot jails for file transfer services

### Docker Setup for Vulnerable Apps (VM 102)

```bash
# Install Docker
apt update && apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable" > /etc/apt/sources.list.d/docker.list
apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Deploy vulnerable applications
cat > docker-compose.yml << 'EOF'
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
EOF

docker compose up -d
```

## Future Enhancements

- [ ] Add Windows Server VM for Active Directory attack scenarios
- [ ] Implement network segmentation with pfSense/OPNsense firewall
- [ ] Deploy SIEM (Wazuh) for blue team detection and monitoring practice
- [ ] Add more vulnerable applications (WebGoat, HackTheBox retired machines)
- [ ] Expand penetration test writeups with documented attack chains and remediation

## Related Repositories

- [`penetration-testing-writeups`](https://github.com/relentiq/penetration-testing-writeups) — Documented attack methodologies against this lab and TryHackMe/eJPT exercises
- [`security-automation-scripts`](https://github.com/relentiq/security-automation-scripts) — Recon, analysis, and automation tools

## Author

**Nikolay** — Security Enthusiast & Automation Developer
- Currently pursuing eJPT certification
- Practicing offensive security through TryHackMe paths and homelab exercises
- Documenting methodology and remediation strategies for portfolio development

## Disclaimer

This project is for **educational purposes only**. All testing is performed against systems I own or have explicit permission to test. Never test against systems without authorization.
