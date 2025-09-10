# Turing RK1 → NVMe Boot (via eMMC staging) — Guide

This repo documents two **clear paths** for getting an RK1 (RK3588) on a Turing Pi 2/2.5 to boot Ubuntu from **NVMe**, while using the onboard **eMMC** only as a minimal bootloader:

- **A) Just Install (recommended):** download prebuilt images and run exact commands.
- **B) Build From Source:** build your own `bootloader-only-rk1.img` (idbloader + U-Boot), then use the same flashing flow.

Why this way? -because i wanted to do everything without physically touching the device itself.
if you have a nvme-usb adapter, just flash the device directly and you can skip the pre-stage-OS thing.

> Assumptions: you run commands from an **Ubuntu** box (22.04/24.04), you have BMC access to your Turing Pi, and your examples target **node 3**. Adapt `--node` if you use a different slot.

---

## Contents

- [A) Just Install](#a-just-install)
  - [A1. Prerequisites](#a1-prerequisites)
  - [A2. Flash a staging OS to eMMC](#a2-flash-a-staging-os-to-emmc)
  - [A3. Prepare the NVMe from the staging OS](#a3-prepare-the-nvme-from-the-staging-os)
  - [A4. Switch eMMC to bootloader-only](#a4-switch-emmc-to-bootloader-only)
  - [A5. Verify NVMe boot](#a5-verify-nvme-boot)
- [B) Build From Source](#b-build-from-source)
  - [B1. Prerequisites](#b1-prerequisites)
  - [B2. Build U‑Boot (RK3588) with Rockchip blobs](#b2-build-u-boot-rk3588-with-rockchip-blobs)
  - [B3. Assemble `bootloader-only-rk1.img`](#b3-assemble-bootloader-only-rk1img)
  - [B4. Flash flow (same as Just Install)](#b4-flash-flow-same-as-just-install)
- [Troubleshooting](#troubleshooting)
- [Notes & Credits](#notes--credits)

---

## A) Just Install

### A1. Prerequisites

**Downloads**
- Ubuntu for RK1 (Joshua Riek builds):  
  https://joshua-riek.github.io/ubuntu-rockchip-download/boards/turing-rk1.html
- **Bootloader-only image** for this repo (contains only idbloader + U‑Boot laid out for eMMC):  
  `images/bootloader-only-rk1.img` (prefer raw `.img` on the BMC; `.xz` may hit RAM limits)

**Ubuntu host packages**
```bash
sudo apt update
sudo apt install -y openssh-client xz-utils
```

**BMC prep** — copy images to the BMC SD card:
```bash
# From your Ubuntu host (replace BMC_IP):
scp ubuntu-24.04-preinstalled-server-arm64-turing-rk1.img \
    root@<BMC_IP>:/mnt/sdcard/Images/
scp images/bootloader-only-rk1.img \
    root@<BMC_IP>:/mnt/sdcard/Images/
```

> 💡 **Why raw .img?** The BMC `tpi flash` can run out of memory when decompressing `.xz` (`"memory limit reached"`). Use raw `.img` for reliability. If you only have `.xz`, uncompress first: `unxz file.img.xz`, or use `xz -b file.img.xz`.

---

### A2. Flash a staging OS to eMMC

On the **BMC**:

```bash
# Example for node 3
tpi power -n 3 off
tpi usb   -n 3 flash

# Flash the full Ubuntu image to eMMC (raw .img recommended)
tpi flash --local \
  --image-path /mnt/sdcard/Images/ubuntu-24.04-preinstalled-server-arm64-turing-rk1.img \
  --node 3 --skip-crc

# Return USB role to normal device mode and power on
tpi usb   -n 3 device
tpi power -n 3 on
```

> Console access: either SSH into the staging OS (via your DHCP lease), or on the BMC:  
> `picocom -b 115200 /dev/ttyS<slot>` (e.g. `/dev/ttyS1` for slot 1, `/dev/ttyS3` for slot 3).  
> If characters drop in `picocom`, try `-emap` or `-f x` (XON/XOFF).

---

### A3. Prepare the NVMe from the staging OS

Do these **on the RK1** (logged into the staging Ubuntu on eMMC):

0) **Download the Ubuntu image inside the staging OS**
If you did not copy the Ubuntu image to the RK1 staging OS, you can download it directly. For example, use `wget` to fetch the image from Joshua Riek's repository:
`wget https://github.com/Joshua-Riek/ubuntu-rockchip/releases/download/v2.4.0/ubuntu-24.04-preinstalled-server-arm64-turing-rk1.img.xz`
After downloading, uncompress the `.xz` file before writing it to NVMe:
`xz -d ubuntu-24.04-preinstalled-server-arm64-turing-rk1.img.xz`


1) **Write the same Ubuntu image to NVMe**
```bash
sudo dd if=ubuntu-24.04-preinstalled-server-arm64-turing-rk1.img \
        of=/dev/nvme0n1 bs=4M status=progress oflag=direct,sync
sync
```

2) **Grow the root partition to full disk** (optional but recommended)
```bash
# Enlarge partition 2 to 100% (use parted or gdisk)
sudo parted /dev/nvme0n1 ---pretend-input-tty <<'EOF'
print
Fix
resizepart 2 100%
quit
EOF

# Enlarge the ext4 filesystem
sudo e2fsck -f /dev/nvme0n1p2
sudo resize2fs /dev/nvme0n1p2
```

3) **Create extlinux.conf on the NVMe root**
```bash
# Mount NVMe root
sudo mkdir -p /mnt/nvmeroot
sudo mount /dev/nvme0n1p2 /mnt/nvmeroot

# Detect kernel version present on NVMe
KVER=$(basename /mnt/nvmeroot/boot/vmlinuz-*-rockchip | sed 's/vmlinuz-//')

# Get PARTUUID of the NVMe root
ROOTPARTUUID=$(blkid -s PARTUUID -o value /dev/nvme0n1p2)

# Write extlinux.conf on the NVMe itself
sudo mkdir -p /mnt/nvmeroot/boot/extlinux
sudo tee /mnt/nvmeroot/boot/extlinux/extlinux.conf >/dev/null <<EOF
default l0
menu title U-Boot menu
prompt 1
timeout 20

label l0
    menu label Ubuntu (NVMe root)
    linux /boot/vmlinuz-$KVER
    initrd /boot/initrd.img-$KVER
    fdtdir /lib/firmware/$KVER/device-tree/
    append root=PARTUUID=$ROOTPARTUUID rootwait rootdelay=5 rw console=ttyS9,115200 console=tty1 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
EOF

sync
sudo umount /mnt/nvmeroot
```

> The kernel, initrd and dtbs referenced above **live on the NVMe root (p2)**. Keeping them on the same partition keeps U‑Boot’s extlinux simple and reliable.

---

### A4. Switch eMMC to bootloader-only

Back on the **BMC**, flash the minimal image that only contains the Rockchip boot stages (no OS):

```bash
tpi power -n 3 off
tpi usb   -n 3 flash

tpi flash --local \
  --image-path /mnt/sdcard/Images/bootloader-only-rk1.img \
  --node 3 --skip-crc

tpi usb   -n 3 device
tpi power -n 3 on
```

---

### A5. Verify NVMe boot

On the running node:

```bash
cat /proc/cmdline      # expect: root=PARTUUID=<nvme0n1p2 UUID>
df -h /                # expect: / on /dev/nvme0n1p2
```

Done!
---

## B) Build From Source

If you prefer to build your own `bootloader-only-rk1.img`:

### B1. Prerequisites

```bash
sudo apt update
sudo apt install -y \
  build-essential gcc-aarch64-linux-gnu bc bison flex swig \
  device-tree-compiler libssl-dev \
  python3 python3-setuptools python3-pip python3-pyelftools \
  git
```

Get the Rockchip blobs:
```bash
mkdir -p ~/work && cd ~/work
git clone https://github.com/rockchip-linux/rkbin.git
# We will use (examples, choose the latest available in rkbin/bin/rk35/):
#   ROCKCHIP_TPL = rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.18.bin
#   BL31        = rk3588_bl31_v1.48.elf
```

Get U‑Boot source (either upstream or a fork with Turing RK1 patches). Upstream generally works for RK3588:
```bash
git clone https://github.com/u-boot/u-boot.git
cd u-boot

# Configure for RK3588 (if you see a turing-rk1-rk3588_defconfig in configs/, use that instead)
make CROSS_COMPILE=aarch64-linux-gnu- rk3588_defconfig

# Build with required Rockchip blobs
make CROSS_COMPILE=aarch64-linux-gnu- \
  ROCKCHIP_TPL=~/work/rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.18.bin \
  BL31=~/work/rkbin/bin/rk35/rk3588_bl31_v1.48.elf \
  -j"$(nproc)"
```

Artifacts:
- `u-boot.itb` (main U‑Boot image)
- `spl/u-boot-spl.bin` (SPL used to create idbloader)
- Optionally `tpl/…` in some trees (not needed if you pass ROCKCHIP_TPL)

If `idbloader.img` was **not** produced automatically, create it:
```bash
./tools/mkimage -n rk3588 -T rksd \
  -d ~/work/rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.18.bin \
  idbloader.img
cat spl/u-boot-spl.bin >> idbloader.img
```

### B2. Build U‑Boot (RK3588) with Rockchip blobs

> Covered above. Ensure the build finishes without “missing external blobs” errors for `ROCKCHIP_TPL` and `BL31`. A missing TEE (OP‑TEE) is fine.

### B3. Assemble `bootloader-only-rk1.img`

Create a small raw image with the correct Rockchip layout for eMMC:

- `idbloader.img` at **sector 64** (offset 32 KiB)
- `u-boot.itb`   at **sector 16384** (offset 8 MiB)
- *(optional)* `trust.img` at sector 24576 (~12 MiB) if your build uses it

```bash
cd ~/work
truncate -s 32M bootloader-only-rk1.img

# idbloader at LBA 64
dd if=/path/to/idbloader.img of=bootloader-only-rk1.img \
   bs=512 seek=64 conv=notrunc status=none

# u-boot.itb at LBA 16384
dd if=/path/to/u-boot.itb of=bootloader-only-rk1.img \
   bs=512 seek=16384 conv=notrunc status=none

# (optional) trust.img at LBA 24576
# dd if=/path/to/trust.img of=bootloader-only-rk1.img \
#    bs=512 seek=24576 conv=notrunc status=none

# Quick sanity check (non-empty bytes at both offsets)
hexdump -C -n 32 -s $((64*512))    bootloader-only-rk1.img | head
hexdump -C -n 32 -s $((16384*512)) bootloader-only-rk1.img | head
```

> You may compress the file (`xz -9`) for storage, but prefer flashing the **raw `.img`** on the BMC to avoid memory limits during decompression.

### B4. Flash flow (same as Just Install)

Follow **A2**, **A3**, **A4** to flash the staging OS, prep NVMe, and then flash the bootloader-only image to the eMMC.

---

## Troubleshooting

- **`tpi flash … "memory limit reached"`**  
  Use a **raw `.img`** instead of `.xz` on the BMC.

- **After flashing loader via rkdeveloptool the USB does not switch Maskrom → Loader**  
  Some loaders are not compatible. Prefer the TuringPi-provided loader when using rkdeveloptool, or avoid this path and use `tpi flash` from the BMC.

- **Boot still comes from eMMC**  
  Ensure you actually flashed the **bootloader-only** image to eMMC, and that NVMe’s `/boot/extlinux/extlinux.conf` uses the **NVMe** PARTUUID.

- **Picocom drops characters**  
  Try `-emap` or `-f x` (XON/XOFF): `picocom -b 115200 -f x /dev/ttyS<slot>`.

---

## Notes & Credits

- Big thanks to the Ubuntu Rockchip builds by **Joshua Riek**.  
- This guide reflects a pragmatic, zero‑touch workflow on a headless Turing Pi 2/2.5: eMMC for bootloader only, NVMe for the OS.
