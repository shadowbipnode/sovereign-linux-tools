# SSH Hardening — Locking Down Remote Access

> Default SSH configuration is an open invitation. This guide covers every layer of SSH hardening for a production Linux server running Bitcoin and Lightning infrastructure.

---

## Why SSH hardening matters

SSH is the most attacked service on any internet-facing server. Within minutes of a new VPS going online, automated scanners are attempting logins on port 22 with common username/password combinations.

The default OpenSSH configuration prioritizes compatibility over security. You need to change that.

This guide assumes Ubuntu 24 LTS. Adjust paths for other distributions.

---

## Step 1 — Move SSH to a non-standard port

Port 22 is scanned by every automated tool on the internet. Moving to a different port eliminates most noise immediately.

```bash
sudo nano /etc/ssh/sshd_config
```

Find and change:
```
#Port 22
```

To a port of your choice (example: 2229):
```
Port 2229
```

**Do not restart SSH yet** — you need to complete the other steps first or you risk locking yourself out.

If you use UFW, allow the new port before restarting:
```bash
sudo ufw allow 2229/tcp
sudo ufw delete allow 22/tcp
```

---

## Step 2 — Disable password authentication

Password authentication is the attack surface. If you can only log in with a key, brute force attacks become irrelevant.

First, ensure you have SSH key authentication working. Generate a key on your local machine if you don't have one:

```bash
ssh-keygen -t ed25519 -C "your-label"
```

Copy the public key to the server:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2229 user@your-server-ip
```

Test that key authentication works **before** disabling passwords:
```bash
ssh -i ~/.ssh/id_ed25519 -p 2229 user@your-server-ip
```

Only after confirming key auth works, edit `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

---

## Step 3 — Disable root login

Root login over SSH should never be allowed. If an attacker gets in as root, the server is fully compromised.

```
PermitRootLogin no
```

If you need root access, log in as a regular user and use `sudo`.

---

## Step 4 — Additional hardening options

Add these to `/etc/ssh/sshd_config`:

```
# Disable unused authentication methods
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no

# Reduce attack surface
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitEmptyPasswords no

# Timeout settings
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30

# Limit authentication attempts
MaxAuthTries 3
MaxSessions 3

# Restrict to specific users (replace with your username)
AllowUsers yourusername
```

**ClientAliveInterval 300** — disconnects idle sessions after 5 minutes.  
**LoginGraceTime 30** — gives attackers only 30 seconds to authenticate.  
**AllowUsers** — only the listed users can log in via SSH. Anyone else is rejected before authentication even begins.

---

## Step 5 — Use Ed25519 keys

If you're still using RSA keys, migrate to Ed25519. It's faster, shorter, and considered more secure against future attacks.

```bash
# Generate Ed25519 key
ssh-keygen -t ed25519 -C "server-access-$(date +%Y)"

# View the public key
cat ~/.ssh/id_ed25519.pub
```

Add the public key to `~/.ssh/authorized_keys` on the server:
```bash
echo "your-public-key-here" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

---

## Step 6 — Apply the configuration

Test the configuration before restarting:
```bash
sudo sshd -t
```

If no errors, restart SSH:
```bash
sudo systemctl restart ssh
```

**Keep your current session open** and test the new configuration in a second terminal:
```bash
ssh -i ~/.ssh/id_ed25519 -p 2229 user@your-server-ip
```

Only close the original session after confirming the new connection works.

---

## Step 7 — Fail2ban for SSH

Even with key-only authentication, automated scanners will hammer your SSH port. Fail2ban blocks IPs after repeated failures.

```bash
sudo apt install fail2ban
```

Create a local configuration file:
```bash
sudo nano /etc/fail2ban/jail.local
```

Add:
```ini
[sshd]
enabled = true
port = 2229
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 2419200
findtime = 600
ignoreip = 127.0.0.1/8 ::1
```

**bantime = 2419200** is 28 days. Three failed attempts and the IP is gone for a month.

Enable and start:
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check status:
```bash
sudo fail2ban-client status sshd
```

---

## Step 8 — SSH client configuration

On your local machine, create or edit `~/.ssh/config` to avoid typing options every time:

```
Host myserver
    HostName your-server-ip
    User yourusername
    Port 2229
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
```

Now you can connect with just:
```bash
ssh myserver
```

---

## Step 9 — Audit your current exposure

Check what's currently listening on your server:
```bash
ss -tlnp | grep ssh
```

Check who is currently logged in:
```bash
who
w
```

Check recent login attempts:
```bash
sudo journalctl -u ssh --since "24 hours ago" | grep "Failed\|Accepted\|Invalid"
```

Check banned IPs:
```bash
sudo fail2ban-client status sshd | grep "Banned IP"
```

---

## Complete sshd_config reference

Here is the complete hardened configuration:

```
Port 2229
AddressFamily inet

HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key

SyslogFacility AUTH
LogLevel VERBOSE

LoginGraceTime 30
PermitRootLogin no
StrictModes yes
MaxAuthTries 3
MaxSessions 3

PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
KerberosAuthentication no
GSSAPIAuthentication no

X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PrintMotd no

ClientAliveInterval 300
ClientAliveCountMax 2

AllowUsers yourusername

AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```

---

## What this protects against

| Attack | Protected |
|--------|-----------|
| Brute force password attacks | ✅ Password auth disabled |
| Root login attempts | ✅ Root login disabled |
| Port scanners | ✅ Non-standard port |
| Credential stuffing | ✅ Key-only auth |
| Idle session hijacking | ✅ ClientAlive timeout |
| Repeated login attempts | ✅ Fail2ban 28-day ban |

---

## What this does NOT protect against

- **Physical access to the server** — SSH hardening is irrelevant if someone has physical access
- **Compromised SSH keys** — protect your private key with a strong passphrase and store it securely
- **Zero-day vulnerabilities in OpenSSH** — keep your system updated with `unattended-upgrades`
- **Supply chain attacks** — verify SSH binary integrity periodically

---

## Verify your hardening

Run a quick self-assessment:
```bash
# Check SSH version
ssh -V

# Verify no password auth
grep "PasswordAuthentication" /etc/ssh/sshd_config

# Verify port
ss -tlnp | grep ssh

# Check fail2ban is running
sudo systemctl is-active fail2ban

# Check banned count
sudo fail2ban-client status sshd | grep "Currently banned"
```

---

## For Bitcoin/Lightning node operators

If you're running a Lightning node, add these additional considerations:

- **Never expose port 22 or your SSH port** in your node's public announcements
- **Use ZeroTier or Tailscale** for remote access instead of exposing SSH directly to the internet when possible
- **Separate SSH keys** for different machines — never reuse the same key across your node and other servers
- **Backup your SSH keys** with the same rigor as your Lightning channel backup — losing access to your server means losing ability to manage your node

```bash
# Backup your SSH private key (store offline)
cp ~/.ssh/id_ed25519 /path/to/offline/backup/
chmod 600 /path/to/offline/backup/id_ed25519
```

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*
