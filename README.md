# sovereign-linux-tools

Practical guides for running a sovereign, private, and self-hosted Linux setup.

No theory. No marketing. Just working configurations and commands tested on real systems.

---

## Guides

| Guide | Description |
|-------|-------------|
| [gocryptfs](./guides/gocryptfs.md) | File-level encryption on Linux — lock sensitive data even while the system is running |
| [signal-opsec](./guides/signal-opsec.md) | Signal notification settings, metadata leaks, iOS vs GrapheneOS |
| [lnd-backup](./guides/lnd-backup.md) | Automated LND backup with GPG encryption and cloud redundancy |
| [sovereign-backup](./guides/sovereign-backup.md) | Triple-redundant encrypted backup for Bitcoin/Lightning nodes |
| [ssh-hardening](./guides/ssh-hardening.md) | Complete SSH hardening for production Linux servers |
| [tor-setup](./guides/tor-setup.md) | Complete guide Tor Setup — Running Bitcoin Core and LND over Tor |
| [tor-transparent-proxy-guide](./guides/tor-transparent-proxy-guide.md) | Complete guide to routing ALL System Traffic Through Tor on Rocky Linux 9 |

*More guides coming.*

---

## Philosophy

Full-disk encryption is table stakes.  
A VPN is not privacy.  
Self-custody means running your own infrastructure.  

These guides cover the practical gap between "I care about privacy" and "my setup actually reflects that."

Tools covered: encryption, node operation, key management, network hardening, backup, and more.

---

## Who this is for

- Bitcoin and Lightning node operators
- Cypherpunks tired of theoretical privacy advice
- Anyone running their own infrastructure and wanting it hardened

---

## Contributing

Found an error or have a better approach? Open an issue or PR.  
These are living documents — they improve with real-world feedback.

---

shadowbip — sovereign Bitcoin node operator, cypherpunk.  
Nostr: `npub1yag9ggwzdrekxput74qq66p88wv8r68r2f3lm3znycqqyh408ufs7htp3e`  
Lightning: `zap@shadowbip.com`  
Web: `https://shadowbip.com`  
Relay: `wss://relay.shadowbip.com`

---

*Verify everything. Trust nothing you haven't run yourself.*
