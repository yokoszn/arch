## Disk Preparation
---

1. **Prepare Disk**  
   Zap the drive:  
   ```
   $ sgdisk -Z /dev/nvme0n1
   ```

2. **Create Partitions**  
   Create a 512MB EFI partition and the root partition:  
   ```
   $ sgdisk -n1:0:+512M -t1:ef00 -c1:EFI -N2 -t2:8304 -c2:LINUXROOT /dev/nvme0n1
   ```

3. **Confirm Partitions**  
   ```
   $ partprobe -s /dev/nvme0n1
   ```

```
lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS

vda    254:0    0   30G  0 disk 
├─nvme0n1p1 254:1    0  512M  0 part
└─nvme0n1p2 254:2    0 469.5G  0 part
```

---

### Encrypt the Root Partition

Encrypt `/dev/nvme0n1p2` with LUKS:  
```
$ cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
$ cryptsetup luksOpen /dev/nvme0n1p2 linuxroot
```

---

### Create Filesystems  

1. Format EFI and root partitions:  
   ```
   $ mkfs.vfat -F32 -n EFI /dev/nvme0n1p1
   $ mkfs.btrfs -f -L linuxroot /dev/mapper/linuxroot
   ```

2. Mount Partitions:  
   ```
   $ mount /dev/mapper/linuxroot /mnt
   $ mkdir /mnt/efi
   $ mount /dev/vda1 /mnt/efi
   $ btrfs subvolume create /mnt/home
   $ btrfs subvolume create /mnt/srv
   $ btrfs subvolume create /mnt/var
   $ btrfs subvolume create /mnt/var/log
   $ btrfs subvolume create /mnt/var/cache
   $ btrfs subvolume create /mnt/var/tmp
   ```

---

## Base Install  

1. Update mirrors:  
   ```
   $ reflector --country AU --age 24 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
   ```

2. Install base system:  
   ```
   $ pacstrap -K /mnt base base-devel linux-hardened linux-firmware intel-ucode vim nano cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo
   ```

---

### System Configuration

1. Update locales:  
   ```
   $ sed -i -e "/^#en_US.UTF-8/s/^#//" /mnt/etc/locale.gen
   $ arch-chroot /mnt locale-gen
   ```

2. Configure first boot:  
   ```
   $ systemd-firstboot --root /mnt --prompt
   ```

3. Create user and configure sudo:  
   ```
   $ arch-chroot /mnt useradd -G wheel -m yokoszn
   $ arch-chroot /mnt passwd yokoszn
   $ sed -i -e '/^# %wheel ALL=(ALL:ALL) NOPASSWD: ALL/s/^# //' /mnt/etc/sudoers
   ```

---

## UKIs + Secure Boot

```
$ echo "quiet rw" >/mnt/etc/kernel/cmdline
```

Let's create the EFI folder structure:
```
$ mkdir -p /mnt/efi/EFI/Linux
```

Open `/mnt/etc/mkinitcpio.conf` in nano / vim .

We need to change the HOOKS in `mkinitcpio.conf` to use systemd, so make yours look like:

```
# vim:set ft=sh
 MODULES=()  
 
 BINARIES=()  
 
 FILES=()  
 
 HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)
```

And now let's update the .preset file, to generate a UKI:

Open `/mnt/etc/mkinitcpio.d/linux.preset` in nano / vim.
(or `/mnt/etc/mkinitcpio.d/linux-hardened.preset` if you have been following along )

comment out "default_image", "default_config" "fallback_image" & "fallback_config"

```
# mkinitcpio preset file for the 'linux-hardened' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-hardened"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-hardened.img"
default_uki="/efi/EFI/Linux/arch-linux-hardened.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-hardened-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-linux-hardened-fallback.efi"
fallback_options="-S autodetect"
```

```
$ arch-chroot /mnt mkinitcpio -P
```

```
$ ls -lR /mnt/efi
```
## Example Result:

```
/mnt/efi:
total 4
drwxr-xr-x 3 root root 4096 Aug 25 20:51 EFI

/mnt/efi/EFI:
total 4
drwxr-xr-x 2 root root 4096 Aug 25 21:02 Linux

/mnt/efi/EFI/Linux:
total 118668
-rwxr-xr-x 1 root root 29928960 Aug  9 14:07 arch-linux.efi
-rwxr-xr-x 1 root root 91586048 Aug  9 14:07 arch-linux-fallback.efi
```

## Services and Boot Loader

```
$ systemctl --root /mnt enable systemd-resolved systemd-timesyncd NetworkManager 
$ systemctl --root /mnt mask systemd-networkd 
$ arch-chroot /mnt bootctl install --esp-path=/efi
```

```
$ sync
$ systemctl reboot --firmware-setup
```

Confirm Secure Boot is in setup mode / disabled: 
```
$ sbctl status
Installed:  ✓ sbctl is installed
Setup Mode: ✗ Enabled
Secure Boot:    ✗ Disabled
Vendor Keys:    none
```
Looks good. Let's first create and enroll our personal Secure Boot keys:

```
$ sudo sbctl create-keys 
$ sudo sbctl enroll-keys -m
```

### Signing EFI Files with `sbctl`


```
$ sudo sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi
$ sudo sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
$ sudo sbctl sign -s /efi/EFI/Linux/arch-linux-hardened.efi
$ sudo sbctl sign -s /efi/EFI/Linux/arch-linux-hardened-fallback.efi
```
### Reinstall Kernel to Confirm Automatic Signing

```
$ sudo pacman -S linux-hardened ...
```

Output confirmation example:
```
(4/4) Signing EFI binaries...
Generating EFI bundles....
✓ Signed /efi/EFI/Linux/arch-linux-fallback.efi
✓ Signed /efi/EFI/Linux/arch-linux.efi
```
### Configure Automatic Unlocking of Root Filesystem

1. **Generate a Recovery Key**

```
$ sudo systemd-cryptenroll /dev/gpt-auto-root-luks --recovery-key
```

---

2. **Enroll Firmware and Secure Boot State**

This step ensures the TPM can unlock the encrypted drive if the expected system firmware and Secure Boot state remain unchanged.  

```
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/gpt-auto-root-luks
```

For additional security, you can require a PIN during boot by including the `--tpm2-with-pin=yes` option:

```
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/gpt-auto-root-luks --tpm2-with-pin=yes
```
