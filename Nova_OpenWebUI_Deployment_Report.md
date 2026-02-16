# Nova: Open WebUI Deployment Report
## Web Interface, Network Integration & Access Configuration

**Setup Date:** February 15, 2026  
**Status:** Operational  
**Interface URL:** `http://nova.ai` (local network)  
**Port:** 3000 (internal), 80 (Nginx reverse proxy)

---

## Executive Summary

This report documents the installation, configuration, and deployment of Open WebUI—a modern, web-based interface for interacting with Large Language Models served via Ollama. Open WebUI provides a ChatGPT-like user experience while maintaining complete local control and data privacy.

The deployment integrates Open WebUI with Ollama backend, Nginx reverse proxy, and local network DNS resolution, creating a seamless user experience accessible from browser and mobile devices.

---

## 1. Open WebUI Architecture & Selection

### 1.1 Why Open WebUI?

**Selection Criteria:**
- **Local-First Design:** All data stays on Nova; no cloud upload
- **Self-Hosted:** Complete control over interface and data
- **Modern UX:** ChatGPT-like interface, intuitive for end users
- **Ollama Integration:** Seamless backend connection
- **Privacy-Focused:** No user tracking, no data collection
- **Open Source:** Transparent codebase, community-driven development

**Alternative Interfaces Considered:**
- **Ollama CLI:** Functional but terminal-only
- **LM Studio UI:** Heavier dependencies, less streamlined
- **Gradio Custom UI:** Requires custom development
- **Commercial ChatGPT:** No data privacy, cloud-dependent

Open WebUI selected for optimal balance of features, usability, and privacy.

### 1.2 Deployment Strategy: Native Installation (Non-Docker)

**Decision:** Install Open WebUI via pip system-wide (not containerized)

**Rationale:**

| Factor | Native Installation | Docker Container |
|--------|-------------------|-----------------|
| **Installation** | Simple (`pip install`) | Requires Docker knowledge |
| **Data Persistence** | Direct filesystem | Volume mounting required |
| **System Integration** | Direct systemd service | Container orchestration |
| **Debugging** | Direct access to logs | Container log aggregation |
| **Complexity** | Minimal | Moderate |

**Current Context:** Single user interface service; native installation appropriate and maintainable.

**Future Consideration:** May containerize if multi-service isolation becomes requirement.

---

## 2. Installation & Initial Setup

### 2.1 Installation via pip

**Command:**
```bash
sudo pip install open-webui --break-system-packages
```

**Why `--break-system-packages`:**
- Pop!_OS uses system-managed Python packages
- Flag allows pip to install alongside system packages
- Necessary for system-wide installation
- Alternative: use virtual environment for user-scoped installation

**Installation Process:**
- Downloads Open WebUI and dependencies (~2 GB including ML libraries)
- Installs PyTorch/Torch for AI acceleration
- Downloads embedding models and dependencies
- Completes in 5-15 minutes (depends on internet speed)

**Verification:**
```bash
which open-webui
open-webui --version
```

**Installed Location:** `/usr/local/bin/open-webui`

### 2.2 Initial Runtime & Data Directory

**Database Location:** `/usr/local/lib/python3.12/dist-packages/open_webui/data/webui.db`

**User Data Location:** `~/.webui_secret_key` (encryption key)

**Critical Issue Encountered:** Database created with root ownership, causing permission denied errors when service ran as non-root user.

**Resolution:** Changed file permissions to allow service user access
```bash
sudo chown -R [service-user] /usr/local/lib/python3.12/dist-packages/open_webui/
chmod -R 755 /usr/local/lib/python3.12/dist-packages/open_webui/data/
```

---

## 3. Service Management via Systemd

### 3.1 Systemd Service Configuration

**Service File:** `/etc/systemd/system/open-webui.service`

```ini
[Unit]
Description=Open WebUI
After=network-online.target ollama.service
Wants=network-online.target
Requires=ollama.service

[Service]
Type=simple
ExecStart=/usr/local/bin/open-webui serve --host 0.0.0.0 --port 3000
Restart=always
RestartSec=10
User=[primary-user]
WorkingDirectory=/home/[primary-user]

[Install]
WantedBy=multi-user.target
```

### 3.2 Service Dependencies & Ordering

**Key Configuration Elements:**

1. **Dependency on Ollama:** `Requires=ollama.service`
   - Open WebUI won't start if Ollama unavailable
   - Prevents orphaned web service without backend

2. **Service Ordering:** `After=ollama.service`
   - Open WebUI starts after Ollama completes initialization
   - Ensures backend ready when web interface initializes

3. **User Context:** `User=[primary-user]` (not root)
   - Critical for file permissions
   - Matches Ollama service user for consistency
   - Prevents privilege escalation risks

### 3.3 Service Commands

```bash
# Enable auto-start on boot
sudo systemctl enable open-webui.service

# Start service
sudo systemctl start open-webui.service

# Check status
sudo systemctl status open-webui.service

# View logs (last 50 lines)
sudo journalctl -u open-webui.service -n 50

# Live log monitoring
sudo journalctl -u open-webui.service -f

# Restart service
sudo systemctl restart open-webui.service

# Stop service
sudo systemctl stop open-webui.service
```

### 3.4 Startup Process & Initialization

**Startup Sequence (from logs):**

1. **Secret Key Loading** (0s)
   ```
   Loading WEBUI_SECRET_KEY from /home/mo/.webui_secret_key
   ```

2. **Database Migration** (2s)
   ```
   INFO [alembic.runtime.migration] Context impl SQLiteImpl.
   Will assume non-transactional DDL.
   ```

3. **CORS Configuration** (2s)
   ```
   WARNING: CORS_ALLOW_ORIGIN IS SET TO '*' - NOT RECOMMENDED FOR PRODUCTION
   ```
   *(Safe for local network; would restrict for internet exposure)*

4. **Dependencies Loading** (4s)
   ```
   RuntimeWarning: Couldn't find ffmpeg or avconv
   USER_AGENT environment variable not set
   ```

5. **Server Start** (4s)
   ```
   INFO: Started server process [PID]
   INFO: Waiting for application startup.
   Application startup complete
   ```

**Typical startup time:** 4-6 seconds

---

## 4. Network Configuration & Access

### 4.1 Port Configuration

**Internal Binding:** Port 3000
```bash
ExecStart=/usr/local/bin/open-webui serve --host 0.0.0.0 --port 3000
```

**Rationale:**
- Standard development port for web applications
- Non-privileged port (doesn't require root)
- Forwarded via Nginx to standard port 80

**Verification:**
```bash
sudo lsof -i :3000
# Should show: python3 [PID] [user] ... (LISTEN)
```

### 4.2 Nginx Reverse Proxy Integration

**Purpose:** Hide internal port 3000; expose on standard port 80 under domain `nova.ai`

**Nginx Configuration:** (See System Setup Report, Section 3.3)

**Flow:**
1. User navigates to `http://nova.ai`
2. Nginx listens on port 80
3. Proxies request to `http://localhost:3000`
4. Open WebUI responds
5. Nginx forwards response to user

**Benefits:**
- Standard HTTP port (no port number in URL)
- Professional appearance (`nova.ai` not `nova.ai:3000`)
- WebSocket support maintained (Upgrade headers forwarded)
- HTTPS-ready (can add SSL later)

**Verification:**
```bash
curl -I http://nova.ai
# Should return 200 OK with Open WebUI headers
```

### 4.3 Local Network Accessibility

**Current Access Methods:**

**Method 1: Direct (requires firewall rule)**
- URL: `http://[nova-ip]:3000`
- Works immediately from any network machine
- Not recommended (bypasses reverse proxy)

**Method 2: Reverse Proxy with DNS (recommended)**
- URL: `http://nova.ai`
- Requires DNS entry: `[nova-ip] nova.ai` in `/etc/hosts`
- Clean URL, professional appearance

**Method 3: Mobile PWA (iPhone/Android)**
- Safari: Add to Home Screen → `http://nova.ai`
- Creates app-like icon on device
- Offline-ready (partial functionality)

### 4.4 Firewall Configuration

**Required Rules:**

| Port | Direction | Protocol | Purpose |
|------|-----------|----------|---------|
| 3000 | Inbound | TCP | Open WebUI (optional, for direct access) |
| 80 | Inbound | TCP | Nginx reverse proxy (required) |
| 11434 | Inbound | TCP | Ollama (required for backend) |

**Configuration:**
```bash
sudo ufw allow 3000/tcp
sudo ufw allow 80/tcp
sudo ufw allow 11434/tcp
```

---

## 5. Backend Integration with Ollama

### 5.1 Connection Configuration

**Auto-Detection:** Open WebUI automatically detects Ollama at startup

**Connection URL:** `http://localhost:11434`

**Verification:**
```bash
# In Open WebUI settings (gear icon)
Settings → Models → Ollama connection status
# Should show: "Connected to Ollama"
```

**Health Check (Backend):**
```bash
curl http://localhost:11434/api/tags
# Should return list of models
```

### 5.2 Model Display & Selection

**Model Discovery:**
1. Open WebUI reads models from Ollama API
2. Displays available models in chat interface dropdown
3. User selects model for conversation

**Expected Models:**
- Mistral 7B Instruct
- Qwen3 8B (when loading completes)

**Troubleshooting:**
```bash
# If models not showing in Open WebUI:

# 1. Verify Ollama has models
ollama list

# 2. Test Ollama API directly
curl http://localhost:11434/api/tags

# 3. Restart Open WebUI
sudo systemctl restart open-webui.service

# 4. Check Open WebUI logs
sudo journalctl -u open-webui.service -n 100 | grep -i model
```

### 5.3 Inference Request Flow

**User Interaction Flow:**

```
User Input (Web UI)
    ↓
Nginx (reverse proxy)
    ↓
Open WebUI (port 3000)
    ↓
Ollama API (port 11434)
    ↓
LLM Model (GPU inference)
    ↓
Response back through chain
    ↓
User sees response in browser
```

**Latency Breakdown (typical):**
- Network roundtrip: <1ms (local network)
- Open WebUI processing: 100-200ms
- Model inference: 10-30 seconds
- **Total:** 10-30 seconds (matches Nova's acceptable latency)

---

## 6. Data & Privacy Considerations

### 6.1 Data Storage

**Conversations:** Stored locally in SQLite database
```
/usr/local/lib/python3.12/dist-packages/open_webui/data/webui.db
```

**User Preferences:** Also in local database

**Models:** On disk (Ollama manages)

**No External Communication:** All data stays on Nova (local network only)

### 6.2 Privacy by Default

**Current Configuration:**
- ✅ No user accounts required (single-user)
- ✅ No cloud upload (local storage only)
- ✅ No analytics tracking
- ✅ No telemetry
- ✅ Open source (transparent)

**CORS Configuration:**
```
CORS_ALLOW_ORIGIN IS SET TO '*' - NOT RECOMMENDED FOR PRODUCTION
```
- Safe for local network development
- Would need restriction for internet exposure
- Can be configured via environment variable

---

## 7. Mobile & Multi-Device Access

### 7.1 Progressive Web App (PWA) on iOS

**Installation (iPhone/iPad):**

1. Open Safari
2. Navigate to `http://nova.ai`
3. Wait for page to fully load
4. Tap Share button (bottom)
5. Scroll down → "Add to Home Screen"
6. Name app (e.g., "Nova AI")
7. Tap "Add"

**Result:** Icon on home screen opens web app in full-screen mode

**Advantages:**
- App-like experience without App Store
- Offline-ready (caches assets)
- Works on any device with browser

**Limitations:**
- Not a native app (performance dependent on browser)
- Background execution limited
- Push notifications not available

### 7.2 Android Access

**Method 1: PWA Installation**
1. Chrome/Firefox browser
2. Navigate to `http://nova.ai`
3. Menu → "Install app" or "Add to Home Screen"

**Method 2: Mobile Apps (if available)**
- Search app store for "Ollama" or "Open WebUI"
- Configure connection: `http://[nova-ip]:11434`

### 7.3 Desktop Access

**Browser Requirements:**
- Chrome/Firefox/Safari (any modern browser)
- JavaScript enabled
- Local network access to Nova

**Recommended:** Use `http://nova.ai` (DNS resolution) rather than IP address

---

## 8. Troubleshooting & Common Issues

### 8.1 Models Not Showing in Web UI

**Symptom:** Open WebUI shows "No models available"

**Diagnosis:**
```bash
# Check Ollama is running
sudo systemctl status ollama.service

# Check models exist
ollama list

# Test API
curl http://localhost:11434/api/tags
```

**Solutions:**
1. **If Ollama not running:** `sudo systemctl start ollama.service`
2. **If no models:** `ollama pull mistral:7b-instruct-q4_K_M`
3. **If API works but UI doesn't see models:**
   - Refresh page (Ctrl+F5)
   - Clear browser cache
   - Restart Open WebUI: `sudo systemctl restart open-webui.service`

### 8.2 Database Permission Error

**Symptom:** Service fails to start with "readonly database"

**Root Cause:** Database owned by wrong user

**Solution:**
```bash
sudo chown -R [service-user] /usr/local/lib/python3.12/dist-packages/open_webui/
sudo systemctl restart open-webui.service
```

### 8.3 Cannot Access via `http://nova.ai`

**Symptom:** Browser shows "can't reach this site"

**Diagnosis:**
```bash
# Check if service running
sudo systemctl status open-webui.service

# Test direct URL
curl http://localhost:3000

# Check DNS resolution
ping nova.ai
```

**Solutions:**
1. **Add DNS entry:** Add `[nova-ip] nova.ai` to `/etc/hosts` on client
2. **Use direct IP:** `http://[nova-ip]:3000` (temporary workaround)
3. **Check Nginx:** `sudo systemctl status nginx`

### 8.4 Slow Inference Response

**Symptom:** Model takes longer than expected to respond

**Diagnosis:**
```bash
# Check GPU utilization
nvidia-smi
# Should show >80% GPU usage during inference

# Check CPU/RAM
top
# Should show reasonable utilization
```

**Likely Causes:**
- GPU not being used (CPU fallback) - check Ollama logs
- Model too large for GPU memory - use smaller model or quantization
- Network slow - shouldn't affect local network
- Concurrent requests - queue prevents parallel inference

### 8.5 High Memory Usage

**Symptom:** Open WebUI consuming >2GB RAM

**Normal:** 1.4-2.0 GB is typical (includes embeddings, model buffers)

**If exceeds 3GB:**
```bash
# Restart service to clear memory
sudo systemctl restart open-webui.service

# Check for memory leaks in logs
sudo journalctl -u open-webui.service -n 200 | grep -i memory
```

---

## 9. Current Deployment Status

### 9.1 Service Status

✅ **Open WebUI Service:** Active (running)

```bash
● open-webui.service - Open WebUI
     Loaded: loaded (/etc/systemd/system/open-webui.service; enabled)
     Active: active (running)
```

### 9.2 Network Accessibility

✅ **Web Interface:** `http://nova.ai` (via Nginx reverse proxy)

✅ **Ollama Backend:** Connected and responding

✅ **Model Display:** Mistral 7B + Qwen3 8B (loading)

### 9.3 Mobile Access

✅ **iOS PWA:** Installed and working

✅ **Multi-device:** Accessible from any local network device

---

## 10. Future Enhancements

### 10.1 Authentication & Multi-User Support

- Currently single-user (no authentication)
- Could add user accounts for family/team use
- Would require additional configuration

### 10.2 HTTPS/TLS Encryption

- Currently HTTP only (local network, acceptable)
- Could add self-signed certificates for encryption in transit
- Would require DNS configuration and certificate management

### 10.3 Advanced Features

- Custom system prompts per user
- Conversation history management
- Model-specific configurations
- Advanced settings interface

### 10.4 Monitoring & Analytics

- Track conversation patterns
- Monitor model performance
- Log inference latency metrics
- Capacity planning data

---

## 11. References & Resources

- Open WebUI GitHub: https://github.com/open-webui/open-webui
- Open WebUI Documentation: https://docs.openwebui.com/
- Nginx Reverse Proxy: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- Ollama API: https://github.com/ollama/ollama/blob/main/docs/api.md

---

## Appendix A: Useful Commands

```bash
# Service management
sudo systemctl start open-webui.service
sudo systemctl stop open-webui.service
sudo systemctl restart open-webui.service
sudo systemctl status open-webui.service
sudo systemctl enable open-webui.service

# Logging
sudo journalctl -u open-webui.service -n 50        # Last 50 lines
sudo journalctl -u open-webui.service -f           # Live logs
sudo journalctl -u open-webui.service --since 1h   # Last hour

# Network testing
curl http://localhost:3000                          # Direct port
curl http://nova.ai                                 # Via Nginx
curl http://localhost:11434/api/tags               # Ollama backend

# System checks
ps aux | grep open-webui                           # Check process
sudo lsof -i :3000                                 # Check port binding
du -sh /usr/local/lib/python3.12/dist-packages/open_webui/  # Disk usage
```

---

## Appendix B: Environment Variables

Open WebUI respects these environment variables (optional):

```bash
WEBUI_SECRET_KEY        # Encryption key (auto-generated if not set)
OLLAMA_BASE_URL         # Override Ollama connection URL
CORS_ALLOW_ORIGIN       # Restrict CORS origins (currently *)
DEBUG                   # Enable debug logging
```

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Status:** Final  
**Sensitive Information:** Redacted (IP addresses, internal hostnames, file paths)
