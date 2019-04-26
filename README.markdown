bootstrap
=========

Bootstrap a bootable system to a file system on a logical volume or file-based image.

Description
-----------

This script creates a root file system directly on a partitionless disk and installs a Debian or Ubuntu base system bootable by QEMU-KVM virtual machines.

If you are familiar with Xen, you probably came across this kind of setup. With Xen you can boot directly from a file-system without a partition table using the `pygrub` bootloader. This is very useful when you need to resize the disk images, as you do not have to alter partition tables and you can do online resizing of virtual disks without shutting down the guest system.

I wanted this for kvm too. Unfortunately, there is no such thing as `pygrub` for KVM. But you still have some options. With KVM you can directly boot using a Kernel from the host file-system. With `libvirt` you can even use a `qemu` hook script to extract the guest Kernel, store it on the hosts file-system and boot it. But after a Kernel upgrade in the guest, you need to stop the VM and start it up again on the host system. If you just reboot from within the guest, `libvirt` will not pickup the new Kernel.

There are two options to also allow reboot from within the guest after a Kernel upgrade:

1. Using the `grub` bootloader, we can work around this, by creating a small boot disk. The disk needs a partition table for Grub to work properly. Grub is installed into the MBR of this disk. The first partition will contain the Kernel, initrd and Grub. The partition is mounted to `/boot`. The rest of the root file-system will be bootstrapped into a separate logical volume or image file, which only contains a file-system. If you carefully size the boot disk, you should not need to resize it again. However, this is still not fully satisfying because at least two disks are needed.

2. Using `extlinux` (`syslinux`) we can solve that problem. It does not need to be installed into an MBR and can be installed on a disk image file or logical volume without a partition table. We end up with a single disk containing a file-system only, which is bootable by QEMU-KVM. This is the default method used in this script.

Requirements
------------

- `debootstrap`

Optional:

- `ubuntu-archive-keyring`
- `parted`
- `kpartx`
- `fallocate`
- `cdebootstrap`

Installation
------------

```bash
git clone https://github.com/rda0/bootstrap.git /opt/bootstrap
ln -s /opt/bootstrap/bootstrap /usr/local/bin/bootstrap
```

Usage
-----

```bash
Usage: bootstrap name pool dist release [options]

    Bootstraps a bootable filesystem to logical volume or file-based image.
    Default bootloader: extlinux

Options:

    name
        Name of the lv or file to be created (`-root` or `-boot` will be appended)
    pool
        Volume group name or file-based image base-path (when `--file` is used)
    dist
        Distribution name (example: `debian`, `ubuntu`)
    release
        Release name (example `stable`, `stretch`, etc)
    -r, --root <size>
        Size of root fs image (default: `2G`)
    -b, --boot <size>
        Enable boot disk (and grub), size of boot disk (example: `512M`)
    -t, --fs-type <type>
        File system type (default: `ext4`)
    --boot-fs-type <type>
        Boot disk file system type (defaults to the value of `--fs-type`)
    -o, --fs-mount-options <options>
        Mount options set in `/etc/fstab` (default: `noatime,nodiratime`)
    --fs-options <options>
        Options passed to `mkfs.<fs-type>` (default: use system defaults)
    --include <packages>
        Comma separated list of packages to include in debootstrap
        (default: `locales,tzdata,python-minimal,python-apt,acpi-support,dbus,ssh,bash-completion,vim,haveged`)
    --exclude <packages>
        Comma separated list of packages to exclude in debootstrap
        (default: `editor,debconf-2.0`)
    -i, --package-install <package_list>
        List of extra packages to install (default: packages/<dist>/<release>/install)
    -u, --package-purge <package_list>
        List of packages to purge (default: packages/<dist>/<release>/purge)
    -n, --network-interface <interface_name>
        Network interface name (default: `eth0`)
    -m, --mirror <mirror_url>
        Mirror to use (default: `http://debian.ethz.ch`)
    -a, --mount-target <path>
        Target mount point (default: `/mnt/bootstrap`)
    -z, --time-zone <zoneinfo>
        Timezone for local time (default: `Europe/Zurich`)
    -y, --yes
        Run non-interactively and answer questions with yes
    -c, --cdebootstrap
        Use cdebootstrap instead of debootstrap
    -f, --file
        Bootstrap to file-based image
    -h, --help
        Print this help message
```

For example, to bootstrap a Debian Stretch using the `extlinux` bootloader:

```bash
./bootstrap debian-stretch-lv vg0 debian stretch
```

This will create the following logical volume containing a bootable system:

```
/dev/vg0/debian-stretch-lv-root
```

To create the same using the `grub` bootloader and a 512M boot disk:

```bash
./bootstrap debian-stretch-lv vg0 debian stretch -b 512M
```

This will create the following logical volumes:

```
/dev/vg0/debian-stretch-lv-boot
/dev/vg0/debian-stretch-lv-root
```

To create a file-based image instead of a logical volume add the `-f` option and specify the base-path:

```bash
./bootstrap debian-buster-img /var/opt/img debian buster -f
```

This will create the following file-based image:

```
/var/opt/img/debian-buster-img-root
```

Configuration
-------------

The network will be configured via DHCP and `sshd` will allow login as root using key based authentication only. The `authorized_keys` file from the host system will be copied into the guest system.

To set a root password to be able to log in using the virtual serial console, create the file `secrets/root_password_hash`, containing a shell variable `ROOT_PASSWORD_HASH` with the hashed root password:

```
ROOT_PASSWORD_HASH='$6$BiHI2LMs$B/L0VkdgRuo/ZTXCpmgA4a9rLXxqVG7R.F6/3L93kKdOPmm8C9nT5VJ/8LL7MxykhqJkGZpOHi8z47m1RAt231'
```

The example above is the default root password hash (password: `toor`) if the file is absent or the variable empty.

Create package lists
--------------------

To bootstrap a system using standard packages, you can create 3 files corresponding to the apt package priorities. Run the following `aptitude` commands on a running system with the target release to generate the list of packages:

```
aptitude search --display-format "%p" '~prequired' > required
aptitude search --display-format "%p" '~pimportant' > important
aptitude search --display-format "%p" '~pstandard' > standard
```

Place the files (`required`, `important`, `standard`) in the following location:

```
packages/<dist>/<release>/
```

To install some extra packages, write a list of packages in:

```
packages/<dist>/<release>/install
```

To purge some unwanted packages, write a list of packages in:

```
packages/<dist>/<release>/purge
```

The default is to bootstrap a minimal base-system (`debootstrap`) with no additional packages except:

- kernel metapackage
- bootloader (`extlinux` or `grub2`)
- `locales`
- `tzdata`

The following extra packages should be installed by default (mainly to support immediate deployment of spawned VMs using Ansible). This is the default for the current releases in `packages/`:

- `python-minimal`: Required for Ansible
- `python-apt`: Required for Ansible
- `acpi-support`: Required for reboot from host-system
- `dbus`: Required for reboot from host-system
- `ssh`: Required for ssh login (Ansible)
- `bash-completion`: Optional
- `vim`: Optional
- `haveged`: Required to allow instantaneous ssh connection without long delays (userspace entropy daemon)
