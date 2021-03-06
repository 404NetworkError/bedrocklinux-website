Title: Bedrock Linux 1.0beta2 Nyla Syslinux Guide
Nav: nyla.nav

Syslinux Installation
=====================

Bedrock Linux itself is largely agnostic of which bootloader you choose to use.
Syslinux installation is documented here rather than another bootloader
primarily due to the fact it is relatively simple and easy to set up by hand.
If you would prefer a different bootloader, such as GRUB or LILO, and you are
confident you are able to set it up (with documentation from another source),
you are welcome to use another bootloader.  If you are dual booting with
another operating system which has its own bootloader and you know how to add
Bedrock Linux to the other operating system's bootloader, you are welcome to do
that as well.  If you are able to setup the system without a bootloader (e.g.
leveraging EFI), that too is fine.

Syslinux is both the name of a project which contains multiple bootloaders and
the name of one of the bootloaders within the project.  If you are using ext2,
ext3, ext4 or BTRFS as your filesystem, the "extlinux" bootloader from Syslinux
should suffice.  If you are using another filesystem, you can either look at another
member of the Syslinux family which supports your filesystem or find a completely
different bootloader. If you do choose to use a different syslinux bootloader,
the steps should be largely the same, replacing "extlinux" with the bootloader
of choice.

Download Syslinux from [the official syslinux
website](http://www.syslinux.org/).  Be sure to read the specific pages for the
given release in case there are any known issues.  You should probably at least
skim the contents of the doc/ directory in the tarball as well.  If you are
using BTRFS, you need at least version 4.0.

Note: These instructions were written for Syslinux 6.01.  6.02 has been found
to have issues `chain.c32`.  6.03 has been released since this document was
written, but it has not yet been tested.

Unpackage syslinux and change directories into the extlinux subdirectory in it:

- {class="cmd"}
- tar xvf syslinux-~(VERSION~).tar.gz
- cd syslinux-~(VERSION~)

Next, compile as a non-root user.  If you are making this for non-EFI systems
run:

- {class="cmd"}
- make clean; make bios

If you are making it for 32-bit EFI systems, run:

- {class="cmd"}
- make clean; make efi32

Or, if you are making it for 64-bit EFI systems, run:

- {class="cmd"}
- make clean; make efi64

Remember which choice you made here, as you will need that information later.

You may receive an error about lacking `nasm` or `uuid-dev` or some other
packages.  If so, install those packages and try again.

You should now have an `extlinux` executable somewhere.

Once it has successfully compiled, switch to a root user and set a new variable
indicating the location of the unpackaged source:

- {class="rcmd"}
- export SYSLINUX=~(/path/to/syslinux/source~)

If you do not have a `$ROOTFS` variable pointing to the Bedrock Linux mount
point from the installation instructions, make one:

- {class="rcmd"}
- export ROOTFS=~(/path/to/bedrock-linux/root~)

Make a directory to install extlinux into:

- {class="rcmd"}
- mkdir -p $ROOTFS/boot/extlinux


Find the `extlinux` executable:

- {class="rcmd"}
- find $SYSLINUX -name extlinux -type f

As root with the ROOTFS variable set, `cd` to the directory containing the
`extlinux` executable and then install it into Bedrock Linux:

- {class="rcmd"}
- ./extlinux --install $ROOTFS/boot/extlinux

Although syslinux is installed on the partition, it still needs to be properly
set up on the harddrive itself. Find the corresponding device file for device
you made bootable when you partitioned/formatted earlier. Note that here we
aren't looking for the partition device file - `/dev/sd~(XN~)` - but rather the
device's file - `/dev/sd~(X~)`. There should be no 0-9 digit in the filename. As
root:

Find the mbr.bin

- {class="rcmd"}
- find $SYSLINUX -name mbr.bin -type f

If there are multiple results - one for each bios, efi32 and efi64 - use the
one corresponding to which system you are using.

`cd` into the directory containing `mbr.bin`.  Copy it over the desired drive's
master boot record.  Be *very* careful here.

- {class="rcmd"}
- cat ./mbr.bin > /dev/sd~(X~) # change X accordingly

Syslinux will need additional support files.  You may read syslinux's
documentation for what each of these do; however, unless you are short on disk
space, it may be easiest to simply grab all of them.  Again, like with mbr.bin,
if there are multiple groups of results we want those corresponding to to your
earlier choice of bios vs efi32 vs efi64.  First, find the bios directory:

- {class="rcmd"}
- find $SYSLINUX -name bios -type d # or efi32 or efi64 as used earlier
- cd ~(output-of-above~)

Next, copy all of the files ending in `.c32` in this directory into the
extlinux install directory.

- {class="rcmd"}
- find . -name "\*.c32" -type f -exec cp {} $ROOTFS/boot/extlinux \;

Now you should create the configuration file for syslinux. Unlike GRUB,
syslinux must be configured by hand. Assuming you want a graphical menu, run
the following to get a basic menu in syslinux:

	{class="rcmd"} cat > $ROOTFS/boot/extlinux/extlinux.conf << EOF
	~(UI menu.c32~)
	MENU TITLE Syslinux Bootloader
	DEFAULT bedrock
	PROMPT 0
	TIMEOUT 50

	LABEL bedrock
		MENU LABEL Bedrock Linux 1.0beta2 Nyla
		LINUX ../~(vmlinuz-KERNEL-VERSION~)
		APPEND root=/dev/sd~(XN~) quiet rw init=/bedrock/sbin/brn
		~(INITRD ../initrd.img-VERSION~)
	EOF

Consider changing the following:

- Change the `/dev/sd~(XN~)` to the partition you set to boot - either
  Bedrock's main/root partition or its `/boot` partition if you made one.
- Change ~(vmlinuz-KERNEL-VERSION~) to the filename of a kernel image installed
  into `$ROOTFS/boot`.  If you don't have any yet, you should get at least one
  during the installation instructions - **once you've done that update this file**.
- Like the kernel image, you'll want to update ~(INITRD ../initrd.img-VERSION~)
  to the initrd you are using, if any.  If you don't have any yet you probably
  will acquire at least one during later installation instructions.  **Update
  this file at that time**.
- You may also change ~(menu.c32~) to ~(vesamenu.c32~) for a fancier menu or
  remove the `~(UI menu.c32~)` line completely for a command line interface.

If you would like to dual-boot with another operating system such as Windows,
append the following:

	{class="rcmd"} cat >> /mnt/bedrock/boot/extlinux/extlinux.conf << EOF
	LABEL ~(LABEL NAME~)
		MENU LABEL ~(Operating System Name~)
		COM32 chain.c32
		APPEND hd~(N N~)
	EOF

Consider changing the following from this addition:

- The ~(LABEL NAME~) is what Syslinux refers to this item - something short and
  simple such as "win7" should suffice.
- The ~(Operating System Name~) is what you will see for the item in the menu
  if you are using `menu.c32` or `vesamenu.c32`.  Something such as "Windows 7"
  should be fine.
- Finally, `hd~(N N~)` refers to the hard drive (the first number) and partition
  (the second number) on which the bootloader you are chainloading is
  installed.  It appears both numbers are zero-indexed, but if this does not
  work for you try with them either or both of them being one-indexed.  For
  example, if the target operating system's bootloader is on `/dev/sda1` (the
  first partition on the first harddrive), you will want `hd0 0`.

You should now have the syslinux bootloader installed.  If it does not seem to
work, consider installing a different syslinux release or trying out another
bootloader.
