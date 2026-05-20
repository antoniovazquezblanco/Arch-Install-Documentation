## Host machine

```bash


# Initramfs
hooks_line="HOOKS=(base systemd autodetect modconf block keyboard sd-vconsole filesystems btrfs fsck)"
sed -i "s/^\(HOOKS.*\)/# \1\n${hooks_line}/g" /etc/mkinitcpio.conf
mkinitcpio -p linux

```
