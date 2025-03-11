Below is a conceptual guide for ensuring your Arch system cannot boot without a specific SD card present—i.e., a “two‐factor” approach that combines:

No kernel or bootloader on the internal disk
A LUKS‐encrypted root on the internal disk
An SD card holding:
The EFI System Partition (with systemd-boot or similar)
The Unified Kernel Image(s) (UKIs) or at least a kernel/initramfs
Optionally, an external keyfile (if you do not use a TPM or want an extra factor).
When the SD card is missing, there is simply no way to boot the kernel off the internal drive.

Important

Your firmware must be set to boot only from the SD card.
If your firmware tries to search the internal disk for a bootloader, you must remove or repurpose the internal EFI partition (so it does not contain a bootloader).
The approach below uses systemd-style “direct” boot with UKIs, but the idea is the same if you prefer GRUB or another boot manager.
1. Partition Layout
We assume you already have an internal drive /dev/nvme0n1 containing a LUKS‐encrypted btrfs root and no EFI partition on it (or at least no valid EFI bootloader).

Your SD card (say /dev/mmcblk0 or /dev/sdc) will have:

A 512 MB (or larger) EFI System Partition (FAT32, type ef00).
Optionally, space for any other partitions on the SD card, or just the one partition if that is all you need.
Make it look like this (adjust device name as needed):

bash
Copy
Edit
sgdisk -Z /dev/mmcblk0
sgdisk -n1:0:+512M -t1:ef00 -c1:EFISD /dev/mmcblk0
mkfs.vfat -F32 -n EFISD /dev/mmcblk0p1
2. Place /boot or UKIs on the SD Card
Since you want to guarantee no boot without the SD card, you must not have your bootloader or kernel on the internal disk. You have two main approaches:

Approach A: Place a Unified Kernel Image (UKI) directly on the SD card
Mount the SD card’s EFI partition somewhere (e.g. /mnt/efisd).

bash
Copy
Edit
mkdir /mnt/efisd
mount /dev/mmcblk0p1 /mnt/efisd
mkdir -p /mnt/efisd/EFI/Linux
Tell mkinitcpio (or dracut, etc.) to generate a unified kernel image directly into /mnt/efisd/EFI/Linux/.

For example, using systemd‐style UKI generation in /etc/mkinitcpio.d/linux-hardened.preset:

ini
Copy
Edit
# mkinitcpio preset file for 'linux-hardened'
ALL_kver="/boot/vmlinuz-linux-hardened"        # kernel to unify
PRESETS=('default')

# Tells mkinitcpio to create a UKI, not a separate initramfs
default_uki="/efi/EFI/Linux/arch-linux-hardened.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
Then run:

bash
Copy
Edit
mount /dev/mmcblk0p1 /mnt/efisd       # the EFI partition on the SD
arch-chroot /mnt mkinitcpio -P       # or do it from inside the installed system
Adjust paths so that the resulting .efi is physically on the SD card.

Install systemd-boot to the SD card**:

bash
Copy
Edit
bootctl install --esp-path=/mnt/efisd
or if done inside a chroot, specify --esp-path.

Sign the UKI if you are using Secure Boot with your own keys, e.g. with sbctl:

bash
Copy
Edit
sbctl sign -s /mnt/efisd/EFI/Linux/arch-linux-hardened.efi
Remove or erase any EFI partition on the internal drive (or at least remove \EFI\Boot\bootx64.efi or any other .efi file) to prevent fallback boot from the internal disk.

Result: The only way for your motherboard to load the EFI loader & kernel is from the SD card’s EFI partition.

Approach B: Keep a normal /boot on the SD card
If you prefer a separate kernel + initramfs (instead of a single UKI), you still do the same concept:

Format and mount the SD card as /boot.
Install your bootloader (systemd-boot or GRUB) into that /boot/EFI on the SD card.
Wipe or do not create an EFI partition on the internal disk.
No matter which approach you pick, the principle is the same:

No kernel/bootloader stored on the internal drive.
Firmware set to boot only from the SD card’s EFI partition.
3. Require the SD Card Also For the LUKS Key (Optional)
Even if the firmware loads your kernel from the SD card, you might also want the LUKS volume itself to require a keyfile on the SD card. This gives you two factors:

Possession of the SD card (for the kernel + keyfile).
Possession of the internal drive (the ciphertext).
If the kernel is always on the SD card anyway, you might simply rely on the fact that if the SD card is missing, there is no kernel to boot. But if you want extra layering, you can embed a keyfile within the initramfs or require it on a mounted partition. For example:

Systemd-cryptenroll with an external keyfile:

bash
Copy
Edit
# Suppose LUKS is on /dev/nvme0n1p2
# 1) Generate a random keyfile
dd if=/dev/urandom of=/tmp/sdcard.key bs=64 count=1

# 2) Add that keyfile as another LUKS passphrase
cryptsetup luksAddKey /dev/nvme0n1p2 /tmp/sdcard.key

# 3) Store that keyfile on the SD card, e.g. /mnt/efisd/lukskey.bin
mv /tmp/sdcard.key /mnt/efisd/lukskey.bin
chmod 600 /mnt/efisd/lukskey.bin

# 4) Adjust kernel command line or systemd-cryptsetup to load that key
#    For example with a systemd-based init, you might specify:
#      rd.luks.key=/dev/disk/by-partlabel=EFISD:/lukskey.bin
#
#    Or embed the key in the UKI itself (via FILES=... in mkinitcpio.conf).
TPM + SD card: If you do have a TPM but still want an SD presence requirement, you can do something like:

Seal the LUKS master key in the TPM with systemd-cryptenroll --tpm2.
Add an additional “key slot” that is stored on the SD card.
Then the system is configured to attempt TPM unlock first but, for instance, also require a second factor from the SD card.
Exactly how you combine a TPM with an external factor depends on your comfort with sealing/unsealing keys, your PCR policy, etc. The simplest path is to rely on the fact that if the SD card is absent, there is no kernel to run, thus no attempt to unlock the LUKS at all.

4. Lock Down Firmware Boot Order
Finally, you should:

Disable or remove any boot entries in the motherboard firmware that point to the internal disk.
Ensure the only valid EFI boot path is the SD card’s EFI partition.
Optionally enable UEFI Secure Boot with your own keys (sbctl or otherwise), signing the .efi on the SD card so that only that trusted image can run.
If your firmware sees no bootloader on the internal disk, but sees a valid /EFI/Boot/bootx64.efi (or systemd-boot) on the SD card, that is the only path. No SD card = no boot.

Putting It All Together
Create no EFI partition on the internal drive (or wipe it so it’s empty).
Create an EFI partition on the SD card, format FAT32.
Install systemd-boot (or GRUB) to that SD card’s EFI partition.
Generate your UKI (or separate kernel + initramfs) to the SD card under /EFI/Linux/....
Sign that UKI if using Secure Boot.
Lock firmware so that the only boot entry is the SD card.
Optionally, require a keyfile from the SD card or use systemd-cryptenroll with a TPM and an external factor.
After these steps, your Arch system:

Will not boot if the SD card is removed (no kernel/bootloader available).
Retains full LUKS encryption on the internal btrfs root.
Optionally uses a TPM or external keyfile on the SD card to enforce additional constraints.
