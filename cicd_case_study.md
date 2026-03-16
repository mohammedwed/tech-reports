# Self-Hosted CI/CD Infrastructure on Repurposed Hardware
## A Case Study in Constrained On-Premises DevOps

| | |
|---|---|
| **Client** | Meridian Training & Consulting |
| **Date** | March 15, 2026 |
| **Environment** | On-Premises / Self-Hosted LAN |
| **Audience** | Infrastructure Engineers, DevOps Teams |
| **Scope** | Design, deployment, and troubleshooting of a self-hosted CI/CD pipeline to support an internal software development team — using exclusively repurposed commodity hardware, open-source tooling, and no cloud spend. |

---

> ⚠️ **Privacy Notice**
>
> This case study has been prepared for knowledge-sharing and documentation purposes. All identifying information — including hostnames, IP addresses, domain names, personnel references, and organisational details — has been redacted or replaced with fictional equivalents to protect the privacy and operational security of the client. The company name "Meridian Training & Consulting" is a pseudonym. Any resemblance to real organisations, systems, or network configurations is coincidental. Reproduction or distribution of this document outside its intended audience is not permitted without prior authorisation.

---

## Table of Contents

1. [Background & Problem Statement](#1-background--problem-statement)
2. [Infrastructure](#2-infrastructure)
3. [Key Constraints](#3-key-constraints)
4. [Architectural Decisions](#4-architectural-decisions)
5. [Implementation Issues & Resolutions](#5-implementation-issues--resolutions)
6. [Limitations](#6-limitations)
7. [Conclusions](#7-conclusions)
8. [Further Work](#8-further-work)
9. [Appendix](#9-appendix)

---

## 1. Background & Problem Statement

Meridian Training & Consulting is a training and consultation firm whose internal development team builds and maintains proprietary course materials, client-facing tools, and internal management systems. As the team's software output grew, the absence of a formal version control and automated pipeline process was creating tangible inefficiencies: inconsistent build environments, manual deployment steps, and no systematic test enforcement before code reached shared environments.

The team required a version-controlled, automated build-and-test pipeline along with a suite of supporting productivity services — documentation, credential management, file sync, and internal DNS. The primary constraint was the absence of a budget for cloud infrastructure or new hardware. The objective was therefore to build a production-grade CI/CD environment from existing machines that had previously been used as personal or ad-hoc workstations.

A secondary objective was data sovereignty: given the sensitive nature of client engagement materials and internal tooling, all source code, credentials, and team artefacts must remain on infrastructure under direct organisational control, with no dependency on third-party SaaS platforms.

This case study documents the architecture decisions made, the constraints that shaped them, the problems encountered during implementation, and the current limitations of the resulting system.

---

## 2. Infrastructure

### 2.1 Hardware Fleet

| Host | Hardware Profile | RAM | OS | Assigned Role |
|------|-----------------|-----|----|---------------|
| **srv-01** | Intel Core (2015-gen, low-power laptop) | — | Ubuntu Server 24.04 (headless) | Primary service host — always on |
| **ws-01** | AMD Ryzen 32-core desktop, RTX-class GPU | 64 GB | Pop!\_OS | Developer workstation |
| **runner-01** | Intel Core i7 4th Gen, Nvidia GT (2 GB) | 8 GB | Ubuntu Server 24.04 (headless) | On-demand CI/CD build runner |
| **gw-01** | Intel i5 6th Gen (Skylake) desktop | 8 GB DDR3 | TBD | Future network gateway / firewall |

### 2.2 Network Addressing

| Host | IP Address | Primary Services |
|------|------------|-----------------|
| srv-01 | 10.10.0.10 | Forgejo, Caddy, AdGuard Home, full Docker stack |
| ws-01 | 10.10.0.20 | Developer workstation — no persistent services |
| runner-01 | 10.10.0.30 | Forgejo Actions runner (WoL-managed, off when idle) |

### 2.3 Service Inventory

All services run on **srv-01** as Docker containers, fronted by Caddy over a shared internal bridge network (`caddy_net`).

| Service | Internal URL | Role |
|---------|-------------|------|
| Forgejo | `https://git.corp.internal` | Git hosting, PR workflow, CI/CD triggers |
| Caddy | — | Reverse proxy, local TLS termination |
| AdGuard Home | — | Network-wide DNS filtering and resolution |
| Vaultwarden | `https://vault.corp.internal` | Team credential management |
| Nextcloud | `https://files.corp.internal` | File cloud, CalDAV, CardDAV |
| Outline | `https://docs.corp.internal` | Internal documentation wiki |
| Linkding | `https://bookmarks.corp.internal` | Shared link / resource library |
| Syncthing | — | Peer-to-peer file sync across nodes |

---

## 3. Key Constraints

Understanding the constraints is essential context for every architectural decision in this case study. They are not incidental — they are the primary drivers of the design.

| ID | Constraint |
|----|-----------|
| **C1** | **Zero hardware budget.** All machines are repurposed. No new servers, NICs, or networking hardware could be procured during the initial build phase. |
| **C2** | **No cloud spend.** All services must run on-premises. No managed CI/CD (GitHub Actions, CircleCI), no cloud storage, no hosted DNS. |
| **C3** | **Single always-on node has weak CPU.** The only machine suitable for 24/7 operation (srv-01) is a 2015-generation laptop-class processor. It cannot sustain CPU-intensive build workloads alongside its service hosting duties without degrading availability. |
| **C4** | **No UPS on the primary workstation.** ws-01 (the most powerful machine) is a desktop with no battery backup. It cannot be relied upon for any persistent service. |
| **C5** | **WiFi-only primary server.** srv-01 has no wired Ethernet connection to the LAN switch, which introduces SSH instability and rules it out as a Wake-on-LAN source via its own NIC. |
| **C6** | **No public domain at time of deployment.** A domain is available but DNS has not yet been transferred to a manageable provider, precluding publicly trusted TLS certificates during this phase. |
| **C7** | **Unrooted mobile devices.** Developer mobile devices cannot trust user-installed CA certificates at the system level, impacting access to internal HTTPS services from mobile clients. |

---

## 4. Architectural Decisions

### Decision 1 — srv-01 as the Sole Always-On Host

**Decision:** Designate srv-01 as the single node running all persistent services 24/7.

**Justification:** Of the available machines, srv-01 is the only one with characteristics suitable for continuous operation: low idle power draw (~5–10 W), fanless operation, compact form factor, and no desktop GPU drawing standby power. ws-01 is a high-power workstation desktop (no battery, no UPS) and runner-01 draws significantly more at idle.

**Trade-off accepted:** srv-01's CPU is weak. Build and test jobs cannot run on this node — they would cause service degradation across the entire stack. This is resolved by Decision 2.

### Decision 2 — On-Demand Build Runner via Wake on LAN

**Decision:** Designate runner-01 as a dedicated CI/CD worker that is powered off when idle and woken via a Wake on LAN (WoL) magic packet at pipeline trigger time, then shut down on job completion.

**Justification:** runner-01 has a meaningfully more capable CPU than srv-01 and 8 GB RAM, adequate for most build and test workloads. Running it 24/7 would consume unnecessary power and reduce hardware lifespan. WoL enables on-demand activation with no human intervention, keeping idle power near zero.

WoL requires a wired Ethernet connection. runner-01 satisfies this — it is connected via physical cable, unlike srv-01.

**Configuration:**

```bash
# runner-01: enable WoL on NIC
sudo ethtool -s eth0 wol g

# Persist via systemd unit: /etc/systemd/system/wol-enable.service
[Unit]
Description=Enable Wake on LAN
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -s eth0 wol g
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
# Wake runner-01 from srv-01
sudo apt install wakeonlan
wakeonlan <runner-01-mac-address>
```

**Pipeline flow:**

```
Developer push  →  Forgejo (srv-01) triggers Actions workflow
                →  WoL magic packet sent to runner-01
                →  runner-01 boots and registers with Forgejo runner daemon
                →  Job executes  (build / test / lint)
                →  runner-01 shuts down on completion
```

**Trade-off accepted:** There is a cold-start latency cost — runner-01 takes 30–60 seconds to boot and register before a job can start. This is acceptable for a small team pipeline; it would not be appropriate for latency-sensitive or high-frequency pipelines.

### Decision 3 — Forgejo as the Git and CI/CD Platform

**Decision:** Use Forgejo (a Gitea fork) as the self-hosted Git platform, with its native Forgejo Actions engine driving the CI/CD pipeline.

**Justification:** Forgejo satisfies several requirements simultaneously: Git hosting with a familiar web UI, pull request workflows, issue tracking, and a GitHub Actions-compatible pipeline syntax. The Actions compatibility is a significant practical benefit — existing workflow files can be adapted with minimal modification, and the team's familiarity with the GitHub Actions YAML format is immediately transferable.

**Alternatives considered:** GitLab CE was evaluated but ruled out due to its substantially higher resource requirements (typically 4+ GB RAM minimum for a usable instance), which would have saturated srv-01. Gitea itself was also considered, but Forgejo's governance model and active community were preferred.

**Runner registration:**

```bash
docker run -d \
  --name forgejo-runner \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v forgejo-runner-data:/data \
  -e FORGEJO_INSTANCE_URL=https://git.corp.internal \
  -e FORGEJO_RUNNER_REGISTRATION_TOKEN=<token-from-forgejo-admin> \
  code.forgejo.org/forgejo/runner:latest
```

**Example workflow** (`.forgejo/workflows/ci.yml`):

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: |
          # e.g. npm test, pytest, cargo test
          make test

      - name: Build
        run: |
          # e.g. npm run build, go build ./...
          make build
```

### Decision 4 — Caddy as the Reverse Proxy and TLS Authority

**Decision:** Use Caddy as the single reverse proxy for all internal services, leveraging its built-in ACME server to act as a local certificate authority.

**Justification:** Caddy's automatic TLS provisioning eliminates the operational overhead of manually managing certificates across services. Its `tls internal` directive issues signed certificates from its own CA root, enabling genuine HTTPS on the LAN without external dependencies. This is superior to plain HTTP, where credentials (Vaultwarden, Forgejo tokens) would traverse the LAN unencrypted.

All service containers share the `caddy_net` Docker bridge, allowing Caddy to reach them by container name — no host port exposure is required for most services.

```bash
docker network create caddy_net
```

**Caddyfile:**

```
git.corp.internal {
  tls internal
  reverse_proxy forgejo:3000
}

files.corp.internal {
  tls internal
  reverse_proxy nextcloud:80
  redir /.well-known/carddav /remote.php/dav 301
  redir /.well-known/caldav  /remote.php/dav 301
  header {
    Strict-Transport-Security max-age=31536000
  }
}

vault.corp.internal {
  tls internal
  reverse_proxy vaultwarden:80
}

docs.corp.internal {
  tls internal
  reverse_proxy outline:3000
}

bookmarks.corp.internal {
  tls internal
  reverse_proxy linkding:9090
}
```

Reload without downtime after any Caddyfile change:

```bash
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

**Trade-off accepted:** The internal CA root certificate must be manually distributed and trusted on each developer machine and CI runner. This is a one-time per-device operation but adds onboarding friction. See Limitation L3 for the CA trust problem on mobile devices.

### Decision 5 — AdGuard Home as the Internal DNS Resolver

**Decision:** Deploy AdGuard Home on srv-01 and configure the LAN's DHCP server to point all clients to it for DNS resolution.

**Justification:** A team-controlled DNS server is a prerequisite for internal hostnames (`*.corp.internal`) to resolve across the LAN without manual `/etc/hosts` entries on every device. AdGuard Home was chosen over plain Bind9 or dnsmasq because it provides DNS-over-HTTPS (DoH) upstream support and a blocklist-based filtering layer — both relevant to the team's network hygiene goals — with minimal configuration.

**Trade-off accepted:** srv-01 is a single point of failure for LAN-wide DNS. If srv-01 goes offline, all internal hostname resolution fails for every device on the network. A secondary DNS resolver is identified as a mitigation in Section 8.1.

---

## 5. Implementation Issues & Resolutions

### 5.1 Repository Migration — DNS Loop (ws-01 → srv-01)

**Context:** Forgejo was initially run on ws-01 for local testing. The migration to srv-01 required Forgejo (running in Docker) to fetch repositories from ws-01.

**Symptom:** Migration requests returned `{"ok":false,"error":"not_found"}` with Node.js/Helmet.js response headers — clearly not a Forgejo response.

**Root cause (layer 1):** The hostname `ws-01` had been added to srv-01's `/etc/hosts` with srv-01's own IP (`10.10.0.10`) rather than ws-01's IP (`10.10.0.20`). All outbound requests to `ws-01:3001` were looping back to srv-01's own Docker containers.

Confirmed via:

```bash
# Returned Helmet.js security headers — identified as Node.js, not Forgejo
curl -I http://ws-01:3001

# Showed docker-proxy bound to port 3001 on srv-01 — confirmed the loop
ss -tlnp | grep 3001
```

**Fix:**

```bash
echo "10.10.0.20  ws-01" | sudo tee -a /etc/hosts
```

**Root cause (layer 2):** Even after correcting `/etc/hosts`, Forgejo inside Docker could not resolve `ws-01` — containers do not inherit the host's `/etc/hosts`. ws-01's Forgejo `app.ini` had `ROOT_URL` set to the hostname, causing all clone URLs to embed it — unresolvable from inside the container network.

**Fix** — updated ws-01's `app.ini` to use its IP address directly:

```ini
[server]
DOMAIN     = 10.10.0.20
SSH_DOMAIN = 10.10.0.20
ROOT_URL   = http://10.10.0.20:3001/
```

Migration succeeded immediately. ws-01's Forgejo instance was then fully decommissioned.

**Lesson:** When migrating between Forgejo instances where either node runs in Docker, all `app.ini` server values must reference IP addresses (or Docker service names within the same network) rather than hostnames — container DNS resolution is isolated from the host.

### 5.2 Forgejo 502 Bad Gateway

**Symptom:** Forgejo returned a 502 immediately after Caddy configuration.

**Root cause:** A single-character typo in the Caddyfile — the reverse proxy target IP contained `10.18.0.10` instead of `10.10.0.10`. Caddy silently proxied to an unreachable host.

**Fix:**

```bash
# After correcting IP in Caddyfile
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

**Lesson:** Caddyfile IP literals should be validated against `ip addr` output before reload. Moving from `docker run` to Docker Compose with container-name-based proxy targets eliminates this class of error entirely.

### 5.3 SSH Instability — WiFi Power Management

**Symptom:** Intermittent SSH session drops to srv-01; latency spikes ranging from 1 ms to 298 ms on an otherwise idle connection.

**Root cause:** The Linux WiFi power management subsystem was duty-cycling the adapter during low-traffic periods, causing the NIC to briefly enter a sleep state mid-session.

**Fix:**

```bash
# Immediate — disable power management on the WiFi interface
sudo iwconfig wlp3s0 power off

# Persistent — override via NetworkManager configuration
echo "[connection]
wifi.powersave = 2" | sudo tee /etc/NetworkManager/conf.d/wifi-powersave-off.conf

sudo systemctl restart NetworkManager
```

**Result:** SSH latency stabilised at 3–4 ms with no further disconnects. ✅

**Lesson:** WiFi power management defaults are optimised for battery life, not server availability. Any server expected to maintain persistent inbound connections over WiFi must have power management explicitly disabled — it is not disabled by default on Ubuntu Server.

### 5.4 Server Boot Hang — `systemd-networkd-wait-online`

**Symptom:** srv-01 stalled for approximately two minutes on every reboot, blocking all service startup.

**Root cause:** `systemd-networkd-wait-online.service` was waiting indefinitely for a network interface that never reached a fully operational state, likely due to WiFi association timing.

**Fix** (executed from recovery mode):

```bash
mount -o remount,rw /
systemctl disable systemd-networkd-wait-online
```

**Lesson:** On systems where network availability at boot is not guaranteed (WiFi-connected servers, machines with intermittent interfaces), `systemd-networkd-wait-online` should be disabled or configured with a strict timeout to prevent cascading service startup failures.

### 5.5 Nextcloud HTTP/HTTPS Protocol Mismatch

**Symptom:** After enabling Caddy's HTTPS proxy for Nextcloud, the application generated internal URLs using `http://`, causing redirect loops and mixed-content browser errors.

**Root cause:** Nextcloud was unaware it was running behind a TLS-terminating proxy and generated URLs based on the HTTP connection it received from Caddy on its internal port.

**Fix:**

```bash
docker exec -u www-data nextcloud php occ config:system:set overwriteprotocol --value="https"
docker restart nextcloud
```

**Lesson:** Any application that generates its own URLs (Nextcloud, Outline, Vaultwarden) must be explicitly told its externally visible protocol when deployed behind a reverse proxy. This is a common misconfiguration category worth checking systematically when onboarding new services behind Caddy.

### 5.6 DNS Resolution Failure for Internal Hostnames (ws-01)

**Symptom:** `nslookup git.corp.internal` on ws-01 returned `SERVFAIL`, despite AdGuard Home on srv-01 correctly resolving the name when queried directly.

**Root cause:** `systemd-resolved` on ws-01 was not forwarding the `corp.internal` domain to the correct upstream nameserver. The `resolved.conf` file had all directives commented out, leaving the stub resolver with no domain routing rules.

**Immediate workaround:**

```bash
echo "10.10.0.10  srv-01 git.corp.internal" | sudo tee -a /etc/hosts
```

**Long-term fix** — configure domain forwarding in `resolved.conf`:

```ini
# /etc/systemd/resolved.conf
[Resolve]
DNS=10.10.0.10
Domains=corp.internal
DNSSEC=no
```

**Lesson:** `systemd-resolved` requires explicit domain-to-nameserver routing rules to forward non-public domains to an internal resolver. The default configuration forwards everything to the globally configured DNS servers, which have no knowledge of `*.corp.internal` names.

---

## 6. Limitations

The following limitations are present in the current state of the system. They are acknowledged constraints rather than overlooked deficiencies, most stemming directly from the hardware and budget constraints established in Section 3.

| ID | Severity | Limitation |
|----|----------|-----------|
| **L1** | 🔴 High | **Single point of failure (srv-01).** All services, DNS resolution, and the CI/CD trigger layer run on a single host. Any srv-01 downtime takes down the entire stack. There is currently no standby node or failover mechanism. |
| **L2** | 🟡 Medium | **WiFi-only primary server.** While SSH instability has been resolved (§5.3), WiFi remains a less reliable medium than wired Ethernet. Packet loss, interference, or access point issues could intermittently affect all services. |
| **L3** | 🟡 Medium | **Internal CA certificates not universally trusted.** Caddy's `tls internal` certificates are not trusted by applications that bypass the OS certificate store — notably Vaultwarden clients on unrooted Android. Team members cannot use the mobile credential vault over HTTPS without device rooting. Consequence of constraint C6. |
| **L4** | 🔴 High | **No remote access.** WireGuard VPN has not yet been configured. All services are accessible only from within the LAN, limiting the pipeline's utility for any distributed or remote working scenario. |
| **L5** | 🔴 High | **No automated backups.** Docker volumes containing repository data, Nextcloud files, Outline documents, and Vaultwarden credentials are not currently backed up. A hardware failure on srv-01 would result in total data loss. |
| **L6** | 🟢 Low | **Runner cold-start latency.** The WoL-based runner introduces a 30–60 second boot delay before any CI job executes. Acceptable at current pipeline frequency; would become problematic at higher throughput. |
| **L7** | 🟡 Medium | **No pipeline observability.** There is no centralised log aggregation, alerting, or uptime monitoring. Service failures are detected reactively rather than proactively. |
| **L8** | 🟡 Medium | **Container configuration not version-controlled.** Services are deployed via `docker run` commands rather than Docker Compose files tracked in Forgejo. The infrastructure state is opaque and difficult to reproduce or audit. |

---

## 7. Conclusions

The deployment demonstrates that a functional, self-hosted CI/CD environment — with version control, automated pipelines, credential management, internal documentation, and file synchronisation — can be built from repurposed commodity hardware at effectively zero cost, using exclusively open-source tooling.

Several architectural patterns proved effective under the given constraints:

**Concentrating persistent services on the lowest-power node** (srv-01) rather than the most powerful one is the correct approach when energy efficiency and uptime are prioritised. Power capacity does not equal server suitability.

**Wake-on-LAN as a compute scheduling mechanism** is a practical, low-friction way to make intermittently available hardware useful for CI workloads without the cost of running it continuously.

**Caddy's automatic internal TLS** significantly reduced the complexity of securing internal services compared to managing certificates manually. The trade-off — CA distribution friction and mobile app incompatibility — is real but manageable at small team scale.

**Shared Docker bridge networking** (all services on `caddy_net`) cleanly separates container-to-container communication from host-level port exposure, and simplifies the Caddyfile to container-name-based proxy targets.

The implementation incidents documented in Section 5 highlight a recurring theme: **the boundary between Docker container networking and the host network** is a reliable source of subtle failures. DNS resolution, hostname rewriting, and certificate trust all behave differently inside a container than on the host — deployments that span both contexts require explicit attention to this boundary.

The system is functional and supports the team's day-to-day development workflow. However, it should be considered a first-phase foundation rather than a production-hardened environment. The high-severity limitations (L1, L4, L5) — single point of failure, no remote access, and no automated backups — represent meaningful operational risk that must be addressed before the team relies on this infrastructure for critical or time-sensitive work.

---

## 8. Further Work

### 8.1 Immediate — Operational Risk Reduction

These items address the high-severity limitations from Section 6 and should be completed before the infrastructure is considered reliable for production use.

- **Restic automated backups** — Schedule nightly backups of all Docker volumes (Forgejo repositories, Nextcloud data, Outline database, Vaultwarden data) to a secondary node or external storage. This is the single highest-priority item: a disk failure on srv-01 currently means total data loss. *(Addresses L5)*
- **WireGuard VPN on srv-01** — Enable remote developer access to all internal services. Required before the pipeline can support any distributed or remote working scenario. *(Addresses L4)*
- **Secondary DNS resolver** — Deploy a minimal dnsmasq or Unbound instance on a second machine to act as a DNS fallback, eliminating the network-wide single point of failure. *(Addresses L1 partially)*

### 8.2 Short-Term — Pipeline Maturity

- **Register Forgejo Actions runner on runner-01** — Complete the WoL trigger integration: wake on pipeline start, shutdown hook on job completion. *(Addresses L6)*
- **Migrate `docker run` deployments to Docker Compose** — All service configurations should be expressed as `docker-compose.yml` files and committed to a private repository on Forgejo, making the infrastructure reproducible and auditable. *(Addresses L8)*
- **Deploy Uptime Kuma** — Provide proactive service health monitoring with alerting, replacing reactive failure detection. *(Addresses L7)*
- **Write canonical workflow templates** — Produce base CI workflow files for the languages/frameworks in active use (Node.js, Python, Go) so new repositories have a working pipeline from day one.

### 8.3 Medium-Term — Security & Access Hardening

- **Transfer DNS to Cloudflare and configure DNS-01 challenge** — A `.org` domain is already registered. Using Caddy's `caddy-dns/cloudflare` plugin for DNS-01 ACME challenges will replace the internal CA with publicly trusted Let's Encrypt certificates, resolving the mobile certificate trust problem without device rooting. *(Addresses L3)*
- **Enable DNS-over-HTTPS on AdGuard Home** — Configure Cloudflare (`https://dns.cloudflare.com/dns-query`) and Quad9 (`https://dns.quad9.net/dns-query`) as encrypted upstream resolvers.
- **Scope runner registration tokens per-repository** — Avoid organisation-wide runner tokens; scope tokens to specific repositories to limit the blast radius of a compromised credential.
- **Rotate all default passwords in Docker environment variables** — Several services were deployed with placeholder credentials. These must be replaced and stored in Vaultwarden.

### 8.4 Long-Term — Infrastructure Expansion

- **Deploy OPNsense on gw-01** — gw-01 (Skylake i5, 8 GB DDR3) is well-suited for a dedicated firewall role, requiring only a second PCIe NIC. OPNsense would provide hardware-enforced network segmentation, IDS/IPS via Suricata, and a purpose-built WireGuard server — removing those responsibilities from srv-01.
- **FileBrowser on srv-01** — Lightweight web-based file management for team members who need ad-hoc file access without a full Nextcloud client.
- **Additional Forgejo Actions runners** — As the team grows or build workloads increase, additional runner nodes can be registered with Forgejo. The WoL pattern established for runner-01 is directly repeatable on any wired Ethernet-capable machine.

---

## 9. Appendix

### A — Docker Network Architecture

```
                     ┌──────────────────────────────────────┐
                     │               srv-01                 │
                     │            (10.10.0.10)              │
                     │                                      │
                     │   ┌──────────────────────────────┐   │
 LAN clients ──────► │   │         caddy_net            │   │
 (HTTPS :443)        │   │     (Docker bridge)          │   │
                     │   │                              │   │
                     │   │  caddy  ◄── TLS termination  │   │
                     │   │    │                         │   │
                     │   │    ├──► forgejo:3000          │   │
                     │   │    ├──► nextcloud:80          │   │
                     │   │    ├──► vaultwarden:80        │   │
                     │   │    ├──► outline:3000          │   │
                     │   │    └──► linkding:9090         │   │
                     │   └──────────────────────────────┘   │
                     │                                      │
                     │   adguard-home  (port 53 / DoH)      │
                     │   syncthing     (LAN sync)           │
                     └──────────────────────────────────────┘
                                        ▲
                              Forgejo Actions trigger
                                        │
                     ┌──────────────────────────────────────┐
                     │              runner-01               │
                     │            (10.10.0.30)              │
                     │                                      │
                     │   Powered off at idle                │
                     │   Woken via WoL on pipeline trigger  │
                     │   Shuts down on job completion       │
                     │                                      │
                     │   forgejo-runner  (Actions worker)   │
                     └──────────────────────────────────────┘
```

### B — Caddy CA Certificate Distribution

The Caddy internal root certificate must be installed on each developer machine and CI runner to establish TLS trust.

**Certificate location on srv-01:**

```
/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt
```

**Install on Ubuntu / Pop!\_OS:**

```bash
sudo cp caddy-local-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

**Firefox** (does not inherit the OS certificate store):
Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import

**Android (unrooted):** Not resolvable via user CA installation. Requires publicly trusted certificates via DNS-01 challenge (see §8.3) or a device that supports per-app CA trust.

### C — Forgejo `app.ini` Reference (srv-01)

```ini
[server]
DOMAIN       = git.corp.internal
SSH_DOMAIN   = git.corp.internal
ROOT_URL     = https://git.corp.internal/

[migrations]
ALLOW_LOCALNETWORKS = true
ALLOWED_DOMAINS     = ws-01
```

---

*Case study compiled: March 15, 2026*
*Prepared by the Infrastructure Team — Meridian Training & Consulting (pseudonym)*
*All identifying information has been redacted. See Privacy Notice.*
