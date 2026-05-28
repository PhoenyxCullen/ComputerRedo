# Computer Redo — Installation Plan
*Reference while installing — keep this repo on phone/Qubes laptop*

---

## Current Drive Inventory

| Device | Size | Type | Model | Current Use |
|--------|------|------|-------|-------------|
| nvme1n1 | 1.9TB | NVMe SSD | TEAM TM8FP6002T | `/` (Btrfs) |
| nvme0n1 | 465.8GB | NVMe SSD | WD Black SN770 500GB | `/ServerVMs` (ext4) |
| sda | 10.9TB | HDD | Seagate ST12000NM0127 | `/home/allen` (Btrfs) |
| sdb | 2.7TB | HDD | Seagate ST3000DM008 | unmounted |
| sdc | 931.5GB | HDD 5400RPM | WD WD10JPVX (2.5" laptop) | `/home`, `/var`, swap, `/boot/efi` |
| WD Passport | 931.5GB | External USB | WD My Passport | Portable backup — in laptop bag, NOT part of redo |

---

## Two VG Design

```
vg_system  ← nvme1n1 + nvme0n1 + sdb + sdc  (all LUKS'd, ~6TB pool)
vg_steam   ← sda only  (no LUKS, Steam library)
```

All system drives go into one pool. LVM flexibility means a large LV can span
drives, and space can be carved out for VMs and Docker as needed post-install.

---

## Target Layout

### vg_system (~6TB pool)
Four drives, each individually LUKS-encrypted, then joined as LVM PVs.

```
nvme1n1  → LUKS (passphrase at boot) → PV ─┐
nvme0n1  → LUKS (keyfile after boot)  → PV ─┤
sdb      → LUKS (keyfile after boot)  → PV ─┤→ vg_system
sdc      → LUKS (keyfile after boot)  → PV ─┘
```

**LUKS unlock strategy:**
- nvme1n1: unlocked first with passphrase at boot prompt
- nvme0n1, sdb, sdc: unlocked automatically via keyfile stored on nvme1n1 after it's open
- See `backup-plan/launch/backup-notes.md` for GPG keyfile setup (LUKS slot 1)

**Logical Volumes — allocated:**

| LV | Size | FS | Mount | Notes |
|----|------|----|-------|-------|
| lv_efi | 1GB | FAT32 | /boot/efi | Unencrypted partition on nvme1n1 (before LUKS) |
| lv_swap | 64GB | swap | [SWAP] | Matches RAM for full hibernate |
| lv_root | 150GB | Btrfs | / and /var | Subvolumes: @root, @var, @snapshots |
| lv_home | 1.5TB | Btrfs | /home, /home/allen | Subvolumes: @home, @allen |

**Btrfs subvolume detail:**
```
lv_root  → Btrfs
  @root         /
  @var          /var
  @snapshots    /.snapshots

lv_home  → Btrfs
  @home         /home
  @allen        /home/allen
```

**Logical Volumes — reserved (create post-install as needed):**

| LV | Suggested Size | Purpose |
|----|----------------|---------|
| lv_vms | 500GB–1TB | Server VMs / libvirt / QEMU |
| lv_docker | 200GB | Docker storage (`/var/lib/docker`) |
| lv_windows | 100–200GB | Windows VM virtual disk |
| (expansion) | remaining ~3TB | Future use |

Leave these unallocated until needed. `lvcreate` and `mkfs` on demand.

---

### vg_steam (sda 10.9TB — no LUKS)
Isolated VG. Steam game files only.

```
sda  → LVM VG: vg_steam
       └─ lv_steam  ~10.9TB  ext4  /home/allen/SteamLibrary
```

- Add as Steam Library Folder: Steam → Settings → Storage
- Steam client/config stays on NVMe (`~/.local/share/Steam`)
- No LUKS — game files are not sensitive, avoids encryption overhead on large sequential reads

---

### WD Passport (external, NOT in redo)
- 931.5GB, currently in laptop bag
- LUKS2 + LVM + Btrfs already set up (see `backup-plan/`)
- Purpose: portable backup, usable on Qubes laptop
- Plug in and mount when doing backups; otherwise keep disconnected

---

## Partition Notes

- **/boot/efi placement:** FAT32 EFI partition must be on an unencrypted partition on nvme1n1 — create this BEFORE setting up LUKS on the rest of the drive. Single small partition (1GB), rest of drive is one LUKS container → PV.
- **Btrfs compression:** `zstd` on @root, @var, @home, @allen. Omit on @snapshots (already compressed). lv_steam uses ext4 — no compression needed for games.
- **Swap size:** 64GB = RAM size, required for full hibernate (`systemctl hibernate`).
- **LVM thin provisioning:** Not recommended here — stick with thick LVs for simplicity and predictable I/O.
- **LUKS keyfile for secondary drives:** Store keyfile in `/etc/crypttab` workflow — nvme1n1 unlocks at boot, keyfile on its filesystem unlocks the rest automatically. See Arch Wiki: `dm-crypt/System configuration#crypttab`.

---

## Pre-Redo Checklist

Before wiping anything:
- [ ] Plug in WD Passport, run backup script from `~/Projects/backup-plan/`
- [ ] Verify backup is current and readable (`btrfs filesystem show`, spot-check files)
- [ ] Check GPG key expiry (expires 2026-09-29 — extend if within 60 days)
- [ ] Export Vivaldi bookmarks and session
- [ ] `pacman -Qe > ~/pacman-explicit.txt`
- [ ] `yay -Qm > ~/aur-packages.txt`
- [ ] Verify dotfiles in backup: `.zshrc`, `.gitconfig`, `.gnupg/`, `.ssh/`, `.config/`

---

## Post-Install Checklist

**Storage:**
- [ ] Verify all LUKS devices unlock correctly at boot
- [ ] Confirm Btrfs subvolumes mounted at correct paths
- [ ] Mount Steam library, add to Steam as library folder
- [ ] Create lv_vms, lv_docker, lv_windows when ready (leave unallocated until then)

**Desktop:**
- [ ] Log into KDE Plasma Wayland session (not X11)
- [ ] Disable KDE audio applet in system tray; confirm pasystray appears
- [ ] Enable KWin desktop effects (wobbly windows, magic lamp, etc.)
- [ ] Install and configure Polonium (deferred — settle in first)
- [ ] Restore gnome-keyring autostart entry
- [ ] Restore WirePlumber cork-disable config

**Apps & Auth:**
- [ ] Start Waydroid session, install Finelo
- [ ] Restore GPG keys (`gpg --import`)
- [ ] Configure git signing (`user.signingkey`, `commit.gpgsign true`)
- [ ] Load GPG subkeys onto YubiKeys
- [ ] Set up `pass` (password-store)
- [ ] Enable SSH via GPG auth subkey

**Pending projects:**
- [ ] Fix Arch-like Qube on Qubes laptop (separate project)

---

## References

- `~/Projects/backup-plan/launch/backup-notes.md` — backup targets, GPG/YubiKey/LUKS keyfile setup
- `~/Projects/StillOS-bios/` — StillOS USB for old 2010 machine
- `~/Projects/LinuxMint-bios/` — Linux Mint USB for old 2010 machine
- Arch Wiki: [dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
- Arch Wiki: [dm-crypt/System configuration#crypttab](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#crypttab)
