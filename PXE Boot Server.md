# PXE Boot Server

As always, we're using Rocky Linux. This guide will install Rocky 8 via pxe boot, but I'm assuming you can read between the lines and install whatever other type of OS you want from here.

> Side note - I think MemTest (<https://memtest.org/>) is a super handy tool to have. I've included a section in this article about Memtest. If you don't need it or want it, then just skip that part - it's not _strictly_ necessary. But super handy.

I'll leave it to you as an exercise for you to figure out how to make a thing that will re-mount the iso images on boot.

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
3. Motivation to make it happen in under 60 minutes :)

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
| | memtest.bin      # optional, from memtest.org
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
    cp /usr/share/syslinux/pxelinux.0  /srv/tftp   
    cp /usr/share/syslinux/ldlinux.c32 /srv/tftp
    cp memtest64.bin /srv/tftp  # optional
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
      KERNEL memtest64.bin
    ```

    For less information about the config than you  might expect: <https://wiki.syslinux.org/wiki/index.php?title=PXELINUX>

4. Make a place on your http server to serve the boot image data from.

    ```bash
    mkdir /var/www/html/pxeboot
    mkdir /var/www/html/pxeboot/rocky8
    mount /root/Rocky8.iso /var/www/html/pxeboot/rocky8
    ```

5. Copy over the `vmlinuz` and `initrd.img` to `/srv/tftp/rocky8`

    ```bash
    cp /var/www/html/pxeboot/rocky8/isolinux/vmlinuz /srv/tftp/rocky8
    cp /var/www/html/pxeboot/rocky8/isolinux/initrd.img /srv/tftp/rocky8
    ```

## Install the TFTP server

> Note - xinetd is dead and gone (rip xinetd)

1. Create a folder for the TFTP files: `mkdir /srv/tftp`
2. Create a TFTP service user

    ```bash
    useradd -r tftp -d /srv/tftp -s /sbin/nologin
    ```

3. Give the tftp user ownership of the tftp directory: `chown -R tftp /srv/tftp ; chmod -R 755 /srv/tftp`
4. Install TFTP Server `sudo yum install tftp-server`
5. Edit the systemd unit file:

    `/usr/lib/systemd/system/tftp.service`

    ```ini
    [Unit]
    Description=Tftp Server
    Requires=tftp.socket
    Documentation=man:in.tftpd

    [Service]
    ExecStart=/usr/sbin/in.tftpd -B 600 -R 9000:9090 -s /srv/tftp -u tftp
    StandardInput=socket

    [Install]
    Also=tftp.socket
    ```

    Fun fact! TFTP listens for new connections on port 69, but then transfers the file on some other port. To specify where that port should be (which you should so you can protect your system via a firewall) use -R startport:endport.

    Fun Fact 2: If your network has small mtu sizes, you probably need to set a more appropriate MTU size for your server. tftp-server uses the -B or --blocksize option for this. -B 600

    Fun Fact 3: you do actually have to run this as root with the -u tftp option. idk wy it just is what it is.

6. Start & Enable the tftp service: `systemctl enable --now tftp`
7. Add the tftp service (and associated ports) to the firwall:

    ```bash
    firewall-cmd --add-service tftp --permanent
    firewall-cmd --add-port 9000-9090/udp --permanent
    firewall-cmd --reload
    ```

### Test it with tftp client

1. Install tftp: `yum install tftp`
2. Use tftp to grab a file:

    ```bash
    tftp localhost
    tftp> get pxelinux.0
    ```

3. Note - your client firewall needs to allow tftp (or just temporarily disable your firewall)

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

## Memtest

This is entirely optional, but if you want to make memtest available via PXE boot then do this:

1. Download the "Binary Files (.bin/.efi)" from <https://memtest.org> . I was able to grab these from <https://memtest.org/download/v6.20/mt86plus_6.20.binaries.zip> (but this may be out of date).
2. Extract the binaries: `unzip mt86plus_6.20.binaries.zip`
3. Copy the desired memtest file over to the TFTP directory: `cp memtest64.bin /srv/tftp`
4. Add that config to your pxelinux.cfg/default file:

    ```cfg
    LABEL 6
      MENU LABEL ^Memtest
      KERNEL memtest64.bin
    ```

5. * Note - you want to use the appropriate file. If you configured PXE for UEFI, then use the memtest64.efi files. If you have 32-bit systems in use, use memtest32.bin.
