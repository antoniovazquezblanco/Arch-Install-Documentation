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
```
```


## Specific instructions

* [Virtualbox Guest](Virtualbox_Guest.md)
* [Microsoft Surface](Surface.md)

