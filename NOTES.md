# Computer Redo — Installation Plan
*Reference while installing — keep this repo on phone/Qubes laptop*

---

## Current Drive Inventory

| Device | Size | Type | Model | Current Use |
|--------|------|------|-------|-------------|
| nvme1n1 | 1.9TB | NVMe SSD | TEAM TM8FP6002T | `/` (Btrfs, root + /var/log subvolumes) |
| nvme0n1 | 465.8GB | NVMe SSD | WD Black SN770 500GB | `/ServerVMs` (ext4) |
| sda | 10.9TB | HDD | Seagate ST12000NM0127 | `/home/allen` (Btrfs) |
| sdb | 2.7TB | HDD | Seagate ST3000DM008 | unmounted (Btrfs) |
| sdc | 931.5GB | HDD 5400RPM | WD WD10JPVX (laptop) | `/home`, `/var`, swap, `/boot/efi` |

---

## Target Layout

### nvme1n1 (1.9TB TEAM) — System Drive
Primary NVMe. LUKS (optional) + LVM + Btrfs. All high-I/O mounts live here.

```
nvme1n1
└─ [LUKS — optional]
   └─ LVM VG: vg_system
      ├─ lv_efi      1GB      FAT32           /boot/efi
      ├─ lv_swap    64GB      swap
      ├─ lv_root   150GB      Btrfs
      │    ├─ @root                            /
      │    ├─ @var                             /var
      │    └─ @snapshots                       /.snapshots
      └─ lv_home   ~1.6TB     Btrfs
           ├─ @home                            /home
           └─ @allen                           /home/allen
```

- `/var` as a Btrfs subvolume on lv_root — keeps package installs and logs on fast NVMe
- `/home` and `/home/allen` are separate subvolumes on lv_home (Option A)
- Independent snapshots per subvolume; LVM handles resizing

### nvme0n1 (465.8GB WD Black) — Server VMs
Dedicated NVMe for VM storage — fast random I/O for virtual disks.

```
nvme0n1
└─ LVM VG: vg_vms
   └─ lv_vms   ~465GB    ext4    /ServerVMs
```

### sda (10.9TB Seagate) — Steam Library Only
Own LVM VG, fully isolated. Sequential HDD I/O is fine for game loading.

```
sda
└─ LVM VG: vg_steam
   └─ lv_steam   ~10.9TB    ext4    /home/allen/SteamLibrary
```

- Add as Steam Library Folder via Steam → Settings → Storage
- Steam client/config stays on NVMe (`/home/allen/.local/share/Steam`)
- Per-game choice of which library to install to

### sdb (2.7TB Seagate) — Bulk Data / Media
Desktop HDD. Good for large files that don't need SSD speed.

```
sdb
└─ LVM VG: vg_data
   └─ lv_data   ~2.7TB    Btrfs (zstd)    /home/allen/Data
```

- Downloads, media, documents, email archives, RPG PDFs, etc.
- Btrfs with zstd compression — good ratio for documents/ebooks

### sdc (931.5GB WD laptop 5400RPM) — Archive / Cold Storage
Slowest drive. Suited for infrequently accessed files only.

```
sdc
└─ LVM VG: vg_archive
   └─ lv_archive   ~900GB    Btrfs (zstd)    /home/allen/Archive
```

- Long-term storage: old backups, ISOs, rarely touched files
- Keep swap off this drive — too slow for swap I/O

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
