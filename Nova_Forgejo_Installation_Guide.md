# Nova: Forgejo Installation Guide
## Forgejo 14.0.2 - Git Server Setup

**Installation Date:** February 18, 2026  
**Forgejo Version:** 14.0.2  
**User Account:** Existing user (mo)  
**Port:** 3001  
**SSH:** Enabled on port 22  
**Configuration:** Using nano editor, absolute paths  

---

## Quick Overview

This guide installs Forgejo as a self-hosted Git server using your existing user account. Key points:
- Direct binary installation (no Docker)
- All data stored in `~/.local/share/forgejo/`
- Configuration in `~/.config/forgejo/app.ini`
- Accessed via `http://nova.ai:3001`
- SSH enabled for git operations

---

## Prerequisites

```bash
# Install required tools
sudo apt update
sudo apt install git curl wget

# Verify your home directory and username
echo "Home: $HOME"
echo "User: $(whoami)"
```

---

## Step 1: Download Forgejo Binary

```bash
# Create installation directory
mkdir -p ~/forgejo-install
cd ~/forgejo-install

# Download Forgejo 14.0.2
wget https://codeberg.org/forgejo/forgejo/releases/download/v14.0.2/forgejo-14.0.2-linux-amd64

# Make executable
chmod +x forgejo-14.0.2-linux-amd64

# Test it works
./forgejo-14.0.2-linux-amd64 --version
```

**Expected output:**
```
Forgejo version 14.0.2
```

### Install Binary to System Location

```bash
# Move to system path
sudo mv forgejo-14.0.2-linux-amd64 /usr/local/bin/forgejo

# Verify
which forgejo
forgejo --version
```

---

## Step 2: Create Data Directories

```bash
# Create directories in your home directory
mkdir -p ~/.local/share/forgejo/data
mkdir -p ~/.local/share/forgejo/log
mkdir -p ~/.config/forgejo

# Verify they exist
ls -la ~/.local/share/forgejo/
ls -la ~/.config/forgejo/
```

---

## Step 3: Create Configuration File

```bash
# Open nano to create config
nano ~/.config/forgejo/app.ini
```

Paste this content (replace `/home/mo` with your home directory from `echo $HOME`):

```ini
[server]
PROTOCOL = http
LISTEN_ADDR = 0.0.0.0
HTTP_PORT = 3001
ROOT_URL = http://nova.ai:3001/
SSH_PORT = 22
APP_DATA_PATH = /home/mo/.local/share/forgejo

[database]
DB_TYPE = sqlite3
PATH = /home/mo/.local/share/forgejo/data/forgejo.db

[repository]
ROOT = /home/mo/.local/share/forgejo/data/repositories

[log]
MODE = file
LEVEL = info
ROOT_PATH = /home/mo/.local/share/forgejo/log
```

**Save:** Ctrl+O, Enter, Ctrl+X

### Verify and Fix Permissions

```bash
# Fix file ownership (if created as root)
sudo chown mo:mo ~/.config/forgejo/app.ini

# Set readable permissions
chmod 644 ~/.config/forgejo/app.ini

# Verify content
cat ~/.config/forgejo/app.ini
```

---

## Step 4: Initialize Forgejo (Web Setup)

Run Forgejo in web mode to initialize the database:

```bash
# Start Forgejo web server
/usr/local/bin/forgejo -c ~/.config/forgejo/app.ini web
```

**Expected output:**
```
2026/02/18 20:XX:XX ...
...
[I] Server started listening at [::]:3001
```

### Complete Web Setup

1. Open browser: `http://localhost:3001`
2. Complete the setup wizard
3. Create admin account
4. Configure instance settings
5. Return to terminal and press `Ctrl+C` to stop

---

## Step 5: Create Systemd Service

```bash
# Get your home directory (for systemd file)
echo $HOME

# Open nano to create systemd service
sudo nano /etc/systemd/system/forgejo.service
```

Paste this content (replace `/home/mo` and `mo` with your values):

```ini
[Unit]
Description=Forgejo (Git Service)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mo
Group=mo
WorkingDirectory=/home/mo

ExecStart=/usr/local/bin/forgejo -c /home/mo/.config/forgejo/app.ini web

Restart=always
RestartSec=10

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

**Save:** Ctrl+O, Enter, Ctrl+X

### Enable and Start Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable auto-start on boot
sudo systemctl enable forgejo.service

# Start the service
sudo systemctl start forgejo.service

# Check status
sudo systemctl status forgejo.service
```

**Expected:**
```
● forgejo.service - Forgejo (Git Service)
     Active: active (running) since...
```

---

## Step 6: Verify Installation

### Check Service is Running

```bash
# Verify process
ps aux | grep forgejo | grep -v grep

# Check port 3001
sudo lsof -i :3001

# Check logs
sudo journalctl -u forgejo.service -n 20
```

### Test Web Access

```bash
# From Nova
curl http://localhost:3001

# From another machine (after adding to /etc/hosts)
echo "[NOVA-IP] nova.ai" | sudo tee -a /etc/hosts
curl http://nova.ai:3001
```

### Check Resources

```bash
# Memory usage
ps aux | grep forgejo | grep -v grep | awk '{print $6 " KB"}'

# Disk usage
du -sh ~/.local/share/forgejo/
```

---

## Step 7: Create User Accounts and Repositories

### Via Web Interface

1. Access `http://nova.ai:3001`
2. Log in with admin account
3. Create repositories
4. Invite users (if needed)

### Via Command Line

```bash
# List users
/usr/local/bin/forgejo -c ~/.config/forgejo/app.ini admin user list

# Create new user
/usr/local/bin/forgejo -c ~/.config/forgejo/app.ini admin user create \
  --username testuser \
  --password SecurePass123! \
  --email testuser@nova.ai
```

---

## Step 8: Test Git Operations

### Clone via HTTPS

```bash
# From another machine
git clone http://nova.ai:3001/admin/test-repo.git
cd test-repo

# Make a commit
echo "# Test" > README.md
git add README.md
git commit -m "Initial commit"

# Push
git push origin main
```

### Clone via SSH (if SSH keys configured)

```bash
git clone git@nova.ai:admin/test-repo.git
```

---

## Troubleshooting

### Forgejo Won't Start

```bash
# Check logs for errors
sudo journalctl -u forgejo.service -n 50 | grep -i error

# Verify config file is readable
cat ~/.config/forgejo/app.ini | head -5

# Check if port 3001 is in use
sudo lsof -i :3001
```

### Permission Denied on Config

```bash
# Fix ownership
sudo chown mo:mo ~/.config/forgejo/app.ini

# Fix permissions
chmod 644 ~/.config/forgejo/app.ini
```

### Cannot Access Web Interface

```bash
# Verify service is running
sudo systemctl status forgejo.service

# Check port binding
sudo lsof -i :3001

# Test local access
curl http://localhost:3001
```

### High Memory Usage

```bash
# Check actual usage
ps aux | grep forgejo | grep -v grep

# Restart service
sudo systemctl restart forgejo.service
```

---

## File Locations

```
Configuration: ~/.config/forgejo/app.ini
Data directory: ~/.local/share/forgejo/
Database: ~/.local/share/forgejo/data/forgejo.db
Repositories: ~/.local/share/forgejo/data/repositories
Logs: ~/.local/share/forgejo/log/
Binary: /usr/local/bin/forgejo
Systemd: /etc/systemd/system/forgejo.service
Web access: http://nova.ai:3001
SSH access: git@nova.ai
```

---

## Common Commands

```bash
# Service management
sudo systemctl start forgejo.service
sudo systemctl stop forgejo.service
sudo systemctl restart forgejo.service
sudo systemctl status forgejo.service

# Logs
sudo journalctl -u forgejo.service -n 50
sudo journalctl -u forgejo.service -f

# Administration
/usr/local/bin/forgejo -c ~/.config/forgejo/app.ini admin user list

# Testing
curl http://localhost:3001
ps aux | grep forgejo
sudo lsof -i :3001
```

---

## Important Notes

1. **Absolute Paths:** All paths in config must be absolute (start with `/home/mo`), not relative (`~`)
2. **APP_DATA_PATH:** Must be set in config to avoid permission errors
3. **File Ownership:** Config file ownership must match your user (mo)
4. **Web Initialization:** Use `forgejo web` for initial setup, not `migrate`
5. **Systemd Service:** Must have correct user, group, and working directory

---

**Installation Complete!** ✅

Access Forgejo at: `http://nova.ai:3001`
