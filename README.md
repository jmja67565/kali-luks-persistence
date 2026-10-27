# 🐉 Kali Live USB Encrypted Persistence (LUKS)

<p align="center">
🌐 <b>Languages:</b>
<a href="README.md">🇬🇧 English</a> ·
<a href="README.zh-CN.md">🇨🇳 简体中文</a> ·
<a href="README.zh-TW.md">🇹🇼 繁體中文</a> ·
<a href="README.ja.md">🇯🇵 日本語</a> ·
<a href="README.ko.md">🇰🇷 한국어</a> ·
<a href="README.de.md">🇩🇪 Deutsch</a> ·
<a href="README.fr.md">🇫🇷 Français</a> ·
<a href="README.es.md">🇪🇸 Español</a> ·
<a href="README.ru.md">🇷🇺 Русский</a> ·
<a href="README.pt-BR.md">🇧🇷 Português</a>
</p>

> Turn a Kali Live USB into a portable system that keeps your changes across reboots — and stays unreadable to anyone who finds the stick.

**Tested with:** Kali 2026.3 Live ISO · 32 GB+ USB 3.0 · written from Windows
**Difficulty:** ★★☆☆☆ · **Time:** ~20 minutes · **Risk:** every command targets the USB stick — identify the wrong disk and you wipe your drive

---

## 🎯 What you end up with

The USB boot menu gains three entries:

```
Live system (amd64)                        ← one-off, everything lost on shutdown
Live system (amd64) persistence            ← plaintext persistence
Live system (amd64) encrypted persistence  ← the goal: passphrase first, then boot
```

Everything you install, configure and save under `~` survives a reboot. If the stick is stolen, all an attacker sees is a LUKS container they cannot open.

---

## 🗂️ Contents

- 🧠 0. Three concepts first
- 🧰 1. Prerequisites
- 💾 2. Write the ISO (DD mode)
- 🖥️ 3. First boot & identifying the stick
- 🧩 4. Create the third partition
- 🔐 5. Encrypt with LUKS
- 📦 6. ext4 + the persistence label
- 📝 7. Write persistence.conf
- ✅ 8. Verify
- 🔒 9. Unmount & close
- 🚀 10. Boot and verify
- 🛡️ 11. Hardening
- 🩺 12. Troubleshooting
- ⚡ 13. Cheat sheet
- ❓ FAQ
- 🔗 References
- 📄 License

---

## 🧠 0. Three concepts first

Understand these three and every command below becomes obvious instead of something you copy blindly.

**① Kali Live is read-only.**
On every boot the system unpacks a root filesystem from the ISO into RAM. Anything you change is gone at shutdown. That is exactly why persistence exists.

**② Persistence = one partition labelled `persistence` + one file.**
At boot, live-boot scans the disks for a partition with `LABEL=persistence`, then overlays it on top of the read-only root filesystem. Reads come from the ISO, writes land on the stick. The partition only needs a file named `persistence.conf` containing a single line: `/ union`.

**③ Encrypted persistence = that partition wrapped in LUKS.**
At boot the initramfs asks for the passphrase, unlocks the container, maps it to `/dev/mapper/<name>`, and *then* treats it exactly like a normal persistence partition.

The key insight: **live-boot only cares about the label. It does not care whether the label lives on a plain partition or on a LUKS mapping.**

Final layout on the stick:

```
Physical layout
┌──────────────────┬──────────┬────────────────────────────────┐
│ sda1   4.6 GB    │ sda2 4 MB│ sda3   remaining space         │
│ ISO9660 image    │ EFI boot │ LUKS container                 │
│ (read-only)      │          │  └─ /dev/mapper/my_usb         │
│                  │          │       ext4 · LABEL=persistence │
└──────────────────┴──────────┴────────────────────────────────┘
```

---

## 🧰 1. Prerequisites

| Item | Requirement | Why |
|---|---|---|
| USB stick | ≥ 32 GB, USB 3.0+ | persistence writes constantly; cheap sticks stall and drop out |
| Image | `kali-linux-2026.3-live-amd64.iso` | must be **live**, not installer |
| Writer | Rufus (Windows) | use **DD mode** |
| Firmware | Secure Boot disabled | Kali Live is not Microsoft-signed |
| Passphrase | stored in a password manager | lose it and the data is gone for good |

**About downloading:** in the official `https://cdimage.kali.org/current/` directory the Live image is distributed **only as a `.torrent`** (about 443 KiB) — the installer is the one shipped as `.iso`. Using a BitTorrent client is the normal path here, not a workaround for a broken mirror.

**Verify the download** — a corrupted ISO is the sneakiest failure mode:

```powershell
# Windows PowerShell
certutil -hashfile .\kali-linux-2026.3-live-amd64.iso SHA256
```

```bash
# Linux / macOS
sha256sum kali-linux-2026.3-live-amd64.iso
```

Compare character by character with `https://cdimage.kali.org/kali-2026.3/SHA256SUMS`.

---

## 💾 2. Write the ISO (DD mode)

In Rufus: **Device** → your stick · **Boot selection** → the ISO · when the mode dialog appears choose **DD Image mode** · Start.

> ⚠️ **Never touch the "Persistent partition size" slider.**
> Rufus implements persistence by patching casper-based distros (Ubuntu, Mint). Kali uses live-boot + isohybrid, so the slider rewrites the partition layout, GRUB can no longer find the kernel, and you land in a bare `grub>` prompt. Redo the write from scratch.

When it finishes, do **not** open the stick with any Windows partitioning tool (Disk Management, DiskGenius, AOMEI…).

---

## 🖥️ 3. First boot & identifying the stick

Boot from the USB and pick **`Live system (amd64)`** (the plain entry).

> Why not the persistence entry? There is no persistence partition yet, so it would be pointless — and partitions must be modified from a **non-persistent session**, otherwise the overlay is actively using the disk you are editing.

Open a terminal and list the disks:

```bash
lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
```

Identify the stick using three signals together:

- `RM=1` (removable)
- one of its partitions is mounted at `/run/live/medium` or `/lib/live/mount/medium` (that is the running ISO)
- the capacity matches the stick in your hand

Sanity-check the write mode while you are here:

```bash
lsblk -f
```

- ✅ `sda1` is `iso9660` and `sda2` is `vfat` (~4 MB) → **DD mode, correct**
- ❌ a single large `vfat` partition → written in ISO mode, go back to step 2

**Automatic detection** (recommended — one typo here wipes your laptop):

```bash
medium=$(findmnt -no SOURCE /run/live/medium 2>/dev/null || findmnt -no SOURCE /lib/live/mount/medium)
usb=/dev/$(lsblk -no PKNAME "$medium")
echo "USB = $usb"
lsblk "$usb"
```

Output looks like this:

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    1  57.3G  0 disk
├─sda1   8:1    1   4.6G  0 part /run/live/medium
└─sda2   8:2    1     4M  0 part
```

Pin it into a variable — every later command references `$usb`, so a typo cannot hit the wrong disk:

```bash
usb=/dev/sda     # use what you saw above
```

---

## 🧩 4. Create the third partition

Inspect the current table:

```bash
sudo fdisk -l $usb
```

Expect `Disklabel type: dos` and two partitions: roughly 4.x GB (ISO) and a few MB (EFI).

Start the editor:

```bash
sudo fdisk $usb
```

Enter these keys in order:

| Key | Meaning |
|---|---|
| `p` ↵ | print the table, confirm only sda1 + sda2 |
| `n` ↵ | new partition |
| `p` ↵ | primary |
| `3` ↵ | partition number (Enter accepts the default 3) |
| ↵ | first sector: default, right after sda2 |
| ↵ | last sector: default, to the end of the disk |
| `w` ↵ | write and exit |

> ⚠️ Why `fdisk` and not GParted?
> A DD-written hybrid ISO is a "disk inside a disk": the whole device is seen as one huge ISO9660 blob, so GParted reports the entire stick as a single partition and shows no free space. That is not a bug, it is the nature of the image. `fdisk` edits the underlying partition table directly and works fine.

**Now the step where most people get stuck.**

After `w` you will likely see:

```
The kernel still uses the old table.
The new table will be used at the next reboot.
Re-reading the partition table failed.: Device or resource busy
```

The reason is simple: **you booted from this very stick, so `sda1` is still mounted** and the kernel refuses to re-read the table. Then:

```bash
lsblk $usb
```

- **`sda3` is there** → continue with step 5
- **it is missing** → don't fight `partprobe` (it fails with the same "busy"), just **reboot into `Live system (amd64)` again**. The table is already on disk; the kernel picks it up at boot. Re-detect the disk (step 3), confirm `sda3` exists, and continue.

---

## 🔐 5. Encrypt with LUKS

Final safety check — print the device and look at it before pressing Enter:

```bash
[ -b ${usb}3 ] && lsblk -o NAME,SIZE,TYPE ${usb}3
```

Then:

```bash
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3
```

Interaction:

1. `WARNING! This will overwrite data on /dev/sda3 irrevocably.` → type **`YES`** (all caps; `yes` is rejected)
2. `Enter passphrase for /dev/sda3:` → your passphrase
3. `Verify passphrase:` → again

**How to choose the passphrase (this is where people fail):**

- Unlocking happens in the initramfs, where the keyboard layout is **US English**. Characters that move between layouts (`ä`, `@` on a German keyboard, …) will not type what you think they type
- Recommended: **12+ characters, lowercase letters and digits only**, or a long passphrase you can actually recite
- `--verify-passphrase` simply asks twice to protect against typos

> This is the passphrase you type on every boot. **Lose it and the data is unrecoverable.** Keep a copy in your password manager.

---

## 📦 6. ext4 + the persistence label

Unlock the container:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
```

Enter the passphrase. On success `/dev/mapper/my_usb` is the *decrypted* device — **every format and write from now on targets it, never `${usb}3` again**.

```bash
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb
sudo e2label /dev/mapper/my_usb
```

`e2label` must print:

```
persistence
```

> Why must the label be `persistence`? live-boot finds the persistence partition by `LABEL=persistence`. `persist`, `Persistence`, or `persistence ` (trailing space) are all silently ignored.

---

## 📝 7. Write persistence.conf

```bash
sudo mkdir -pv /mnt/my_usb
sudo mount -v /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync
```

`/ union` means: overlay the entire root filesystem `/` onto this partition, so all writes land here.

> ⚠️ Write it with `echo` inside Linux. If you create the file in Windows Notepad it gets CRLF line endings, live-boot parses `/ union\r`, the path does not exist, and persistence silently does nothing. This is the single most common failure.

---

## ✅ 8. Verify

Both checks must pass:

```bash
sudo blkid /dev/mapper/my_usb
```

```
/dev/mapper/my_usb: LABEL="persistence" TYPE="ext4"
```

```bash
xxd /mnt/my_usb/persistence.conf
```

```
00000000: 2f20 756e 696f 6e0a                      / union.
```

The file must end with `0a` (LF). If you see `0d0a`, it is CRLF — delete it and rewrite.

One more, easy to read at a glance:

```bash
stat -c '%s bytes' /mnt/my_usb/persistence.conf   # must be 8
```

---

## 🔒 9. Unmount & close

```bash
sudo umount -v /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

Order matters: **unmount the filesystem first, close the mapping second** — otherwise you get `Device or resource busy`.

---

## 🚀 10. Boot and verify

Pick **`Live system (amd64) encrypted persistence`** in the boot menu.

You should be prompted with something like:

```
Enter passphrase for /dev/sda3:
```

Type the passphrase, reach the desktop, then verify two things.

**① The data really is encrypted** (a stolen stick is useless):

```bash
lsblk -f
```

`sda3` must show `crypto_LUKS`, with the `my_usb` mapping below it carrying `ext4` and the label `persistence`.

**② Persistence really works:**

```bash
echo hello > ~/enc-test
sudo reboot
```

Boot the same entry again, unlock, then:

```bash
cat ~/enc-test     # prints hello → done
```

---

## 🛡️ 11. Hardening

### 11.1 Back up the LUKS header

A damaged header means the entire partition is unreadable, and it lives in the first few megabytes where a single stray write can destroy it.

```bash
sudo cryptsetup luksHeaderBackup ${usb}3 --header-backup-file ~/luks-header-backup.img
```

Store that `.img` on another stick or in cloud storage. **It protects against header corruption, not against a forgotten passphrase.**

### 11.2 Set a nuke password

Install the package that provides the feature first (it is a Debian patch, the command does not exist by default):

```bash
sudo apt install -y cryptsetup-nuke-password
sudo cryptsetup luksAddNuke ${usb}3
```

Set an emergency password. If you are ever forced to unlock the stick, typing this one wipes the LUKS header and nobody can decrypt the data.

### 11.3 Add a backup passphrase

```bash
sudo cryptsetup luksAddKey ${usb}3
```

### 11.4 Inspect the key slots

```bash
sudo cryptsetup luksDump ${usb}3
```

Each slot is listed — main passphrase, backup key, nuke password.

---

## 🩺 12. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No `encrypted persistence` entry in the boot menu | different menu nesting under UEFI, or wrong ISO | look inside the `Live system` submenu; confirm you booted the stick |
| `/dev/sda3` missing after `w` | kernel cannot re-read a busy partition table | reboot into Live (step 4) |
| `partprobe` reports "busy" | the stick's `sda1` is mounted because you booted from it | ignore, reboot |
| Passphrase rejected over and over | initramfs uses a US layout; passphrase contains layout-dependent characters | re-key with plain ASCII letters and digits |
| Changes disappear after reboot | booted plain `Live system`; or wrong label; or CRLF config | redo the two checks in step 8 |
| `blkid` shows no `persistence` label | label wrong, or you formatted outside the mapping | `e2label /dev/mapper/my_usb persistence` |
| `fdisk` warns `GPT PMBR size mismatch` | the stick carries a leftover GPT table (the Kali image itself is a DOS table, so this should not happen) | `sudo sgdisk -e $usb` to relocate the backup GPT, then `fdisk` again |
| `luksFormat` says "Device is in use" | the partition was auto-mounted | `sudo umount ${usb}3`, retry |
| Boot hangs at `cryptsetup` for a while | normal — unlocking in a Live environment is slow | wait 10–30 s |

---

## ⚡ 13. Cheat sheet

Once `${usb}3` exists, the whole sequence from encryption to cleanup is:

```bash
usb=/dev/sda                      # re-detect on every boot

# encrypt
sudo cryptsetup --verbose --verify-passphrase luksFormat ${usb}3

# open, format, label
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo mkfs.ext4 -L persistence /dev/mapper/my_usb

# mount and configure
sudo mkdir -p /mnt/my_usb
sudo mount /dev/mapper/my_usb /mnt/my_usb
echo "/ union" | sudo tee /mnt/my_usb/persistence.conf
sync

# verify
sudo blkid /dev/mapper/my_usb          # LABEL="persistence" TYPE="ext4"
xxd /mnt/my_usb/persistence.conf       # ends with 0a

# clean up
sudo umount /mnt/my_usb
sudo cryptsetup luksClose my_usb
sudo reboot
```

---

## ❓ FAQ

**I don't want encryption — plain persistence only?**
Skip `luksFormat` and `luksOpen` in steps 5–6, run `sudo mkfs.ext4 -L persistence ${usb}3` directly, then continue from step 7. Boot the `persistence` entry.

**A new Kali release is out — do I have to redo everything?**
Yes. Re-writing the ISO with DD resets the partition table and the persistence partition goes with it. Copy your `~` off the stick first.

**The persistence partition is full — can I grow it?**
Extend it to the free space at the end of the stick, then:

```bash
sudo cryptsetup luksOpen ${usb}3 my_usb
sudo cryptsetup resize my_usb
sudo resize2fs /dev/mapper/my_usb
```

**I forgot the LUKS passphrase.**
There is no recovery. No brute-force shortcut, and a header backup does not help (it only guards against corruption). This is why the passphrase belongs in a password manager.

**Can I "install" the system so `apt upgrade` works normally?**
That is a different route: install Kali onto the stick with the installer ISO, treating it as a regular disk. More stable, but every kernel upgrade needs GRUB care and it wears the stick faster. The hybrid Live approach is better for a portable toolkit.

**Will persistence wear out my USB stick?**
It adds writes, but normal use is fine. Avoid no-name sticks, do not fill the stick completely (a full overlay behaves strangely), and check `df -h` occasionally.

**Can I modify the persistence partition while booted with persistence?**
No. The overlay is using it; repartitioning or reformatting from that session will very likely corrupt your data. Always do it from a plain `Live system` session.

---

## 🔗 References

- Kali docs: USB persistence - <https://www.kali.org/docs/usb/usb-persistence/>
- Kali docs: encrypted persistence - <https://www.kali.org/docs/usb/usb-persistence-encryption/>
- Ventoy persistence (the other route: a `.dat` file, no partition editing) - <https://www.ventoy.net/en/plugin_persistence.html>

## 📄 License

Released under the [MIT License](LICENSE). Copyright (c) 2026 jmja67565.

Pull requests and issue reports about pitfalls you hit are welcome.
