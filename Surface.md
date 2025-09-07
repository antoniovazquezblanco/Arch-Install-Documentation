# Archlinux Microsoft Surface setup

Surface devices are not very well supported. It is advisable to be up to date by following the linux-surface project.

- <https://github.com/linux-surface/linux-surface/wiki/Installation-and-Setup>

## Preparation

1. Prepare an Archlinux boot USB (<https://wiki.archlinux.org/title/USB_flash_installation_medium>).
2. Boot Windows and update all the firmwares if there are any pending updates.
3. Power off the surface, press volume up + power, release power to boot into UEFI.
4. Security > Change SecureBoot > Disable SecureBoot.
5. Boot configuration > Enable boot from USB devices
6. Swipe left on USB storage to boot from USB.

## Installation

```bash
# Keyboard locale
loadkeys es

# Configure wireless connection...
echo '[General]'                        > /etc/iwd/main.conf
echo 'EnableNetworkConfiguration=true' >> /etc/iwd/main.conf
systemctl restart iwd
iwctl --passphrase $passphrase station wlan0 connect $essid

# Set a password and start ssh
passwd
systemctl start sshd
ip a
```

```bash
# Connect to the machine
ssh root@192.168.1.xxx

# Sync time via NTP
timedatectl

# Update database and keyring
pacman -Sy --noconfirm archlinux-keyring
curl -s https://raw.githubusercontent.com/linux-surface/linux-surface/master/pkg/keys/surface.asc | sudo pacman-key --add -
sudo pacman-key --lsign-key 56C464BAAC421453

# Add linux-surface repo
echo ""                                             >> /etc/pacman.conf
echo "[linux-surface]"                              >> /etc/pacman.conf
echo "Server = https://pkg.surfacelinux.com/arch/"  >> /etc/pacman.conf
pacman -Sy --noconfirm

# Create partitions
# boot - EFI partition 512MiB
# arch - The remaining space
sgdisk --clear \
       --new=1:0:+512MiB --typecode=1:ef00 --change-name=1:boot \
       --new=2:0:0       --typecode=2:8300 --change-name=2:arch \
       /dev/nvme0n1

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
pacstrap /mnt base base-devel linux-surface linux-surface-headers linux-firmware linux-firmware-marvell intel-ucode iptsd btrfs-progs sudo networkmanager zsh zsh-completions grml-zsh-config vim wget git usbutils htop pipewire-jack gdm wayland gnome-shell gnome-calculator gnome-control-center gnome-disk-utility gnome-keyring gnome-logs gnome-settings-daemon gnome-system-monitor gnome-tweaks ghostty firefox ttf-dejavu gnu-free-fonts ttf-droid ttf-liberation ttf-roboto ttf-cascadia-code ttf-hack ttf-roboto-mono

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
HOSTNAME="SurfaceArch"
echo "${HOSTNAME}" > /etc/hostname
echo "127.0.0.1 ${HOSTNAME}.localdomain ${HOSTNAME}" >> /etc/hosts

# Initramfs
mkinitcpio -p linux-surface

# Bootloader
bootctl --path=/boot install --no-variables
mkdir -p /boot/loader
echo "default ArchOS" > /boot/loader/loader.conf
echo "timeout 0" >> /boot/loader/loader.conf

mkdir -p /boot/loader/entries
echo "title Arch Linux" > /boot/loader/entries/arch.conf
echo "linux /vmlinuz-linux-surface" >> /boot/loader/entries/arch.conf
echo "initrd /intel-ucode.img" >> /boot/loader/entries/arch.conf
echo "initrd /initramfs-linux-surface.img" >> /boot/loader/entries/arch.conf
echo "options root=LABEL=arch rw rootflags=subvol=/@" >> /boot/loader/entries/arch.conf

echo "title Arch Linux Fallback" > /boot/loader/entries/arch-fallback.conf
echo "linux /vmlinuz-linux-surface" >> /boot/loader/entries/arch-fallback.conf
echo "initrd /intel-ucode.img" >> /boot/loader/entries/arch.conf
echo "initrd /initramfs-linux-surface-fallback.img" >> /boot/loader/entries/arch-fallback.conf
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

# Update database and keyring
pacman -Sy --noconfirm archlinux-keyring
curl -s https://raw.githubusercontent.com/linux-surface/linux-surface/master/pkg/keys/surface.asc | sudo pacman-key --add -
sudo pacman-key --lsign-key 56C464BAAC421453

# Add linux-surface repo
echo ""                                             >> /etc/pacman.conf
echo "[linux-surface]"                              >> /etc/pacman.conf
echo "Server = https://pkg.surfacelinux.com/arch/"  >> /etc/pacman.conf
pacman -Sy --noconfirm

# Enable desired services
systemctl enable gdm
systemctl enable NetworkManager

# Install the AUR helper
cd /tmp
su $USER -P -c "wget https://aur.archlinux.org/cgit/aur.git/snapshot/pikaur.tar.gz && tar -xf pikaur.tar.gz && rm *.tar.gz && cd pikaur && makepkg -s"
pacman -U /tmp/pikaur/*.zst
cd ~
rm -rf /tmp/pikaur

# Install aur packages
pikaur -S libwacom-surface surface-control

# Exit chroot
exit

# Umount, close and reboot
umount -R /mnt
reboot
```
