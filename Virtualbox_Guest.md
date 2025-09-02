# Archlinux Virtualbox Guest setup

I commonly use the machine via SSH in a graphical detached state.

## Prepare the machine

Create a VM with the following settings:

```
General
 ├─ Advanced
 │   ├─ Shared clipboard: Bidirectional
 │   └─ Drag and drop: Bidirectional
 ├─ System
 │   ├─ Motherboard
 │   │   ├─ Base memory: As desired...
 │   │   ├─ Floppy: Disable
 │   │   └─ Enable EFI: Enable
 │   └─ Processor
 │       └─ Processors: As desired...
 ├─ Display
 │   └─ Screen
 │       ├─ Video memory: As desired...
 │       ├─ Graphics controller: VMSVGA
 │       └─ Enable 3D accel: Disable
 ├─ Network
 │   ├─ Adapter 1: Will be used for network access
 │   │   └─ Attached to: NAT
 │   └─ Adapter 2: Host only SSH access
 │       └─ Attached to: Host-only Adapter
 ├─ USB
 │   ├─ Enable USB: Enable
 │   └─ Mode: USB 3.0 xHCI
 └─ Shared Folders
     └─ Add as desired with access full and automount.
```

## Target machine

Boot instalation media in UEFI mode. Make minor configuration and bring ssh up...

```bash
loadkeys es
passwd
systemctl start sshd
ip a
```

## Host machine

```bash
ssh root@192.168.56.xxx

# Select a drive, lsblk command may help you
DRIVE="/dev/sda"
HOSTNAME="ArchVM"
USER="dummy"

# Sync time via NTP
timedatectl set-ntp true

# Update database and keyring
pacman -Sy --noconfirm archlinux-keyring

# Create partitions
# boot - EFI partition 512MiB
# arch - The remaining space
sgdisk --clear \
       --new=1:0:+512MiB --typecode=1:ef00 --change-name=1:boot \
       --new=2:0:0       --typecode=2:8300 --change-name=2:arch \
       $DRIVE

# Format partitions
mkfs.fat -F32 -n boot /dev/disk/by-partlabel/boot
mkfs.btrfs --label arch /dev/disk/by-partlabel/arch -f

# Create subvolumes
mount -t btrfs LABEL=arch /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@pkg
btrfs subvolume create /mnt/@.snapshots
umount -R /mnt

# Mount partitions
mount -t btrfs -o subvol=@,defaults,compress=zstd:1,noatime LABEL=arch /mnt
mkdir -p /mnt/{boot,home,var/log,var/cache/pacman/pkg,.snapshots}
mount -t btrfs -o subvol=@home,noatime,compress=zstd:1 LABEL=arch /mnt/home
mount -t btrfs -o subvol=@log,noatime,compress=zstd:1 LABEL=arch /mnt/var/log
mount -t btrfs -o subvol=@pkg,noatime,compress=zstd:1 LABEL=arch /mnt/var/cache/pacman/pkg/
mount -t btrfs -o subvol=@.snapshots,noatime,compress=zstd:1 LABEL=arch /mnt/.snapshots
mount LABEL=boot /mnt/boot

# Install the system
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs sudo networkmanager zsh zsh-completions grml-zsh-config vim wget git usbutils htop openssh

# Generate fstab
genfstab -L -p /mnt >> /mnt/etc/fstab

# Chroot into the new system
arch-chroot /mnt

# Shell change
chsh -s /usr/bin/zsh

# Set timezone
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "es_ES.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "KEYMAP=es" >> /etc/vconsole.conf
echo "FONT=Lat2-Terminus16" >> /etc/vconsole.conf
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "LC_ADDRESS=es_ES.UTF-8" >> /etc/locale.conf
echo "LC_MONETARY=es_ES.UTF-8" >> /etc/locale.conf
echo "LC_MEASUREMENT=es_ES.UTF-8" >> /etc/locale.conf
echo "LC_NUMERIC=es_ES.UTF-8" >> /etc/locale.conf
echo "LC_PAPER=es_ES.UTF-8" >> /etc/locale.conf
echo "LC_TELEPHONE=es_ES.UTF-8" >> /etc/locale.conf
echo "LC_TIME=es_ES.UTF-8" >> /etc/locale.conf


# Set hostname
echo "${HOSTNAME}" > /etc/hostname
echo "127.0.0.1 ${HOSTNAME}.localdomain ${HOSTNAME}" >> /etc/hosts

# Initramfs
hooks_line="HOOKS=(base systemd autodetect modconf block keyboard sd-vconsole filesystems btrfs fsck)"
sed -i "s/^\(HOOKS.*\)/# \1\n${hooks_line}/g" /etc/mkinitcpio.conf
mkinitcpio -p linux

# Bootloader
bootctl --path=/boot install --no-variables
mkdir -p /boot/loader
echo "default ArchOS" > /boot/loader/loader.conf
echo "timeout 0" >> /boot/loader/loader.conf

root_uuid="$(blkid /dev/sda2 -o export | grep -oP '^UUID=\K.+')"
mkdir -p /boot/loader/entries
echo "title Arch Linux" > /boot/loader/entries/arch.conf
echo "linux /vmlinuz-linux" >> /boot/loader/entries/arch.conf
echo "initrd /initramfs-linux.img" >> /boot/loader/entries/arch.conf
echo "options root=LABEL=arch rw rootflags=subvol=/@" >> /boot/loader/entries/arch.conf

echo "title Arch Linux Fallback" > /boot/loader/entries/arch-fallback.conf
echo "linux /vmlinuz-linux" >> /boot/loader/entries/arch-fallback.conf
echo "initrd /initramfs-linux-fallback.img" >> /boot/loader/entries/arch-fallback.conf
echo "options root=LABEL=arch rw rootflags=subvol=/@" >> /boot/loader/entries/arch.conf

# User management
passwd
useradd -m -G users,wheel,audio,video,input -s /usr/bin/zsh $USER
passwd $USER

# Sudoers
cp /etc/sudoers /tmp/sudoers.bak
sed -i '/%wheel ALL=(ALL:ALL) ALL/s/^# //g' /tmp/sudoers.bak
visudo -cf /tmp/sudoers.bak
if [ $? -eq 0 ]; then
  sudo cp /tmp/sudoers.bak /etc/sudoers
else
  echo "Could not modify /etc/sudoers file. Please do this manually."
fi
rm /tmp/sudoers.bak

# Enable desired services
systemctl enable NetworkManager

# Install the AUR helper
cd /tmp
su $USER -P -c "wget https://aur.archlinux.org/cgit/aur.git/snapshot/pikaur.tar.gz && tar -xf pikaur.tar.gz && rm *.tar.gz && cd pikaur && makepkg -s"
pacman -U /tmp/pikaur/*.zst
cd ~
rm -rf /tmp/pikaur

# Exit chroot
exit

# Umount, close and reboot
umount -R /mnt
reboot
```

After reboot in the VM:

```bash
# Configure network
# Set a fixed address in the host only adapter and mark as never use for routing
sudo nmtui

# Configure ssh
echo "ListenAddress 192.168.56.102" >> /etc/ssh/sshd_config

systemctl enable sshd
```