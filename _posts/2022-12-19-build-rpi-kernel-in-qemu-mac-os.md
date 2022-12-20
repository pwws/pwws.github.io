---
layout: post
title:  "Setup Ubuntu VM for Raspberry Pi kernel builds and development on a Mac with Apple Silicon"
date:   2022-08-28 19:07:27 +0200
categories: jekyll update
---

## Intro

Soon after switching from a Linux laptop to an M1 Mac, I've run into two problems with hacking my Raspberry Pi:

1. Mac has no built-in support for ext4 filesystem, so when you plug in an SD card with RPi Linux, you'll only be able to read the //boot/ partition using Finder.
   There is some project that adds ext4 to Mac OS, but only in read mode and even reading from ext4 partition didn't work for me.
   You can also buy *extFS for Mac by Paragon Software* for about 40 dollars, but why would you do that if you can just spend 40 hours instead and have it for free, huh?

2. You won't simply compile Linux kernel on your Mac.
   If you try to follow instructions for building Linux on Linux, using your Mac, you'll see a myriad of errors.
   I won't go into the details, and just say that after an hour of attempts and reading the Internet, I gave up, as suggested by people on the web.

   You could build kernel on your Raspberry Pi directly, but it'd take much more time than on your Mac.

To work around these problems, I've decided to install a Linux in VM.
I've tried a couple of solutions:

- Parallels VM - works great, but after 14-day trial you have to buy a pretty expensive license.
- Virtualbox - as of November 2022, I was unable to run Linux on M1 Mac.
- multipass - a tool for quickly spawning Ubuntu VMs, based on qemu.
  I didn't manage to access SD card from guest VM.
  It seems to be a fantastic tool, but for purposes different than I had.
- [UTM](https://mac.getutm.app/) - another frontend for qemu.
  Nice GUI, but again I couldn't get to my SD card from the guest.
- [qemu](https://www.qemu.org/) - super powerful, but with steep learning curve.
  Spoiler: it worked for me and solved both problems outlined above.

Below, you'll find my braindump from a long day of experiments, that culminated in building a couple of kernels for my RPi and installing them on SD directly from my Mac.

## Install Ubuntu in qemu

I've [this tutorial](https://adonis0147.github.io/post/qemu-macos-apple-silicon/) ([backup](https://web.archive.org/web/20221219200927/https://adonis0147.github.io/post/qemu-macos-apple-silicon/)).

1. Make a directory where you'll gather Ubuntu/EFI images:

```bash
mkdir ~/qemu-ubuntu
cd ~/qemu-ubuntu
```

2. Download EFI image:

```bash
curl -L https://releases.linaro.org/components/kernel/uefi-linaro/latest/release/qemu64/QEMU_EFI.fd -o QEMU_EFI.fd
```

3. Create disk image where Ubuntu will be installed: 

```bash
qemu-img create -f raw -o preallocation=full ubuntu.img 40G
```
    Mind that we used `preallocation=full`.
    It means, qemu will create a 40 GB (or whatever you specify) file on your disk.
    You can use other preallocation methods, to significantly decrease the disk usage (disk performance will suffer though).

3. Boot qemu with Ubuntu live CD.
I'm using 22.04.1 **ARM64** server version, but any reasonably fresh should do:

```bash
qemu-system-aarch64 \
        -machine virt,accel=hvf \
        -cpu host \
        -smp 8 \
        -m 8G \
        -drive if=virtio,cache=none,format=raw,file=./ubuntu.img \
        -cdrom /Users/\$USER/Downloads/ubuntu-22.04.1-live-server-arm64.iso  \
      -nographic \
      -bios QEMU_EFI.fd
```

4. Install Ubuntu using the install wizard

### Running Ubuntu in qemu

Now, we'll run Ubuntu with SD Card connected to the host Mac OS. 

```bash
diskutil list # find the SD card in the output. In our case it's /dev/disk16*

# /boot partition (/dev/disk16s1) is mounted automatically in Mac OS - we need to unmount it
sudo umount /Volumes/boot

sudo qemu-system-aarch64 \
    -machine virt,accel=hvf \
    -cpu host \
    -smp 8 \
    -m 8G \
    -drive if=virtio,cache=none,format=raw,file=./ubuntu.img \
    -nographic \
    -bios QEMU_EFI.fd \
    -drive file=/dev/disk16,format=raw,if=virtio \ # use dev file appropriate for your system
    -net nic -net user,hostfwd=tcp::2222-:22 # this will forward the SSH port from VM to 2222 on your localhost
```

### Mounting SD card in virtualized Ubuntu in qemu

```bash
lsblk # find the SD card device file; we'll use vdb with vdb1 and vdb2 partitions

sudo mkdir /mnt/fat32
sudo mkdir /mnt/ext4

sudo mount -t vfat /dev/vdb1 /mnt/fat32
sudo mount -t ext4 /dev/vdb2 /mnt/ext4

# now we should be able to see and modify the RPi filesystems in /mnt/{fat32, ext4}
```

### Build kernel

Follow the instructions from the official docs.

Here they are in short form, for RPi 3:

1. Install kernel build dependencies:


```bash
sudo apt install git bc bison flex libssl-dev make
```

2. Clone the kernel source from the official Raspberry Pi repo.
   I recommend using `--depth` to fetch just the last commit and `--branch` to choose the branch from which we want to build.
   It will save you a looooot of time and disk space.

```bash
git clone --depth=1 --branch <branch> https://github.com/raspberrypi/linux
```

3. Apply the default configuration: 

```bash
cd linux
KERNEL=kernel8
make bcm2711_defconfig
```

4. Build the kernel, modules and dtbs:

```bash
make -j12 Image.gz modules dtbs
```

`-j12` means, that make will spawn up to 12 parallel build jobs.
I have came up with 12 using the rule of thumb of using 1.5 * CPU cores.

5. Copy modules, kernel and other files where they belong on the SD card:

```bash
sudo env PATH=$PATH make ARCH=arm64 INSTALL_MOD_PATH=/mnt/ext4 modules_install

sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img # make a backup of the last used kernel image
sudo cp arch/arm64/boot/Image /mnt/fat32/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /mnt/fat32/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* /mnt/fat32/overlays/
sudo cp arch/arm64/boot/dts/overlays/README /mnt/fat32/overlays/

sudo umount /mnt/fat32
sudo umount /mnt/ext4
```

Now you can unplug the SD card and insert it into your Raspberry Pi.
If you have followed this instruction carefully, there's some chance your RPi will boot with the new kernel :)

# TODO

```bash
```

