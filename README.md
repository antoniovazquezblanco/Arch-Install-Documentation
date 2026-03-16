# Arch Install Documentation

My Archlinux install instructions, scripts and config files.

Description of the different uses of my machines:
* Surface: Gnome, disk encryption, secure boot.
* Virtualbox Guest: Use via SSH in a graphical detached state.
* X1 Carbon (3rd Gen): Gnome, disk encryption, secure boot.


## Prepare a boot medium

Download an up to date ISO from <https://archlinux.org/download/>.
You can either use the ISO or prepare a [boot USB](https://wiki.archlinux.org/title/USB_flash_installation_medium).


## Target preparation

<details>
<summary>Surface</summary>

1. Boot Windows and update all the firmwares if there are any pending updates.
2. Power off the surface, press volume up + power, release power to boot into UEFI.
3. Security > Change SecureBoot > Disable SecureBoot.
4. Boot configuration > Enable boot from USB devices.
5. Swipe left on USB storage to boot from USB.

</details>

<details>
<summary>Virtualbox Guest</summary>

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

</details>

<details>
<summary>X1 Carbon (3rd Gen)</summary>

1. Reboot into BIOS by pressing F1.
2. Disable Secure Boot.
3. Reboot into Boot Menu by pressing F12.
4. Boot into Arch install medium.

</details>


## Initial target setup

Common setup would be to setup a network connection and enable ssh.
It is faster to copy commands to an ssh window than typing on the target.

```bash
# Set keyboard locale
loadkeys es

# (Optional) Configure wireless connection...
echo '[General]'                        > /etc/iwd/main.conf
echo 'EnableNetworkConfiguration=true' >> /etc/iwd/main.conf
systemctl restart iwd
iwctl --passphrase $passphrase station wlan0 connect $essid

# Set a password and start ssh
passwd
systemctl start sshd
ip a
```


## Setup

Connect to the target machine via ssh.

```bash
ssh root@archiso
```

Start the setup process.

```bash
# Sync time via NTP
timedatectl set-ntp true

# Update database and keyring
pacman -Sy --noconfirm archlinux-keyring

# (Optional) Add linux-surface repo
curl -s https://raw.githubusercontent.com/linux-surface/linux-surface/master/pkg/keys/surface.asc | sudo pacman-key --add -
sudo pacman-key --lsign-key 56C464BAAC421453
echo ""                                             >> /etc/pacman.conf
echo "[linux-surface]"                              >> /etc/pacman.conf
echo "Server = https://pkg.surfacelinux.com/arch/"  >> /etc/pacman.conf
pacman -Sy --noconfirm

# Select a drive, lsblk command may help you
DRIVE="/dev/sda"
#DRIVE="/dev/nvme0n1"

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

# Select a kernel package
PKGKERNEL="linux linux-headers"
#PKGKERNEL="linux-surface linux-surface-headers"

# Select firmware packaes
PKGFW="intel-ucode linux-firmware-intel"

# System package strapping
pacstrap -K /mnt base ${PKGKERNEL} ${PKGFW} btrfs-progs sudo networkmanager wget usbutils less htop openssh

# Generate fstab
genfstab -L -p /mnt >> /mnt/etc/fstab

# Chroot into the new system
arch-chroot /mnt

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
#HOSTNAME="VirtualArch"
#HOSTNAME="SurfaceArch"
#HOSTNAME="X1Arch"
HOSTNAME="VaultArch"
echo "${HOSTNAME}" > /etc/hostname
echo "127.0.0.1 ${HOSTNAME}.localdomain ${HOSTNAME}" >> /etc/hosts

# Select a kernel name
KENRELNAME="linux"
#KENRELNAME="linux-surface"

# Initramfs
mkinitcpio -p ${KERNELNAME}

# Bootloader
bootctl --path=/boot install --no-variables
mkdir -p /boot/loader
echo "default ArchOS" > /boot/loader/loader.conf
echo "timeout 0" >> /boot/loader/loader.conf
mkdir -p /boot/loader/entries
echo "title Arch Linux" > /boot/loader/entries/arch.conf
echo "linux /vmlinuz-${KERNELNAME}" >> /boot/loader/entries/arch.conf
echo "initrd /intel-ucode.img" >> /boot/loader/entries/arch.conf
echo "initrd /initramfs-${KERNELNAME}.img" >> /boot/loader/entries/arch.conf
echo "options root=LABEL=arch rw rootflags=subvol=/@" >> /boot/loader/entries/arch.conf
echo "title Arch Linux Fallback" > /boot/loader/entries/arch-fallback.conf
echo "linux /vmlinuz-${KERNELNAME}" >> /boot/loader/entries/arch-fallback.conf
echo "initrd /intel-ucode.img" >> /boot/loader/entries/arch.conf
echo "initrd /initramfs-${KERNELNAME}-fallback.img" >> /boot/loader/entries/arch-fallback.conf
echo "options root=LABEL=arch rw rootflags=subvol=/@" >> /boot/loader/entries/arch.conf

# User management
USER="dummy"
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

# Add linux-surface repo
pacman -Sy --noconfirm archlinux-keyring
curl -s https://raw.githubusercontent.com/linux-surface/linux-surface/master/pkg/keys/surface.asc | sudo pacman-key --add -
sudo pacman-key --lsign-key 56C464BAAC421453
echo ""                                             >> /etc/pacman.conf
echo "[linux-surface]"                              >> /etc/pacman.conf
echo "Server = https://pkg.surfacelinux.com/arch/"  >> /etc/pacman.conf
pacman -Sy --noconfirm

# Networking
pacman -S networkmanager
systemctl enable NetworkManager
systemctl enable bluetooth

# Zsh
pacman -S zsh zsh-completions grml-zsh-config
chsh -s /usr/bin/zsh

# Development
pacman -S base-devel vim neovim git 

# Desktop
pacman -S gdm wayland gnome-shell gnome-calculator gnome-control-center gnome-disk-utility gnome-keyring gnome-logs gnome-settings-daemon gnome-system-monitor gnome-tweaks ghostty firefox 
systemctl enable gdm

# Install fonts
pacman -S ttf-dejavu gnu-free-fonts ttf-droid ttf-liberation ttf-roboto ttf-cascadia-code ttf-hack ttf-roboto-mono

# Install the AUR helper
cd /tmp
su $USER -P -c "wget https://aur.archlinux.org/cgit/aur.git/snapshot/pikaur.tar.gz && tar -xf pikaur.tar.gz && rm *.tar.gz && cd pikaur && makepkg -s"
pacman -U /tmp/pikaur/*.zst
cd ~
rm -rf /tmp/pikaur

# (Optional) Surface drivers
pacstrap iptsd
pikaur -S libwacom-surface surface-control

# (Optional) For VMs configure SSH daemon
# Set a fixed address in the host only adapter and mark as never use for routing
sudo nmtui
echo "ListenAddress 192.168.56.102" >> /etc/ssh/sshd_config
systemctl enable sshd

# Exit chroot
exit

# Umount, close and reboot
umount -R /mnt
reboot
```


## Specific instructions

* [Virtualbox Guest](Virtualbox_Guest.md)
* [Microsoft Surface](Surface.md)

