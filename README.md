# Building GHC 8 on SmartOS x64

- [Current Target Versions](#current-target-versions)
- [Route:](#route)
- [Space Occupation:](#space-occupation)
- [Steps](#steps)
  - [About downloading command line](#about-downloading-command-line)
  - [Prepare the build machine](#prepare-the-build-machine)
    - [Create the build virtual machine](#create-the-build-virtual-machine)
    - [Install smartos](#install-smartos)
  - [Prepare the build stage](#prepare-the-build-stage)
  - [Prepare the build zone](#prepare-the-build-zone)
    - [With stable connection to Joyent server](#with-stable-connection-to-joyent-server)
    - [With unstable connection to Joyent server](#with-unstable-connection-to-joyent-server)
  - [Create the build zone](#create-the-build-zone)
  - [Download GHC Files](#download-ghc-files)

## Current Target Versions

- SmartOS:
  http://us-east.manta.joyent.com/Joyent_Dev/public/SmartOS/smartos.html#20200117T180335Z
- GHC:
  https://www.haskell.org/ghc/download_ghc_8_8_2.html

## Route:

> :: install 7.6.3 -> boot 7.10.3 -> boot 8.2.2 -> boot 8.4.4 -> boot 8.6.5 -> boot 8.8.2

Why? GHC must be bootstrapped with an already working GHC, and each version has it own
minimum version requirement, thus this path of bootstrapping.

## Space Occupation:

Minimum of **60GB** disk space recommended for the `zones` pool of the SmartOS build (may
be virtual) machine

- Build Stage: 20GB
- Build Zone: 12.4GB

## Steps

### About downloading command line

> `curl -C -` in a `while` loop can workaround an unstable internet connection to the servers.
>
> `-#` forces curl to show overall progress of downloading.

You can just use `curl -OL xxx` instead if having a fast & stable connection.

### Prepare the build machine

> If you prefer to build with a physical machine, you must already be knowledgable
> enough to figure out the differences than with a virtual machine.

- Download SmartOS iso image
- (download the usb image & `dd` to a usb stick as boot device, might be better for a physical machine)

```bash
while ! curl -# -C - -OL http://us-east.manta.joyent.com/Joyent_Dev/public/SmartOS/20200117T180335Z/smartos-20200117T180335Z.iso ; do sleep 1; done
```

#### Create the build virtual machine

- With the downloaded `iso` mounted to its virtual cdrom

As you may have knew, **SmartOS** is not to be booted from disks, the iso (cdrom)
will continue serving as the boot device even after installed the os, so if the
hypervisor (virt-manager in my case) ejects the cdrom after first boot / os install,
make sure to have it connected manually, forever.

> note however if with KVM, you want to change the virtual cdrom's bus to `SCSI`,
> as on one hand the default `IDE` bus boots the os noticably slower, on the other
> hand I've found the alternative `SATA` bus can induce high probability in crash
> at boot, if you assign more than 1 cpu to the VM. You'd always want to use more
> CPU cores to make the builds faster, so the best bet is to use `SCSI` attached
> virtual cdrom for faster os boot as well.
>
> it seems `UEFI` boot with **OVMF** doesn't support boot from `SCSI` cdrom,
> you'd better stick with the default `BIOS` boot option.

- And with one or more virtual disks

> the more physical disks your smartos has, the better it performs, also true
> for VMs, where you can create one virtual disk storage file on each physical
> disk attached to the host machine. this is how ZFS rocks at zero-cost.

`VirtIO` bus recommended for virtual disks

- The more CPUs and the more RAM the better

> I used 8GB RAM and 6 virtual CPUs with type `host-passthrough` FYI.

#### Install smartos

Boot the virtual machine and follow the prompts, it's straight forward.

> My virtual machine got an IP address of `192.168.122.69` after installed,
> yours likely to be different, substitue the IP to yours for command lines
> in the following sections.

### Prepare the build stage

It's a good idea to have a dedicated filesystem in the global zone for the
build staging area, which is shared to a dedicated zone for the builds.

```bash
# zfs create -o mountpoint=/build zones/build
```

Clone and scp this repository to the build stage:

> If you'd prefer install **git** to your GZ, you can clone directly there

```bash
$ git clone https://gitlab.haskell.org/complyue/smart-ghc8
$ scp -r smart-ghc8 root@192.168.122.69:/build/
```

### Prepare the build zone

#### With stable connection to Joyent server

Things are straight forward and easy with a stable connection

- Decide the best zone image to use

```bash
# imgadm avail name=base-64-lts
UUID                                  NAME         VERSION  OS       TYPE          PUB
c02a2044-c1bd-11e4-bd8c-dfc1db8b0182  base-64-lts  14.4.0   smartos  zone-dataset  2015-03-03
24648664-e50c-11e4-be23-0349d0a5f3cf  base-64-lts  14.4.1   smartos  zone-dataset  2015-04-17
b67492c2-055c-11e5-85d8-8b039ac981ec  base-64-lts  14.4.2   smartos  zone-dataset  2015-05-28
96bcddda-beb7-11e5-af20-a3fb54c8ae29  base-64-lts  15.4.0   smartos  zone-dataset  2016-01-19
088b97b0-e1a1-11e5-b895-9baa2086eb33  base-64-lts  15.4.1   smartos  zone-dataset  2016-03-04
1f32508c-e6e9-11e6-bc05-8fea9e979940  base-64-lts  16.4.1   smartos  zone-dataset  2017-01-30
390639d4-f146-11e7-9280-37ae5c6d53d4  base-64-lts  17.4.0   smartos  zone-dataset  2018-01-04
c193a558-1d63-11e9-97cf-97bb3ee5c14f  base-64-lts  18.4.0   smartos  zone-dataset  2019-01-21
e75c9d82-3156-11ea-9220-c7a6bb9f41b6  base-64-lts  19.4.0   smartos  zone-dataset  2020-01-07
```

Record its uuid, at the time of this writing, that is
`e75c9d82-3156-11ea-9220-c7a6bb9f41b6`, which is described as:

> A 64-bit SmartOS image with just essential packages installed. Ideal for
> users who are comfortable with setting up their own environment and tools.

- Import the image locally

```bash
# imgadm import e75c9d82-3156-11ea-9220-c7a6bb9f41b6
Importing e75c9d82-3156-11ea-9220-c7a6bb9f41b6 (base-64-lts@19.4.0) from "https://images.joyent.com"
Gather image e75c9d82-3156-11ea-9220-c7a6bb9f41b6 ancestry
Must download and install 1 image (183.7 MiB)
 ...
```

#### With unstable connection to Joyent server

If you are like me, with a rather unstable connection (even over VPN), do this:

- In a browser (with proxy probably), goto:

  - https://images.joyent.com/images?sort=published_at.desc&name=base-64-lts

    to decide the best zone image to use, record its uuid

- Download its manifest & filesystem payload separately:

```bash
curl -o e75c9d82-3156-11ea-9220-c7a6bb9f41b6.manifest -L https://images.joyent.com/images/e75c9d82-3156-11ea-9220-c7a6bb9f41b6
while ! curl -# -C - -o e75c9d82-3156-11ea-9220-c7a6bb9f41b6.gz -L https://images.joyent.com/images/e75c9d82-3156-11ea-9220-c7a6bb9f41b6/file ; do sleep 1; done
```

- Install the image locally

```bash
imgadm install -m e75c9d82-3156-11ea-9220-c7a6bb9f41b6.manifest -f e75c9d82-3156-11ea-9220-c7a6bb9f41b6.gz
```

### Create the build zone

Modify `hswander.json` to your needs, and create the vm

```bash
[root@smartwander ~]# cd /build/smart-ghc8/
[root@smartwander /build/smart-ghc8]# cat smartos-arts/hswander.json
{
 "brand": "joyent",
 "image_uuid": "e75c9d82-3156-11ea-9220-c7a6bb9f41b6",
 "alias": "hswander",
 "hostname": "hswander",
 "max_physical_memory": 8196,
 "quota": 20,
 "resolvers": ["192.168.122.1", "223.5.5.5"],
 "nics": [
  {
    "nic_tag": "admin",
    "ip": "dhcp"
  }
 ]
}

[root@smartwander /build/smart-ghc8]# vmadm create -f smartos-arts/hswander.json
Successfully created VM a088383f-7e61-cd6a-e654-f845c7545ae9
```

Record the new vm's uuid, yours must be different than mine (which is
`a088383f-7e61-cd6a-e654-f845c7545ae9`), make sure substitute it for command lines
in following sections.

Stop the vm and grant it access to the build stage

> Note SmartOS is happily to tab-complete the vm's uuid and many other uuids
> on the bash command line, will save you many typings (or copy&paste efforts)

```bash
[root@smartwander /build/smart-ghc8]# vmadm list
UUID                                  TYPE  RAM      STATE             ALIAS
a088383f-7e61-cd6a-e654-f845c7545ae9  OS    8196     running           hswander
[root@smartwander /build/smart-ghc8]# vmadm stop a088383f-7e61-cd6a-e654-f845c7545ae9
Successfully completed stop for VM a088383f-7e61-cd6a-e654-f845c7545ae9
[root@smartwander /build/smart-ghc8]#
[root@smartwander /build/smart-ghc8]# zonecfg -z a088383f-7e61-cd6a-e654-f845c7545ae9
zonecfg:a088383f-7e61-cd6a-e654-f845c7545ae9> add fs
zonecfg:a088383f-7e61-cd6a-e654-f845c7545ae9:fs> set type=lofs
zonecfg:a088383f-7e61-cd6a-e654-f845c7545ae9:fs> set special=/build
zonecfg:a088383f-7e61-cd6a-e654-f845c7545ae9:fs> set dir=/build
zonecfg:a088383f-7e61-cd6a-e654-f845c7545ae9:fs> end
zonecfg:a088383f-7e61-cd6a-e654-f845c7545ae9> commit
zonecfg:a088383f-7e61-cd6a-e654-f845c7545ae9> exit
[root@smartwander /build/smart-ghc8]#
[root@smartwander /build/smart-ghc8]# vmadm start a088383f-7e61-cd6a-e654-f845c7545ae9
Successfully started VM a088383f-7e61-cd6a-e654-f845c7545ae9
```

Login to the build zone

```bash
[root@smartwander /build/smart-ghc8]# zlogin a088383f-7e61-cd6a-e654-f845c7545ae9
[Connected to zone 'a088383f-7e61-cd6a-e654-f845c7545ae9' pts/2]
   __        .                   .
 _|  |_      | .-. .  . .-. :--. |-
|_    _|     ;|   ||  |(.-' |  | |
  |__|   `--'  `-' `;-| `-' '  ' `-'
                   /  ; Instance (base-64-lts 19.4.0)
                   `-'  https://docs.joyent.com/images/smartos/base

[root@hswander ~]# df -h /build
Filesystem      Size  Used Avail Use% Mounted on
-                48G  184M   48G   1% /build
[root@hswander ~]# df -h /opt
Filesystem                                  Size  Used Avail Use% Mounted on
zones/a088383f-7e61-cd6a-e654-f845c7545ae9   21G  623M   20G   3% /
```

Install build tools

```bash
[root@hswander ~]# pkgin update
reading local summary...
processing local summary...
processing remote summary (https://pkgsrc.joyent.com/packages/SmartOS/2019Q4/x86_64/All)...
pkg_summary.xz                                                                                                                                                    100% 2172KB  12.9KB/s   02:49
[root@hswander ~]#
[root@hswander ~]# while ! pkgin -y in build-essential ; do sleep 1; done
calculating dependencies...done.

3 packages to refresh:
  libxml2-2.9.10nb1 nghttp2-1.40.0 curl-7.67.0

40 packages to install:
  libidn-1.34nb1 p5-Net-SSLeay-1.85nb2 p5-Net-LibIDN-0.12nb11 p5-Mozilla-CA-20180117nb2 p5-Socket6-0.29nb1 p5-Net-IP-1.26nb7 p5-MIME-Base64-3.15nb5 p5-IO-Socket-INET6-2.72nb5 p5-Digest-MD5-2.55nb4
  mit-krb5-1.16.2nb3 p5-GSSAPI-0.28nb10 p5-Digest-HMAC-1.03nb9 p5-Net-Domain-TLD-1.75nb3 p5-Net-DNS-1.21 p5-IO-CaptureOutput-1.1105 p5-TimeDate-2.30nb6 p5-IO-Socket-SSL-2.060nb1
  gettext-tools-0.20.1 pcre2-10.34 p5-Net-SMTP-SSL-1.04nb3 p5-MailTools-2.20nb2 p5-Error-0.17028 p5-Email-Valid-1.202nb3 p5-Authen-SASL-2.16nb7 expat-2.2.8 libtool-info-2.4.6
  libtool-fortran-2.4.6nb1 libtool-base-2.4.6nb2 pkgconf-1.6.0 m4-1.4.18nb2 libtool-2.4.6 gmake-4.2.1nb1 git-docs-2.24.1 git-base-2.24.1 gcc7-7.4.0nb3 bison-3.4.2 binutils-2.26.1nb1
  automake-1.16.1nb1 autoconf-2.69nb9 build-essential-1.3

3 to refresh, 0 to upgrade, 40 to install
140M to download, 386M to install
 ...
```

> With an unstable connection to Joyent server, you'd be downloading some
> large files from
> https://pkgsrc.joyent.com/packages/SmartOS/2019Q4/x86_64/All/
> manually and install them with `pkg_add` until the meta package
> `build-essential` has all its dependencies (constitutes) installed.

### Download GHC Files

- GHC 7.6.3 binary package from pkgsrc

> 7.6.3 is the highest binary version from **pkgsrc**, and not available from latest repositories,
> must download and install with `pkg_add`, `pkgin install ghc` is not working now.

```bash
while ! curl -# -C - -OL https://pkgsrc.joyent.com/packages/SmartOS/2019Q2/x86_64/All/ghc-7.6.3nb13.tgz ; do sleep 1; done
```

- GHC source tar balls

```bash
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/7.10.3/ghc-7.10.3-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.2.2/ghc-8.2.2-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.4.4/ghc-8.4.4-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.6.5/ghc-8.6.5-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.8.2/ghc-8.8.2-src.tar.xz ; do sleep 1; done
```
