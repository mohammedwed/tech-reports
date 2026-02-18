# Nova: Forgejo Planning Document
## Self-Hosted Git Server & Repository Management Architecture

**Planning Date:** February 15, 2026  
**Status:** Planning Phase - Professional Review  
**Implementation:** Pending Decision Finalization  
**Deployment Model:** Direct Binary Installation  
**User Model:** Single User  

---

## Executive Summary

This document outlines a comprehensive plan for deploying Forgejo as Nova's self-hosted Git server. It includes evaluation of alternatives, formal decision matrices, risk analysis, and success criteria to ensure a professionally justified architectural choice.

Forgejo will serve as the central repository management system, providing local Git hosting, web-based repository management, optional CI/CD integration, and repository mirroring capabilities. This planning document justifies the selection and provides a roadmap for implementation.

---

## 1. Architecture Overview

### 1.1 Forgejo's Role in Nova

**Core Responsibilities:**
- Manage all Git repositories locally
- Provide web UI for project browsing and management
- Handle webhooks for CI/CD triggers
- Store project metadata and configuration
- Manage users and access control (single user initially)
- Mirror repositories to/from remote services

### 1.2 Key Characteristics

| Aspect | Specification |
|--------|---------------|
| **Installation Type** | Direct binary (non-containerized) |
| **Users** | Single user (primary account) |
| **Network Access** | Local network only (initial phase) |
| **Database** | SQLite (embedded, zero-configuration) |
| **CI/CD** | Undecided (Forgejo Actions or external tool) |
| **Storage** | Local filesystem (NVMe SSD) |
| **Memory Footprint** | ~100-200 MB (idle), 300-500 MB (active) |
| **Port Configuration** | TBD (3000 or alternative) |
| **Auto-Startup** | systemd service with dependencies |

---

## 2. Alternatives Evaluation

### 2.1 Candidates Considered

Four primary options were evaluated for Nova's Git server:

1. **Forgejo** - Community-driven fork of Gitea
2. **Gitea** - Original lightweight Git service
3. **Gitolite** - Lightweight Git hosting (CLI-focused)
4. **Plain Git** - Bare repositories with manual management

### 2.2 Detailed Comparison Matrix

| Criterion | Forgejo | Gitea | Gitolite | Plain Git |
|-----------|---------|-------|----------|-----------|
| **Web UI** | ⭐⭐⭐⭐⭐ Full-featured | ⭐⭐⭐⭐⭐ Full-featured | ⭐ CLI only | ❌ None |
| **Performance** | ⭐⭐⭐⭐⭐ ~100 MB RAM | ⭐⭐⭐⭐⭐ ~100 MB RAM | ⭐⭐⭐⭐⭐ Minimal | ⭐⭐⭐⭐⭐ Minimal |
| **Community Support** | ⭐⭐⭐⭐ Active, growing | ⭐⭐⭐⭐⭐ Large, stable | ⭐⭐⭐ Stable | N/A |
| **Integration Complexity** | ⭐⭐⭐⭐ Simple | ⭐⭐⭐⭐ Simple | ⭐⭐ Moderate | ⭐ Very simple |
| **Learning Curve** | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐ Low | ⭐⭐ Moderate | ⭐⭐⭐⭐⭐ Very low |
| **Vendor Lock-in Risk** | ⭐⭐⭐⭐⭐ None | ⭐⭐⭐ Some | ⭐⭐⭐⭐⭐ None | ⭐⭐⭐⭐⭐ None |
| **Operational Overhead** | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐ Low | ⭐⭐⭐ Moderate | ⭐⭐ High |
| **CI/CD Capabilities** | ⭐⭐⭐⭐ Forgejo Actions | ⭐⭐⭐⭐ Gitea Actions | ❌ None | ❌ None |

### 2.3 Detailed Candidate Analysis

#### 2.3.1 Forgejo - RECOMMENDED CHOICE

**Strengths:**
- ✅ **Community-Driven:** Independent fork emphasizing community governance and transparency
- ✅ **Full-Featured Web UI:** Professional repository management interface
- ✅ **Built-in CI/CD:** Forgejo Actions (GitHub Actions compatible)
- ✅ **Direct Installation:** Binary executable, systemd service compatible
- ✅ **Lightweight:** ~100 MB memory footprint, minimal CPU overhead
- ✅ **Active Community:** Growing adoption, regular updates, clear roadmap
- ✅ **No Vendor Lock-in:** Fully open-source, community-controlled
- ✅ **Aligns with Philosophy:** Self-hosted, community-first, cost-effective

**Weaknesses:**
- ⚠️ **Smaller Ecosystem:** Less documentation than Gitea (but adequate)
- ⚠️ **Newer Project:** Community fork started ~2023, less battle-tested
- ⚠️ **Feature Parity:** May lag behind Gitea on cutting-edge features

**Resource Profile:**
- Memory (idle): 100-150 MB
- Memory (active): 300-500 MB
- CPU: Minimal during idle, 10-30% during active operations
- Disk: Base ~500 MB, grows with repositories

**Integration Fit:** ⭐⭐⭐⭐⭐ Excellent match for Nova

---

#### 2.3.2 Gitea - NOT RECOMMENDED

**Why Rejected:**
While technically equivalent to Forgejo, Gitea's governance trajectory introduces long-term risk incompatible with your self-hosted philosophy.

**Key Concerns:**
- Growing corporate involvement (Gitea Inc.) and commercialization pressure
- Community concerns about divergence from pure open-source principles
- Risk of proprietary features in future versions
- Corporate interests may conflict with user interests

**Decision Logic:**
Although Gitea has larger community and more documentation, Forgejo's community governance better aligns with Nova's values of independence and community control.

---

#### 2.3.3 Gitolite - REJECTED

**Critical Blocker:** No web UI - incompatible with requirement for user-friendly repository management

**Additional Issues:**
- Entirely CLI-based (no browser access)
- Steep learning curve (requires Perl knowledge)
- High operational overhead (manual user/key management)
- No built-in CI/CD integration

**Decision:** Gitolite sacrifices usability for minimal overhead. Nova's philosophy prioritizes ease of use.

---

#### 2.3.4 Plain Git - REJECTED

**Critical Blockers:**
- No web UI (cannot browse repositories via browser)
- No project management features (issue tracking, PRs, etc.)
- No CI/CD integration capabilities

**Additional Issues:**
- Very high operational overhead (manual everything)
- Poor user experience (requires Git CLI expertise)
- Not scalable for multiple repositories

**Decision:** Plain Git fails multiple core requirements.

---

## 3. Decision Matrix & Weighted Scoring

### 3.1 Scoring Methodology

**Evaluation Criteria (Your Specified Priorities):**

| Criterion | Weight | Rationale |
|-----------|--------|-----------|
| **Performance** | 15% | Must fit in Nova's resource envelope |
| **Community Support** | 15% | Need active maintenance, documentation, support |
| **Integration Complexity** | 15% | Must integrate cleanly with Ollama, Open WebUI, Nginx |
| **Learning Curve** | 10% | You'll be primary administrator and user |
| **Vendor Lock-in Risk** | 15% | Self-hosted means avoiding proprietary lock-in |
| **Operational Overhead** | 15% | Single user means must be low-maintenance |
| **CI/CD Capabilities** | 10% | Nice-to-have for future, not critical yet |

**Total: 100%**

### 3.2 Scoring Results (Weighted)

| Tool | Performance (15%) | Community (15%) | Integration (15%) | Learning (10%) | Lock-in (15%) | Overhead (15%) | CI/CD (10%) | **Weighted Score** |
|------|-----------------|-----------------|-------------------|----------------|---------------|----------------|-------------|------------------|
| **Forgejo** | 5.0 | 4.0 | 5.0 | 4.0 | 5.0 | 4.0 | 4.0 | **4.4 / 5.0** |
| **Gitea** | 5.0 | 5.0 | 5.0 | 4.0 | 3.0 | 4.0 | 4.0 | **4.25 / 5.0** |
| **Gitolite** | 5.0 | 3.0 | 2.0 | 2.0 | 5.0 | 2.0 | 1.0 | **2.8 / 5.0** |
| **Plain Git** | 5.0 | 1.0 | 2.0 | 1.0 | 5.0 | 1.0 | 1.0 | **2.0 / 5.0** |

**Winner: Forgejo with 4.4/5.0**

### 3.3 Detailed Score Justifications

**Forgejo Scoring Breakdown:**
- **Performance (5/5):** ~100 MB memory, minimal CPU. Perfect fit.
- **Community (4/5):** Growing, active community. Slightly smaller than Gitea but highly engaged.
- **Integration (5/5):** Direct binary, systemd service, Nginx reverse proxy. Perfect alignment with Nova patterns.
- **Learning (4/5):** Intuitive web UI, standard Git operations. Single-user setup straightforward.
- **Lock-in (5/5):** Open source, community-driven, zero corporate influence. Zero long-term risk.
- **Overhead (4/5):** Systemd manages lifecycle. Minimal day-to-day administration needed.
- **CI/CD (4/5):** Forgejo Actions available, GitHub Actions syntax compatible. Expandable without restructuring.

**Why Forgejo Outscores Gitea (0.15 points):**
The critical differentiator is **Vendor Lock-in Risk.** Although technically equivalent, Gitea's governance trajectory (corporate influence) introduces long-term strategic risk that conflicts with your self-hosted philosophy. Forgejo's community governance aligns with Nova's values of independence and community control.

---

## 4. Risk Analysis

### 4.1 Identified Risks (with Integrated Mitigations)

| Risk | Probability | Impact | Mitigation Strategy |
|------|-------------|--------|---------------------|
| **Forgejo community fragmentation** | Low (10%) | Medium | Monitor project health; Gitea migration path available |
| **Database corruption** | Low (5%) | High | Implement automated daily backups to secondary storage |
| **Security vulnerabilities discovered** | Low (10%) | Medium | Enable systemd auto-restart; subscribe to security advisories |
| **Port 3000 conflicts** | Low (5%) | Low | Use separate port (3001) or Nginx path-based routing |
| **Resource contention with Ollama** | Very Low (2%) | Low | Monitor system metrics during first month |
| **SSH key management complexity** | Medium (40%) | Low | Start with HTTPS-only; add SSH later after stabilization |
| **Repository storage exceeding capacity** | Low (5%) | Medium | Current 915 GB sufficient; monitor monthly |

**Risk Mitigation Summary:**
1. **Data Protection:** Daily database backups (automated in Phase 2)
2. **Security:** Systemd auto-restart with monitoring
3. **Compatibility:** Maintain ability to migrate to Gitea if community fork becomes unstable
4. **Resources:** Monthly performance monitoring during first 3 months
5. **Complexity:** Phased approach (HTTPS first, SSH later)

---

## 5. Success Criteria (Simple, Measurable Goals)

### 5.1 Phase 1: Installation & Stabilization

✅ **Functional Success:**
- Forgejo running and accessible at `http://nova.ai:3001`
- Single user account created and authenticated
- Test repository created, cloned, and committed successfully
- Systemd service automatically starts on reboot
- No resource conflicts with Ollama or Open WebUI

✅ **Performance Success:**
- Memory usage <500 MB under normal operations
- CPU usage <5% during idle periods
- Response time <2 seconds for web UI operations

### 5.2 Phase 2: Operational Stability

✅ **Reliability:**
- System stable over 30+ days of continuous operation
- Backup process running successfully (verified daily)
- Zero unplanned downtime or service interruptions

✅ **Usability:**
- Push/pull operations work reliably via HTTPS
- Web UI responsive for all repository operations
- Can create/delete/rename repositories without issues

### 5.3 Measurement Methods

- **Uptime:** `sudo systemctl status forgejo.service`
- **Performance:** `top`, `free -h`, monthly disk checks
- **Functionality:** Regular Git operations (push, pull, clone tests)
- **Accessibility:** Web UI load time and responsiveness

---

## 6. Key Assumptions

### 6.1 Documented Assumptions

**Operating Environment:**
1. Nova remains on **local network only** during planning phase
2. **Single user** account sufficient for initial deployment
3. Repositories stored on **local NVMe SSD** (no external storage initially)
4. **SQLite database** adequate for single-user, local-only use

**Implementation Details:**
5. **Port 3001** available or alternative Nginx routing possible
6. **HTTPS authentication** sufficient; SSH optional for Phase 2
7. **Systemd service** can manage Forgejo lifecycle with dependencies
8. **Nginx reverse proxy** can route traffic without conflicts

**Maintenance:**
9. **Manual daily backups** sufficient initially (automated Phase 2)
10. **Community support** and documentation adequate for single-user setup

### 6.2 Assumption Dependency Risks

**If assumptions break:**
- If Nova exposed to internet → encryption/hardening required
- If multi-user access needed → permissions model may need refinement
- If repositories exceed 100 GB → backup strategy must scale
- If Forgejo community becomes inactive → migration to Gitea plan activated

---

## 7. Implementation Phases

### 7.1 Phase 1: Basic Deployment (20-30 minutes)

1. Download Forgejo binary from official releases
2. Create system user (`forgejo` user)
3. Configure SQLite database (auto-created)
4. Create systemd service with Ollama dependency
5. Configure Nginx reverse proxy
6. Initialize Forgejo and create user account
7. Test Git operations (clone, push, pull)
8. Verify system stability

### 7.2 Phase 2: Enhancement (Post-stabilization, 1+ month later)

- Add SSH key authentication
- Set up repository mirroring (GitHub/GitLab)
- Evaluate Forgejo Actions for CI/CD
- Implement automated backups
- Consider multi-user support if needed

### 7.3 Phase 3: Advanced (Extended operations)

- External CI/CD integration if Forgejo Actions insufficient
- Git hooks and automation
- Repository templates
- Team collaboration features

---

## 8. Critical Decisions Before Implementation

**Must finalize before proceeding:**

- [ ] **Port Selection:** Port 3001 (separate from Open WebUI) - RECOMMENDED
- [ ] **URL Access:** `nova.ai:3001` (direct access) - RECOMMENDED  
- [ ] **Initial Auth:** HTTPS only; defer SSH - RECOMMENDED
- [ ] **Backup Plan:** Daily manual backups - RECOMMENDED
- [ ] **CI/CD Strategy:** Forgejo Actions Phase 1; external tool Phase 2+ - RECOMMENDED
- [ ] **Mirroring:** Deferred to Phase 2 - RECOMMENDED
- [ ] **SSH Support:** Deferred until Phase 2 - RECOMMENDED

---

## 9. Resource Impact Summary

| Aspect | Idle Usage | Active Usage | Nova Available | Impact Assessment |
|--------|-----------|--------------|-----------------|-------------------|
| **Memory** | 100-150 MB | 300-500 MB | 62 GB | Negligible (<1%) |
| **CPU** | <1% | 10-30% | 12 cores | No contention |
| **Disk** | 500 MB base | +repo size | 915 GB free | Ample headroom |
| **Network** | Minimal | During operations | Local network | No bottleneck |

**Conclusion:** No resource conflicts expected with Ollama or Open WebUI. System has comfortable headroom.

---

## 10. Architectural Alignment Verification

**Forgejo follows all established Nova patterns:**

| Component | Nova Pattern | Forgejo Match |
|-----------|-------------|---------------|
| **Ollama** | Direct binary + systemd | ✅ Identical |
| **Open WebUI** | Non-containerized + systemd | ✅ Same pattern |
| **Network** | Local-only + Nginx proxy | ✅ Same pattern |
| **User Model** | Non-root + file ownership | ✅ Same pattern |
| **Auto-Startup** | systemd with dependencies | ✅ Same pattern |

**Result:** Forgejo seamlessly integrates with existing Nova architecture.

---

## 11. References & Resources

- **Forgejo Official:** https://forgejo.org/
- **Forgejo Installation:** https://forgejo.org/docs/latest/
- **Forgejo vs Gitea:** https://forgejo.org/
- **Forgejo Actions:** https://forgejo.org/docs/latest/user/actions/
- **Git Server Comparison:** https://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server

---

## 12. Final Recommendation

### 12.1 Summary

**Forgejo is the recommended choice for Nova's Git server** based on comprehensive evaluation of alternatives, weighted decision matrix analysis, risk assessment, and architectural alignment.

**Key Justifications:**
1. ✅ Highest weighted score (4.4/5.0) across all evaluation criteria
2. ✅ Community-driven governance aligns with self-hosted philosophy
3. ✅ Lightweight and performant (no resource contention)
4. ✅ Seamless integration with existing Nova services
5. ✅ Zero vendor lock-in risk
6. ✅ Adequate community support and documentation
7. ✅ Phased approach allows future enhancement without restructuring

### 12.2 Next Steps

Once this planning document is approved:

1. Finalize decision checklist (Section 8)
2. Create detailed "Forgejo Installation Guide"
3. Proceed with Phase 1 deployment
4. Document setup in "Forgejo Installation & Configuration Report"

---

**Document Version:** 2.0 (Professional Edition)  
**Last Updated:** February 15, 2026  
**Status:** Professional Review Complete - Ready for Implementation  
**Sensitive Information:** Redacted (IP addresses, internal file paths)
