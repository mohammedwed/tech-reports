# Nova: System Setup Report
## Hardware, OS Configuration & Network Infrastructure

**Setup Date:** February 15, 2026  
**Status:** Operational  
**System Name:** Nova  
**Location:** Local Network

---

## Executive Summary

Nova is a freshly configured multi-purpose Linux server built on high-end hardware, designed to support four primary workloads: software development, application hosting, CI/CD testing infrastructure, and Large Language Model (LLM) serving.

This report documents the initial system setup, hardware architecture, operating system selection rationale, network configuration, and lessons learned during the deployment process.

---

## 1. Hardware Architecture

### 1.1 System Specifications

| Component | Specification | Justification |
|-----------|---------------|---------------|
| **CPU** | AMD Ryzen 9 7900X (12-core, 24-thread) | Repurposed from existing hardware; excellent multi-core performance for parallel workloads |
| **Base Clock** | 3.7 GHz | Sufficient for baseline operations |
| **Boost Clock** | 5.7 GHz | High single-thread performance for LLM inference optimization |
| **RAM** | 62 GiB DDR5-4800 MT/s | Ample capacity for 4 concurrent workloads without memory pressure |
| **GPU** | Nvidia RTX 4060 | 6GB VRAM; purpose-fit for quantized LLM inference |
| **Storage** | 931.5 GiB NVMe SSD | Fast I/O for development builds, CI/CD artifacts, model caching |
| **Virtualization** | AMD-V enabled | Support for containerization and future VM workloads |

### 1.2 Hardware Synergy Analysis

The hardware combination is well-balanced for Nova's multi-purpose design:

- **CPU-GPU Balance:** 12 cores handle parallel tasks while RTX 4060 accelerates LLM inference without bottlenecking either component
- **RAM-Workload Fit:** 62 GiB total capacity with strategic allocation:
  - LLM inference: 5-8 GB (system RAM + GPU VRAM)
  - Development environment: 8-10 GB
  - CI/CD pipeline: 10-15 GB (parallel builds)
  - Application hosting: 10-15 GB
  - System overhead & buffer: 15+ GB
- **Storage Capacity:** 931.5 GiB provides >900 GB free after OS/tools installation, suitable for models, build artifacts, and application data

### 1.3 Hardware Vulnerabilities & Mitigations

The Ryzen 9 7900X includes modern CPU security mitigations:
- Spectre v1/v2: Mitigated via IBRS/STIBP
- Meltdown: Not affected
- TSA: Vulnerable (no microcode), but mitigated in practice on current systems

No hardware modifications required for the local-network deployment model.

---

## 2. Operating System Selection & Configuration

### 2.1 OS Choice: Pop!_OS

**Selected:** Pop!_OS (latest stable)

**Rationale:**
- **Nvidia Driver Support:** Primary consideration; Pop!_OS provides excellent out-of-the-box Nvidia GPU support critical for RTX 4060 integration
- **Developer-Friendly:** Pre-configured packages and tooling for development environments
- **Ubuntu Base:** Leverages Ubuntu ecosystem stability while adding developer features
- **Active Community:** Strong community support for development and server use cases

**Alternatives Considered & Rejected:**
- Ubuntu Server: More minimal, fewer pre-configured dev tools
- Debian: Older package versions, less Nvidia optimization
- CentOS/RHEL: Enterprise-focused, less suitable for development

### 2.2 Installation Configuration

**Encryption:** Disabled during OS setup

**Rationale:**
- Local network access only (low security risk on deployment date)
- Performance optimization (eliminate encryption overhead)
- Simplicity during initial setup and testing phase
- Intentional trade-off: reconsider if network exposure changes to internet-facing

**Note:** Encryption migration path identified; can be enabled at later stage if requirements change.

### 2.3 Network Configuration

**Network Scope (Current):** Local network only

**IP Address:** [REDACTED - Local IP]

**Hostname:** Nova

**Network Accessibility:** SSH (port 22), Ollama API (port 11434), Web UI (port 3000)

All services configured for intra-network access only. No internet exposure planned initially.

---

## 3. Network Infrastructure & Access Configuration

### 3.1 Firewall Configuration

**Firewall Tool:** UFW (Uncomplicated Firewall)

**Allowed Ports:**

| Port | Service | Protocol | Purpose |
|------|---------|----------|---------|
| 22 | SSH | TCP | Remote administration |
| 11434 | Ollama API | TCP | LLM inference API |
| 3000 | Open WebUI | TCP | Web interface (via Nginx reverse proxy) |
| 80 | HTTP (Nginx) | TCP | Reverse proxy for web interface |

**Configuration Commands:**
```bash
sudo ufw enable
sudo ufw allow 22/tcp
sudo ufw allow 11434/tcp
sudo ufw allow 3000/tcp
sudo ufw allow 80/tcp
```

### 3.2 Local DNS Resolution

**Method 1: Client-Side /etc/hosts (Implemented)**

On client machines accessing Nova:
```
[REDACTED - Local IP]    nova.ai
```

**Benefit:** Works immediately on client machines; no server-side DNS infrastructure needed for local-only access.

**Future Enhancement:** Deploy local DNS server (dnsmasq) if network-wide resolution needed across multiple machines.

### 3.3 Nginx Reverse Proxy Configuration

**Purpose:** Expose Open WebUI on standard port 80 under domain `nova.ai`

**Configuration File:** `/etc/nginx/sites-available/nova`

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name nova.ai;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Key Features:**
- WebSocket support (Upgrade header forwarding)
- Real IP tracking for upstream services
- HTTPS-ready (proto forwarding)

**Verification:**
```bash
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 3.4 Remote Access Strategy

**Current:** Local network only

**Future:** Planning for secure remote SSH access; encryption will be reconsidered before internet exposure.

**Planned Approach:** SSH key-based authentication with firewall rules restricting access to known IP ranges.

---

## 4. System Setup & Initialization

### 4.1 Installation Steps

1. **Pop!_OS Installation:** Standard installation with no encryption
2. **System Updates:**
   ```bash
   sudo apt update
   sudo apt upgrade
   ```
3. **Essential Tools Installation:**
   ```bash
   sudo apt install openssh-server ufw nginx curl wget git
   ```
4. **Firewall Setup:** UFW rules configured (see Section 3.1)
5. **Nginx Installation & Configuration:** Reverse proxy setup (see Section 3.3)

### 4.2 User Configuration

**Primary User:** Standard user account (non-root)

**Service User:** All systemd services run as primary user (not root)

**Rationale:** Non-root service execution prevents privilege escalation risks and ensures proper file access for local development environments.

### 4.3 System Uptime & Stability

- **Initial Uptime:** 25 minutes at setup completion
- **Current Status:** Stable; services configured for auto-restart and boot persistence

---

## 5. Lessons Learned

### 5.1 User Permissions & Service Isolation

**Critical Issue Encountered:** Services installed as root created files in root-owned directories; when systemd services later ran as non-root user, permission errors occurred.

**Resolution:** 
- Reconfigure systemd service files with `User=[non-root user]`
- Set proper ownership of application data directories
- Ensure consistent user context across service lifecycle

**Takeaway:** Establish user context early; avoid privilege switching during service operation.

### 5.2 Encryption vs. Performance Trade-off

**Decision:** Skip encryption during OS setup for development environment

**Rationale:** Local network, controlled environment, performance optimization priority

**Future Consideration:** Document encryption migration path for if/when network exposure increases

### 5.3 Network Configuration Approach

**Iterative Process:** Started with terminal-only access, then added:
1. SSH for remote administration
2. API access (port 11434)
3. Web interface (port 3000)
4. Reverse proxy (Nginx)
5. Local DNS resolution

**Lesson:** Plan network access strategy incrementally; avoid over-provisioning security for development phase.

---

## 6. Current System State

### 6.1 Services Status

- **SSH Server:** Active, listening on port 22
- **Firewall:** Enabled, rules configured
- **Nginx:** Active, reverse proxy operational
- **System:** Fully initialized and ready for development/hosting workloads

### 6.2 Performance Baseline

- **CPU Utilization:** Minimal (idle state)
- **RAM Utilization:** <10% before workloads
- **Storage Utilization:** 1.6% (915+ GB free)
- **Network Latency:** Sub-millisecond (local network)

---

## 7. Future Considerations

### 7.1 Network Expansion

- **Internet Exposure:** Enable encryption before exposing to internet
- **Remote SSH Access:** Implement key-based authentication
- **Local DNS Server:** Deploy dnsmasq if multi-machine DNS needed
- **VPN/Tunneling:** Consider for secure remote access

### 7.2 Storage & Scaling

- Current 931 GiB sufficient for foreseeable workloads
- Monitor usage as LLM models and CI/CD artifacts grow
- NVMe SSD performance adequate for all planned workloads

### 7.3 Monitoring & Observability

- Currently minimal monitoring
- Future: Implement system metrics collection (CPU, RAM, disk, network)
- Consider centralized logging if multiple services proliferate

---

## 8. References & Resources

- Pop!_OS Documentation: https://pop.system76.com/docs/
- UFW Firewall Guide: https://help.ubuntu.com/community/UFW
- Nginx Reverse Proxy: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- AMD Ryzen Security: https://www.amd.com/en/technologies/cpu-security

---

## Appendix A: System Information Commands

Useful commands for reproducing this configuration:

```bash
# Hardware information
lscpu
dmidecode -t memory
lsblk -d -o name,rota,size,type
nvme list

# Network configuration
hostname -I
ip addr show
sudo ufw status
sudo systemctl status nginx

# Service status
sudo systemctl status ollama.service
sudo systemctl status open-webui.service

# System utilization
top
df -h
free -h
```

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Status:** Final  
**Sensitive Information:** Redacted (IP addresses, internal hostnames)
