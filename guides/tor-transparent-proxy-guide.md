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

The scripts automatically detect your network gateway, so the setup works on any network — home, office, hotspot, or public WiFi.

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
| Bootstrap safety | Kill switch only activates after Tor is fully connected |
| Network portability | Scripts auto-detect gateway — works on any network |

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

IPv6 is a major leak vector. Tor does not handle IPv6 traffic, so any IPv6-capable application can bypass the proxy entirely. This is the single most common source of IP leaks — even with iptables rules active, IPv6 traffic flows through a completely separate stack (ip6tables) that must be blocked independently.

Disable immediately:

```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1
```

Disable on your specific interfaces (adjust names to match your system — find them with `ip link show`):

```bash
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

> **Important:** The `sysctl` setting `net.ipv6.conf.all.disable_ipv6 = 1` is not always enough. NetworkManager can re-enable IPv6 per-connection, and some interfaces may not respect the `all` flag. Always disable IPv6 on each specific interface AND in NetworkManager for reliable protection.

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

> **Why port 5399 instead of 5353?** Port 5353 is used by Avahi (mDNS) and Chromium-based browsers (including Brave). Using 5399 avoids conflicts that prevent Tor from starting.

---

## Step 3 — Fix SELinux Policies

Rocky Linux 9 runs SELinux in Enforcing mode by default. Tor needs permission to bind to the new ports (9040 and 5399). Without this step, Tor will fail to start with "Permission denied" errors.

Add the ports to SELinux policy:

```bash
sudo semanage port -a -t tor_port_t -p tcp 9040
sudo semanage port -a -t tor_port_t -p tcp 5399
sudo semanage port -a -t tor_port_t -p udp 5399
```

If `semanage port -a` fails with "already defined", use `-m` (modify) instead:

```bash
sudo semanage port -m -t tor_port_t -p tcp 5399
```

If Tor still fails to start after adding ports, generate a custom SELinux module from the denial logs:

```bash
# Temporarily set permissive to capture all denials
sudo setenforce 0
sudo systemctl restart tor

# Generate and install the policy module
sudo ausearch -m avc -ts today | grep tor | audit2allow -M tor_transparent
sudo semodule -i tor_transparent.pp

# Re-enable enforcing
sudo setenforce 1
sudo systemctl restart tor
```

Verify Tor is running with all three ports:

```bash
sudo systemctl status tor
ss -tlnp | grep -E "5399|9050|9040"
ss -ulnp | grep 5399
```

Expected output — three listeners:

```
LISTEN  127.0.0.1:9040   (TransPort - TCP)
LISTEN  127.0.0.1:9050   (SocksPort - TCP)
UNCONN  127.0.0.1:5399   (DNSPort - UDP)
```

---

## Step 4 — Disable Avahi (mDNS)

Avahi broadcasts your hostname and available services on the local network. This is a privacy leak and also conflicts with DNS port usage.

```bash
sudo systemctl stop avahi-daemon
sudo systemctl disable avahi-daemon
sudo systemctl mask avahi-daemon
```

---

## Step 5 — Create the Transparent Proxy Script

This script activates the transparent proxy. Key features:

- **Bootstrap verification:** Checks that Tor has fully connected before enabling the kill switch. If Tor can't connect, it exits without blocking your traffic.
- **IPv6 blocking:** Drops all IPv6 traffic via ip6tables as a second line of defense.
- **DNS locking:** Sets `resolv.conf` to `127.0.0.1` and makes it immutable.
- **DNS redirect ordering:** Redirects DNS to `127.0.0.1:53` *before* the loopback exclusion rule — this is critical (see [Design Decisions](#design-decisions) below).

```bash
sudo tee /usr/local/bin/tor-transparent.sh << 'SCRIPT'
#!/bin/bash
TOR_USER="toranon"
TOR_TRANS_PORT="9040"
TOR_DNS_PORT="5399"
NON_TOR="127.0.0.0/8 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16"

# Verify Tor is running
if ! systemctl is-active tor &>/dev/null; then
    echo "[!] Tor is not running. Starting..."
    systemctl start tor
    sleep 5
fi

# Verify bootstrap completed
if ! journalctl -u tor --no-pager -n 20 | grep -q "Bootstrapped 100%"; then
    echo "[!] Tor has not finished bootstrapping. Waiting 15s..."
    sleep 15
    if ! journalctl -u tor --no-pager -n 20 | grep -q "Bootstrapped 100%"; then
        echo "[ERROR] Tor cannot connect. Exiting without activating kill switch."
        exit 1
    fi
fi

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

### Design Decisions

**DNS redirect before loopback exclusion:** When `resolv.conf` points to `127.0.0.1`, applications send DNS queries to `127.0.0.1:53`. Without the early redirect rule, the loopback exclusion (`RETURN` for `127.0.0.0/8`) would skip the DNS redirect entirely, and queries would fail because nothing listens on `127.0.0.1:53`. By placing the DNS redirect rule *before* the loopback exclusion, DNS queries to `127.0.0.1:53` are properly redirected to Tor's DNSPort on `127.0.0.1:5399`.

**Bootstrap verification:** The script checks that Tor has completed its bootstrap (connected to the Tor network) before activating the kill switch. Without this check, activating the kill switch on a network where Tor can't connect would leave you with no internet and no way to recover remotely.

**Kill switch:** The final `DROP` rule ensures that if Tor crashes, all traffic is blocked rather than leaking in the clear.

---

## Step 6 — Create the Disable Script

This script deactivates the transparent proxy and automatically detects your current network gateway to restore DNS — so it works on any network, not just your home LAN.

```bash
sudo tee /usr/local/bin/tor-transparent-off.sh << 'SCRIPT'
#!/bin/bash
# Clean iptables
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X
iptables -P OUTPUT ACCEPT
# Restore IPv6
ip6tables -F
ip6tables -P OUTPUT ACCEPT
ip6tables -P INPUT ACCEPT
ip6tables -P FORWARD ACCEPT
# Unlock and restore DNS automatically
chattr -i /etc/resolv.conf 2>/dev/null
# Auto-detect current gateway
GW=$(ip route | grep "default" | head -1 | awk '{print $3}')
if [ -n "$GW" ]; then
    echo -e "nameserver $GW\nsearch Home" > /etc/resolv.conf
else
    echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8\nsearch Home" > /etc/resolv.conf
fi
echo "[OK] Tor Transparent Proxy DISABLED"
echo "DNS: $(grep nameserver /etc/resolv.conf)"
SCRIPT

sudo chmod +x /usr/local/bin/tor-transparent-off.sh
```

---

## Step 7 — Create the Emergency Recovery Script

If everything breaks and you have no network, use this from a local TTY (`Ctrl+Alt+F2`):

```bash
sudo tee /usr/local/bin/net-emergency.sh << 'SCRIPT'
#!/bin/bash
echo "[!] EMERGENCY NETWORK RECOVERY"
# Unlock resolv.conf
chattr -i /etc/resolv.conf 2>/dev/null
# Clean iptables
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X
iptables -P OUTPUT ACCEPT
# Restore IPv6
ip6tables -F
ip6tables -P OUTPUT ACCEPT
ip6tables -P INPUT ACCEPT
ip6tables -P FORWARD ACCEPT
# Auto-detect gateway
GW=$(ip route | grep "default" | head -1 | awk '{print $3}')
if [ -n "$GW" ]; then
    echo -e "nameserver $GW\nsearch Home" > /etc/resolv.conf
else
    echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8\nsearch Home" > /etc/resolv.conf
fi
echo "[OK] Network restored"
echo "DNS: $(grep nameserver /etc/resolv.conf)"
echo "GW:  $GW"
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

The `tor-transparent.sh` script handles locking `resolv.conf` with `chattr +i` automatically when activated.

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

# === Tor / VPN / Network ===
alias toron='sudo /usr/local/bin/tor-transparent.sh'
alias toroff='sudo /usr/local/bin/tor-transparent-off.sh'
alias torstatus='echo "=== IP ===" && curl -s --max-time 15 https://check.torproject.org/api/ip && echo "" && echo "=== Rules ===" && sudo iptables -L OUTPUT -n --line-numbers && echo "=== Tor ===" && systemctl is-active tor'
alias dnstor='sudo chattr -i /etc/resolv.conf 2>/dev/null; sudo bash -c "echo -e \"nameserver 127.0.0.1\nsearch Home\" > /etc/resolv.conf"; sudo chattr +i /etc/resolv.conf; echo "DNS via Tor (locked)"'
alias dnsreset='sudo chattr -i /etc/resolv.conf 2>/dev/null; GW=$(ip route | grep default | head -1 | awk "{print \$3}"); sudo bash -c "echo -e \"nameserver ${GW:-1.1.1.1}\nsearch Home\" > /etc/resolv.conf"; echo "DNS reset to ${GW:-1.1.1.1}"'
alias netfix='sudo /usr/local/bin/net-emergency.sh'
alias vpnon='sudo /usr/local/bin/tor-transparent-off.sh && sudo systemctl restart mullvad-daemon && sleep 2 && mullvad connect && sleep 5 && mullvad status'
alias vpnoff='mullvad disconnect && sleep 2'
EOF

source ~/.bashrc
```

| Alias | Function |
|-------|----------|
| `toron` | Activate transparent proxy + kill switch |
| `toroff` | Deactivate proxy, auto-detect gateway, restore DNS |
| `torstatus` | Check current IP, iptables rules, and Tor status |
| `dnstor` | Force DNS through Tor and lock resolv.conf |
| `dnsreset` | Auto-detect gateway and restore DNS |
| `netfix` | Emergency — restore everything to defaults |
| `vpnon` | Disable Tor + start Mullvad VPN |
| `vpnoff` | Disconnect Mullvad VPN |

---

## Step 11 — Mullvad VPN Integration (Optional)

If you have Mullvad VPN, you can switch between Tor and Mullvad. They should **never run simultaneously** — they conflict on routing and DNS.

**Workflow:**

| From | To | Command |
|------|----|---------|
| Tor | Mullvad | `vpnon` |
| Mullvad | Tor | `vpnoff && toron` |
| Tor | Normal | `toroff` |
| Normal | Tor | `toron` |

> **Note:** Mullvad uses nftables for its firewall rules. If Mullvad shows "Unable to apply firewall rules", restart the daemon after disabling Tor: `toroff && sudo systemctl restart mullvad-daemon && mullvad connect`

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

### Tor won't start — port 5353 already in use

Avahi or a Chromium-based browser is using port 5353. That's why this guide uses port 5399 instead. If you still have issues:

```bash
ss -ulnp | grep 5353
sudo systemctl mask avahi-daemon
```

### DNS resolution fails

Check if the DNS redirect is working:

```bash
# Direct test to Tor DNS
dig +short google.com @127.0.0.1 -p 5399

# If this works but normal dig fails, check the iptables redirect order
sudo iptables -t nat -L OUTPUT -n --line-numbers
```

The DNS redirect to `127.0.0.1` must appear **before** the loopback RETURN rule. If the order is wrong, DNS queries to `127.0.0.1:53` will be skipped by the loopback exclusion and never reach Tor's DNSPort.

### IPv6 leak after reboot

NetworkManager can re-enable IPv6. Ensure it's disabled per-connection:

```bash
for conn in $(nmcli -t -f NAME c show --active); do
    sudo nmcli connection modify "$conn" ipv6.method disabled
done
sudo nmcli general reload
```

### toron fails on a different network

If Tor can't bootstrap (e.g., the network blocks Tor relay ports), the script will exit without activating the kill switch. You'll see:

```
[ERROR] Tor cannot connect. Exiting without activating kill switch.
```

In this case, consider using Tor bridges:

```bash
# Add to /etc/tor/torrc.d/transparent.conf
UseBridges 1
ClientTransportPlugin obfs4 exec /usr/bin/obfs4proxy
Bridge obfs4 <bridge-address>
```

Get bridges from [https://bridges.torproject.org](https://bridges.torproject.org).

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
| `/etc/tor/torrc.d/transparent.conf` | Tor transparent proxy configuration |
| `/etc/sysctl.d/99-disable-ipv6.conf` | Permanently disable IPv6 |
| `/etc/NetworkManager/conf.d/no-dns.conf` | Prevent NM from overwriting DNS |
| `/usr/local/bin/tor-transparent.sh` | Activation script (with bootstrap check) |
| `/usr/local/bin/tor-transparent-off.sh` | Deactivation script (with auto-gateway detection) |
| `/usr/local/bin/net-emergency.sh` | Emergency recovery script (with auto-gateway detection) |
| `/etc/systemd/system/tor-transparent.service` | Boot persistence via systemd |
| `/etc/resolv.conf` | Locked to `127.0.0.1` when Tor is active |

---

## Security Considerations

- **UDP traffic is blocked.** Tor only supports TCP. Applications relying on UDP (VoIP, gaming, some video streaming) will not work.
- **Tor is not a silver bullet.** Browser fingerprinting, WebRTC leaks, and application-level leaks can still compromise anonymity. Harden your browser (disable WebRTC, remote HTML loading, telemetry).
- **Exit node trust.** Tor exit nodes can see unencrypted traffic. Always use HTTPS.
- **Local network services** listening on `0.0.0.0` are accessible from your LAN. Consider binding them to `127.0.0.1` if not needed externally.
- **ZeroTier / VPN tunnels** bypass Tor by design (their traffic stays within the local network ranges excluded by the iptables rules). Be aware of what traffic goes through these tunnels.
- **This setup does not anonymize traffic from other devices** on your network — only the local machine.
- **The bootstrap check is not bulletproof.** If Tor bootstraps successfully but later loses connectivity, the kill switch will block all traffic (which is the intended behavior — fail closed, not fail open).

---

## License

This guide is released under [MIT License](LICENSE). Use at your own risk.
