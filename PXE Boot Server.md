# PXE Boot Server

As always, we're using Rocky Linux. This guide will install Rocky 8 via pxe boot, but I'm assuming you can read between the lines and install whatever other type of OS you want from here.

## Quick summary: the end result

1. You press F12 on your client to boot from the Network.
2. Your DHCP server issues it an IP address and tells it that the "next server" (the boot server) is at [your-pxe-server].
3. The Client downloads the pxelinux.0 file using tftp.
4. The Client runs the pxelinux.0 executable, which tells it to grab the rest of the boot config from [your-pxe-server]/pxelinux.cfg/default
5. The client grabs the boot (kernel & initramfs) files via HTTP, hosted at <HTTP://deploy/OS/linux> and boots.
6. The client grabs the Kickstart file via HTTP, from <HTTP://deploy/kickstart.cfg>
7. The client follows those kickstart instructions to install linux
8. The client reboots and is a running linux machine.

## Prerequisites

Before starting you need:

1. Linux Server
2. A Linux installer iso (like Rocky8.iso)
3. Drive to make it happen in under 60 minutes :)

PXE servers on linux require a few things things: TFTP and HTTP. You will also want to have the syslinux package so that you can grab the pxelinux.0 file (read on in the PXE section). And you'll need ISO images for everything you want to boot (hint - you don't just serve the ISO, you have to mount it and serve stuff out of the iso. It's kinda like carving up a crab and serving its heart on a platter to a patron, rather than handing the patron the crab). You'll also need to configure your DHCP server (more info about that later).

## The directory structure

```txt
/
| /srv/tftp
| | /rocky8
| |  | vmlinuz       # comes from Rocky8.iso/isolinux/vmlinuz
| |  ` initrd.img    # comes from Rocky8.iso/isolinuz/initrd.img
| | /pxelinux.cfg
| |  ` default
| | ldlinux.c32      # Comes from /usr/bin/syslinux/ldlinux.c32
| ` pxelinux.0       # Comes from /usr/bin/syslinux/pxelinux.0
| 
` /var/www/html
  | /pxeboot
  |  ` /rocky8        # Mount of Rocky 8 iso image
  ` /kickstart
    ` Rocky8KS.cfg    # Kickstart file for Rocky 8
```

## PXE

PXE starts with TFTP. So we'll set up a directory for tftp to host a bunch of files (including the PXE executable) and the put in all the necessary configuration things.

1. Setup the TFTP directory structure

    ```bash
    mkdir /srv/tftp
    mkdir /srv/tftp/pxelinux.cfg
    ```

2. Get and copy the BIOS PXE boot files:

    ```bash
    yum install -y syslinux
    cp /usr/share/syslinux/pxelinux.0 /srv/tftp
    ```

3. Configure the `/srv/tftp/pxelinux.cfg/default`

    ```txt
    DEFAULT menu.c32
    PROMPT 0
    TIMEOUT 300
    ONTIMEOUT 2

    MENU TITLE PXE Boot Menu
    LABEL 1
      MENU LABEL ^1 - Install Rocky Linux 8.8 from a local repository
      KERNEL rocky8/vmlinuz
      APPEND initrd=rocky8/initrd.img showopts method=http://[server-ip]/pxeboot/rocky8/ devfs=nomount ks=http://[server-ip]/kickstart/Rocky8KS.conf

    LABEL 2
      MENU LABEL ^2 - Boot from local media
      LOCALBOOT 0x80

    label 3
      MENU LABEL ^Memtest
      KERNEL memtest
    ```

    For less information about the config than you  might expect: <https://wiki.syslinux.org/wiki/index.php?title=PXELINUX>

4. Make a place on your http server to serve the boot image data from.

    ```bash
    mkdir /var/www/html/pxeboot
    mkdir /var/www/html/pxeboot/rocky8
    mount /root/Rocky8.iso /var/www/html/pxeboot/Rocky8
    ```

5. Copy over the `vmlinuz` and `initrd.img` to `/srv/tftp/rocky8`

## Install the TFTP server

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

## Kickstart File

Ok, but like, how does the linux installer know what all to do?

What an awesome question! Unfortunately I think this article is getting a bit long... jk! I wouldn't make you go read a separate article for something that's pretty much only gonna be used in this situation lol.

About the Kickstart install: <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-howto>

Kickstart syntax: <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax>

As a side note- I'd highly recommend you check out <http://static.open-scap.org/ssg-guides/ssg-rhel8-guide-stig.html> for information about how to set up a NIST-800 compliant system.

Better yet... Use the RHEL Kickstart maker tool: <https://access.redhat.com/labs/kickstartconfig/>

```cfg
# System Auth Information
auth --enableshadow --pasalgo=sha512

%addon org_fedora_oscap
content-type = scap-security-guide
profile = xccdf_org.ssgproject.content_profile_rhvh-vpp
%end
etc
```

## Configure DHCP

Welcome to the real world where our DHCP servers run on Cisco routers!

JK, but the internet already has soooo much documentation about how to use a linux server as a dhcp server, so here's how you'd do it on a Cisco router:

```cisco
configure terminal
ip dhcp pool your-pool-name
   bootfile pxelinux.0
   next-server [your-pxe-server-ip]
   option domain-name-servers [your-dns-server-ip]
   default-router [your-gateway-ip]
   network [your-network-subnet-mask]
   lease 0 2 0 4
end
```
