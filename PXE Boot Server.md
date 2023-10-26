# PXE Boot Server

As always, we're using Rocky Linux.

## Prerequisites

PXE servers on linux require 2 things: TFTP and HTTP. You will also want to have the syslinux package so that you can grab the pxelinux.0 file (read on in the PXE section). You'll also need to configure your DHCP server (more info about that later).

## PXE

PXE starts with TFTP. So we'll set up a directory for tftp to host a bunch of files (including the PXE executable) and the put in all the necessary configuration things.

1. Setup the TFTP directory structure

    ```bash
    mkdir /srv/tftp
    mkdir /srv/tftp/pxelinux.cfg
    ```

2. Get and copy the UEFI boot files (I assume you're using grub, unfortunately):

    ```bash
    mkdir /srv/tftp/EFI
    mkdir /srv/tftp/EFI/BOOT
    cp /boot/efi/EFI/BOOT/BOOTX64.EFI /srv/tftp/EFI/BOOT/
    ```

3. Configure the `/srv/tftp/EFI/BOOT/grub.cfg`

    ```txt
    default menu.c32
    prompt 0
    timeout 300
    ontimeout 2

    menu title PXE Boot Menu
    label 1
      menu label ^1 - Install Rocky Linux 8.8 from a local repository
      kernel rhel8/vmlinuz
      append initrd=rhel8/initrd.img showopts method=http://[server-ip]/rhel/ devfs=nomount ks=http://[server-ip]/ks/Rocky8KS.conf

    label 2
      menu label ^2 - Boot from local media
      localboot 0x80
    ```

For more information about the config: <https://wiki.syslinux.org/wiki/index.php?title=PXELINUX>

### Install the TFTP server

1. Create a folder for the TFTP files: `mkdir /srv/tftp`
2. Create a TFTP service user

    ```bash
    useradd -r tftp -d /srv/tftp -s /sbin/nologin
    ```

3. Give the tftp user ownership of the tftp directory: `chown -R tftp /srv/tftp ; chmod -R 600 /srv/tftp`
4. Install TFTP Server `sudo yum install tftp-server xinetd`
5. Edit the configuration file:

    `/etc/xinetd.d/tftp`

    ```conf
    service tftp
    {
        socket_type     = dgram
        protocol        = udp
        wait            = yes
        user            = tftp # is root by default
        server          = /usr/sbin/in.tftpd
        server_args     = -s /srv/tftp
        disable         = no # is yes by default
        per_source      = 11
        cps             = 100 2
    }
    ```

6. Start & Enable the xinetd (tftp) service: `systemctl enable --now xinetd`

### Test it with tftp client

1. Install tftp: `yum install tftp`
2. Use tftp to grab a file:

    ```bash
    tftp localhost
    tftp> get pxelinux.0
    ```
