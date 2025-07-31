# Repeatable Headless Ubuntu Server SSH Bootstrap Guide

## Purpose
Make it trivial to securely provision and access a headless Ubuntu Server over SSH, with:
- Key-based authentication
- Easy recurring access via shortcut
- Basic hardening (firewall, updates, brute-force protection)
- Optional conveniences (Ubuntu Pro, mDNS, static/reserved IP)
- Clear fallback paths and diagnostics

This is written so you or anyone else can rerun it from scratch without guessing why a step exists.

---

## Assumptions / Prerequisites
- A separate admin machine (Mac/Linux/Windows+WSL) from which you will SSH.  
- Ubuntu Server installed (e.g., 24.04 LTS) on the target machine.  
- Network connectivity between admin and server (same LAN or routable).  
- You intend to use public-key-only SSH (password login disabled after verification).  
- You have (or will create) a GitHub account for easy SSH key injection during install.

---

## 1. Generate a dedicated SSH keypair (one-time per admin machine)

```bash
ssh-keygen -t ed25519 -C "ubuntu-server" -f ~/.ssh/ubuntu_server_ed25519
```

### What this does & why
- `ed25519` is modern, secure, and compact.  
- `-C` adds a comment so you can identify the key later.  
- `-f` keeps it separate from default keys so you can reason about which key is for which server.

Load it into your local SSH agent for seamless use:

```bash
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/ubuntu_server_ed25519   # macOS-specific; omit `--apple-use-keychain` elsewhere
```

### Why
Agent caching avoids typing the passphrase repeatedly. On macOS, the keychain integration persists your unlocked key across restarts securely.

Lock down the private key file:

```bash
chmod 600 ~/.ssh/ubuntu_server_ed25519
```

### Why `chmod 600`?
SSH refuses to use a private key file if it's readable by others. `600` means: owner can read/write (`rw-`), group and others have no access. This prevents other users on your machine from stealing the key.

---

## 2. Publish your public key so the server can accept it

### Recommended during install: GitHub import
1. Copy the public key:
   ```bash
   cat ~/.ssh/ubuntu_server_ed25519.pub
   ```
2. On GitHub: go to **Settings → SSH and GPG keys → New SSH key**, paste the key, name it (e.g., `ubuntu-server`).  
3. During the Ubuntu Server installer, choose **“Import SSH key from GitHub”** and supply your GitHub username. The installer will pull and install the key for your user.

### Alternate: Launchpad / Ubuntu One
If you prefer Launchpad/Ubuntu One, add the same public key to your profile and import it during install via the Launchpad option.

### Fallback (post-install manual install)
If you end up with temporary password access or console access:
```bash
cat ~/.ssh/ubuntu_server_ed25519.pub | ssh <user>@<server-ip>   'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys'
```

Then fix ownership if needed:
```bash
ssh <user>@<server-ip> "sudo chown -R <user>:<user> /home/<user>/.ssh"
```

### Why
This explicitly places your public key into the server’s `authorized_keys` so SSH key authentication works. Correct permissions are required or SSH will ignore the file.

---

## 3. Shortcut future SSH connections with `~/.ssh/config`

Add this to your admin machine's SSH config (`~/.ssh/config`):

```text
Host myserver
  HostName 10.0.0.105              # or ubuntu-server.local if using mDNS
  User eissayouj                   # server username
  IdentityFile ~/.ssh/ubuntu_server_ed25519
  AddKeysToAgent yes
  UseKeychain yes                 # macOS-specific; omit if not applicable
```

Lock down the config file:
```bash
chmod 600 ~/.ssh/config
```

### Why
- Makes `ssh myserver` enough instead of long flags.  
- Centralizes identity, username, and host settings.  
- `AddKeysToAgent` / `UseKeychain` improves usability on macOS so the key is auto-loaded.

---

## 4. Discover the server’s IP

### On the server (if you have console or shell access)
```bash
hostname -I
```
or
```bash
ip -4 addr show
```
The `inet` line for the active interface is the IP (e.g., `10.0.0.105`).

### Alternative if you only have network access from another machine
- Check your router’s DHCP leases and identify the machine by hostname or MAC.  
- Scan your subnet:
  ```bash
  brew install nmap   # macOS example
  nmap -p 22 --open 10.0.0.0/24
  ```

### Why
You need the correct IP to SSH to; dynamic DHCP can change it unless reserved.

---

## 5. Optional: Friendly name via mDNS (.local) resolution

On the server:
```bash
sudo hostnamectl set-hostname ubuntu-server
sudo apt update
sudo apt install avahi-daemon
sudo systemctl enable --now avahi-daemon
```

### Why
Installs Avahi so you can access the machine as `ubuntu-server.local` instead of memorizing IPs.

---

## 6. First-time SSH connection and host key verification

Connect using:
```bash
ssh myserver
```
Or explicitly:
```bash
ssh -i ~/.ssh/ubuntu_server_ed25519 eissayouj@10.0.0.105
```

On first connection you'll be prompted to accept the host key; type `yes`.

Optional manual verification on the server:
```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```
Ensure the fingerprint matches what your client warned about.

### Why
Caches the host key to detect MITM later. Manual verification ensures you’re not being intercepted.

---

## 7. Bootstrap & harden the server

```bash
sudo apt update && sudo apt upgrade -y
```

Enable automatic security updates:
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Install firewall, fail2ban, and tooling:
```bash
sudo apt install ufw fail2ban ubuntu-advantage-tools
sudo ufw allow OpenSSH
sudo ufw enable
```

(Optional) Attach Ubuntu Pro and enable extended security:
```bash
sudo pro attach <YOUR_TOKEN>
sudo pro enable esm-infra esm-apps livepatch
```

### Why
- System updates & unattended upgrades patch known vulnerabilities.  
- UFW restricts exposed ports.  
- Fail2ban throttles repeated login attempts.  
- Ubuntu Pro extends security coverage and enables live kernel patching.

---

## 8. SSH service hardening

Edit `/etc/ssh/sshd_config` and ensure the following are set:
```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```
Then restart SSH:
```bash
sudo systemctl restart ssh
```

### Why
Reduces attack surface: disables root login and password-based auth, forcing key-only login. Only do this after confirming key access works.

---

## 9. IP stability

### Preferred: DHCP reservation
Reserve the server’s IP in your router using its MAC address so it doesn’t change.

### Alternate: Static IP via Netplan
Edit `/etc/netplan/00-installer-config.yaml`, for example:
```yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: no
      addresses: [10.0.0.105/24]
      gateway4: 10.0.0.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
```
Apply:
```bash
sudo netplan apply
```

### Why
Keeps your alias consistent and avoids rediscovering IPs.

---

## 10. Repeatable bootstrap script example

```bash
#!/bin/bash
set -e

# Update
sudo apt update && sudo apt upgrade -y

# Enable unattended upgrades
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades

# Install core security tools
sudo apt install -y ufw fail2ban ubuntu-advantage-tools
sudo ufw allow OpenSSH
sudo ufw --force enable

# Attach Ubuntu Pro (replace placeholder with real token)
sudo pro attach YOUR_TOKEN_HERE
sudo pro enable esm-infra esm-apps livepatch

# Restart SSH to pick up any config changes
sudo systemctl restart ssh
```

**Important:** Do not commit private keys or tokens to source control.

---

## 11. Troubleshooting

### Permission denied (publickey)
- Wrong username? Try the exact account (e.g., `eissayouj`).  
- SSH agent not offering key? Use `ssh-add` or specify `-i`.  
- Public key not installed correctly: ensure `~/.ssh/authorized_keys` has the key, with `.ssh` at `700` and file at `600`, owned by the proper user.

### Host key changed warning
If you reinstall or regenerate host keys:
```bash
ssh-keygen -R <ip-or-hostname>
```
Then reconnect to accept new key.

### Cannot reach server
- `ping <ip>` to test network.  
- `nmap -p22 <ip>` or `nc -vz <ip> 22` to see if SSH port is open.  
- Check server firewall (`sudo ufw status`).  
- Ensure both devices are on same LAN/subnet.

---

## 12. Summary
You now have a repeatable, secure workflow that:
- Generates and secures SSH keys  
- Injects the public key via GitHub or manually  
- Uses SSH config aliasing for easy connects  
- Verifies host identity  
- Bootstraps and hardens the system (firewall, updates, brute-force defense)  
- Optionally uses Ubuntu Pro and mDNS name resolution  
- Ensures IP stability and includes fallbacks

---

## Security reminders
- Never commit your private key (`~/.ssh/ubuntu_server_ed25519`) or Ubuntu Pro token to GitHub.  
- Use per-server keypairs for compartmentalization if managing many machines.  
- Rotate keys if compromise is suspected.

---

## Next layers you can build
- Containerized services with Docker / Docker Compose  
- Reverse proxy with TLS (Caddy, Nginx + Certbot)  
- File sync (Nextcloud), media (Jellyfin), VPN (WireGuard), monitoring (Netdata)  
- Automated backups and remote access policies
