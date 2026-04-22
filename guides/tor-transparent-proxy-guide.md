# Routing ALL System Traffic Through Tor on Rocky Linux 9

A complete guide to configuring a transparent Tor proxy on Rocky Linux 9, forcing all TCP traffic and DNS queries through the Tor network with a kill switch to prevent leaks.

> **Tested on:** Rocky Linux 9.7 (Blue Onyx) — Kernel 5.14.0 — SELinux Enforcing

## Table of Contents

- [Overview](#overview)
- [What This Setup Does](#what-this-setup-does)
- [Prerequisites](#prerequisites)
- [Step 1 — Disable IPv6](#step-1--disable-ipv6)
- [Step 2 — Install and Configure Tor](#step-2--install-and-configure-tor)
- [Step 3 — Fix SELinux Policies](#step-3--fix-selinux-policies)
- [Step 4 — Disable Avahi (mDNS)](#step-4--disable-avahi-mdns)
- [Step 5 — Create the Transparent Proxy Script](#step-5--create-the-transparent-proxy-script)
- [Step 6 — Create the Disable Script](#step-6--create-the-disable-script)
- [Step 7 — Create the Emergency Recovery Script](#step-7--create-the-emergency-recovery-script)
- [Step 8 — Lock Down DNS](#step-8--lock-down-dns)
- [Step 9 — Enable at Boot](#step-9--enable-at-boot)
- [Step 10 — Shell Aliases](#step-10--shell-aliases)
- [Step 11 — Mullvad VPN Integration (Optional)](#step-11--mullvad-vpn-integration-optional)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Files Modified](#files-modified)
- [Security Considerations](#security-considerations)

---

## Overview

This guide configures your Rocky Linux 9 system to route **all** outgoing TCP traffic and DNS queries through Tor using a transparent proxy. A kill switch ensures that if Tor goes down, no traffic leaks out in the clear.

## What This Setup Does

| Layer | Protection |
|-------|-----------|
| All TCP traffic | Redirected through Tor TransPort (9040) |
| All DNS queries | Resolved through Tor DNSPort (5399) |
| Kill switch | iptables DROP rule blocks any non-Tor traffic |
| IPv6 | Completely disabled (sysctl + ip6tables DROP) |
| DNS override | `resolv.conf` locked to `127.0.0.1` with `chattr +i` |
| mDNS/Avahi | Disabled and masked |
| Boot persistence | systemd service restores rules on reboot |
| Exit node filtering | Excludes nodes in high-surveillance countries |

---

## Prerequisites

```bash
sudo dnf install -y epel-release
sudo dnf install -y tor policycoreutils-python-utils
```

Verify Tor is installed:

```bash
rpm -q tor
```

Identify the Tor user (you'll need this later):

```bash
ps -eo user,pid,comm | grep tor
# Typically "toranon" on Rocky Linux 9
```

---

## Step 1 — Disable IPv6

IPv6 is a major leak vector. Tor does not handle IPv6 traffic, so any IPv6-capable application can bypass the proxy entirely.

```bash
# Disable immediately
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1

# Disable on specific interfaces (adjust names to your system)
sudo sysctl -w net.ipv6.conf.wlo1.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.enp0s20f0u1.disable_ipv6=1
```

Make it permanent:

```bash
sudo tee /etc/sysctl.d/99-disable-ipv6.conf << 'EOF'
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.wlo1.disable_ipv6 = 1
net.ipv6.conf.enp0s20f0u1.disable_ipv6 = 1
EOF

sudo sysctl --system
```

Also disable IPv6 in NetworkManager for each active connection:

```bash
sudo nmcli connection modify "YourConnectionName" ipv6.method disabled
```

Verify no global IPv6 addresses remain:

```bash
ip -6 addr show scope global
# Should return nothing
```

> **Note:** Replace `wlo1` and `enp0s20f0u1` with your actual interface names. Find them with `ip link show`.

---

## Step 2 — Install and Configure Tor

Create the transparent proxy configuration:

```bash
sudo tee /etc/tor/torrc.d/transparent.conf << 'EOF'
# Transparent Proxy
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsOnResolve 1
TransPort 9040 IsolateClientAddr IsolateClientProtocol IsolateDestAddr IsolateDestPort
DNSPort 5399

# Security
SocksPort 9050
SafeSocks 1
TestSocks 1

# Exclude exit nodes in high-surveillance countries
ExcludeExitNodes {ru},{cn},{ir},{by},{kz}
StrictNodes 1
EOF
```

> **Why port 5399 instead of 5353?** Port 5353 is used by Avahi (mDNS) and Chromium-based browsers. Using 5399 avoids conflicts.

---

## Step 3 — Fix SELinux Policies

Rocky Linux 9 runs SELinux in Enforcing mode. Tor needs permission to bind to the new ports.

```bash
# Add ports to SELinux policy
sudo semanage port -a -t tor_port_t -p tcp 9040
sudo semanage port -a -t tor_port_t -p tcp 5399
sudo semanage port -a -t tor_port_t -p udp 5399
```

If `semanage port -a` fails with "already defined", use `-m` instead:

```bash
sudo semanage port -m -t tor_port_t -p tcp 5399
```

If Tor still fails to start, generate a custom SELinux module:

```bash
# Temporarily set permissive to capture denials
sudo setenforce 0
sudo systemctl restart tor

# Generate and install the policy module
sudo ausearch -m avc -ts today | grep tor | audit2allow -M tor_transparent
sudo semodule -i tor_transparent.pp

# Re-enable enforcing
sudo setenforce 1
sudo systemctl restart tor
```

Verify Tor is running:

```bash
sudo systemctl status tor
ss -tlnp | grep -E "5399|9050|9040"
ss -ulnp | grep 5399
```

Expected output — three listeners:

```
LISTEN  127.0.0.1:9040   (TransPort)
LISTEN  127.0.0.1:9050   (SocksPort)
UNCONN  127.0.0.1:5399   (DNSPort - UDP)
```

---

## Step 4 — Disable Avahi (mDNS)

Avahi broadcasts your hostname and services on the local network — a privacy leak.

```bash
sudo systemctl stop avahi-daemon
sudo systemctl disable avahi-daemon
sudo systemctl mask avahi-daemon
```

---

## Step 5 — Create the Transparent Proxy Script

```bash
sudo tee /usr/local/bin/tor-transparent.sh << 'SCRIPT'
#!/bin/bash
TOR_USER="toranon"
TOR_TRANS_PORT="9040"
TOR_DNS_PORT="5399"
NON_TOR="127.0.0.0/8 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16"

### CLEANUP ###
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X

### BLOCK IPv6 ###
ip6tables -F
ip6tables -P OUTPUT DROP
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -A OUTPUT -o lo -j ACCEPT
ip6tables -A INPUT -i lo -j ACCEPT

### LOCK DNS ###
chattr -i /etc/resolv.conf 2>/dev/null
echo -e "nameserver 127.0.0.1\nsearch Home" > /etc/resolv.conf
chattr +i /etc/resolv.conf

### NAT — REDIRECT ###
# Let Tor's own traffic pass (avoid loops)
iptables -t nat -A OUTPUT -m owner --uid-owner $TOR_USER -j RETURN

# Redirect DNS to 127.0.0.1 BEFORE the loopback exclusion
# (resolv.conf points to 127.0.0.1, so DNS goes to loopback)
iptables -t nat -A OUTPUT -d 127.0.0.1 -p udp --dport 53 -j REDIRECT --to-ports $TOR_DNS_PORT
iptables -t nat -A OUTPUT -d 127.0.0.1 -p tcp --dport 53 -j REDIRECT --to-ports $TOR_DNS_PORT

# Exclude local networks
for NET in $NON_TOR; do
    iptables -t nat -A OUTPUT -d $NET -j RETURN
done

# Redirect all remaining DNS and TCP
iptables -t nat -A OUTPUT -p udp --dport 53 -j REDIRECT --to-ports $TOR_DNS_PORT
iptables -t nat -A OUTPUT -p tcp --dport 53 -j REDIRECT --to-ports $TOR_DNS_PORT
iptables -t nat -A OUTPUT -p tcp --syn -j REDIRECT --to-ports $TOR_TRANS_PORT

### FILTER — KILL SWITCH ###
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -m owner --uid-owner $TOR_USER -j ACCEPT
for NET in $NON_TOR; do
    iptables -A OUTPUT -d $NET -j ACCEPT
done
iptables -A OUTPUT -j DROP

echo "[OK] Tor Transparent Proxy ACTIVE with kill switch + IPv6 blocked"
SCRIPT

sudo chmod +x /usr/local/bin/tor-transparent.sh
```

### Key Design Decisions

**DNS redirect before loopback exclusion:** When `resolv.conf` points to `127.0.0.1`, applications send DNS queries to `127.0.0.1:53`. Without the early redirect rule, the loopback exclusion (`RETURN` for `127.0.0.0/8`) would skip the DNS redirect entirely, and queries would fail because nothing listens on `127.0.0.1:53`. By placing the DNS redirect rule before the loopback exclusion, DNS queries to `127.0.0.1:53` are properly redirected to Tor's DNSPort on `127.0.0.1:5399`.

**Kill switch:** The final `DROP` rule ensures that if Tor crashes, all traffic is blocked rather than leaking in the clear.

---

## Step 6 — Create the Disable Script

```bash
sudo tee /usr/local/bin/tor-transparent-off.sh << 'SCRIPT'
#!/bin/bash
# Clean iptables
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X
iptables -P OUTPUT ACCEPT
# Unlock and restore DNS
chattr -i /etc/resolv.conf
echo -e "nameserver 192.168.0.1\nsearch Home" > /etc/resolv.conf
echo "[OK] Tor Transparent Proxy DISABLED — normal traffic restored"
SCRIPT

sudo chmod +x /usr/local/bin/tor-transparent-off.sh
```

> **Note:** Replace `192.168.0.1` with your router's IP address.

---

## Step 7 — Create the Emergency Recovery Script

If everything breaks and you have no network, use this from a local TTY (`Ctrl+Alt+F2`):

```bash
sudo tee /usr/local/bin/net-emergency.sh << 'SCRIPT'
#!/bin/bash
echo "[!] EMERGENCY NETWORK RECOVERY"
chattr -i /etc/resolv.conf
echo -e "nameserver 192.168.0.1\nsearch Home" > /etc/resolv.conf
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X
iptables -P OUTPUT ACCEPT
ip6tables -F
ip6tables -P OUTPUT ACCEPT
ip6tables -P INPUT ACCEPT
ip6tables -P FORWARD ACCEPT
echo "[OK] Network restored to default state"
echo "DNS: $(cat /etc/resolv.conf | grep nameserver)"
SCRIPT

sudo chmod +x /usr/local/bin/net-emergency.sh
```

---

## Step 8 — Lock Down DNS

Prevent NetworkManager from overwriting `resolv.conf`:

```bash
sudo tee /etc/NetworkManager/conf.d/no-dns.conf << 'EOF'
[main]
dns=none
EOF

sudo systemctl restart NetworkManager
```

The `tor-transparent.sh` script handles locking `resolv.conf` with `chattr +i` automatically.

---

## Step 9 — Enable at Boot

Enable Tor:

```bash
sudo systemctl enable tor
```

Create a systemd service for the iptables rules:

```bash
sudo tee /etc/systemd/system/tor-transparent.service << 'EOF'
[Unit]
Description=Tor Transparent Proxy iptables rules
After=tor.service network-online.target
Requires=tor.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/tor-transparent.sh
ExecStop=/usr/local/bin/tor-transparent-off.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable tor-transparent.service
```

---

## Step 10 — Shell Aliases

Add convenience aliases to your `~/.bashrc`:

```bash
cat >> ~/.bashrc << 'EOF'

# === Tor Transparent Proxy ===
alias toron='sudo /usr/local/bin/tor-transparent.sh'
alias toroff='sudo /usr/local/bin/tor-transparent-off.sh'
alias torstatus='echo "=== IP ===" && curl -s --max-time 15 https://check.torproject.org/api/ip && echo "" && echo "=== Rules ===" && sudo iptables -L OUTPUT -n --line-numbers && echo "=== Tor ===" && systemctl is-active tor'
alias dnstor='sudo chattr -i /etc/resolv.conf && sudo bash -c "echo -e \"nameserver 127.0.0.1\nsearch Home\" > /etc/resolv.conf" && sudo chattr +i /etc/resolv.conf && echo "DNS via Tor (locked)"'
alias dnsreset='sudo chattr -i /etc/resolv.conf && sudo bash -c "echo -e \"nameserver 192.168.0.1\nsearch Home\" > /etc/resolv.conf" && echo "DNS restored to router"'
alias netfix='sudo /usr/local/bin/net-emergency.sh'
EOF

source ~/.bashrc
```

| Alias | Function |
|-------|----------|
| `toron` | Activate transparent proxy + kill switch |
| `toroff` | Deactivate proxy, restore normal traffic |
| `torstatus` | Check current IP, rules, and Tor status |
| `dnstor` | Force DNS through Tor |
| `dnsreset` | Restore DNS to local router |
| `netfix` | Emergency — restore everything to defaults |

---

## Step 11 — Mullvad VPN Integration (Optional)

If you have Mullvad VPN, you can switch between Tor and Mullvad. They should **never run simultaneously** — they conflict on routing and DNS.

Add these aliases:

```bash
cat >> ~/.bashrc << 'EOF'

# === Tor <-> Mullvad Switch ===
alias vpnon='sudo /usr/local/bin/tor-transparent-off.sh && sudo systemctl restart mullvad-daemon && sleep 2 && mullvad connect && sleep 5 && mullvad status'
alias vpnoff='mullvad disconnect && sleep 2'
EOF

source ~/.bashrc
```

**Workflow:**

| From | To | Command |
|------|----|---------|
| Tor | Mullvad | `vpnon` |
| Mullvad | Tor | `vpnoff && toron` |
| Tor | Normal | `toroff` |
| Normal | Tor | `toron` |

---

## Verification

### Quick Check

```bash
torstatus
```

### Full Verification

```bash
echo "=== Public IP ==="
curl -s https://check.torproject.org/api/ip

echo "=== DNS Leak Test ==="
dig +short whoami.akamai.net @ns1-1.akamaitech.net

echo "=== IPv6 Leak ==="
ip -6 addr show scope global
curl -s --max-time 5 https://ipv6.icanhazip.com 2>/dev/null && echo "IPv6 LEAK!" || echo "IPv6 blocked OK"

echo "=== Kill Switch ==="
sudo iptables -L OUTPUT -n --line-numbers

echo "=== DNS Config ==="
cat /etc/resolv.conf
lsattr /etc/resolv.conf

echo "=== Services ==="
systemctl is-active tor
systemctl is-enabled tor-transparent
```

**Expected results:**

- `"IsTor": true` — all traffic exits via Tor
- DNS resolves to a non-ISP IP
- No global IPv6 addresses
- `resolv.conf` shows `nameserver 127.0.0.1` with immutable attribute (`----i---------`)
- iptables OUTPUT chain ends with `DROP`

---

## Troubleshooting

### Tor won't start — "Permission denied" on bind

SELinux is blocking Tor from binding to the new ports.

```bash
sudo ausearch -m avc -ts recent | grep tor
sudo ausearch -m avc -ts today | grep tor | audit2allow -M tor_fix
sudo semodule -i tor_fix.pp
sudo systemctl restart tor
```

### DNS resolution fails

Check if the DNS redirect is working:

```bash
# Direct test to Tor DNS
dig +short google.com @127.0.0.1 -p 5399

# If this works but normal dig fails, the iptables redirect isn't catching it
sudo iptables -t nat -L OUTPUT -n | grep 5399
```

### IPv6 leak after reboot

NetworkManager can re-enable IPv6. Ensure it's disabled per-connection:

```bash
for conn in $(nmcli -t -f NAME c show --active); do
    sudo nmcli connection modify "$conn" ipv6.method disabled
done
sudo nmcli general reload
```

### Lost all connectivity

From a local TTY (`Ctrl+Alt+F2`):

```bash
sudo /usr/local/bin/net-emergency.sh
```

### Mullvad says "Unable to apply firewall rules"

Restart the Mullvad daemon after disabling Tor:

```bash
toroff
sudo systemctl restart mullvad-daemon
sleep 3
mullvad connect
```

---

## Files Modified

| File | Purpose |
|------|---------|
| `/etc/tor/torrc.d/transparent.conf` | Tor transparent proxy config |
| `/etc/sysctl.d/99-disable-ipv6.conf` | Permanently disable IPv6 |
| `/etc/NetworkManager/conf.d/no-dns.conf` | Prevent NM from overwriting DNS |
| `/usr/local/bin/tor-transparent.sh` | Activation script |
| `/usr/local/bin/tor-transparent-off.sh` | Deactivation script |
| `/usr/local/bin/net-emergency.sh` | Emergency recovery script |
| `/etc/systemd/system/tor-transparent.service` | Boot persistence |
| `/etc/resolv.conf` | Locked to `127.0.0.1` |

---

## Security Considerations

- **UDP traffic is blocked.** Tor only supports TCP. Applications relying on UDP (VoIP, gaming, some video streaming) will not work.
- **Tor is not a silver bullet.** Browser fingerprinting, WebRTC leaks, and application-level leaks can still compromise anonymity. Harden your browser (disable WebRTC, remote HTML loading, telemetry).
- **Exit node trust.** Tor exit nodes can see unencrypted traffic. Always use HTTPS.
- **Local network services** (ports on `0.0.0.0`) are accessible from your LAN. Consider binding them to `127.0.0.1` if not needed externally.
- **ZeroTier / VPN tunnels** bypass Tor by design. Be aware of what traffic goes through these tunnels.
- **This setup does not anonymize traffic from other devices** on your network — only the local machine.

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*


