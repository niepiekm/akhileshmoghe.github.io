---
title:  "A Demystifying Introduction to Initrd and Initramfs"
permalink: /_post/embedded/linux/initramfs-initrd
date:   2023-08-02 1:33:22 +0530
categories:
  - Linux
  - Embedded Linux
  - Systems Engineering
toc: true
toc_label: "Contents"
toc_icon: "file-alt"
toc_sticky : true
tags:
  - Linux
  - Embedded Linux
  - Systems Engineering
author: Akhilesh Moghe
show_author_profile: true
---

In Linux systems, __*<u>initrd</u>*__ (<u>initial ramdisk</u>) or __*<u>Initramfs</u>*__ are used for loading a temporary root file system into memory, to be further used as part of the Linux startup process.
Both are commonly used to make preparations before the real root file system can be mounted. The initramfs is a complete set of directories that you would find on a normal root filesystem. It is bundled into a single `cpio` archive and compressed with one of several compression algorithms.


## Why Initrd or Initramfs came in existance
- Many Linux distributions create a single, generic <u>Linux kernel image</u> to boot on a wide variety of hardware.
- The __*device drivers*__ for this generic kernel image are included as *<u>loadable kernel modules</u>* because statically compiling many drivers into one kernel causes the kernel image to be much larger, to boot on computers with limited memory, sometimes even results in boot crashes.
- Many a times, the root file system may be on a __LVM__, __NFS__ (on diskless workstations), or on an __encrypted partition__. All of these require special preparations to mount.
- To avoid having to hardcode handling for so many unique scenarios into the kernel, a temporary root file-system is utilized during the initial boot stage, known as __*<u>early user space</u>*__. In order to mount the actual root file-system, user-space tools that perform hardware detection, module loading, and device discovery can be found in this root file-system.
- So in a broader sense, the __*Initramfs*__ or __*Initrd*__ has essentially one purpose: <u>locating and mounting the real root file system</u> so that the boot process can transition to it.
- To elaborate in detail with example of __*Encrypted Partitions*__ mounting, some system configurations like using a `cryptodevices` requires a <u>user space utility to provoke the kernel to configure the devices appropriately</u>, as <u>they need to have a password from the user</u>. This password requesting utility being a user space utility, could pose <u>a chicken and egg problem</u> i.e your rootfs contains the user space utilities, but the rootfs cannot come up till the user space utilities are available. In such cases, the __*Initramfs*__ plays a mediator in between giving <u>a temporary rootfs</u> which has the user space utilities needed for mounting the real rootfs.

---

## Linux boot process with Initrd or Initramfs
- The __Image__ of this Initial ROOT File System (along with the kernel image) is required be stored somewhere accessible by the __*Linux bootloader*__ or the boot firmware of the computer.
- This __Image__ storage location can be the *<u>root file system</u>* itself, a *<u>boot image on an optical disc</u>*, on a *<u>small partition on a local disk</u>* (usually using __ext4__ or __FAT__ file systems), or a *<u>TFTP server</u>* (for systems that can boot from Ethernet).
- The __*Bootloader*__ will <u>start the kernel</u> by sending in the memory address of the image after loading the kernel and initial root file system image into memory.
  - __U-Boot__ Example: `bootz ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr}`
- The kernel attempts to determine the image's format from its initial few data blocks at the end of the boot process, which can either be the Initrd or Initramfs.\
&nbsp;
- <u>In case of</u> __*<u>Initrd</u>*__:
  - The initrd image may be a file system image (optionally compressed), which is made available in a special __*block device (/dev/ram)*__ which is then mounted as the initial root file system.
  - :pencil: The *device driver for this file system must be compiled statically into the kernel*.
  - Many distributions use <u>compressed </u>__<u>ext2</u>__<u> file system images</u>, while the __Debian__ uses [cramfs](/_post/linux/file_systems_types#cramfs) (Compressed ROM/RAM file system) in order to boot on memory constrained systems, since the __<u>cramfs</u>__<u> image can be mounted in-place without requiring extra space for decompression</u>.
  - Once the initial root file system is up, the <u>kernel executes</u> [`/linuxrc`](https://www.novell.com/documentation/suse91/suselinux-adminguide/html/ch12s04.html) <u>as its first process</u>.
    - `/linuxrc` must be located in the root directory of the Initrd and needs to be executable which is run with root permissions by the kernel.
    - In case `/linuxrc` is __dynamically linked__, all required shared libraries from `/lib` must be available in Initrd also.
    - `/linuxrc` can also be a __shell script__, in that case a shell must exist in `/bin`.
    - In [SUSE Linux](https://www.suse.com/products/), a __statically-linked__ `/linuxrc` is used to keep initrd as small as possible.
  - When `/linuxrc` process exits, the <u>kernel assumes that the real root file system has been mounted and executes</u> `/sbin/init` to begin the normal user-space boot process.
  - On an __*Initrd*__, the new final `root` is mounted at a temporary mount point and rotated into place with [`pivot_root(8)`](https://man7.org/linux/man-pages/man8/pivot_root.8.html) (to change the root filesystem). This leaves the *<u>Initial root file system at a mount point</u>* (such as `/initrd`) where normal *<u>boot scripts can later unmount it</u>* to free up memory held by the __*Initrd*__.\
&nbsp;
- <u>For</u> __*<u>Initramfs</u>*__:
  - __*Initramfs*__ is available since the *<u>Linux kernel 2.6.13</u>*.
  - The __*Initramfs*__ image may be a [cpio archive](<TBD>) (optionally compressed).
  - The archive is unpacked by the kernel into a special instance of a [tmpfs](/_post/linux/file_systems_types#tmpfs) that becomes the initial root file system.
  - :pencil: The __*Initramfs*__ has the advantage over the __*Initrd*__ of <u>not requiring an</u> __<u>intermediate file system</u>__ or __<u>block device drivers</u>__ <u>to be compiled into the kernel</u>.
  - <u>Red Hat Linux</u> distribution uses the [dracut](https://en.wikipedia.org/wiki/Dracut_(initramfs)) package to create an initramfs image.
  - With __*Initramfs*__, the kernel executes `/init` as its *<u>first process that is not expected to exit</u>*. The `/init` program is typically a <u>shell script</u>.
  - For some applications like *<u>Ubuntu live cds</u>*, __*Initramfs*__ can use the [casper](https://manpages.ubuntu.com/manpages/focal/man7/casper.7.html) utility to create a writable environment using [unionfs](/_post/linux/file_systems_types#unionfs) *<u>to overlay</u>*__*<u> a persistence layer</u>*__*<u> over a</u>*__*<u> read-only</u>*__*<u> root filesystem</u>* image. For example, <u>overlay data can be stored on a USB flash drive</u>, while a compressed [SquashFS](/_post/linux/file_systems_types#squashfs) read-only image stored on a live CD acts as a root filesystem.
  - :pencil: On an __*Initramfs*__, the *<u>initial root file system cannot be rotated away</u>* like that on __Initrd__ using `pivot_root(8)`. Instead, Initramfs is simply emptied and the *<u>final root file system mounted over the top</u>*.\
&nbsp;
- kernel can unpack Initrd or Initramfs images compressed with gzip, bzip2, LZMA, XZ, LZO, LZ4 and zstd.
- __*Initrd*__ and __*Initramfs*__ implement `/linuxrc` or `/init` <u>as a shell script</u>.
- Both include a <u>minimal shell</u> (usually `/bin/ash`) along with some <u>essential user-space utilities</u> (usually [BusyBox](https://en.wikipedia.org/wiki/BusyBox) toolkit).
- To further save space, the <u>shell, utilities and their supporting libraries are typically compiled with</u> *<u>space optimizations enabled</u>* (such as `gcc -O` flag) and <u>linked against</u> [klibc](https://en.wikipedia.org/wiki/Klibc), <u>a minimal version of the</u> __*<u>libc</u>*__ library written specifically for this purpose.

---

## Few more details about:

### <u>Initial RAM Disk (Initrd)</u>:
- __*Initrd*__ provides the capability to load a __RAM disk__ Image and <u>mount it as the Root File System</u> by the Bootloader. Then user-space programs can be run from it.
- Afterwards, a new larger Root File System can be mounted from a different device. The *previous root (from initrd) is then moved to a directory and can be subsequently unmounted*.
- __*Initrd*__ is mainly designed to allow system startup to occur in two phases, where the kernel comes up with a minimum set of compiled-in drivers, and where additional modules are loaded from initrd.\
&nbsp;
- __IMPORTANT__: __*<u>Initrd mechanism has been deprecated and the support for it is removed from Linux Kernel from 2021.</u>*__
  - __*Initramfs*__ had been introduced since __*Linux Kernel 2.6*__.
  - It is the recommended way for implementing initial root file system all modern Linux distributions.
  - Even if you see files named as `initrd-$(uname -r)` in `/boot` folders on modern Linux distributions like Ubuntu/Debian, keep in mind that this is not an initrd but actually an __*Initramfs*__ only. That means, it can be a concatenation of multiple [cpio]() archives, each of which may or may not be compressed.
  - References:
    - [[PATCH 14/23] initrd: mark initrd support as deprecated](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2233909.html)
    - [Re: using deprecated initrd support, will be removed in 2021](https://lkml.org/lkml/2021/3/4/173)

---

### <u>Init RAM File System (Initramfs)</u>:
- __Linux Kernel version 2.6__ onwards, contain a [gzipped](/_post/embedded/linux/initramfs-cpio#gzip-command) [cpio](/_post/embedded/linux/initramfs-cpio#cpio-command) format archive, which is extracted into ROOTFS when the kernel boots up.
- An __*Initramfs*__ archive is a <u>complete self-contained ROOTFS for Linux</u>.
- The __Linux Kernel version 2.6__ build process *<u>always creates a gzipped cpio format initramfs
archive and links it into the resulting kernel binary</u>*. By default, <u>this archive is empty (consuming 134 bytes on x86)</u>.
- After extracting, the kernel checks to see if ROOTFS contains a file `/init`, and if so it executes it as `PID=1`. This `init` process is responsible for bringing rest of the system up, including locating and <u>mounting the real root device</u> (if any).
- If ROOTFS *<u>does not contain</u>* an `init` program after the embedded `cpio` archive is extracted into it, the <u>kernel will try to locate and mount a ROOT partition</u>, then `exec` some variant of `/sbin/init` out of that.
- __Using *External initramfs images*__:
  - If the kernel has __*<u>initrd </u>*__ <u>support enabled</u>, an external __*cpio.gz archive*__ can also be passed into a v2.6 kernel in place of an Initrd. The <u>kernel will autodetect the type as</u> __*Initramfs*__, and <u>extract the external </u>__*<u>cpio archive</u>*__<u> as a ROOTFS before trying to run</u> `/init`.
  - This approach of separately packaging of __*Initramfs*__ has an *<u>Advantage</u>* that, you can run a <u>non-GPL code from within</u> __*<u>Initramfs</u>*__,<u> without conflating it with the GPL licensed Linux kernel binary</u>.
  - External __*Initramfs*__ can supplement the kernel's built-in initramfs image. The external archive will overwrite any conflicting files in the built-in initramfs archive.
- __*Initramfs*__ usually contains either [klibc](https://en.wikipedia.org/wiki/Klibc) or [uClibc](https://en.wikipedia.org/wiki/UClibc) which are C libraries designed to statically link early userspace code against, along with some related utilities. Also contains [BusyBox](https://en.wikipedia.org/wiki/BusyBox).

---

## Why Kernel replaced <u>Initrd</u> with <u>Initramfs</u>
- __RAMdisk vs RAMfs__
  - A __RAMdisk__ (like __*Initrd*__) is a *<u>RAM based </u>*__*<u>Block device</u>*__, which means it's a <u>fixed size of memory</u> that can be <u>formatted and mounted like a disk</u>.
  - The contents of the __RAMdisk__ have to be formatted with [`mke2fs`](/_post/linux/disk_img_creation_process#mkfs) and mounted with [`losetup`](/_post/linux/disk_img_creation_process#losetup) tools, and like all block devices it requires a <u>filesystem driver</u> to interpret the data at runtime.
  - Normally these tools are kept on a file system as user-space program, but as there's no real file system mounted at this boot stage, <u>these tools and drivers are required to be compiled in the Kernel itself</u>.
  - __*Fixed size*__ requirement of __RAMdisk__ puts a limitation on the size, that will either wastes the excess memory or will not be adequate for the purpose. You *<u>can't expand or shrink it without formatting again</u>*.
  - __RAMdisk__ also *<u>wastes memory as it gets cached</u>*, as Linux caches all the files read from or written to block devices. This is the downside of the __RAMdisk__ being treated as a block device.
  - Whereas, __*Initramfs*__ is an instance of [tmpfs](/_post/linux/file_systems_types#tmpfs), which *<u>automatically grow or shrink to fit the size of the data</u>* they contain. Adding files to a __*Initramfs*__ (or extending existing files) automatically allocates more memory, and deleting or truncating files frees that memory.
  - With __*Initramfs*__, as there's *<u>no block device</u>*, there's *<u>no duplication of data between block device and cache</u>*.
  - __*Initramfs*__ doesn't need any file system driver built in the Kernel.\
&nbsp;
- <u>Few other design limitations of </u>__*<u>Initrd</u>*__<u> overcame by </u>__*<u>Initramfs</u>*__:
  - The `/linuxrc` <u>tries to determine the several available devices for real ROOTFS</u> before returning the identified device number to the Kernel so the kernel could mount the real root device and execute the real init program.
  - __*Initrd*__ assums that real *<u>ROOTFS will always be a Block device</u>*, rather than a network share.
  - Also __*Initrd*__ was never considered to be the real ROOTFS for memory constrained embedded devices.
  - In __*Initrd*__, `/linuxrc` <u>is not run like an</u> `init` <u>program with PID=1</u>, which actually provides special properties reserved for `init` like it cannot be killed with `kill -9` command.
  - In __*Initramfs*__, the Kernel doesn't care where the real ROOTFS is (it's __*Initramfs*__ until `init` program from real ROOTFS is executed).
  - In __*Initramfs*__, the `init` program of __*Initramfs*__ is always run as a real `init`, with PID=1.
  - When __*Initramfs*__ `init` needs to hand that special Process ID off to another program, it can use the `exec()` syscall just like everybody else.

---

## Use cases for Initramfs or Initrd
### 1. *<u>Minimizing Linux Kernel size</u>*
- The idea here is that there's a lot of initialization tasks done in the __*Linux Kernel*__ that could be just as easily done in userspace.
- Many modules required for mounting different typesf of ROOTFS like __NFS__, __LVM__, etc can be offloaded to __*Initramfs*__ or __*Initrd*__ instead of compiling all of them into the kernel, as not all of these modules will be required for every type of ROOTFS mounting.

### 2. *<u>Mounting Real ROOTFS</u>*
- Some Linux distributions like __*Debian*__ generates a customized __*Initrd*__ image which contains only necessary stuffs to boot some particular computer, such as [ATA](https://en.wikipedia.org/wiki/Advanced_Technology_Attachment), [SCSI](https://en.wikipedia.org/wiki/SCSI) and <u>filesystem kernel modules</u>. Such images typically embed the location and type of the root file system.
- Other Linux distributions like __*Ubuntu*__ generates a generic __*Initrd*__ image. These images start only with the device name of the root file system (or its UUID) and discovers everything else at boot time. In this case, the software must perform a complex cascade of tasks to get the root file system mounted. Few examples of what else would be required to be done to boot the ROOTFS are:
  - Load all storage device drivers that are required by the boot process. For this, kernel modules of common storage devices can be added to __*Initrd*__ image and then an event-driven hotplug agent like [udev](https://en.wikipedia.org/wiki/Udev) can be used to load the modules matching the computer's detected hardware.
  - If *<u>Boot Splash Screen</u>* is required, the video hardware must be initialized along with user-space utilities to paint the splash screen animation during the boot process.
  - If the __NFS__ needs to be accessed, __*Initrd*__ must bring up the primary network interface, invoke a <u>DHCP client</u>, to obtain a DHCP lease, extract the name of the NFS shared location and the address of the NFS server from the lease, and mount the NFS shared location.
  - If root file system is on a __logical volume__, the __LVM utilities__ must be invoked to scan for the activate the volume group containing it.
  - If root file system is on an __encrypted block device__, the __*Initrd*__ needs to invoke a utility script to prompt the user to type in a passphrase and/or insert a hardware token (such as a smart card or a USB security dongle), and then create a decryption target with the device mapper.

### 3. *<u>Maintenance Tasks on ROOTFS before mounting</u>*
- When the Root file system finally becomes visible, *<u>any maintenance tasks that cannot be ran on a mounted Root file system are required to be done</u>*, the Root file system is mounted *<u>read-only</u>*, and any processes that must continue running (such as the splash screen utility) are hoisted into the newly mounted root file system.

### 4. *<u>Recovery disks creation</u>*
- __*Initrd*__ or __*Initramfs*__ can be used to create a *<u>Recovery Disk Image</u>* or *<u>Backup a Partition</u>*.
- Because with __*Initrd*__ or __*Initramfs*__ mechanism, the real ROOTFS is not required to be loaded, so the backup of the ROOTFS can be created and saved to some external device like USB/CD-ROM.
- The system loaded from initrd can invoke a user-friendly dialog or it can also perform some form of auto-detection for available backup media.

### 5. *<u>Linux Installers</u>*
- Linux distribution Installers typically <u>run entirely from an</u> __*Initramfs*__, as they must be able to host the installer interface and supporting tools before any persistent storage has been set up.
- CD-ROM distributors may use __*Initrd*__ for better installation from CD, by bootstrapping a bigger RAM disk via initrd from CD; or by booting via a bootloader like [loadlin](https://en.wikipedia.org/wiki/Loadlin) or directly from the CD-ROM.
- [Tiny Core Linux (TCL)](https://en.wikipedia.org/wiki/Tiny_Core_Linux) and [Puppy Linux](https://en.wikipedia.org/wiki/Puppy_Linux) can run entirely from __*Initrd*__.


---

## References
- [initrd](https://tldp.org/LDP/Linux-Filesystem-Hierarchy/html/initrd.html)
- [Introducing initramfs, a new model for initial RAM disks](https://archive.ph/20130104033427/http://www.linuxfordevices.com/c/a/Linux-For-Devices-Articles/Introducing-initramfs-a-new-model-for-initial-RAM-disks/#selection-254.1-267.56)
- [Kernel.org: ramfs-rootfs-initramfs](https://www.kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt)

