# Arch Linux Base Install Readme

This is my personal installation tutorial for [Arch Linux](https://www.archlinux.org/). It assumes familiarity with the Arch [Beginner's Guide](https://wiki.archlinux.org/index.php/Beginners'_guide) and [Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide).

It uses efi and systemd boot and makes many assumptions (german keyboard & timezone, nvme drive, enable trim on ssd, etc..), so be aware that not all commands may map to your local environment.

## Preparations

- Flash a recent archiso to an usb drive, verify and boot it.

- Load german keyboard layout

```bash
$ loadkeys de-latin1
```

- Get internet access

```bash
# Start DCHP daemon
$ systemctl start dhcpcd.service
# Connect to wifi with iw
$ systemctl start iwd.service
$ iwctl
[iwd] device list
[iwd] station wlan0 get-networks
[iwd] station wlan0 connect "SSID"
[iwd] exit
# ping a server to confirm
$ ping archlinux.org
```

- Set timezone

```bash
$ timedatectl set-ntp true
$ timedatectl set-timezone Europe/Berlin
```

## Disk setup

for an EFI boot

[Arch Wiki for LVM on LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)

```
+----------------+ +-------------------------------------------+
| Boot partition | | Logical volume 1    | Logical volume 2    |
|                | |                     |                     |
| /boot          | | [SWAP] 8 GB         | /                   |
|                | |                     |                     |
|                | | /dev/CryptLVM/swap  | /dev/CryptLVM/root  |
| (may be on     | |_ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _|
| other device)  | |                                           |
|                | |          LUKS2 encrypted partition        |
| /dev/nvme0n1p1 | |            /dev/nvme0n1p2                 |
+----------------+ +-------------------------------------------+
```

- [Create partions with fdisk](https://wiki.archlinux.org/index.php/Fdisk):

```bash
$ fdisk /dev/nvme0n1

# Create /boot partition
Command (m for help): g # Creates a gpt layout & deletes the disk!!
Command (m for help): n
Partion number: 1
First sector: (default)
Last sector: +512M

Command (m for help): t
Partion type: 1

# Create / (root) partition
Command (m for help): n
Partion number: 2
First sector: (default)
Last sector: (default)

# Check partiion layout and write to disk
Command (m for help): p
Command (m for help): w
```

- Encrypt the root partition with `cryptsetup` and open it:

```bash
$ cryptsetup luksFormat /dev/nvme0n1p2
$ cryptsetup open /dev/nvme0n1p2 cryptlvm
```

- Prepare the logical volumes, as [described in the Arch Wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Preparing_the_logical_volumes).

```bash
$ pvcreate /dev/mapper/cryptlvm
$ vgcreate CryptLVM /dev/mapper/cryptlvm
$ lvcreate -L 8G CryptLVM -n swap
$ lvcreate -l 100%FREE CryptLVM -n root

# Format your filesystems on each logical volume:
$ mkfs.ext4 /dev/CryptLVM/root
$ mkswap /dev/CryptLVM/swap

# Mount your filesystems:
$ mount /dev/CryptLVM/root /mnt
$ swapon /dev/CryptLVM/swap
```

- [Preparing the boot partition](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Preparing_the_boot_partition_2)

```bash
$ mkfs.fat -F32 /dev/nvme0n1p1
$ mkdir /mnt/boot
$ mount /dev/nvme0n1p1 /mnt/boot
```

## Install Arch Linux

Proceed with the [Arch Linux Installation](https://wiki.archlinux.org/index.php/Installation_guide#Installation).

- Install essential packages

```bash
$ pacstrap /mnt base linux linux-firmware
$ pacstrap /mnt ethtool lvm2 dhclient dhcpcd dnsmasq dnsutils efibootmgr grub intel-ucode iwd man-db man-pages mesa netctl ntp openssh parted sudo terminus-font texinfo tmux usbutils vi vim vulkan-intel vulkan-mesa-layer wpa_supplicant
```

- Fstab

```bash
$ genfstab -U /mnt >> /mnt/etc/fstab
```

- Change root

```bash
$ arch-chroot /mnt
```

- Time zone

Set the time zone:

```bash
$ ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
# Run hwclock(8) to generate /etc/adjtime:
$ hwclock --systohc

```

This command assumes the hardware clock is set to UTC. See System time#Time standard for details.

- Localization

Uncomment `en_US.UTF-8 UTF-8`, `de_DE.UTF-8 UTF-8` and other needed locales in `/etc/locale.gen`, and generate them with:

```bash
$ locale-gen
```

Create the locale.conf(5) file, and set the LANG variable accordingly:

/etc/locale.conf:
`LANG=en_US.UTF-8`

Make the changes persistent and use a bigger font for HiDPI in `/etc/vconsole.conf`:

```
KEYMAP=de-latin1
FONT=ter-124n
```

- Network configuration

Create the hostname file:

/etc/hostname:
`maschine`


- [Configuring mkinitcpio](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio_2)

For systemd-based initramfs, change `/etc/mkinitcpio.conf` to the following:

```
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 filesystems fsck)
```

With intel graphics, also put:
```
MODULES=(intel_agp i915)
```

Regenerate all presets (default & fallback), with:

```bash
$ mkinitcpio -P
```

- Root password

```bash
$ passwd
```

- [Configuring the boot loader](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_the_boot_loader_2)

We are using GRUB.

```bash
$ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=MaschineGrub
```

In order to unlock the encrypted root partition at boot, the following kernel parameter needs to be set by the boot loader:

/etc/default/grub:
```
GRUB_CMDLINE_LINUX="rd.luks.name=device-UUID=cryptlvm root=/dev/CryptLVM/root rd.luks.options=discard"
```

The device-UUID refers to the UUID of /dev/nvme0n1p2. Grab it via `ls -l /dev/disk/by-uuid`.
You can use a `tmux` session for split terminals.

Regenerate the grub config:

```bash
$ grub-mkconfig -o /boot/grub/grub.cfg
```

Intel microcode updates are configured automatically, because we installed the `intel-ucode` beforehand.

- Reboot!! :)

```bash
$ exit
$ umount -R /mnt
$ reboot
```

Remember to unplug the archiso usb drive.


## Post installation steps

- First thing after reboot is to get internet back:

```bash
$ systemctl enable dhcpcd.service
$ systemctl start dhcpcd.service
$ systemclt start iwd.service
$ iwctl
[iwd] device list
[iwd] station wlan0 get-networks
[iwd] station wlan0 connect "SSID"
[iwd] exit
# ping a server to confirm
$ ping archlinux.org
```

- Update the whole system `pacman -Syu`

- Install Ansible and git `pacman -Sy ansible git`

- Run the playbook :)

## SSH

### Create SSH keys

Install openssh. Then create a new key for each service with:

```
ssh-keygen -t rsa -b 4096 -C "EMAIL@ADDRESS"
```

Optionally add the keys to your KeePassXC database.

### SSH Agent

Add a [systemd user unit](https://wiki.archlinux.org/index.php/Systemd/User) to run the [ssh-agent](https://wiki.archlinux.org/index.php/SSH_keys#SSH_agents). Write the following to `~/.config/systemd/user/ssh-agent.service`.

```bash
[Unit]
Description=SSH key agent

[Service]
Type=simple
Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

[Install]
WantedBy=default.target
```

Enable and run the service with `systemctl --user enable --now ssh-agent`.

Then export the SSH_AUTH_SOCK environment variable in `/etc/profile`:

```
# SSH-Agent
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"
```

After you logout and login again, you can start using the SSH Agent from KeePassXC.
