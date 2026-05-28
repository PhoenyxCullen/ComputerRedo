# Computer Redo — Installation Plan
*Reference while installing — keep this repo on phone/Qubes laptop*

---

## Current Drive Inventory

| Device | Size | Model | Current Use |
|--------|------|-------|-------------|
| nvme1n1 | 1.9TB | TEAM TM8FP6002T | `/` (Btrfs, root + /var/log subvolumes) |
| nvme0n1 | 465.8GB | WD Black SN770 500GB | `/ServerVMs` (ext4) |
| sda | 10.9TB | Seagate ST12000NM0127 | `/home/allen` (Btrfs) |
| sdb | 2.7TB | Seagate ST3000DM008 | unmounted (Btrfs) |
| sdc | 931.5GB | WD WD10JPVX | `/home`, `/var`, swap, `/boot/efi` |

---

## Target Layout

### nvme1n1 (1.9TB) — System Drive
Wipe and set up with LUKS (optional) + LVM + Btrfs.

```
nvme1n1
└─ [LUKS — optional]
   └─ LVM VG: vg_system
      ├─ lv_swap    ~64GB     swap
      ├─ lv_root   ~150GB     Btrfs
      │    ├─ @root           /
      │    └─ @snapshots      /.snapshots
      ├─ lv_var     ~50GB     ext4 or Btrfs @var
      ├─ lv_home   remaining  Btrfs
      │    ├─ @home           /home
      │    └─ @allen          /home/allen
      └─ lv_efi      1GB      FAT32  /boot/efi
```

- `/home` and `/home/allen` are **separate Btrfs subvolumes on the same LV** (Option A)
- Independent snapshots: can snapshot `@allen` without touching `@home`
- Independent compression policies per subvolume
- LVM handles resizing if lv_home needs to grow

### sda (10.9TB Seagate) — Steam Only
Dedicated drive, own LVM VG, fully isolated from system.

```
sda
└─ LVM VG: vg_steam
   └─ lv_steam   ~10.9TB    ext4   /home/allen/SteamLibrary
```

- Mounted at `/home/allen/SteamLibrary` — added as a Steam Library Folder via Steam → Settings → Storage
- Steam client and config stay on the NVMe (`/home/allen/.local/share/Steam`); game files install to this drive
- Per-game choice of which library to install to
- Own VG means it can be added/removed without touching vg_system
- ext4 recommended — no snapshot benefit for a games drive

### nvme0n1 (465.8GB WD Black) — Server VMs
Keep as dedicated VM storage.

```
nvme0n1
└─ LVM VG: vg_vms  (or keep as single partition)
   └─ lv_vms   465GB    ext4 or Btrfs   /ServerVMs
```

### sdb (2.7TB Seagate) — TBD
Currently unmounted. Options:
- Media/data storage
- Additional Btrfs pool (could be added to vg_system as a second PV)
- Secondary backup target

---

## Desktop Environment

**KDE Plasma (Wayland session)**
- KWin compositor (Wayland) — required for Waydroid
- Icon-Only Task Manager (already configured)
- Polonium — i3-style tiling (installed, configure post-install)
- pasystray — system tray audio control replacing KDE audio applet (installed)
- gnome-keyring autostart — fixes Vivaldi session key issue on KDE

**Audio:**
- WirePlumber cork/ducking disabled via `~/.config/wireplumber/wireplumber.conf.d/50-disable-cork.conf`

**Waydroid:**
- Requires KDE Plasma Wayland session
- Images already installed: `system.img` + `vendor.img` present

---

## Partition Notes

- **LUKS:** Optional — adds full-disk encryption with one passphrase at boot. Negligible performance impact on modern NVMe. Slot 0 = passphrase, Slot 1 = GPG keyfile (see backup-plan project for GPG unlock setup).
- **Btrfs compression:** Use `zstd` on `@root`, `@home`, `@allen`. Skip on `@snapshots` (already compressed data). Skip or use `lzo` on `lv_steam` if using Btrfs there.
- **Swap size:** 64GB matches RAM (64GB DDR5) — needed for full hibernate support.
- **/boot/efi:** 1GB FAT32, on nvme1n1 (system drive). EFI/UEFI boot only.

---

## Pre-Redo Checklist

Before wiping anything:
- [ ] Run backup script from `~/Projects/backup-plan/` — verify WD Passport backup is current
- [ ] Check GPG key expiry (expires 2026-09-29 — extend if within 60 days)
- [ ] Export Vivaldi session/bookmarks
- [ ] Note pacman explicit packages: `pacman -Qe > ~/pacman-explicit.txt`
- [ ] Note AUR packages: `yay -Qm > ~/aur-packages.txt`
- [ ] Verify `/home/allen/` backup includes dotfiles (.zshrc, .gitconfig, .gnupg, .ssh)

---

## Post-Install Checklist

- [ ] Set up KDE Plasma Wayland as default session
- [ ] Install pasystray, disable KDE audio applet in system tray
- [ ] Install Polonium, configure tiling (deferred — do after settling in)
- [ ] Restore gnome-keyring autostart entry
- [ ] Restore WirePlumber cork-disable config
- [ ] Start Waydroid session, install Finelo
- [ ] Restore GPG keys, configure git signing
- [ ] Load GPG subkeys onto YubiKeys
- [ ] Set up pass (password-store)
- [ ] Re-enable SSH via GPG auth subkey

---

## References

- `~/Projects/backup-plan/launch/backup-notes.md` — backup targets, GPG/YubiKey setup, pre-redo backup plan
- `~/Projects/StillOS-bios/` — StillOS USB for the old 2010 machine (separate project)
- `~/Projects/LinuxMint-bios/` — Linux Mint USB for the old 2010 machine (separate project)
