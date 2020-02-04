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
  - [Download Media Files](#download-media-files)

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

It's better to have a dedicated filesystem in the global zone for the build
staging area, which is shared to a dedicated zone for the builds.

```bash
# zfs create -o mountpoint=/build zones/build
```

Clone and scp this repository to the build stage:

```bash
$ git clone 
$ scp -r smart-ghc8 root@192.168.122.69:/build/
```

### Prepare the build zone

```bash

```


```bash

```


```bash

```


```bash

```

### Download Media Files

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

- (Optional if you have an unstable connection to joyent image server) Base 64 zone image

```bash
curl -o e75c9d82-3156-11ea-9220-c7a6bb9f41b6.manifest -L https://images.joyent.com/images/e75c9d82-3156-11ea-9220-c7a6bb9f41b6
while ! curl -# -C - -o e75c9d82-3156-11ea-9220-c7a6bb9f41b6.gz -L https://images.joyent.com/images/e75c9d82-3156-11ea-9220-c7a6bb9f41b6/file ; do sleep 1; done
imgadm install -m e75c9d82-3156-11ea-9220-c7a6bb9f41b6.manifest -f e75c9d82-3156-11ea-9220-c7a6bb9f41b6.gz
```
