# Nova: Forgejo Actions & Runner Configuration Report
## CI/CD Pipeline Setup and Configuration

**Setup Date:** February 27, 2026
**Status:** Operational
**Runner:** nova-runner (v12.7.0)
**Execution Mode:** Host (bare-metal)

---

## Executive Summary

This report documents the configuration of Forgejo Actions and the act runner on Nova, enabling automated CI/CD pipelines triggered by repository events. The setup provides automated testing on every push, running pytest against the codebase using a Python virtual environment — without installing dependencies system-wide.

The deployment follows Nova's established pattern of direct binary installation managed via systemd, avoiding containerization overhead and maintaining consistency with existing services.

---

## 1. Architecture Overview

### 1.1 CI/CD Flow

```
Developer pushes code
        ↓
Forgejo (port 3001)
        ↓
Forgejo Actions (event trigger)
        ↓
act_runner (nova-runner)
        ↓
Pipeline execution (host)
        ↓
Results reported to Forgejo UI
```

### 1.2 Key Characteristics

| Aspect | Specification |
|--------|---------------|
| **Runner Name** | nova-runner |
| **Runner Version** | v12.7.0 |
| **Execution Mode** | Host (bare-metal, no Docker) |
| **Trigger** | Push events |
| **Python Environment** | Virtual environment (per-run) |
| **Test Framework** | pytest |
| **Config Location** | `/home/mo/.config/forgejo-runner/` |
| **Service Management** | systemd |

---

## 2. Enabling Forgejo Actions

### 2.1 Configuration

Actions is enabled via `/home/mo/.config/forgejo/app.ini`:

```ini
[actions]
ENABLED = true
```

**Service restart required after change:**

```bash
sudo systemctl restart forgejo
```

### 2.2 Verification

After restart, the **Actions** tab appears in repository navigation, and runners are manageable via **Admin Panel → Actions → Runners**.

---

## 3. Act Runner Installation

### 3.1 Binary Download

The Forgejo act runner is installed as a standalone binary. Always check the latest release at `https://code.forgejo.org/forgejo/act_runner/releases` before downloading.

```bash
cd /tmp
curl -L https://code.forgejo.org/forgejo/act_runner/releases/download/[version]/act_runner-[version]-linux-amd64 -o act_runner
sudo mv act_runner /usr/local/bin/act_runner
sudo chmod +x act_runner
act_runner --version
```

**Installed Version:** v12.7.0

**Installed Location:** `/usr/local/bin/act_runner`

### 3.2 Runner Registration

A dedicated config directory is created before registration:

```bash
mkdir -p /home/mo/.config/forgejo-runner
cd /home/mo/.config/forgejo-runner
act_runner register
```

**Registration prompts:**

| Prompt | Value |
|--------|-------|
| **Instance URL** | `http://localhost:3001` |
| **Token** | Obtained from Admin Panel → Actions → Runners |
| **Runner Name** | nova-runner |
| **Labels** | Default (accepted) |

Registration creates a `.runner` file in the working directory containing the runner credentials and configuration.

**Token Location:** Admin Panel → Actions → Runners → Create new runner

---

## 4. Service Management via Systemd

### 4.1 Service File

**Location:** `/etc/systemd/system/forgejo-runner.service`

```ini
[Unit]
Description=Forgejo Act Runner
After=network-online.target forgejo.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/act_runner daemon
Restart=always
RestartSec=10
User=mo
WorkingDirectory=/home/mo/.config/forgejo-runner

[Install]
WantedBy=multi-user.target
```

**Critical Configuration Notes:**

1. **WorkingDirectory:** Must point to the directory containing the `.runner` credentials file
2. **After=forgejo.service:** Runner starts only after Forgejo is available
3. **User=mo:** Runs as primary user, consistent with other Nova services
4. **Restart=always:** Recovers automatically from failures

### 4.2 Service Commands

```bash
# Enable auto-start on boot
sudo systemctl enable forgejo-runner

# Start service
sudo systemctl start forgejo-runner

# Check status
sudo systemctl status forgejo-runner

# View logs
sudo journalctl -u forgejo-runner -f

# Restart service
sudo systemctl restart forgejo-runner
```

### 4.3 Expected Status

```
● forgejo-runner.service - Forgejo Act Runner
     Loaded: loaded (/etc/systemd/system/forgejo-runner.service; enabled)
     Active: active (running)
```

Runner should appear as **Idle** in Forgejo Admin Panel → Actions → Runners when no jobs are running.

---

## 5. Pipeline Configuration

### 5.1 Workflow File Location

Workflows are stored in the repository under `.forgejo/workflows/`. Each `.yaml` file in this directory is treated as a pipeline definition.

### 5.2 Working Pipeline

**File:** `.forgejo/workflows/test.yaml`

```yaml
name: Test Pipeline
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        run: |
          git init
          git remote add origin http://localhost:3001/${{ github.repository }}.git
          git fetch
          git checkout ${{ github.sha }}

      - name: Set up Python
        run: |
          python3 -m venv .venv
          .venv/bin/pip install pytest

      - name: Run tests
        run: .venv/bin/pytest models/
```

### 5.3 Pipeline Design Decisions

**Manual Git Checkout (no actions/checkout):**

`actions/checkout@v4` requires Node.js. Rather than installing Node.js system-wide, a manual git checkout is used instead, keeping Nova's principle of avoiding unnecessary system-wide installations. The `github.repository` and `github.sha` context variables are used for Forgejo Actions compatibility.

**Python Virtual Environment:**

pytest is installed into a per-run virtual environment (`.venv`) rather than system-wide. This keeps the host clean and ensures dependency isolation between projects.

**Targeted Test Path:**

pytest is pointed at `models/` explicitly to avoid collecting test files from directories that import GUI dependencies (e.g., tkinter) which are unavailable in a headless server environment.

---

## 6. Lessons Learned

### 6.1 Node.js Dependency in actions/checkout

**Issue:** `actions/checkout@v4` fails with `Cannot find: node in PATH`

**Root Cause:** The action requires Node.js as a runtime, which was not installed on Nova.

**Resolution:** Replaced `actions/checkout` with a manual `git init / git fetch / git checkout` sequence using Forgejo context variables. Avoids system-wide Node.js installation while maintaining full checkout functionality.

### 6.2 Context Variable Naming

**Issue:** `gitea.repository` and `gitea.sha` caused YAML schema validation errors.

**Root Cause:** Newer Forgejo versions use `github.*` prefix for Actions context variables (for GitHub Actions compatibility).

**Resolution:** Use `github.repository` and `github.sha` instead of `gitea.*` equivalents.

### 6.3 System Package vs. Virtual Environment

**Issue:** Installing `python3-tk` and `pytest` via pip or system-wide conflicts with Nova's principle of clean host installations.

**Resolution:** Use `python3 -m venv .venv` with `--system-site-packages` omitted to create a fully isolated environment per run. System packages are never polluted by pipeline dependencies.

### 6.4 tkinter in Headless Environment

**Issue:** `test_models.py` imports from `main.py`, which imports `tkinter` at the top level, causing collection errors even though tests don't use the GUI.

**Resolution:** Refactored `main.py` to guard the tkinter import inside `if __name__ == "__main__":`, keeping business logic classes (`InMemorySlotRepository`, `ParkingLot`) importable without triggering GUI dependencies.

### 6.5 Duplicate Runner Registrations

**Issue:** Multiple failed registration attempts created several offline runners in the Admin Panel.

**Resolution:** Removed duplicate runners via Admin Panel → Actions → Runners → Delete. Only one runner (nova-runner) retained.

---

## 7. Current Deployment Status

### 7.1 Runner Status

✅ **nova-runner:** Idle (connected, awaiting jobs)

### 7.2 Pipeline Status

✅ **Test Pipeline:** Passing (18 tests collected, 18 passed)

### 7.3 Verified Capabilities

- ✅ Push event triggers pipeline automatically
- ✅ Repository checkout via local Forgejo instance
- ✅ Python virtual environment creation per run
- ✅ pytest execution with targeted test path
- ✅ Pass/fail results reported in Forgejo Actions UI
- ✅ Runner survives service restarts and reboots

---

## 8. Future Enhancements

### 8.1 Additional Pipeline Stages

- Linting (flake8, ruff) as a separate job
- Code coverage reporting
- Build artifact generation

### 8.2 Node.js Installation (Optional)

If future pipelines require `actions/checkout` or other Node-based actions, install Node.js system-wide:

```bash
sudo apt install nodejs
```

This would allow use of the standard Actions ecosystem without manual git checkout steps.

### 8.3 Multiple Pipelines

- Separate pipelines per project/repository
- Branch-specific triggers (e.g., only run on `main`)
- Pull request event triggers

### 8.4 Notifications

- Webhook notifications on pipeline failure
- Integration with local monitoring

---

## 9. Reference: Useful Commands

```bash
# Runner management
act_runner --version                          # Check version
act_runner register                           # Register new runner
act_runner daemon                             # Run manually (outside systemd)

# Service management
sudo systemctl status forgejo-runner          # Check status
sudo systemctl restart forgejo-runner         # Restart runner
sudo journalctl -u forgejo-runner -f          # Live logs
sudo journalctl -u forgejo-runner -n 50       # Last 50 lines

# Forgejo Actions config
nano /home/mo/.config/forgejo/app.ini         # Edit Forgejo config
sudo systemctl restart forgejo                # Apply config changes

# Runner credentials
ls /home/mo/.config/forgejo-runner/           # List runner files
cat /home/mo/.config/forgejo-runner/.runner   # View runner registration
```

---

**Document Version:** 1.0
**Last Updated:** February 27, 2026
**Status:** Final
**Sensitive Information:** Redacted (tokens, internal hostnames)
