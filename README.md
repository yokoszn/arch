
---
This configuration is designed to maximize security and integrity without sacrificing convenience:

* **Secure Boot** ensures only trusted, signed boot software is executed, preventing unauthorized bootloader or kernel tampering.
* **Unified Kernel Image (UKI)** bundles the kernel, initramfs, and related resources into a single EFI-executable file, simplifying Secure Boot signing and improving boot reliability.
* **LUKS2 Disk Encryption** protects data at rest with strong encryption. LUKS2 is chosen for its support of TPM-based unlocking and modern features.
* **TPM 2.0 Integration** allows the system’s TPM to automatically unlock the encrypted disk when the system’s firmware and boot chain are in a known trusted state. This provides a passwordless boot (with fallbacks) bound to hardware security.
* **Btrfs with Subvolumes** offers a modern CoW (Copy-on-Write) filesystem with advanced features like atomic snapshots, compression, and easy partitioning via subvolumes. Subvolumes will be used to separate system and home data, facilitating backups and snapshots.
* **Hardened Kernel** (linux-hardened) applies extra patches and strict settings to mitigate exploits and enhance kernel security, providing stronger defaults than the standard kernel.

## Introduction

This guide provides a comprehensive walkthrough for installing Arch Linux with an advanced security-focused setup. We will configure the system with UEFI Secure Boot, full disk encryption (LUKS2), a Btrfs filesystem with multiple subvolumes, and automatic decryption via TPM 2.0. The boot process will use Unified Kernel Images (UKI) and the **systemd-boot** bootloader. 

Each section of this guide will explain not only the steps to achieve the setup but also the rationale behind choices and the workings of each tool. **Important standards** such as the Discoverable Partitions Specification (DPS) and the Unified Kernel Image format will be explained in context. By the end, you will have a secure Arch Linux installation suitable for sensitive environments or as a robust daily-driver system.

> **Note:** This is a technical guide intended for users already comfortable with Linux basics and Arch Linux installation procedures. It assumes UEFI-capable hardware with TPM 2.0 support. For brevity, general Linux installation steps (like setting locale or creating a user) are mentioned but not deeply explained. Always refer to the official Arch Wiki for the most up-to-date details on specific commands and configuration files.

## Prerequisites and Preparation

Before beginning the installation, ensure you have the following:

* **Arch Linux Installation Media** – A recent Arch Linux ISO booted in **UEFI mode** (check that `/sys/firmware/efi/efivars` exists to confirm UEFI boot).
* **Internet Connection** – Wired or wireless network access from the live environment (for installing packages).
* **UEFI Secure Boot Capable Firmware** – Secure Boot can be toggled off initially, but the hardware must support enabling it later.
* **TPM 2.0 Module** – A TPM 2.0 chip enabled in firmware. This will be used for binding disk unlock to device state.
* **Sufficient Disk Space** – We will format and repartition the disk (all data will be erased). Ensure any important data is backed up.

It’s also recommended to familiarize yourself with UEFI firmware menus (for enabling Secure Boot and enrolling keys) and have a **backup of recovery keys/passwords** for the encrypted disk. In case the TPM auto-unlock fails (e.g., after hardware changes or firmware updates), you will need the LUKS passphrase to unlock the disk.

Throughout this guide, replace device identifiers and UUIDs with those specific to your system (the examples assume a single drive `/dev/sda` or an NVMe drive `/dev/nvme0n1. )

## Disk Partitioning and LUKS2 Encryption

**Goal:** Set up GPT partitions for UEFI, create an encrypted LUKS2 container for the OS, and prepare it for Btrfs.

### 1. Partition the Disk (GPT with EFI System Partition)

First, convert the disk to GPT and create the necessary partitions:

* **EFI System Partition (ESP):** This will hold the UEFI bootloader and the unified kernel images. We recommend 512 MB or 1 GB in size, formatted as FAT32.
* **Root Partition:** Allocate the rest of the disk for the Arch Linux installation. This partition will be encrypted with LUKS2 and later formatted with Btrfs.

Using `parted` (you can also use `fdisk` or `sgdisk`), partition the disk as follows:


**Prepare Disk**  
   Zap the drive:  
```bash   
   $ sgdisk -Z /dev/sda
   ```

2. **Create Partitions**  
   Create a 512MB EFI partition and the root partition:  
   ```
   $ sgdisk -n1:0:+512M -t1:ef00 -c1:EFI -N2 -t2:8304 -c2:LINUXROOT /dev/sda
   ```

```
$ mkfs.vfat -F32 -n EFI /dev/sda1
```

Leave `/dev/sda2` unformatted for now—we will set up encryption on it next.
### This creates:

* Partition 1 (`/dev/sda1`): EFI System Partition, \~512 MiB, and marks it with the “boot” flag (GPT GUID for ESP).
* Partition 2 (`/dev/sda2`): “cryptroot”, occupying the rest of the disk, intended for the encrypted Btrfs root.
* 
2. **Confirm Partitions**  
   ```
   $ partprobe -s /dev/sda
   ```

```
lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS

vda    254:0    0   30G  0 disk 
├─sda1 254:1    0  512M  0 part
└─sda2 254:2    0 469.5G  0 part
```


## Looking good, next, let's encrypt our root partition with LUKS, using `cryptsetup`:


> **Why GPT and a separate EFI partition?** UEFI systems require an EFI System Partition to store bootloader executables. GPT is recommended for UEFI to comply with standards and because it supports partition GUIDs used in the Discoverable Partitions Specification. The separate ESP allows the bootloader and kernel (as a UKI) to remain unencrypted (so firmware can read them), while the OS data is on an encrypted partition.

### 2. Set Up LUKS2 Full Disk Encryption & Root Partition

Initialize LUKS2 on the root partition to encrypt it:

```bash
cryptsetup luksFormat --type luks2 /dev/sda2
```

* When prompted, choose a strong passphrase. This will be your fallback to unlock the disk if TPM auto-unlock fails. LUKS2 is specified (though it’s default) because TPM integration requires LUKS2 metadata support (LUKS1 doesn’t support TPM2 keyslots).

Open the LUKS container and map it to a device (e.g., `cryptroot`):

```bash
cryptsetup luksOpen /dev/sda2 linuxroot
```

After this, `/dev/mapper/linuxroot` will represent the decrypted view of the storage. We’ll build our Btrfs filesystem on this mapped device.

> **About LUKS2 and why it’s used:** LUKS2 provides improved metadata redundancy, flexible key management, and support for hardware security tokens (TPM, smart cards, etc.) through systemd-cryptenroll. In our case, LUKS2 will allow binding a key to TPM 2.0 for automatic unlocking. Earlier LUKS1 volumes must be upgraded to LUKS2 to support TPM unlocking, so we use LUKS2 from the start. The encryption uses AES-256 in XTS mode by default, providing strong security for data at rest.

## Btrfs Filesystem and Subvolume Setup

**Goal:** Format the LUKS container with Btrfs and create subvolumes for a structured filesystem layout.

With the LUKS container open at `/dev/mapper/linuxroot`, create a Btrfs filesystem on it:

1. Format BTRFS root partition:  
   ```
   $ mkfs.btrfs -f -L linuxroot /dev/mapper/linuxroot
   ```
Nice!, now let's mount our partitions, and create our BTRFS subvolumes.

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

*(You can create additional subvolumes as needed. Common ones include `@var` or `@log` to exclude log files from snapshots, and `@snapshots` for snapshot storage. This guide will use just `@` and `@home` for simplicity.)*

> **Why Btrfs with subvolumes?** Btrfs is a modern filesystem offering advanced features: copy-on-write, snapshots, subvolumes, transparent compression, and checksumming for data integrity. By using subvolumes, we can keep **`/`** (the OS) separate from **`/home`** (user data). This makes it easier to take snapshots of the system or reinstall the OS without touching user files. Snapshots can be used for quick backups or even system rollback in conjunction with tools like Snapper or Timeshift. The subvolume layout we use is a simple flat layout where each subvolume is a top-level entity (children of the Btrfs top-level). This is a straightforward setup that works well for general use. If needed, additional subvolumes (e.g., for `/var` or for Docker storage) can be created to isolate those from system snapshots.

## Installing the Base System

With the storage prepared and mounted, we can install the Arch Linux base system.

1. **Select Mirrors (Optional):** Ensure `/etc/pacman.d/mirrorlist` is up to date in the live environment for faster package downloads (you can skip if you have a recent Arch ISO or acceptable speeds).

```
   $ reflector --country AU --age 24 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
```

2. **Install Base Packages:** Use `pacstrap` to install the base system packages into `/mnt`. We will include the base system, the hardened kernel, and other essential packages:

   ```bash
     $ pacstrap -K /mnt base base-devel linux-hardened linux-firmware intel-ucode vim nano cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo
   ```

   * `base` – minimal base packages for Arch.
   * `linux-hardened` – the hardened Linux kernel (with its headers for any module compilation). This replaces the default `linux` kernel.
   * `linux-firmware` – firmware files for hardware support.
   * `btrfs-progs` – Btrfs management tools (for managing subvolumes, snapshots, etc.).
   * `mkinitcpio` – for generating initramfs (included by default with base, but explicitly ensuring it’s present).

   * other utilities like an editor (`vim`) and `sudo` can be included as needed.

   > **Hardened Kernel rationale:** The `linux-hardened` kernel is an Arch-provided alternative kernel that applies a set of security hardening patches to mitigate kernel and userspace exploits. It also enables stricter kernel configuration options by default (for example, disabling unprivileged user namespaces, enabling additional memory protections, etc.) to reduce the attack surface. Using a hardened kernel further secures our system at the cost of very minimal performance impact. It’s an ideal choice for security-conscious setups, complementing Secure Boot and encryption. (If desired, Arch’s regular or LTS kernel could be used instead by installing `linux` or `linux-lts`, but this guide focuses on the hardened variant.)

---
## System Configuration
Now we configure system basics and set up the boot process.

Set your locale, timezone, and hostname as per a standard Arch install:

* **Locale:** Edit `/etc/locale.gen` to uncomment your locale (e.g., `en_US.UTF-8 UTF-8`), then run `locale-gen`. Create `/etc/locale.conf` with `LANG=en_US.UTF-8` (adjust to your locale).
* **Timezone:** Symlink your timezone, for example: `ln -sf /usr/share/zoneinfo/Region/City /etc/localtime` and then run `hwclock --systohc` to set hardware clock.
* **Hostname:** Set your desired hostname in `/etc/hostname`, and map it to `127.0.1.1` in `/etc/hosts` along with `localhost`.

*(These steps are identical to the official Arch installation guide, for convienience you can edit the below commands as needed):

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

## Mkinitcpio Hooks and Initramfs Generation

The initramfs (initial RAM filesystem) is a temporary environment that boots the system before the real filesystem is available. We need to configure `mkinitcpio` to include the appropriate hooks for systemd-based early boot and encryption, and to produce a unified kernel image.

 First, let's create our kernel cmdline file. This doesn't need to contain anything because we're using Discoverable Partitions, we just do it so `mkinitcpio` doesn't complain:
 ```
 $ echo "quiet rw" >/mnt/etc/kernel/cmdline
```

And let's create the EFI folder structure:
```
$ mkdir -p /mnt/efi/EFI/Linux
```

Edit `/etc/mkinitcpio.conf`:

We need to change the HOOKS in `mkinitcpio.conf` to use systemd, so make yours look like:

```
# vim:set ft=sh
 MODULES=()  
 
 BINARIES=()  
 
 FILES=()  
 
 HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)
```

  **Explanation of key hooks:**

  * `systemd`: Enables a systemd-based initramfs (using **systemd-udev** and **systemd-init** instead of BusyBox). This is required for using systemd-cryptsetup to unlock LUKS, especially with TPM integration.
  * `sd-encrypt`: Uses systemd-cryptsetup in the initramfs to unlock encrypted volumes (instead of the older `encrypt` hook). This hook recognizes LUKS2 and integrates with systemd-cryptenroll (TPM) seamlessly.
  * `keyboard` and `sd-vconsole`: Ensure keyboard layout and console are set up, so you can type the decryption passphrase if needed (especially important for non-US keyboards).
  * `block` and `filesystems`: Standard hooks to detect block devices and filesystems.
  * `fsck`: To run filesystem checks (optional, but included here for completeness).

* **Modules:** In most cases, you don’t need to specify modules manually for Btrfs or encryption, as `autodetect` and the hooks will include the necessary drivers. (The `btrfs` module will be added automatically by the `filesystems` hook.)

Save the changes to `/etc/mkinitcpio.conf`.

Next, configure **unified kernel image (UKI)** creation. Arch’s `mkinitcpio` can build a UKI by combining the kernel, initramfs, and other sections into an EFI binary. We’ll set that up:

* Find the preset for your kernel in `/etc/mkinitcpio.d/`. Since we installed `linux-hardened`, open `/etc/mkinitcpio.d/linux-hardened.preset` in an editor.
* Edit the preset to define an **EFI image output**. For example, adjust it like below (actual file may differ slightly):

```ini
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

  In the above:

  * `ALL_kver` points to the kernel image installed by linux-hardened (the vmlinuz).
  * `ALL_microcode` picks up any CPU microcode files in /boot. This ensures Intel/AMD microcode is included in the unified image (make sure to install `intel-ucode` or `amd-ucode` package as needed).
  * We define `default_efi_image` as the path for the unified kernel image we want to create. According to the Discoverable Partitions Specification and systemd-boot conventions, placing the UKI in `EFI/Linux/` on the ESP makes it discoverable without a manual entry. Here we name it `ArchLinux-hardened.efi`. (The directory `/boot/EFI/Linux/` corresponds to `<ESP>/EFI/Linux/` since we have ESP mounted at /boot.)
  * `default_options="--splash ..."` is optional; it adds an Arch logo splash in boot menu. You may omit this or use a different BMP image if desired. It does not affect functionality.

And now, let's generate our UKIs:
```
$ arch-chroot /mnt mkinitcpio -P
```

Once that is complete, confirm they have been created

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


> **Unified Kernel Image (UKI):** A UKI is a single UEFI-executable file containing the kernel, initramfs, and optionally an embedded command line and OS release information. By using UKIs, we simplify Secure Boot (only one file to sign) and make the bootloader configuration trivial (systemd-boot can auto-detect these images). It also ensures all components (kernel, initrd, microcode) are measured together by the TPM for a consistent security state.

## Services and Boot Loader

OK, we're just about done in the archiso, we just need to enable some services, and install our bootloader:

```
$ systemctl --root /mnt enable systemd-resolved systemd-timesyncd NetworkManager 
$ systemctl --root /mnt mask systemd-networkd 
$ arch-chroot /mnt bootctl install --esp-path=/efi
```

OK, let's reboot, and then finish off the installation. Whilst you're rebooting, head into your UEFI/BIOS and put Secure Boot into "Setup Mode", you'll need to check with your PC/Motherboard manufacturer for exact details on how to do that:

```
$ sync
$ systemctl reboot --firmware-setup
```

*(If your UEFI firmware allows it, this step will put the machine into “Setup Mode” and replace the default keys with your newly generated ones. In some cases, you may need to manually confirm or carry out key enrollment via firmware menus.

Confirm Secure Boot is in setup mode / disabled: 
```
$ sbctl status
Installed:  ✓ sbctl is installed
Setup Mode: ✗ Enabled
Secure Boot:    ✗ Disabled
Vendor Keys:    none
```
Looks good. Let's first create and enroll our personal Secure Boot keys:

## Enabling UEFI Secure Boot

Now that the system can boot with the unified kernel image, we will enable UEFI Secure Boot. Secure Boot will ensure that only our signed bootloader and kernel (UKI) can run, preventing unauthorized code at boot time. We’ll use **sbctl** (Secure Boot Control) to manage Secure Boot keys and sign binaries.

**Overview:** We will generate our own Platform Keys (PK, KEK, db) and enroll them, effectively becoming our own certificate authority for Secure Boot on this machine. Then we will sign the UKI and systemd-boot with our keys. Optionally, we could incorporate existing Microsoft keys to dual-boot Windows (sbctl can enroll with `-m` for including Microsoft keys), but this guide focuses on a Linux-only approach.

```
$ sudo sbctl create-keys 
$ sudo sbctl enroll-keys -m
```

We use the `-m` option to enroll the Microsoft vendor key along with our self-created platform key. If you're absolutely certain that none of your hardware has OPROMs signed by Microsoft, you can leave this option out. **WARNING**: Mistakes can potentially brick your system. Although it’s not ideal, it’s generally safer to include the Microsoft vendor key. _You have been warned!_

* We do *not* need to create a file in `/boot/loader/entries/` for the UKI, because systemd-boot will see `ArchLinux-hardened.efi` in `EFI/Linux` and list it in the menu automatically. If you had multiple kernels or wanted custom kernel options for a specific entry, those would warrant a custom entry file, but here the embedded cmdline is sufficient.

At this stage, a basic boot is possible (though not yet with Secure Boot or TPM unlock enabled). The firmware will use systemd-boot, which will in turn find our unified kernel and boot it.

Before moving on, ensure you set a root password (`passwd`) and create any initial user accounts you need (with `useradd` and `passwd`). This is important because once TPM unlocking is enabled, the system might boot straight to the login prompt without asking for a disk password, so you’ll need your user credentials to log in.


### Signing EFI Files with `sbctl`

Next, we’ll use `sbctl` to sign the required `.efi` files. These include the systemd-boot `.efi` file and the UKI files generated by `mkinitcpio`. Using the `-s` option ensures `sbctl` automatically resigns them whenever the kernel or bootloader is updated through `pacman`.

```
$ sudo sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi
$ sudo sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
$ sudo sbctl sign -s /efi/EFI/Linux/arch-linux-hardened.efi
$ sudo sbctl sign -s /efi/EFI/Linux/arch-linux-hardened-fallback.efi
```

### Reinstall Kernel to Confirm Automatic Signing

To ensure everything is working as expected, reinstall the kernel. During this process, the UKI should be resigned automatically:

```
$ sudo pacman -S linux-hardened
```

Output confirmation example:
```
(4/4) Signing EFI binaries...
Generating EFI bundles....
✓ Signed /efi/EFI/Linux/arch-linux-fallback.efi
✓ Signed /efi/EFI/Linux/arch-linux.efi
```

### Configure Automatic Unlocking of Root Filesystem

After rebooting, configure the system to automatically unlock the root filesystem by binding a LUKS key to the TPM. This involves generating a new key, adding it to the volume for unlocking, and binding it to PCRs 0 and 7 (corresponding to the system firmware and Secure Boot state).

1. **Generate a Recovery Key**

First, create a recovery key for emergency use in case of future issues. Keep this key secure and hidden.

```
$ sudo systemd-cryptenroll /dev/gpt-auto-root-luks --recovery-key
```

---

2. **Enroll Firmware and Secure Boot State**

This step ensures the TPM can unlock the encrypted drive if the expected system 
firmware and Secure Boot state remain unchanged.  

sbctl also provides a hook to auto-sign these on future updates, which was set up during installation. This means whenever the kernel or bootloader updates via pacman, sbctl’s hook will resign the new versions automatically. This ensures you don’t end up with an unsigned kernel after updates. (The `sbctl status` and `sbctl verify` commands can be used anytime to audit signing status.)

```
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/gpt-auto-root-luks
```

For additional security, you can require a PIN during boot by including the `--tpm2-with-pin=yes` option:

```
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/gpt-auto-root-luks --tpm2-with-pin=yes
```

### 3. Reboot and Test

Reboot the system now. Because Secure Boot is enabled and our UKI is signed, the boot process should proceed as before. When it comes time to unlock the LUKS2 volume:

* **TPM unlock attempt:** The initramfs (systemd-cryptsetup) will see the TPM LUKS keyslot and attempt to use the TPM. If the system’s PCR7 matches the expected value (Secure Boot is in the state when the key was enrolled) and the firmware/boot chain hasn’t deviated, the TPM will unseal the key. The disk will unlock **without prompting for a password**.

* You should observe that the boot continues and eventually you get to a login prompt or display manager without any LUKS passphrase request. This indicates success – the TPM provided the decryption key.

* **Fallback to passphrase:** If the TPM does **not** release the key (e.g., PCR values don’t match), you will be prompted for the LUKS passphrase as usual. This could happen if Secure Boot was disabled, if the boot sequence changed (e.g., you booted from a different medium or changed the bootloader without resigning), or after certain firmware updates. In such cases, entering the correct passphrase will still unlock the disk (the original keyslot is still there), and the system will boot. You can then decide to re-enroll the TPM key if needed.

### 4. TPM PCR and Security Considerations

It’s important to understand the security implications of auto-unlocking with TPM:

* The LUKS decryption key is now sealed in the TPM against certain PCR values. As long as an attacker cannot replicate those PCR values on boot, they cannot get the TPM to release the key. PCR7 being tied to Secure Boot state means that if an attacker tries to boot with Secure Boot off or with their own unsigned/insecure bootloader, PCR7 will differ and TPM won’t unseal. This effectively ties disk unlock to the presence of our Secure Boot trust chain.
* We did not include PCR0 (which measures BIOS/UEFI firmware) in the policy by default. This means a firmware update (which changes PCR0) should not invalidate the TPM secret. However, it also means an attacker with physical access who can, say, downgrade or tamper with firmware (without affecting Secure Boot keys) might still satisfy PCR7. This is a complex scenario, but generally PCR7 alone provides a good balance between security and practicality.
* The **expected behavior at boot** is that any change in the Secure Boot keys or state will prevent auto-unlock. For example, disabling Secure Boot or enrolling new keys might change PCR7 and thus require re-enrollment. Similarly, if you later decide to dual-boot with other keys or switch kernels without properly signing, the auto-unlock might fail until things are consistent again.
* **Cold boot attack consideration:** With auto-unlock, the disk is decrypted automatically on boot. If an attacker steals the device *while it is off*, they could simply turn it on and, if they can get past the OS login, the data is accessible (because the disk unlocked). This is a trade-off for convenience. Mitigations include using strong OS account passwords, maybe full disk hibernation which requires re-unlock, or not using TPM auto-unlock on extremely sensitive systems. TPM unlocking is best suited to protect against *offline* tampering (someone can’t alter your bootloader to steal your key, because Secure Boot + TPM stops that) and to ensure the system hasn’t been tampered with (Measured Boot). It does **not** protect the data if an attacker can simply boot your legitimate system (unlocked) and then extract data. So, treat your user login with similar seriousness as you would an encryption passphrase in terms of access security.

If you ever need to remove the TPM key or change the PCR policy, you can use systemd-cryptenroll to `--wipe-slot=tpm2` for that LUKS device, and then re-enroll with different PCRs. Running `cryptsetup luksDump /dev/sda2` will show a keyslot with type `systemd-tpm2` when a TPM key is enrolled, confirming the setup.

* **Discoverable Partitions Spec (DPS) Note:** Our setup implicitly adheres to aspects of the DPS. We used a standard ESP partition for boot. We did not create a separate `XBOOTLDR` partition because we placed the unified kernel in the ESP itself (which systemd-boot supports). DPS defines a scheme for auto-locating the root partition if no explicit info is given, using partition GUIDs and embedded info in the UKI. In our case, we explicitly specify `root=/dev/mapper/root`, so systemd knows where our root is. If we omitted that, systemd’s **GPT auto-generator** could try to find a partition of type Linux root and use it, but since our root is inside LUKS, it would be more complex. For completeness: DPS assigns a unique GUID for encrypted partitions and for Linux root partitions. If one were to follow DPS fully, the LUKS container’s partition type GUID could be set to the Linux LUKS GUID, and the root subvolume’s filesystem UUID could be referenced. However, those details are beyond our scope – we rely on direct configuration.

* **Updates:** In the future, when updating the kernel (`linux-hardened`) via pacman, the `mkinitcpio` preset will automatically generate a new UKI (`ArchLinux-hardened.efi`). Thanks to sbctl’s pacman hook, it will get signed right after installation. The next reboot should just work. The TPM-sealed key will usually continue to work since PCR7 won’t change if you’re using the same Secure Boot keys to sign the new kernel. If you do something like change Secure Boot keys or move to a new machine, you’d have to reenroll the TPM key.)
## Conclusion

You have set up a base Arch Linux system with a robust security posture:

* The system only boots trusted, signed code (Secure Boot with custom keys).
* All data is encrypted at rest with LUKS2, and the encryption key is protected by the TPM tied to your system’s secure state.
* The use of Unified Kernel Images simplifies kernel updates and boot management, as new kernels are automatically bundled and detected without manual configuration.
* Btrfs provides flexibility for managing your data (you can leverage snapshots for safe upgrades or backups).
* The hardened kernel adds an extra layer of defense against exploits out of the box.

Such a setup adheres to modern Linux security standards and is suitable for environments where both security and convenience are desired. It also illustrates the use of current Linux technologies (like systemd-boot, UKIs, and TPM integration) that are increasingly being adopted by various distributions.

Going forward, remember to keep your Secure Boot keys safe (if you ever need to re-enroll them) and maintain backups of your data. If a major change (like firmware update or Secure Boot key change) causes the TPM unlock to stop working, you can always unlock with your passphrase and then reenroll the TPM key to realign with the new PCR state. This ensures that even as your system evolves, you can keep the convenience of auto-unlock without sacrificing security.

---

Consult the [Arch Wiki Hardware Pages](https://wiki.archlinux.org/title/Category:Laptops) for additional configuration tips tailored to your device, such as touchpad gestures, power management, or device-specific quirks.
## Resources:

- [Unified Extensible Firmware Interface/Secure Boot](https://archlinux.org/packages/?name=intel-media-driver)
- [Secure your boot process: UEFI + Secureboot + EFISTUB + Luks2 + lvm + ArchLinux](https://nwildner.com/posts/2020-07-04-secure-your-boot-process/)  
- [How is hibernation supported on machines with UEFI Secure Boot?](https://security.stackexchange.com/questions/29122/how-is-hibernation-supported-on-machines-with-uefi-secure-boot)  
- [Authenticated Boot and Disk Encryption on Linux](https://0pointer.net/blog/authenticated-boot-and-disk-encryption-on-linux.html)
