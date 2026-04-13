# Sovereign Backup — Triple Cloud Redundancy for Bitcoin Nodes

> A backup system is only as good as the last time you tested it.  
> This guide covers automated, encrypted, triple-redundant backup for a Bitcoin/Lightning node stack.

---

## What this covers

- Automated hourly backup via cron
- GPG encryption for sensitive files
- Smart change detection — only re-encrypts files that actually changed
- Three independent cloud destinations (Google Drive, MEGA, Backblaze B2)
- Telegram alerts on failure
- 30-day history with automatic cleanup

---

## What gets backed up

| File | Destination | Encrypted |
|------|-------------|-----------|
| `channel.backup` | GDrive + B2 | No (LND encrypts internally) |
| `lnd.conf` | GDrive + B2 | Yes (GPG) |
| `.env` files | GDrive + B2 | Yes (GPG) |
| `bitcoin.conf` | GDrive + B2 | No |
| `docker-compose.yml` | GDrive + B2 | No |
| LNBits SQLite DB | GDrive + B2 + history | No |
| Nextcloud MariaDB dump | GDrive + B2 + history | No |

**Why `channel.backup` is not GPG-encrypted:**  
LND already encrypts it internally with your wallet password. Anyone who obtains it cannot use it without that password. Leaving it in cleartext allows rclone to detect changes and skip unnecessary uploads.

**Why config files ARE GPG-encrypted:**  
They may contain credentials in plaintext. GPG encryption ensures that even if cloud storage is compromised, credentials remain safe.

---

## Prerequisites

```bash
# rclone
curl https://rclone.org/install.sh | sudo bash

# sqlite3 (for LNBits backup)
sudo apt install sqlite3

# megacmd (for MEGA sync)
curl -O https://mega.nz/linux/repo/xUbuntu_24.04/amd64/megacmd-xUbuntu_24.04_amd64.deb
sudo apt install ./megacmd-xUbuntu_24.04_amd64.deb
mega-login your@email.com 'yourpassword'
```

---

## Cloud configuration

### Google Drive

```bash
rclone config
# Type: drive
# Follow OAuth flow in browser
```

### Backblaze B2

1. Create account at backblaze.com (10GB free, no credit card)
2. Create a bucket
3. Create an Application Key with read/write access to that bucket only
4. Configure rclone:

```bash
rclone config
# Type: b2
# Account: your keyID
# Key: your applicationKey
```

### MEGA

```bash
sudo apt install megacmd
mega-login your@email.com 'yourpassword'
mega-ls  # verify connection
```

Note: MEGA is used via megacmd, not rclone, due to API compatibility issues.

---

## GPG setup for automated backup

The script runs as root via cron. The GPG key must be in root's keyring.

```bash
# Generate key (if needed)
gpg --full-generate-key
# RSA 4096, no expiry

# Export public key
gpg --export --armor your@email.com > /tmp/pubkey.asc

# Import into root's keyring
sudo gpg --homedir /root/.gnupg --import /tmp/pubkey.asc
rm /tmp/pubkey.asc
```

**Critical:** export and store your private key offline before using it.

```bash
gpg --export-secret-keys --armor your@email.com > private_key_backup.asc
```

Store this file:
- On an offline USB drive
- In a password manager with file attachment support
- On a second device

If the private key is lost, all GPG-encrypted backups are unrecoverable.

---

## The backup script

Adapt paths and credentials to your setup before deploying.

```bash
#!/bin/bash
# --- PATH CONFIGURATION ---
SCB="/opt/bitcoin-node/lnd/data/chain/bitcoin/mainnet/channel.backup"
LND_CONF="/opt/bitcoin-node/lnd/lnd.conf"
BITCOIN_CONF="/opt/bitcoin-node/bitcoin/bitcoin.conf"
DOCKER_CONF="/opt/bitcoin-node/docker-compose.yml"
LNBITS_DB="/opt/lnbits-dedicated/data/database.sqlite3"
DEST_REMOTE="gdrive:Your_Backup_Folder"
TEMP_DB="/tmp/lnbits_snap.sqlite3"
NEXTCLOUD_DB_DUMP="/tmp/nextcloud_db.sql"
RCLONE_CONF="--config /home/user/.config/rclone/rclone.conf"
HASH_DIR="/var/cache/node_backup_hashes"
DATE=$(date +%Y-%m-%d_%H-%M)

# --- TELEGRAM NOTIFICATIONS ---
TG_TOKEN=$(grep TELEGRAM_BOT_TOKEN /path/to/.env | cut -d= -f2)
TG_CHAT_ID=$(grep TELEGRAM_CHAT_ID /path/to/.env | cut -d= -f2)

notify_telegram() {
    local message=$1
    if [ -n "$TG_TOKEN" ] && [ -n "$TG_CHAT_ID" ]; then
        curl -s -X POST "https://api.telegram.org/bot${TG_TOKEN}/sendMessage" \
            -d "chat_id=${TG_CHAT_ID}" \
            -d "text=${message}" \
            --silent > /dev/null
    fi
}

mkdir -p "$HASH_DIR"

# --- GPG ENCRYPTION WITH CHANGE DETECTION ---
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
        notify_telegram "⚠️ BACKUP: GPG encryption failed for $(basename $file)"
    fi
}

echo "--- Backup started ($DATE) ---"

# 1. NODE FILES
echo "[1/3] Node files..."

rclone copyto $SCB $DEST_REMOTE/node/channel.backup $RCLONE_CONF
rclone copyto $BITCOIN_CONF $DEST_REMOTE/node/bitcoin.conf $RCLONE_CONF
rclone copyto $DOCKER_CONF $DEST_REMOTE/node/docker-compose.yml $RCLONE_CONF

encrypt_and_upload $LND_CONF $DEST_REMOTE/node/lnd.conf
encrypt_and_upload /path/to/.env $DEST_REMOTE/node/node.env

# 2. LNBITS DATABASE
echo "[2/3] LNBits database..."
if [ -f "$LNBITS_DB" ]; then
    sqlite3 "$LNBITS_DB" ".backup '$TEMP_DB'"
    if [ $? -eq 0 ] && [ -f "$TEMP_DB" ]; then
        rclone copyto "$TEMP_DB" "$DEST_REMOTE/lnbits/lnbits_database.sqlite3" $RCLONE_CONF
        rclone copyto "$TEMP_DB" "$DEST_REMOTE/history/lnbits_$DATE.sqlite3" $RCLONE_CONF
        rm "$TEMP_DB"
        echo "[OK] LNBits DB saved."
    else
        echo "[ERROR] LNBits snapshot failed."
        notify_telegram "⚠️ BACKUP: LNBits snapshot failed - $DATE"
    fi
fi

# 3. NEXTCLOUD DATABASE
echo "[3/3] Nextcloud database..."
DB_PASS=$(grep NEXTCLOUD_DB_PASSWORD /path/to/.nextcloud_backup.env | cut -d= -f2)
docker exec -e MYSQL_PWD="$DB_PASS" nextcloud-db-1 mariadb-dump -u nextclouduser nextcloud > "$NEXTCLOUD_DB_DUMP"
if [ -s "$NEXTCLOUD_DB_DUMP" ]; then
    rclone copyto "$NEXTCLOUD_DB_DUMP" "$DEST_REMOTE/nextcloud/nextcloud_db.sql" $RCLONE_CONF
    rclone copyto "$NEXTCLOUD_DB_DUMP" "$DEST_REMOTE/history/nextcloud_db_$DATE.sql" $RCLONE_CONF
    rm "$NEXTCLOUD_DB_DUMP"
    echo "[OK] Nextcloud DB saved."
else
    echo "[ERROR] Nextcloud DB dump failed."
    notify_telegram "⚠️ BACKUP: Nextcloud DB dump failed - $DATE"
    rm -f "$NEXTCLOUD_DB_DUMP"
fi

# CLEANUP
echo "Cleaning history > 30 days..."
rclone delete --min-age 30d $DEST_REMOTE/history/ $RCLONE_CONF

# SYNC → MEGA
MEGA_TEMP="/tmp/node_mega_temp"
mkdir -p "$MEGA_TEMP"
echo "Syncing to MEGA..."
rclone sync $DEST_REMOTE "$MEGA_TEMP" $RCLONE_CONF --exclude "history/**"
if [ $? -eq 0 ]; then
    mega-put -c "$MEGA_TEMP" /Node_Backups
    if [ $? -eq 0 ]; then
        echo "[OK] MEGA sync completed."
    else
        echo "[ERROR] MEGA upload failed."
        notify_telegram "⚠️ BACKUP: MEGA upload failed - $DATE"
    fi
    rm -rf "$MEGA_TEMP"
else
    echo "[ERROR] GDrive download for MEGA failed."
    notify_telegram "⚠️ BACKUP: GDrive download for MEGA failed - $DATE"
    rm -rf "$MEGA_TEMP"
fi

# SYNC → BACKBLAZE B2
echo "Syncing to Backblaze B2..."
rclone sync $DEST_REMOTE b2:your-bucket-name $RCLONE_CONF --exclude "history/**"
if [ $? -eq 0 ]; then
    echo "[OK] Backblaze sync completed."
else
    echo "[ERROR] Backblaze sync failed."
    notify_telegram "⚠️ BACKUP: Backblaze sync failed - $DATE"
fi

echo "--- Backup completed ($DATE) ---"
```

---

## Cron setup

```bash
sudo crontab -e
```

Add:

```
0 * * * * /bin/bash /path/to/backup.sh >> /var/log/node_backup.log 2>&1
```

Runs every hour. Log file lets you verify backups without checking manually.

---

## Why three destinations

| Destination | Provider | Privacy | Free tier | Notes |
|-------------|----------|---------|-----------|-------|
| Google Drive | Google (US) | Low | 15GB | Primary, fastest sync |
| MEGA | MEGA (NZ) | High | 20GB | E2E encrypted |
| Backblaze B2 | Backblaze (US) | Medium | 10GB | Reliable, S3-compatible |

All sensitive files are GPG-encrypted before leaving the server — cloud provider privacy becomes irrelevant for those files.

MEGA and Backblaze exclude `history/**` to save space — only current files are mirrored there.

---

## Verification

Don't assume backups work — verify them regularly.

```bash
# Check channel.backup is being updated
ls -la /path/to/lnd/data/chain/bitcoin/mainnet/channel.backup

# Check GDrive
rclone ls gdrive:Your_Backup_Folder/node/

# Test GPG decryption
rclone copy gdrive:Your_Backup_Folder/node/lnd.conf.gpg /tmp/
gpg --decrypt /tmp/lnd.conf.gpg > /tmp/test.conf
cat /tmp/test.conf
rm /tmp/test.conf /tmp/lnd.conf.gpg
```

If decryption fails, fix it before you need it.

---

## Recovery

```bash
# Restore channel.backup to a new LND instance
lncli restorechanbackup --multi_file /path/to/channel.backup

# Decrypt config files
gpg --decrypt lnd.conf.gpg > lnd.conf
```

After `restorechanbackup`, LND broadcasts force-close transactions for all channels. Funds return to your on-chain wallet after the timelock expires — typically 144 blocks (~24 hours), but can be longer.

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*
