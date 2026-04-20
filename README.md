# BIOS Unlock Guide — Lenovo IdeaPad 330 15-ARR
### For Linux beginners | Ubuntu 24

---

## What is this and why would I do it?

Your laptop's BIOS has hidden settings that Lenovo locks away from the normal menu.
By writing to EFI variables directly, you can unlock things like:

- **Virtualization (SVM)** — needed for VirtualBox, QEMU, Android emulators
- **Performance mode** — forces higher sustained CPU speeds
- **Above 4GB MMIO** — needed for GPU passthrough
- **Fan control** — quiet vs performance thermal profiles
- **Disable CPU cores or SMT** — niche tuning

This guide uses a special GRUB shell loaded from a USB stick to send those writes.
You do **not** need to modify any files on your main drive.

> **Risk level:** Low if you follow this guide exactly. The worst realistic outcome
> from a wrong value is a boot loop, which you can fix by resetting CMOS
> (unplug battery + hold power button 30 seconds). Always read before writing.

---

## Step 1 — Prepare the USB stick (do this on your laptop in Ubuntu)

You need a USB stick of any size. **Its contents will be erased.**

### 1a. Find the USB device name

Plug in the USB stick, then run:

```bash
lsblk
```

You will see output like:
```
NAME   SIZE TYPE MOUNTPOINT
sda    238G disk
sdb      8G disk        ← this is your USB
└─sdb1   8G part /media/...
```

Your USB is probably `sdb`. **Double-check — do not use `sda` (that is your main drive).**

### 1b. Wipe and format as FAT32

Replace `sdb` with your actual USB device name:

```bash
sudo wipefs -a /dev/sdb
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart ESP fat32 1MiB 100%
sudo parted /dev/sdb --script set 1 esp on
sudo mkfs.fat -F32 /dev/sdb1
```

### 1c. Mount the USB

```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb
```

### 1d. Download the GRUB shell with setup_var support

```bash
sudo mkdir -p /mnt/usb/EFI/BOOT
sudo curl -L "https://github.com/datasone/grub-mod-setup_var/releases/download/1.4/modGRUBShell.efi" \
     -o /mnt/usb/EFI/BOOT/BOOTx64.efi
```

Verify the file downloaded correctly (should be around 1–2 MB):

```bash
ls -lh /mnt/usb/EFI/BOOT/BOOTx64.efi
```

### 1e. Unmount the USB

```bash
sudo umount /mnt/usb
```

The USB is ready.

---

## Step 2 — Disable Secure Boot in BIOS

You need to disable Secure Boot once, or the laptop will refuse to boot your USB.

1. Shut down the laptop completely
2. Power on and immediately press **F2** repeatedly until the BIOS menu appears
3. Navigate to **Security** tab using arrow keys
4. Find **Secure Boot** → set to **Disabled**
5. Press **F10** to save and exit

> You can re-enable Secure Boot later if you want. It has no effect on Ubuntu
> once your OS is already installed and working.

---

## Step 3 — Boot from the USB stick

1. Plug in your prepared USB stick
2. Power on and immediately press **F12** repeatedly
3. A boot menu appears — select your USB drive (it may say "EFI USB" or the brand name)
4. You should see a **GRUB shell** — a black screen with a `grub>` prompt

If you see a normal Ubuntu boot instead, go back to BIOS (F2) and check that
the USB is listed in the Boot Order, and that Secure Boot is off.

---

## Step 4 — Read and write BIOS settings

You are now in the GRUB shell. Type carefully — there is no autocorrect.

### The golden rule: always read first

Before changing any value, read what it currently is:

```
setup_var2 0x238
```

You should see output like:
```
UEFI variable value (0-indexed offset 0x238): 0x01
```

`0x01` means the setting is currently enabled. `0x00` means disabled.
If you see an error, try `setup_var3` instead of `setup_var2`.

### How to write a value

```
setup_var2 0x<OFFSET> 0x<VALUE>
```

Then reboot:
```
reboot
```

---

## Step 5 — Common settings to change

All of these use the `Setup` BIOS variable. Use `setup_var2` for each.

---

### Enable Virtualization (AMD SVM)
Needed for: VirtualBox, QEMU, KVM, Android emulators

```
setup_var2 0x238
```
Read the value. If it says `0x00`, it is disabled. Enable it:
```
setup_var2 0x238 0x1
```

---

### Switch to Performance thermal mode
Makes the fan run more aggressively so the CPU runs faster for longer.

```
setup_var2 0x248
```
`0x01` = Performance (louder fan, faster CPU), `0x00` = Quiet mode
```
setup_var2 0x248 0x1
```

---

### Enable Above 4GB MMIO
Needed for: GPU passthrough (VFIO), running a dedicated GPU in a VM

```
setup_var2 0x9E
```
`0x01` = Enabled
```
setup_var2 0x9E 0x1
```

---

### Swap Fn key behavior
`0x01` = Hotkey mode (F1=mute, F2=volume, etc. — press Fn for actual F1–F12)
`0x00` = Legacy mode (F1–F12 work directly, Fn for media keys)

```
setup_var2 0x240
setup_var2 0x240 0x0
```

---

### Disable SVM Lock
Allows the OS (Linux kernel) to toggle virtualization without going through BIOS.
Useful for advanced KVM setups. Only needed if you know what this means.

```
setup_var2 0xEF
setup_var2 0xEF 0x0
```

---

## Step 6 — CPU settings (advanced)

These use the **AmdSetup** variable — a different variable from the one above.
The `setup_var2` command needs to know which variable to write to.

> **Warning:** These settings directly affect CPU operation. Start with reading
> values. Do not guess — bad CPU settings can cause instability or a boot loop
> (fixable with CMOS reset).

### Disable Core Performance Boost (reduce heat/power)

```
setup_var2 0x4
setup_var2 0x4 0x0
```
`0x00` = Boost disabled (runs at base clock only), `0x01` = Auto (default, boost on)

### Enable/Disable SMT (Hyperthreading)

```
setup_var2 0x6E
setup_var2 0x6E 0x0
```
`0x00` = SMT disabled (fewer threads, less heat), `0x01` = Auto (SMT on, default)

After disabling SMT you **must do a full power cycle** (shut down, unplug, battery out
30 seconds, reassemble) — a normal reboot is not enough.

---

## Troubleshooting

### "variable not found" or error on setup_var2
Try `setup_var3` with the same command. Some Insyde BIOS versions need it:
```
setup_var3 0x238
```

### System won't boot after a change
1. Boot the USB again into GRUB shell
2. Read the offset you changed
3. Write the original value back (usually `0x01` or `0x00`)
4. Reboot

### Complete CMOS reset (if system won't boot at all)
1. Shut down, unplug power cable
2. Remove the bottom panel (Phillips screwdriver)
3. Disconnect the CMOS battery (small round coin cell, or disconnect the main battery connector)
4. Hold the power button for 30 seconds
5. Reconnect battery, reassemble, boot
6. BIOS will reset to factory defaults

### "Secure Boot violation" when booting USB
Go to BIOS (F2) → Security → Secure Boot → Disabled, save (F10), try again.

---

## Reference: all confirmed offsets for this laptop

All offsets below are for the `Setup` variable (use `setup_var2`).

| Offset | Setting | 0x00 | 0x01 |
|--------|---------|------|------|
| `0x238` | AMD SVM (Virtualization) | Disabled | **Enabled** (default) |
| `0x9E` | Above 4GB MMIO | Disabled | **Enabled** (default) |
| `0xEF` | SVM Lock | Unlocked | **Locked** (default) |
| `0x14D` | SMM Code Lock | Unlocked | **Locked** (default) |
| `0x240` | Fn Key mode | Legacy Fx | **Hotkey** (default) |
| `0x242` | USB Power Share | Disabled | **Enabled** (default) |
| `0x244` | USB Charge (battery mode) | Disabled | **Enabled** (default) |
| `0x246` | Battery Charging Logo | Disabled | **Enabled** (default) |
| `0x248` | System Performance Mode | Quiet | **Performance** (default) |
| `0x24A` | Fan/Thermal Mode | Quiet | **Performance** (default) |
| `0x23A` | PowerXpress / Switchable GPU | iGPU only | **Discrete+iGPU** (default) |
| `0x23E` | Secure Rollback Prevention | Disabled | **Enabled** (default) |
| `0x21E` | Secure Boot | Disabled | **Enabled** (default) |

AMD CPU settings (use `setup_var2`, AmdSetup variable):

| Offset | Setting | 0x00 | 0x01 |
|--------|---------|------|------|
| `0x4` | Core Performance Boost | Disabled | **Auto/On** (default) |
| `0x6E` | SMT (Hyperthreading) | Disabled | **Auto/On** (default) |

---

## After you're done

- Leave Secure Boot **disabled** or re-enable it — your choice, Ubuntu works either way
- The USB stick can be reused normally (reformat it)
- Your changes survive reboots and Ubuntu updates — they are stored in BIOS NVRAM
- To undo any change, boot the USB again and write the original value

Full offset reference is in `BIOS_OFFSETS.md` in the same folder as this guide.
