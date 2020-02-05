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
  - [Build GHC 7.10.3](#build-ghc-7103)
  - [Build GHC 8.2.2](#build-ghc-822)

## Current Target Versions

- SmartOS:
  http://us-east.manta.joyent.com/Joyent_Dev/public/SmartOS/smartos.html#20200117T180335Z
- GHC:
  https://www.haskell.org/ghc/download_ghc_8_8_2.html

## Route:

> :: install 7.6.3 -> boot 7.10.3 -> boot 8.2.2 -> boot 8.4.4 -> boot 8.6.5 -> boot 8.8.2

Why? To build GHC from source, it must be bootstrapped with an already working GHC, and
each version comes with it own minimum version requirement, thus this path of bootstrapping.

## Space Occupation:

Minimum of **60GB** disk space is recommended for the `zones` pool of the **SmartOS**
build machine (virtual or physical)

- Build Stage: 20 GB
- Build Zone: 12.4 GB

## Steps

### About downloading command line

> `curl -C -` in a `while` loop can workaround an unstable internet connection to the servers.
>
> `-#` forces curl to show overall progress of downloading.

You can just use `curl -OL xxx` instead, if having a fast & stable connection.

### Prepare the build machine

> If you prefer to build with a physical machine, you must already be knowledgable
> enough to figure out the differences than with a virtual machine.

- Download SmartOS iso image
- (download the usb image & `dd` to a usb stick as boot device, might be better for a physical machine)

```bash
while ! curl -# -C - -OL http://us-east.manta.joyent.com/Joyent_Dev/public/SmartOS/20200117T180335Z/smartos-20200117T180335Z.iso ; do sleep 1; done
```

#### Create the build virtual machine

- With the downloaded `iso` mounted through a virtual cdrom
- And have the cdrom as the sole boot device

As you may already know, **SmartOS** is not to be booted from storage disks, the iso
(cdrom) will continue serving as the boot device even after installed the os, so if
the hypervisor (virt-manager in my case) ejects the cdrom after first boot / os
install, make sure to have it connected manually, forever.

> note however if with KVM, you want to change the virtual cdrom's bus to `SCSI`,
> as on one hand the default `IDE` bus boots the os noticeably slower, on the other
> hand I've found the alternative `SATA` bus can induce high probability in crash
> at boot, if you assign more than 1 cpu to the VM. You'd always want to use more
> CPU cores to make the builds faster, so the best bet is to use `SCSI` attached
> virtual cdrom for faster os boot as well.
>
> it seems `UEFI` boot with **OVMF** doesn't support boot from `SCSI` cdrom,
> you'd better stick with the default `BIOS` boot option (not this selection is
> only present at the time of vm creation with virt-manager, not to be changed
> after vm created).

- And with one virtual disk or the better, more virtual disks each on a separate physical one

> the more physical disks your smartos has, the better it performs, also true
> for VMs, where you can create one virtual disk storage file on each physical
> disk attached to the host machine. this is how ZFS rocks at zero-cost.

`VirtIO` bus recommended for virtual disks

- The more CPUs and the more RAM, the better

> I used 8GB RAM and 6 virtual CPUs with type `host-passthrough` FYI.

#### Install smartos

Boot the virtual machine and follow the prompts, it's straight forward.

> My virtual machine got an IP address of `192.168.122.69` after installed,
> yours likely to be different, substitue the IP to yours for command lines
> in the following sections.

### Prepare the build stage

It's a good idea to have a dedicated filesystem in the global zone as the
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

- GHC source tarballs

```bash
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/7.10.3/ghc-7.10.3-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.2.2/ghc-8.2.2-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.4.4/ghc-8.4.4-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.6.5/ghc-8.6.5-src.tar.xz ; do sleep 1; done
while ! curl -# -C - -OL https://downloads.haskell.org/~ghc/8.8.2/ghc-8.8.2-src.tar.xz ; do sleep 1; done
```

### Build GHC 7.10.3

According to https://www.mail-archive.com/smartos-discuss@lists.smartos.org/msg05016.html
, `ghc-7.6.3` needs a `-lssp` option added to its compiling command line;
while `7.10.3` (and later versions during our builds) seems don't really
need a similar patch.

```bash
[root@hswander /build]# patch /opt/local/lib/ghc-7.6.3/settings smart-ghc8/ghc-arts/ghc-7.6.3_settings.patch
patching file /opt/local/lib/ghc-7.6.3/settings
```

> `gtar` can better guess the compression method applied to tarballs, easier
> on command line.

```bash
[root@hswander /build]# gtar xf ghc-7.10.3-src.tar.xz
[root@hswander /build]# cd ghc-7.10.3
```

> We have better sources for haddock of the stable versions, by not
> building it, some time are saved and we don't bother to setup
> the more complex dependencies for haddock on SmartOS.

```bash
[root@hswander /build/ghc-7.10.3]# echo 'HADDOCK_DOCS = NO' > mk/build.mk
```

I found `--disable-ld-override` not necessary, but we should specify
`--prefix` to let the installing not target system locations.

```
[root@hswander /build/ghc-7.10.3]# ./configure --prefix /opt/local/ghc7.10.3 2>&1 | tee /build/log-7.10.3-configure.txt
checking for gfind... /opt/local/bin/gfind
 ...
----------------------------------------------------------------------
Configure completed successfully.

   Building GHC version  : 7.10.3
          Git commit id  : 97e7c293abbde5223d2bf0516f8969bdd1a9a7a2

   Build platform        : x86_64-unknown-solaris2
   Host platform         : x86_64-unknown-solaris2
   Target platform       : x86_64-unknown-solaris2

   Bootstrapping using   : /opt/local/bin/ghc
      which is version   : 7.6.3

   Using gcc                 : /opt/local/bin/gcc
      which is version       : 7.4.0
   Building a cross compiler : NO
   cpp       : /opt/local/bin/gcc
   cpp-flags : -E -undef -traditional
   ld        : /usr/bin/ld
   Happy     :  ()
   Alex      :  ()
   Perl      : /opt/local/bin/perl
   dblatex   :
   xsltproc  :

   Using LLVM tools
      llc   :
      opt   :

   HsColour was not found; documentation will not contain source links

   Building DocBook HTML documentation : NO
   Building DocBook PS documentation   : NO
   Building DocBook PDF documentation  : NO
----------------------------------------------------------------------

For a standard build of GHC (fully optimised with profiling), type (g)make.

To make changes to the default build configuration, copy the file
mk/build.mk.sample to mk/build.mk, and edit the settings in there.

For more information on how to configure your GHC build, see
   http://ghc.haskell.org/trac/ghc/wiki/Building

[root@hswander /build/ghc-7.10.3]#
```

> The number to `-j` should correspond to how many virtual CPUs
> allocated to the build vm.

```bash
[root@hswander /build/ghc-7.10.3]# gmake -j6 2>&1 | tee /build/log-7.10.3-make.txt
 ...
```

make the install

```bash
[root@hswander /build/ghc-7.10.3]# gmake install 2>&1 | tee /build/log-7.10.3-install.txt
 ...
[root@hswander /build/ghc-7.10.3]# /opt/local/ghc7.10.3/bin/ghci
GHCi, version 7.10.3: http://www.haskell.org/ghc/  :? for help
Prelude> 3*7
21
Prelude>
Leaving GHCi.
```

(optionally) make the bindist

```bash
[root@hswander /build/ghc-7.10.3]# gmake binary-dist
 ...
[root@hswander /build/ghc-7.10.3]# du -h ghc-7.10.3-*
123M	ghc-7.10.3-x86_64-unknown-solaris2.tar.bz2
```

### Build GHC 8.2.2

```bash
[root@hswander /build]# gtar xf ghc-8.2.2-src.tar.xz
[root@hswander /build]# cp smart-ghc8/ghc-arts/mk_build.mk ghc-8.2.2/mk/build.mk
[root@hswander /build]# patch -p0 < smart-ghc8/ghc-arts/ghc-8.2.2_rules_distdir-way-opts.mk.patch
patching file ghc-8.2.2/rules/distdir-way-opts.mk
[root@hswander /build]# cd ghc-8.2.2
[root@hswander /build/ghc-8.2.2]# export PATH=/opt/local/ghc7.10.3/bin:$PATH
[root@hswander /build/ghc-8.2.2]#
[root@hswander /build/ghc-8.2.2]# ./configure --prefix /opt/local/ghc8.2.2 2>&1 | tee /build/log-8.2.2-configure.txt
 ...
----------------------------------------------------------------------
Configure completed successfully.

   Building GHC version  : 8.2.2
          Git commit id  : 0156a3d815b784510a980621fdcb9c5b23826f1e

   Build platform        : x86_64-unknown-solaris2
   Host platform         : x86_64-unknown-solaris2
   Target platform       : x86_64-unknown-solaris2

   Bootstrapping using   : /opt/local/ghc7.10.3/bin/ghc
      which is version   : 7.10.3

   Using (for bootstrapping) : /opt/local/bin/gcc
   Using gcc                 : gcc
      which is version       : 7.4.0
   Building a cross compiler : NO
   Unregisterised            : NO
   hs-cpp       : gcc
   hs-cpp-flags : -E -undef -traditional
   ar           : /opt/local/bin/ar
   ld           : ld
   nm           : /opt/local/bin/nm
   objdump      : /opt/local/bin/objdump
   ranlib       : /opt/local/bin/ranlib
   windres      :
   dllwrap      :
   Happy        :  ()
   Alex         :  ()
   Perl         : /opt/local/bin/perl
   sphinx-build :
   xelatex      :

   Using LLVM tools
      llc   :
      opt   :

   HsColour was not found; documentation will not contain source links

   Tools to build Sphinx HTML documentation available: NO
   Tools to build Sphinx PDF documentation available: NO
----------------------------------------------------------------------

For a standard build of GHC (fully optimised with profiling), type (g)make.

To make changes to the default build configuration, copy the file
mk/build.mk.sample to mk/build.mk, and edit the settings in there.

For more information on how to configure your GHC build, see
   http://ghc.haskell.org/trac/ghc/wiki/Building

[root@hswander /build/ghc-8.2.2]#
[root@hswander /build/ghc-8.2.2]# gmake -j6 2>&1 | tee /build/log-8.2.2-make.txt
 ...
"/opt/local/ghc7.10.3/bin/ghc" -o utils/deriveConstants/dist/build/tmp/deriveConstants -hisuf hi -osuf  o -hcsuf hc -static  -H32m -O -Wall   -package-db libraries/bootstrapping.conf  -hide-all-packages -i -iutils/deriveConstants/. -iutils/deriveConstants/dist/build -Iutils/deriveConstants/dist/build -iutils/deriveConstants/dist/build/deriveConstants/autogen -Iutils/deriveConstants/dist/build/deriveConstants/autogen     -optP-include -optPutils/deriveConstants/dist/build/deriveConstants/autogen/cabal_macros.h -package-id base-4.8.2.0-22c9761e23a803f9ae779a2c20934380 -package-id containers-0.5.6.2-11bc7e6db5f4aab8daf90becef78c10e -package-id process-1.2.3.0-c30e4c640a5d1dd03e371e2b728281e1 -package-id filepath-1.4.0.0-f97d1e4aebfd7a03be6980454fe31d6e -XHaskell2010  -no-user-package-db -rtsopts       -odir utils/deriveConstants/dist/build -hidir utils/deriveConstants/dist/build -stubdir utils/deriveConstants/dist/build    -optl-Wl,-m64 -static  -H32m -O -Wall   -package-db libraries/bootstrapping.conf  -hide-all-packages -i -iutils/deriveConstants/. -iutils/deriveConstants/dist/build -Iutils/deriveConstants/dist/build -iutils/deriveConstants/dist/build/deriveConstants/autogen -Iutils/deriveConstants/dist/build/deriveConstants/autogen     -optP-include -optPutils/deriveConstants/dist/build/deriveConstants/autogen/cabal_macros.h -package-id base-4.8.2.0-22c9761e23a803f9ae779a2c20934380 -package-id containers-0.5.6.2-11bc7e6db5f4aab8daf90becef78c10e -package-id process-1.2.3.0-c30e4c640a5d1dd03e371e2b728281e1 -package-id filepath-1.4.0.0-f97d1e4aebfd7a03be6980454fe31d6e -XHaskell2010  -no-user-package-db -rtsopts       utils/deriveConstants/dist/build/Main.o
"/opt/local/ghc7.10.3/bin/ghc" -hisuf hi -osuf  o -hcsuf hc -static  -H32m -O -Wall   -package-db libraries/bootstrapping.conf  -hide-all-packages -i -iutils/genprimopcode/. -iutils/genprimopcode/dist/build -Iutils/genprimopcode/dist/build -iutils/genprimopcode/dist/build/genprimopcode/autogen -Iutils/genprimopcode/dist/build/genprimopcode/autogen     -optP-include -optPutils/genprimopcode/dist/build/genprimopcode/autogen/cabal_macros.h -package-id base-4.8.2.0-22c9761e23a803f9ae779a2c20934380 -package-id array-0.5.1.0-960bf9ae8875cc30355e086f8853a049 -XHaskell2010  -no-user-package-db -rtsopts       -odir utils/genprimopcode/dist/build -hidir utils/genprimopcode/dist/build -stubdir utils/genprimopcode/dist/build    -c utils/genprimopcode/./Main.hs -o utils/genprimopcode/dist/build/Main.o
ghc: fd:12: hGetContents: resource exhausted (Resource temporarily unavailable)
libraries/hpc/ghc.mk:3: libraries/hpc/dist-boot/build/.depend-v.haskell: No such file or directory
gmake[1]: *** [utils/deriveConstants/ghc.mk:19: utils/deriveConstants/dist/build/tmp/deriveConstants] Error 2
gmake[1]: *** Deleting file 'utils/deriveConstants/dist/build/tmp/deriveConstants'
gmake[1]: *** Waiting for unfinished jobs....
gmake: *** [Makefile:125: all] Error 2
[root@hswander /build/ghc-8.2.2]#
```

There's a strange error that `/opt/local/ghc7.10.3/bin/ghc` would crash after
the target file can actually be generated corrrectly, we have to tell `make`
to ignore the crash and go on as if normally.

> This renders the finally built **ghc8.2.2** worrying to be used seriously,
> but as we are only to bootstrap **ghc8.4.4** with it, and even the built
> **ghc8.4.4** is only used to bootstrap **ghc8.6.5** or **ghc8.8.2**, let's
> live with this concern without going too deep into it.

```bash
[root@hswander /build/ghc-8.2.2]# gmake -i -j6 2>&1 | tee /build/log-8.2.2-make-i.txt
 ...
```
