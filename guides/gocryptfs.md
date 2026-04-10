# gocryptfs — File-Level Encryption on Linux

> Full-disk encryption protects you when the machine is off.  
> Once you're logged in, everything is readable — to you, to any process, to anyone with a shell.  
> **gocryptfs fills that gap.**

---

## Why file-level encryption matters

Full-disk encryption (LUKS, VeraCrypt whole-disk) is a good baseline. But it only protects you when the machine is powered off or locked.

Once the system is running and you're authenticated:
- Your entire filesystem is decrypted and accessible
- Any process running under your user can read everything
- A rogue script, compromised application, or remote attacker with a shell sees it all

File-level encryption solves this by keeping sensitive data encrypted **even while the system is running** — you unlock only what you need, only when you need it, and lock it back when done.

---

## What is gocryptfs

**gocryptfs** is a FUSE-mounted, file-level encryption tool for Linux.

- Encrypts at the file level — each file is an individual encrypted object on disk
- Files are only readable when you explicitly mount the vault with your password
- When unmounted, the data is unreadable noise
- No root required to mount
- Works transparently with all existing tools (editors, scripts, file managers)

**Cryptography:**
- `AES-256-GCM` for file contents
- `EME wide-block` for file name encryption
- `scrypt` for password hashing

It was designed as a security-hardened replacement for the older EncFS project, fixing its known vulnerabilities.

---

## Installation

```bash
sudo apt install gocryptfs        # Debian/Ubuntu
sudo pacman -S gocryptfs          # Arch
sudo dnf install gocryptfs        # Fedora
```

---

## Setup

### 1. Create the encrypted vault

```bash
mkdir secret-folder
gocryptfs --init secret-folder
```

You will be asked for a password. After that, gocryptfs prints your **master key**:

```
Your master key is:
    8f68fc74-9e35626d-36923ce7-9c7adb42-
    31d456a8-90e4367e-d4dcfd2e-067c0ced
```

> ⚠️ **Save this master key offline.** If you forget your password, this is the only way to recover your data. Store it in a password manager or on paper in a secure location.

### 2. Mount the vault

```bash
mkdir open-folder
gocryptfs secret-folder/ open-folder/
```

Enter your password. The decrypted contents are now accessible at `open-folder/`.  
Work normally — your editor, scripts, and tools all work as expected.

### 3. Unmount when done

```bash
umount open-folder/
```

Once unmounted, `open-folder/` is empty. `secret-folder/` contains only encrypted noise.

```bash
ls open-folder/
# (empty)

ls secret-folder/
# Fes37GGPQvKs0h4H1AIlBILPWErpmm-Pn9hJuzba4B8  gocryptfs.conf  gocryptfs.diriv
```

---

## Cloud sync (safe)

Because gocryptfs encrypts at the file level, you can sync `secret-folder/` to any cloud provider (rclone, Syncthing, Nextcloud) without risk. The cloud never sees plaintext — only encrypted blobs.

```bash
rclone sync secret-folder/ gdrive:my-encrypted-vault
```

---

## Best practices

| Practice | Why |
|----------|-----|
| Store master key offline | Password loss = permanent data loss |
| Unmount after use | Minimizes exposure window |
| Use a strong password | scrypt is solid but don't rely on it alone |
| Sync only the encrypted folder | Never sync the mount point |
| Combine with full-disk encryption | Defense in depth |

---

## Quick reference

```bash
# Initialize a new vault
gocryptfs --init secret-folder/

# Mount
gocryptfs secret-folder/ open-folder/

# Unmount
umount open-folder/

# Mount with reverse mode (encrypt on-the-fly for backup)
gocryptfs --reverse plain-folder/ encrypted-view/
```

---

## Why this matters for sovereignty

Full-disk encryption is table stakes. But if your session is open — and it usually is — an attacker with access to your running system reads everything.

File-level encryption adds a second layer: **your sensitive data is locked even when you're logged in.** Seed phrases, GPG keys, node configs, macaroons — none of it needs to sit in plaintext on a running system.

Lock what matters. Unmount when done.

---

*Part of [sovereign-linux-tools](https://github.com/shadowbip/sovereign-linux-tools) — practical guides for digital sovereignty.*
