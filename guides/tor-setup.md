# Tor Setup — Running Bitcoin Core and LND over Tor

> Your Bitcoin node broadcasts your IP address to every peer it connects to. This guide covers how to route all node traffic through Tor, making your node's location and identity invisible to the network.

---

## Why Tor for a Bitcoin node

When you run Bitcoin Core without Tor, every peer you connect to sees your IP address. This means:

- Your ISP can see that you run a Bitcoin node
- Anyone on the network can map your IP to your node
- Blockchain analytics companies correlate your IP with your transactions
- Your node's location is publicly visible on sites like Bitnodes

With Tor, your node connects through a chain of encrypted relays. Your real IP is never exposed to peers. Inbound connections reach you via a `.onion` address that reveals nothing about your location or identity.

---

## Prerequisites

- Ubuntu 24 LTS (or similar Debian-based system)
- Bitcoin Core installed and synced
- LND installed (optional, for Lightning setup)
- Root or sudo access

---

## Step 1 — Install Tor

```bash
sudo apt update
sudo apt install tor -y
```

Verify installation:
```bash
tor --version
sudo systemctl status tor
```

---

## Step 2 — Configure Tor

Edit the Tor configuration file:
```bash
sudo nano /etc/tor/torrc
```

Add or uncomment these lines:

```
# Enable SOCKS proxy for outbound connections
SOCKSPort 9050

# Enable control port (required for LND)
ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1

# Create hidden service for Bitcoin Core
HiddenServiceDir /var/lib/tor/bitcoin/
HiddenServicePort 8333 127.0.0.1:8333

# Create hidden service for LND (if using Lightning)
HiddenServiceDir /var/lib/tor/lnd/
HiddenServicePort 9735 127.0.0.1:9735
```

Restart Tor:
```bash
sudo systemctl restart tor
sudo systemctl enable tor
```

---

## Step 3 — Get your .onion addresses

After restarting Tor, your hidden service addresses are generated automatically:

```bash
# Bitcoin Core onion address
sudo cat /var/lib/tor/bitcoin/hostname

# LND onion address
sudo cat /var/lib/tor/lnd/hostname
```

Save these addresses — you will need them for configuration and to share with peers.

---

## Step 4 — Configure Bitcoin Core

Edit your `bitcoin.conf`:
```bash
nano ~/.bitcoin/bitcoin.conf
```

Add these settings:

```ini
# Route all outbound connections through Tor
proxy=127.0.0.1:9050

# Enable onion routing
onion=127.0.0.1:9050

# Listen on your .onion address
listenonion=1

# Disable DNS seeding (leaks your IP before Tor connects)
dnsseed=0
forcednsseed=0

# Only connect to onion peers (onion-only mode)
# Remove this line if you want dual-stack (clearnet + onion)
onlynet=onion

# Bind to localhost only
bind=127.0.0.1

# Your onion address (replace with your actual .onion address)
# externalip=youraddress.onion
```

Restart Bitcoin Core:
```bash
bitcoin-cli stop
bitcoind -daemon
```

Verify Tor is working:
```bash
bitcoin-cli getnetworkinfo | grep -A5 "onion"
```

You should see `"reachable": true` for the onion network.

---

## Step 5 — Configure LND over Tor

Edit your `lnd.conf`:
```bash
nano ~/.lnd/lnd.conf
```

Add these settings:

```ini
[tor]
# Enable Tor
tor.active=1

# Use Tor v3 hidden services
tor.v3=1

# Tor SOCKS proxy
tor.socks=127.0.0.1:9050

# Tor control port
tor.control=127.0.0.1:9051

# Stream isolation — each connection uses a different Tor circuit
tor.streamisolation=1

[Bitcoin]
bitcoin.active=1
bitcoin.mainnet=1
bitcoin.node=bitcoind
```

For **onion-only mode** (maximum privacy), add:
```ini
[Application Options]
# Only advertise onion address
externalip=youraddress.onion

# Do not use clearnet
nolisten=true
```

For **dual-stack mode** (better connectivity, slightly less private):
```ini
[Application Options]
# Advertise both clearnet and onion
externalip=your.ip.address
externalip=youraddress.onion
```

Restart LND:
```bash
lncli stop
lnd &
```

---

## Step 6 — Verify your setup

**Check Bitcoin Core is routing through Tor:**
```bash
bitcoin-cli getpeerinfo | grep -E '"addr"|"network"' | head -20
```

All peers should show `"network": "onion"` if you are in onion-only mode.

**Check your node's network status:**
```bash
bitcoin-cli getnetworkinfo
```

Look for:
- `"proxy": "127.0.0.1:9050"` — Tor proxy active
- `"onion"` section with `"reachable": true`

**Check LND is using Tor:**
```bash
lncli getinfo | grep -E "uris|identity"
```

Your node URI should end in `.onion:9735`.

**Test Tor connectivity:**
```bash
curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org/api/ip
```

This should return a Tor exit node IP, not your real IP.

---

## Step 7 — Add your node to the Tor group (for LND cookie auth)

LND needs to read Tor's cookie file for control port authentication:

```bash
# Add your user to the tor group
sudo usermod -aG tor $(whoami)

# Verify
groups $(whoami)
```

Log out and back in for the group change to take effect.

---

## Onion-only vs Dual-stack

| Mode | Privacy | Connectivity | Recommended for |
|------|---------|--------------|-----------------|
| Onion-only | Maximum | Fewer peers, slower sync | Privacy-first setups |
| Dual-stack | High | More peers, faster sync | Routing nodes, higher uptime |

**Onion-only** hides your IP completely but limits you to peers that support Tor. Initial sync is slower.

**Dual-stack** exposes your IP for clearnet connections but gives you access to the full network. Your onion address is still available for privacy-conscious peers.

For a routing Lightning node focused on uptime and liquidity, dual-stack is often the better choice. For maximum sovereignty, go onion-only.

---

## What Tor protects

- ✅ Your IP address is hidden from Bitcoin peers
- ✅ Your ISP cannot see you are connecting to the Bitcoin network
- ✅ Your Lightning node location is not publicly visible
- ✅ Inbound connections reach you without revealing your IP
- ✅ Blockchain analytics cannot correlate your IP with your transactions

## What Tor does NOT protect

- ❌ Transaction graph analysis — Tor hides your IP, not your on-chain activity
- ❌ Lightning channel graph — your channels and capacity are still public
- ❌ Timing attacks at the Tor exit — if you use clearnet Lightning invoices
- ❌ Compromised Tor nodes — Tor is not bulletproof, just much harder to trace
- ❌ Your node identity — your public key is still visible on the Lightning graph

---

## Common issues and fixes

**Bitcoin Core not connecting to peers:**
```bash
# Check Tor is running
sudo systemctl status tor

# Check SOCKS port is listening
ss -tlnp | grep 9050

# Test Tor manually
curl --socks5-hostname 127.0.0.1:9050 https://wtfismyip.com/text
```

**LND fails to start with Tor error:**
```bash
# Check LND has access to Tor cookie
ls -la /var/lib/tor/
sudo chmod 644 /var/lib/tor/control.authcookie

# Verify group membership
groups
```

**Slow initial sync in onion-only mode:**

This is expected. Onion-only mode limits your peer pool. You can temporarily switch to dual-stack for the initial sync, then switch back to onion-only once synced.

**Check if your IP is leaking:**
```bash
# Before enabling Tor
curl https://wtfismyip.com/text

# After enabling Tor (should return a different IP)
curl --socks5-hostname 127.0.0.1:9050 https://wtfismyip.com/text
```

---

## Docker setup

If you run Bitcoin Core and LND in Docker, configure Tor on the host and pass the SOCKS proxy address to the containers:

```yaml
services:
  bitcoin:
    command:
      - -proxy=172.17.0.1:9050
      - -onion=172.17.0.1:9050
      - -listenonion=1
      - -dnsseed=0

  lnd:
    command: >
      lnd
      --tor.active
      --tor.v3
      --tor.socks=172.17.0.1:9050
      --tor.control=172.17.0.1:9051
```

Replace `172.17.0.1` with your Docker bridge gateway IP:
```bash
docker network inspect bridge | grep Gateway
```

---

## Monitoring your Tor node

Check how many onion peers you have:
```bash
bitcoin-cli getpeerinfo | grep '"network": "onion"' | wc -l
```

Check Tor circuit status:
```bash
sudo -u debian-tor tor-prompt
# Then type: GETINFO circuit-status
```

Monitor Tor logs:
```bash
sudo journalctl -u tor --since "1 hour ago" -f
```

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*
