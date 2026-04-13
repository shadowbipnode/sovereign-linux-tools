# LND Backup — The Only Guide That Matters

> If you lose your `channel.backup` file and your node data at the same time, your funds in open channels are gone. Not recoverable. Not negotiable.
>
> This guide covers what to back up, why, and how to automate it properly.

---

## What you actually need to back up

Running an LND node means you have two categories of data:

**Critical — loss means loss of funds:**
- `channel.backup` — Static Channel Backup (SCB). This file lets you force-close all channels and recover funds if the node is unrecoverable. LND encrypts it internally with your wallet password.
- `wallet.db` — your on-chain wallet. If you have your seed phrase, you can regenerate this. If you don't, back it up.

**Important — loss means reconfiguration work:**
- `lnd.conf` — your node configuration
- `.env` files — credentials and API keys
- `docker-compose.yml` — your stack configuration
- Database dumps (LNBits, Nextcloud, etc.)

**Not worth backing up:**
- `channel.db` — too large, changes constantly, not useful for recovery
- Log files

---

## The mistake I made

When I started routing Lightning payments, I opened channels and assumed the node would just keep running. I didn't think about what happens if the VPS provider has a catastrophic failure, or if I accidentally wipe the wrong volume.

I was copying `channel.backup` manually, occasionally, to a single location. Not automated. Not verified. Not redundant.

I got lucky. The node never failed catastrophically. But when I actually looked at my backup situation, I realized I had no reliable recovery path.

**The lesson:** set up automated backup before you open your first channel. Not after.

---

## The channel.backup file

LND generates and updates `channel.backup` every time a channel state changes. It's located at:

```
/path/to/lnd/data/chain/bitcoin/mainnet/channel.backup
```

This file is already encrypted by LND using your wallet password. You can safely upload it to cloud storage without additional encryption — without your wallet password, it's useless to anyone else.

**What it does:**
If your node dies and you can't recover the channel database, you can use `channel.backup` to initiate a force-close on all your channels. Your peers will cooperate (they have no incentive not to) and your funds will be returned to your on-chain wallet after the timelock expires.

**What it doesn't do:**
It does not recover your routing history, your fee settings, or your channel database. It only recovers funds.

---

## Automated backup with rclone and GPG

Here's the complete backup script I run every hour via cron.

### Prerequisites

```bash
# Install rclone
curl https://rclone.org/install.sh | sudo bash

# Install sqlite3 (for LNBits backup)
sudo apt install sqlite3

# Configure rclone with your cloud provider
rclone config
```

### The backup script

```bash
#!/bin/bash
# --- PATH CONFIGURATION ---
SCB="/path/to/lnd/data/chain/bitcoin/mainnet/channel.backup"
LND_CONF="/path/to/lnd/lnd.conf"
LNBITS_DB="/path/to/lnbits/data/database.sqlite3"
DEST_REMOTE="gdrive:LND_Backups"
TEMP_DB="/tmp/lnbits_snap.sqlite3"
RCLONE_CONF="--config /home/user/.config/rclone/rclone.conf"
HASH_DIR="/var/cache/lnd_backup_hashes"
DATE=$(date +%Y-%m-%d_%H-%M)

mkdir -p "$HASH_DIR"

# --- GPG ENCRYPTION WITH CHANGE DETECTION ---
# Only re-encrypts and uploads if the file has changed
GPG_RECIPIENT="your@email.com"

encrypt_and_upload() {
    local file=$1
    local dest=$2
    local hash_file="$HASH_DIR/$(echo $file | tr '/' '_').sha256"
    local current_hash=$(sha256sum "$file" 2>/dev/null | cut -d' ' -f1)
    local stored_hash=$(cat "$hash_file" 2>/dev/null)

    if [ "$current_hash" = "$stored_hash" ]; then
        echo "[SKIP] $(basename $file) unchanged."
        return
    fi

    gpg --homedir /root/.gnupg --encrypt --armor --trust-model always \
        --recipient $GPG_RECIPIENT --output "${file}.gpg" "$file" 2>/dev/null

    if [ $? -eq 0 ]; then
        rclone copyto "${file}.gpg" "$dest.gpg" $RCLONE_CONF
        rm "${file}.gpg"
        echo "$current_hash" > "$hash_file"
        echo "[OK] $(basename $file) encrypted and uploaded."
    else
        echo "[ERROR] GPG encryption failed for $file"
    fi
}

# --- CHANNEL.BACKUP (uploaded in cleartext — LND encrypts it internally) ---
rclone copyto $SCB $DEST_REMOTE/node/channel.backup $RCLONE_CONF

# --- SENSITIVE CONFIG FILES (GPG encrypted, skipped if unchanged) ---
encrypt_and_upload $LND_CONF $DEST_REMOTE/node/lnd.conf

# --- LNBITS DATABASE SNAPSHOT ---
if [ -f "$LNBITS_DB" ]; then
    sqlite3 "$LNBITS_DB" ".backup '$TEMP_DB'"
    if [ $? -eq 0 ] && [ -f "$TEMP_DB" ]; then
        rclone copyto "$TEMP_DB" "$DEST_REMOTE/lnbits/lnbits_database.sqlite3" $RCLONE_CONF
        rclone copyto "$TEMP_DB" "$DEST_REMOTE/history/lnbits_$DATE.sqlite3" $RCLONE_CONF
        rm "$TEMP_DB"
    fi
fi

# --- CLEANUP: keep only 30 days of history ---
rclone delete --min-age 30d $DEST_REMOTE/history/ $RCLONE_CONF

echo "Backup completed: $DATE"
```

### Why GPG for config files but not channel.backup

`channel.backup` is already encrypted by LND. Uploading it in cleartext is safe — no one can use it without your wallet password.

Config files (`lnd.conf`, `.env` files) may contain credentials in plaintext. These should be GPG-encrypted before leaving your server.

The hash check prevents re-encrypting and re-uploading unchanged files on every run. Since GPG produces different ciphertext for the same input on each run (random nonce), rclone cannot detect changes in encrypted files — we handle this manually with SHA256 hashes.

---

## Setting up the cron job

```bash
sudo crontab -e
```

Add:

```
0 * * * * /bin/bash /path/to/backup.sh >> /var/log/lnd_backup.log 2>&1
```

This runs every hour. The log file lets you verify backups are working without checking manually.

---

## Setting up GPG for automated backup

The backup script runs as root via cron. Your GPG key must be in root's keyring.

```bash
# Generate a GPG key (if you don't have one)
gpg --full-generate-key
# Choose RSA 4096, no expiry

# Export your public key
gpg --export --armor your@email.com > /tmp/backup_pubkey.asc

# Import into root's keyring
sudo gpg --homedir /root/.gnupg --import /tmp/backup_pubkey.asc
rm /tmp/backup_pubkey.asc
```

**Critical:** export and store your private key offline before using it for backup encryption.

```bash
gpg --export-secret-keys --armor your@email.com > backup_private.asc
```

Store this file:
- On a USB drive kept offline
- In a password manager that supports file attachments
- On a second device

If you lose the private key, the encrypted backups are unrecoverable.

---

## Redundant cloud destinations

Single cloud = single point of failure. Use at least two:

```bash
# After backing up to primary (Google Drive in this example),
# sync to a second destination
rclone sync gdrive:LND_Backups backblaze:lnd-backups-mirror $RCLONE_CONF --exclude "history/**"
```

Exclude `history/**` from the mirror — you only need current files on the secondary, not 30 days of snapshots.

---

## Recovery procedure

If your node dies and you need to recover:

**Step 1 — Restore channel.backup to a new LND instance:**

```bash
# Copy channel.backup to your new node
lncli restorechanbackup --multi_file /path/to/channel.backup
```

**Step 2 — Wait for force-closes:**
LND will broadcast force-close transactions for all channels. Your peers will see these and cooperate. Funds return to your on-chain wallet after the timelock expires (typically 144 blocks — about 24 hours, but can be longer depending on channel settings).

**Step 3 — Decrypt and restore config files:**

```bash
gpg --decrypt lnd.conf.gpg > lnd.conf
```

---

## Verification

Don't assume your backups work — verify them.

**Check that channel.backup is being updated:**

```bash
ls -la /path/to/lnd/data/chain/bitcoin/mainnet/channel.backup
# Timestamp should update whenever channel state changes
```

**Check the cloud backup:**

```bash
rclone ls gdrive:LND_Backups/node/
# channel.backup should be present and recently modified
```

**Test GPG decryption monthly:**

```bash
gpg --decrypt gdrive_local_copy/node/lnd.conf.gpg > /tmp/test_decrypt.conf
cat /tmp/test_decrypt.conf
rm /tmp/test_decrypt.conf
```

If decryption fails, fix it before you need it.

---

## What SCB recovery does NOT give you

Be realistic about what Static Channel Backup recovery means:

- ✅ Recovers funds from all open channels
- ✅ Works even if peers are uncooperative (force-close is unilateral)
- ❌ Does not recover routing history or earned fees
- ❌ Does not recover channel database or HTLC state
- ❌ Requires waiting for timelock expiry (can be days)
- ❌ Incurs on-chain fees for each force-close transaction

SCB is a last resort, not a migration tool. Keep your node healthy so you never need it.

---

## Bottom line

Your `channel.backup` file is worth more than your node hardware. Back it up automatically, verify it regularly, store it redundantly.

The five minutes it takes to set up a cron job is the cheapest insurance you can buy for your Lightning funds.

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*
