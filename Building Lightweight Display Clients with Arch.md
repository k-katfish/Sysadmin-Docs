# Building a massively lightweight display client with Arch

Look. I just want a screen to display a webpage. Not that hard.

## 1. Install ArchBTW

The offical name for Arch Linux, for those who don't know, is ArchBTW, the full official name is "IuseArchBTW".

1. Make ISO. If you can't get past this step, go get good at doing linux, then come back here in 30 days and try again.
2. Format your destination disk: `fdisk /dev/sdX`

```
g
n [enter] [enter] +512M [enter]
t [enter] 1 [enter]
n [enter] [enter] [enter] [enter]
w
```

3. Make your filesystems

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
```

4. Mount your partitions

```bash
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

5. Install Linux

```bash
pacstrap -K /mnt base linux linux-firmware vim ttf-dejavu ttf-liberation
```

Wait...

6. Create the fstab file & chroot into it

```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt/etc/fstab
```

7. While chroot'd into the new system:

```
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --sysohc
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "mynewhostname" >> /etc/hostname
```

8. Boot things

```
bootctl install
```

`/boot/loader/loader.conf`

```conf
default arch.conf
timeout 0
console-mode max
editor no
```

`/boot/loader/entries/arch.conf`

```text
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root="UUID=u-u-i-d" rw quiet splash
```

9. Set root password

```bash
passwd
```

10. Make display user (will autologin later)

```bash
useradd -m display
passwd display
```

11. Configure the Network

```bash
pacman -Syu dhcpcd
vim /etc/dhcpcd.conf

# add these lines to the end
interface eth0
static ip_address=1.2.3.4/32
static routers=1.2.3.4
static domain_name_servers=1.2.3.4
```

```bash
# Enable that service, and name resolution
sudo systemctl enable dhcpcd
sudo systemctl enable systemd-resolved
```

DON'T QUIT/REBOOT YET :)

## 2. Setup auto login & web launching

1. Enable autologin on tty1

`/etc/systemd/system/getty@tty1.service.d/autologin.conf`

```ini
[Service]
ExecStart=
ExecStart=-/usr/bin/agetty --autologin display --noclear %I 38400 linux
```

2. Install the xorg-server and xinit packages

```bash
pacman -Syu xorg-server xinit
```

2. su to the display user and create a few configs:

`su display`

`/home/display/.xinitrc`

```rc
chromium --kiosk --window-size=1920,1080 http://website
```

`/home/display/.bashrc`

Add the following:

```bash
if [ -z "$DISPLAY" ] && [ "$(tty)" == "/dev/tty1" ]; then
    startx
fi
```

3. Install chromium

```bash
pacman -Syu chromium
```

## 3. Get regular updates

`/etc/systemd/system/updates.service`

```ini
[Unit]
Description=Do Updates

[Service]
ExecStart=/sbin/pacman -Syu --noconfirm
User=root
```

`/etc/systemd/system/updates.timer`

```ini
[Unit]
Description=timer to run updates

[Timer]
OnBootSec=5min
OnUnitActiveSec=1d

[Install]
WantedBy=timers.target
```

```bash
systemctl enable updates.timer
```

Man if that worked I'm calling myself "Arch Guru"

If this step fails you may have to install the `accountsservices` package and enable the `accounts-daemon` 
