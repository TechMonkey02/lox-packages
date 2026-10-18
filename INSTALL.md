# Installing LOX

> ⚠️ LOX is early and in testing. A **spare drive** is safest. To share a drive with Windows,
> follow [Dual-boot with Windows](#dual-boot-with-windows) first. Choosing **Erase** in
> `lox-install` deletes everything on the disk, including Windows.

## What you need
- An x86_64 PC booting in **UEFI** mode (any AM5 board is fine)
- A USB stick, 4 GB or larger (its contents get erased)
- An internet connection during install (wired is easiest)
- A target drive of **32 GB or more**, 100 GB+ recommended for games
- NVIDIA: **RTX 20 / GTX 16 series or newer**. Older cards aren't supported by the open driver.

## 1. Get the ISO
Download `lox-<date>-x86_64.iso` and `SHA256SUMS` from the newest release at
<https://github.com/TechMonkey02/lox-packages/releases>, and check the download:
`sha256sum -c SHA256SUMS` on Linux, or `Get-FileHash lox-*.iso` in PowerShell (compare the hash
by eye).

## 2. Write it to a USB stick
On Windows, either:
- **Rufus**: pick the ISO. When asked, choose **DD image mode** (not ISO mode).
- **Ventoy**: install Ventoy to the stick once, then just copy ISOs onto it.

On Linux: `sudo dd if=lox-*.iso of=/dev/sdX bs=4M status=progress oflag=sync`
(double-check `/dev/sdX` is the USB stick).

## 3. BIOS / UEFI settings (AM5 boards)
Open the firmware setup (usually **Del** or **F2** at power-on):
- **CSM / Legacy boot: disabled** (UEFI only)
- **Secure Boot: disabled** (LOX doesn't sign its bootloader yet)
- Optional but good for gaming: **Re-Size BAR: enabled** (needs **Above 4G Decoding: enabled**),
  **EXPO** profile for your RAM
- **Update the BIOS first** if it's more than a year old. Older AM5 BIOSes have a Resizable BAR bug
  that leaves NVIDIA cards black in Linux (see [NVIDIA card shows no picture](#nvidia-card-shows-no-picture)).
- Then boot the USB stick (boot menu is usually **F8**, **F11** or **F12**). Pick the **UEFI:** entry.

## 4. Boot the live system
You'll see the LOX boot menu, then a text console with the LO₂ banner.

- **Wired:** network comes up by itself.
- **Wi-Fi:** run `iwctl`, then
  `station wlan0 connect "Your Network"` and type the password. `exit` when done.

Check with `ping -c 3 archlinux.org`.

## 5. Install
```
lox-install
```
1. Pick the disk number.
2. If the disk already has partitions, choose how to install:
   - **Alongside the existing systems**: keeps Windows and everything else, uses the free space
     (see [Dual-boot with Windows](#dual-boot-with-windows))
   - **Erase the whole disk**: everything on it is deleted
3. Type `yes` to confirm.
4. The archinstall menu opens with LOX's settings filled in. Set at least:
   - **Authentication**: create your user (give it sudo/admin rights) and a password
   - **Timezone**
   - Optional: keyboard layout, language

   Leave **Disk configuration** as it is: it's already set up for the choice you made.
5. Choose **Install**. It downloads ~3 GB, so give it 10–30 minutes.
6. When it's done, reboot and remove the USB stick.

NVIDIA cards are detected automatically. The installer prints
"NVIDIA GPU: installing the NVIDIA driver" when it finds one.

## 6. First boot
- GRUB shows **LOX Linux** (boots automatically) and **Advanced options** (LTS kernel fallback).
- Log in on the KDE Plasma login screen. The first login sets up the LOX look (dark theme, LOX
  wallpaper); change anything you like in System Settings afterwards.
- A first snapshot, "Fresh LOX install", is taken automatically in the background. From the next
  boot on, GRUB also shows **LOX Linux snapshots**.

## Updating
Everything from Arch updates normally: `sudo pacman -Syu`, or Discover.

That includes LOX's own package, `lox-base`, which comes from the signed `[lox]` repo
(`https://techmonkey02.github.io/lox-packages`). Check your version with `pacman -Q lox-base`.

**Installed before lox-base 0.4.0?** Older installs don't know the `[lox]` repo or its signing
key yet. Do this once, and from then on `pacman -Syu` keeps lox-base up to date too. First add
the LOX key, and check that the fingerprint `--add` prints is
`B4AC 08CA 0FE9 EC5C 08D5 BB89 3861 8448 89E8 0628`:
```
curl -fLO https://techmonkey02.github.io/lox-packages/lox.gpg
sudo pacman-key --add lox.gpg
sudo pacman-key --lsign-key B4AC08CA0FE9EC5C08D5BB893861844889E80628
```
Then install lox-base 0.4.0, which sets up the repo:
```
sudo pacman -U https://techmonkey02.github.io/lox-packages/x86_64/lox-base-0.4.0-1-any.pkg.tar.zst
sudo systemctl enable lox-gpu-check.service
sudo pacman -Syu
```
Coming from 0.2.0, also check that `/etc/default/grub` has the line
`GRUB_GFXMODE=1280x720,auto`. If not, add it and run `sudo grub-mkconfig -o /boot/grub/grub.cfg`.
New installs get all of this automatically.

## If an update breaks something
Every `pacman` update takes a snapshot before and after.
1. Reboot. In GRUB pick **LOX Linux snapshots**, then a snapshot from before the update.
   It boots as a temporary system (changes there aren't kept).
2. If that works, make it permanent:
   ```
   sudo snapper list          # find the snapshot number
   lox-rollback <number>      # then reboot
   ```
   Your home folder is never rolled back. Only the system is.

Still stuck: boot **Advanced options → LTS kernel**, which is often enough when a new kernel misbehaves.

## NVIDIA card shows no picture
Symptoms: nothing on monitors connected to the NVIDIA card, `nvidia-smi` says it can't talk to
the driver, and LOX shows a "NVIDIA graphics card not working" notification (visible with a
monitor on the motherboard's video port).

Cause: with **Resizable BAR** on, the firmware gives the card a large memory window (8 GB+) but
forgets to tell Linux where it's allowed to put it (missing 64-bit range in the ACPI tables).
Windows doesn't care; Linux refuses the window and the NVIDIA driver can't start. It's a firmware
bug, seen e.g. on an ASUS TUF GAMING B650-PLUS WIFI with BIOS 1811 (2023).

To confirm: `journalctl -b -k | grep 'BAR 1'` shows `can't claim; no compatible bridge window`.

Fixes, best first:
1. **Update the motherboard BIOS** to the newest version from the manufacturer's support page,
   then re-enable Above 4G Decoding and Re-Size BAR if the update reset them. This fixed it on the
   ASUS board above: the RTX 4060 Ti works in LOX with Resizable BAR on.
2. If that doesn't help: BIOS → **Re-Size BAR Support: Disabled**. The card then works in Linux
   with a normal-sized window; the cost is a few percent in some games, in Windows too.

What doesn't work: the kernel options `pci=realloc` and `pci=big_root_window` (no effect), and
`pci=nocrs` (breaks other devices, e.g. Wi-Fi).

## Dual-boot with Windows
LOX can go on the same drive as Windows. It uses free (unallocated) space, adds its boot loader
next to Windows' on the EFI partition, and adds a **Windows** entry to its GRUB menu.
Your Windows partitions aren't touched.

**In Windows, before installing:**
1. **Back up** anything you can't lose. Partitioning is where mistakes are expensive.
2. **BitLocker:** if C: is encrypted, save the recovery key (account.microsoft.com/devices/recoverykey).
   Windows may ask for it once after the boot order changes.
3. **Turn off Fast Startup:** Control Panel → Power Options → *Choose what the power buttons do* →
   *Change settings that are currently unavailable* → untick **Turn on fast startup**.
4. **Make free space:** Disk Management (`diskmgmt.msc`) → right-click C: → **Shrink Volume**.
   Leave at least **32 GB** (100 GB+ for games) as **Unallocated**. Don't create or format a
   partition in it. If you already made one for LOX, right-click it → **Delete Volume** so it
   becomes unallocated.

**Then** run `lox-install`, pick the Windows drive and choose **Alongside the existing systems**.
If that option isn't offered, the installer says why (e.g. not enough unallocated space).

**After installing:**
- GRUB starts by default and has a **Windows** entry. To boot Windows by default instead,
  change the boot order in the firmware setup and use the firmware boot menu (F8/F11/F12) for LOX.
- A big Windows update can put Windows Boot Manager first again. Fix it in the firmware setup.
- **Clock off by some hours in Windows?** Windows keeps the hardware clock in local time, Linux
  in UTC. Easiest fix, in an admin terminal on Windows:
  `reg add "HKLM\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_DWORD /f`
