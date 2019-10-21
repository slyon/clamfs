# ClamFS
ClamFS - User-space fs with on-access antivirus scanning

## Description

ClamFS is a [FUSE-based user-space file system](https://en.wikipedia.org/wiki/Filesystem_in_Userspace)
for Linux and [BSD](https://www.freshports.org/security/clamfs/)
with on-access anti-virus file scanning through [clamd daemon](https://manpages.debian.org/testing/clamav-daemon/clamd.8.en.html)
(a file scanning service developed by [ClamAV Project](https://www.clamav.net/)).

## Features

 * User-space file system (no kernel patches, recompilation, etc.)
 * Configuration stored in XML files
 * FUSE (and libfuse) used as file system back-end
 * Scan files using ClamAV
 * ScanCache (LRU with time-based and out-of-memory expiration) speeds up file access
 * Sends mail to administrator when detect virus

## Getting Started

These instructions will get you a copy of the project up and running on your
local machine.

### Installing packages

#### Arch

[ClamFS package](https://aur.archlinux.org/packages/clamfs/)
is available from [AUR](https://aur.archlinux.org/) repository.

#### Debian, Ubuntu, etc.

[Debian GNU/Linux](https://packages.debian.org/clamfs),
[Ubuntu](https://packages.ubuntu.com/clamfs) and
[Devuan](https://pkginfo.devuan.org/cgi-bin/d1pkgweb-query?search=clamfs)
have `clamfs` package in their repositories.

```
sudo apt install clamfs clamav-daemon clamav-freshclam
```

#### Gentoo

Gentoo provides [sys-fs/clamfs package](https://packages.gentoo.org/packages/sys-fs/clamfs).

#### FreeBSD, DragonFly BSD

[FreeBSD](https://www.freshports.org/security/clamfs/) and [DragonFly BSD]() has `security/clamfs` in ports.

Install package...
```
pkg install clamfs
```

... or install from ports.
```
cd /usr/ports/security/clamfs ; make install clean
```

### Building from sources

#### Prerequisites

To build ClamFS on any GNU/Linux or *BSD you need:
 * [FUSE](https://github.com/libfuse/libfuse)
 * [RLog](https://www.arg0.net/rlog)
 * [POCO](https://pocoproject.org/)
 * [Boost](https://www.boost.org/)

To run ClamFS `clamd` service from [ClamAV project](https://www.clamav.net/)
is required.

Note 1: POCO versions up to 1.2.8 contain 4-BSDL licensed files and thus you
should avoid linking it against any GPL licensed code. I strongly advise using
version 1.2.9 or newer (as license issues has been fixed).

Note 2: ClamFS version up to 1.0.1 required also
[GNU CommonCPP](https://www.gnu.org/software/commoncpp/)
library. This dependency was dropped in version 1.1.0 (with commit 3bdb8ec).

#### Installing dependencies

##### Debian, Ubuntu, etc.

To build ClamFS on Debian GNU/Linux and Ubuntu install these packages:
 * libfuse-dev
 * librlog-dev
 * libpoco-dev
 * libboost-dev

As a run-time dependency install:
 * clamav-daemon
 * fuse

Run following command to install al dependencies.
```
sudo apt-get -y --no-install-recommends install \
      build-essential automake libfuse-dev \
      librlog-dev libpoco-dev libboost-dev \
      clamav-daemon clamav-freshclam
```

##### FreeBSD, DragonFly BSD

To build ClamFS on FreeBSD and DragonFly BSD you need those ports:
 * sysutils/fusefs-libs
 * devel/rlog
 * devel/poco-ssl (or devel/poco)
 * devel/boost-libs

As a run-time dependency you need:
 * security/clamav

Note: older FreeBSD version required port named `sysutils/fusefs-kmod`.
This is no longer the case as `fuse` module is part of kernel.

#### Downloading

Just download the release package and extract it with `tar`.

```
tar xf clamfs-<version>.tar.gz
```

Or clone repository.

```
git clone https://github.com/burghardt/clamfs.git
```

#### Building

If using cloned repository rebuild autotools configuration with `autogen.sh`
script. If using release tarballs skip this step.
```
sh autogen.sh
```

Configure package with `configure` script.
```
sh configure
```

Finally build sources with `make`.
```
make -j
```

#### Installing

Run `make install` (as root) to install binaries.
```
sudo make install
```

### Usage

ClamFS requires only one argument - configuration file name. Configuration is
stored as XML document. Sample configuration is available in `doc` directory,
in file named [clamfs.xml](doc/clamfs.xml).

#### Sample output

```
17:11:44 (clamfs.cxx:993) ClamFS v1.1.0-snapshoot (git-7d4beda)
17:11:44 (clamfs.cxx:994) Copyright (c) 2007-2019 Krzysztof Burghardt <krzysztof@burghardt.pl>
17:11:44 (clamfs.cxx:995) https://github.com/burghardt/clamfs
17:11:44 (clamfs.cxx:1004) ClamFS need to be invoked with one parameter - location of configuration file
17:11:44 (clamfs.cxx:1005) Example: src/clamfs /etc/clamfs/home.xml
```

#### Configuration

Please refer to [clamfs.xml](doc/clamfs.xml) for comprehensive list of
configuration options. Only three options are mandatory:
 * `<clamd socket="" />` to set path to `clamd` socket
 * `<filesystem root="" />` to set place from ClamFS will read files
 * `<filesystem mountpoint="" />` to set mount point where virtual filesystem
   will be attached in directory tree

#### Additional configuration steps for FreeBSD

FreeBSD's `fuse` kernel module has to be loaded before starting ClamFS. This
can be done ad-hoc with `kldload fuse` command.

To have it loaded at boot time, add the following line to `/boot/loader.conf`.
```
fuse_load="YES"
```

Or append fuse module to `kld_list` in `/etc/rc.conf`.
```
kld_list="fuse"
```

Also configure ClamAV daemon and signature downloader service to start during
boot with following options appended to `/etc/rc.conf`.
```
clamav_clamd_enable="YES"
clamav_freshclam_enable="YES"
```

Finally start required services with following commands.
```
service kld start
service clamav-freshclam start
service clamav-daemon start
```

#### Mounting and unmounting ClamFS file systems

To mount ClamFS filesystem run ClamFS with configuration filename as a parameter.
```
clamfs /etc/clamfs/netshare.xml
```

To unmount ClamFS use `fusermount` with `-u` flag and
`<filesystem mountpoint="/net/share" />` value as a parameter.
```
sudo fusermount -u /net/share
```

## Fine tuning

### Starting without clamd available

A new “check” option was added to allow you to mount a ClamFS file system when
clamd is not available, such as during an early stage of the boot process.
To disable ClamAV Daemon (clamd) check on ClamFS startup set option check to
no:
```
<clamd socket="/var/run/clamav/clamd.ctl" check="no" />
```

### Mounting file systems from /etc/fstab

With `check=no` mounting ClamFS file systems form /etc/fstab is possible using
fuse mount helper (/sbin/mount.fuse). ClamFS will be started on boot with
configuration file defined here provided as its argument. Simple definition
of ClamFS mount point in /etc/fstab looks like:
```
clamfs#/etc/clamfs/share.xml  /clamfs/share  fuse  defaults  0  0
```

### Read-only mounts

The “readonly” option was added to the filesystem options allowing you to
create a read-only protected file system. Just extend filesystem definition
in config file with `readonly` option set to `yes`:
```
<filesystem root="/share" mountpoint="/clamfs/share" readonly="yes" />
```

### Program name reported as unknown when virus found

```
16:33:24 (clamav.cxx:152) (< unknown >:19690) (root:0) /tmp/eicar.com: Eicar-Test-Signature FOUND
```

To see program name instead of `< unknown >` in log messages on FreeBSD one
need to mount `/proc` filesystem. Add following line to `/etc/fstab`.
```
proc /proc procfs rw 0 0
```
And mount `/proc` with `mount /proc`.

Program name should be reported correctly with mounted `/proc`.
```
16:37:31 (clamav.cxx:152) (hexdump:19740) (root:0) /tmp/eicar.com: Eicar-Test-Signature FOUND
```

### Using ClamFS with WINE

Please refer to my blog post [Wine with on-access ClamAV scanning](https://blog.burghardt.pl/2007/11/wine-with-on-access-clamav-scanning/)
if you are interested in running ClamFS to protect WINE installation.

## License

This project is licensed under the GPLv2 License - see the
[COPYING](COPYING) file for details.

## Historical repositories at SourceForge

Long time ago [ClamFS](http://clamfs.sourceforge.net/) was developed on
[SourceForge](https://sourceforge.net/projects/clamfs/) and some CVS and
SVN repositories still resides there. Right now all development takes place
on GitHub.