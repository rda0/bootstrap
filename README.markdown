bootstrap-lv
============

Bootstrap a bootable system to an lvm filesystem.

Description
-----------

I created this script to bootstrap a bootable system into an lvm disk to be used as KVM virtual machine disk image.

If you are familiar with Xen, you probably came across this kind of setup. With Xen you can boot directly from a file-system without a partition table using the `pygrub` bootloader. This is very useful when it comes to resizing the logical volume, as you do not have to mess with partition tables and you can do online resizing of virtual disks.

Of course I wanted this for kvm too. Unfortunately, there is no such thing as `pygrub` for KVM. But you still have some options. With KVM you can directly boot using a Kernel from the host file-system. With `libvirt` you can even use a `qemu` hook script to extract the guest Kernel, store it on the hosts file-system and boot it. But after a Kernel upgrade in the guest, you need to stop the VM and start it up again on the host system. If you just reboot from within the guest, `libvirt` will not pickup the new Kernel. Of course I wanted that reboot.

There are two options to achieve this:

1. Using the `grub` bootloader, we can work around this, by creating a small boot disk. The disk needs a partition table for Grub to work properly. Grub is installed into the MBR of this disk. The first partition will contain the Kernel, initrd and Grub. The partition is mounted to `/boot`. The rest of the root file-system will be bootstrapped into a separate logical volume, which only contains a file-system. If you carefully size the boot lv, you probably never have to resize that disk. However, we are still not fully satisfied, because we still need two lvs (disks).

2. Using `extlinux` (`syslinux`) we can solve that problem. It does not need to be installed into an MBR and we can directly install it into our file-system. We end up with a single logical volume, containing only a file-system that is bootable by QEMU-KVM. This is the default method used in this script.

Requirements
------------

- `cdebootstrap`

Optional:

- `ubuntu-archive-keyring`
- `parted`
- `kpartx`

Usage
-----

```bash
Usage: ./bootstrap vm vg dist release [options]

    Bootstraps a bootable kvm vm to lvm filesystem.
    Default bootloader: extlinux

Options:

    vm
        Name of the vm (used for lvm naming)
    vg
        Volume group name
    dist
        Distribution name (example: `debian`, `ubuntu`)
    release
        Release name (example `stable`, `stretch`, etc)
    -r, --root <size>
        Size of root fs lv (default: `2G`)
    -b, --boot <size>
        Enable boot disk (and grub), size of boot disk lv (example: `512M`)
    -t, --fs-type <type>
        Type for root file system (default: `ext4`)
    -i, --package-install <package_list>
        List of extra packages to install (default: bootstrap-packages/<dist>/<release>/install)
    -u, --package-purge <package_list>
        List of packages to purge (default: bootstrap-packages/<dist>/<release>/purge)
    -n, --network-interface <interface_name>
        Network interface name (default: `eth0`)
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

Configuration
-------------

The network will be configured via DHCP and `sshd` will allow login as root using key based authentication only. The `authorized_keys` file from the host system will be copied into the guest system.

To set a root password to be able to log in using the virtual serial console, create the file `secrets/root_password_hash`, containing a shell variable `ROOT_PASSWORD_HASH` with the hashed root password:

```
ROOT_PASSWORD_HASH='$6$BiHI2LMs$B/L0VkdgRuo/ZTXCpmgA4a9rLXxqVG7R.F6/3L93kKdOPmm8C9nT5VJ/8LL7MxykhqJkGZpOHi8z47m1RAt231'
```

The example above is the default root password hash (password: `toor`) if the file is absent or the variable empty.
