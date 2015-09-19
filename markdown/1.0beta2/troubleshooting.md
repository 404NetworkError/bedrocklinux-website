Title: Bedrock Linux 1.0beta2 Nyla Tips, Tricks and Troubleshooting
Nav: nyla.nav

Bedrock Linux 1.0beta2 Nyla Tips, Tricks and Troubleshooting
============================================================

This page contains tips, tricks and troubleshooting advice for various software
and ~{strata~} for Bedrock Linux 1.0beta2 Nyla.

- [Tips](#tips)
	- [Stratum Aliases](#stratum-aliases)
- [General issues](#general)
	- [Proprietary Nvidia Drivers](#proprietary-nvidia-drivers)
	- [/dev/fd errors](#dev-fd-errors)
	- [No keyboard or mouse in xorg](#no-kbd-mouse)
	- [root/sudo path issues](#root-path)
	- ["grantpt failed" error](#grantpt-failed)
	- [time issues](#time)
- [stratum specific issues](#stratum-specific)
	- [Debian-based Linux distributions](#debian-based)
		- [Ubuntu/Upstart fix](#upstart-fix)
		- [Locale packages](#locale)
		- [Statoverride](#statoverride)
		- [Ubuntu resolv.conf](#ubuntu-resolvconf)
	- [Arch Linux](#arch)
		- [Pacman Filesystem Errors](#pacman-filesystem-errors)
	- [Gentoo Linux](#gentoo)
		- [/var/tmp out of space](#portage-out-of-space)
	- [Fedora](#fedora)
		- [Problems with using yum.](#yum-problems)
	- [CRUX](#crux)
		- [Slow boot](#crux-slow-boot)
		- [Freeze on shutdown](#crux-shutdown-freeze)

## {id="tips"} Tips

### {id="stratum-aliases"} Stratum Aliases

Rather than typing `brc ~(stratum~)`, one can shave some keystroke by generating
aliases for all of the ~{strata~}, like so:

	for STRATUM in $(bri -l | grep -v '^local$')
	do
		alias $STRATUM="brc $STRATUM"
		alias s$STRATUM="sudo brc $STRATUM"
	done

Consider placing that loop, or something similar, in your shell's rc file.

## {id="general"} General issues

### {id="proprietary-nvidia-drivers"} Proprietary Nvidia Drivers

The official Nvidia proprietary drivers works well in Bedrock Linux if set up
properly.

Note, the proprietary nvidia drivers are functionally two components: the
userland component and the kernel module.  The goal is to get the kernel module
in `/lib/modules` so it can be utilized by the rest of the system and to get
the userland component into (1) the ~{stratum~} that provides xorg and (2)
other ~{strata~} which you would like to have graphics acceleration.  Finally,
note that mixing nvidia driver version probably isn't a good idea; it may be
best to stick with a single version everywhere.

First, download the appropriate release of the nVidia Linux drivers as can be
found [here](http://www.nvidia.com/object/unix.html).  Keep it somewhere that
will continue across reboots, as you may reboot soon.

Note that nvidia's proprietary drivers do not play nicely with the nouveau
drivers, and so the nouveau drivers must be disabled.  Create or append to the
file at `/etc/modprobe.d/blacklist` the following:

	blacklist nouveau

If nouveau is currently loaded, it will have to be removed.  If you have
difficulty `{class="rcmd"} rmmod`'ing it because it is in use, reboot.  If it
appears your initrd is loading it, add "rdblacklist=nouveau" to your
bootloader's kernel line.

Next, the proprietary driver module.  In the ~{stratum~} that provides the
kernel (so the versions match), install the proprietary nvidia driver module by
doing one of the following:

- Using the official proprietary nvidia driver with the `-K` option to install
  only the kernel.
- Using the official proprietary nvidia driver with*out* the
  `--no-kernel-module` option so that it installs both the userland and,
  importantly, the kernel module.

Finally, install the userland component in all of the ~{strata~} which you
would like to have acceleration in xorg.  For each of these ~{strata~}run the
nVidia proprietary driver installer with the `--no-kernel-module` option.  If
you have a 32-bit ~{stratum~}, on a 64 bit system, you can use the x86 nvidia
driver prefixed with "linux32" so it doesn't complain about being on a 64 bit
system.  If you are installing this into a ~{stratum~} while the system is
already running xorg, as long as the ~{stratum~} in which you are installing
these drivers is not the one providing xorg you can probably get away with
using the `--no-x-check` flag.

You are then free to start and use xorg with GPU acceleration.

### {id="dev-fd-errors"} /dev/fd errors

If you receive errors along these lines:

	/dev/fd/~(N~): No such file or directory

where ~(N~) is a number, this is most likely due to the fact that the device
manager you are using is not setting up `/dev/fd` as some programs expect.
This can be solved (for the current session) by running:

- {class="rcmd"}
- rm -r /dev/fd
- ln -s /proc/self/fd /dev

To solve this permanently, one could simply add those two lines to
`/etc/rc.local`.

### {id="no-kbd-mouse"} No keyboard or mouse in xorg

If you run `startx` and do not have a keyboard or mouse:

- First, don't panic about your system being hard locked.  You can regain
  keyboard control and go to a tty by hitting alt-sysrq-r followed by the keys
  to go to the tty (such as ctrl-alt-F1).  Read up about
  [magic sysrq on linux](http://en.wikipedia.org/wiki/Magic_SysRq_key) if
  you're not familiar with it.

- Try using `udev` if you aren't already.  Boot with a ~{stratum~} from some
  distro that defaults to starting `udev` at boot - most major ones do.

- Ensure you have the relevant keyboard and mouse packages installed.  On
  Debian-based systems, these would be `xserver-xorg-input-kbd` and
  `xserver-xorg-input-mouse`.

- While this should not be necessary if you are using udev, you may want to
  consider setting `AutoAddDevices` and `AllowEmptyInput` to `False` in your
  `xorg.conf` file.  If this file already exists, it is probably at
  `/etc/X11/xorg.conf` in the ~{stratum~} that provides `startx`; otherwise, you'll
  have to create it.  Try adding the following to the relevant `xorg.conf` file
  and starting the xserver:

		Section "ServerFlags"
			Option "AutoAddDevices" "False"
			Option "AllowEmptyInput" "False"
		EndSection

### {id="root-path"} Root sometimes loses PATH items

There are two common ways to switch from a normal user to root, both of which
can potentially change your `$PATH` away from what is desired.  To see the proper
path for the root user, login directly to a tty and run `{class="rcmd"} echo $PATH`.

If you use sudo, make sure you have a "secure\_path" line in `/etc/sudoers` which includes the entire root PATH, such as:

`Defaults secure_path="/bedrock/bin:/bedrock/sbin:/bedrock/brpath/pin/bin:/bedrock/brpath/pin/sbin:/usr/local/bin:/opt/bin:/usr/bin:/bin:/usr/local/sbin:/opt/sbin:/usr/sbin:/sbin:/bedrock/brpath/bin:/bedrock/brpath/sbin"`

If you use su *without the -l flag*, consider changing the relevant lines in `/etc/login.defs` to the following:

`ENV_SUPATH PATH=/bedrock/bin:/bedrock/sbin:/bedrock/brpath/pin/bin:/bedrock/brpath/pin/sbin:/usr/local/bin:/opt/bin:/usr/bin:/bin:/usr/local/sbin:/opt/sbin:/usr/sbin:/sbin:/bedrock/brpath/bin:/bedrock/brpath/sbin`

`ENV_PATH PATH=/bedrock/bin:/bedrock/brpath/pin/bin:/usr/local/bin:/opt/bin:/usr/bin:/bin:/bedrock/brpath/bin:`

### {id="grantpt-failed"} "grantpt failed" error

If you see "grantpt failed" errors, such as when starting a terminal emulator
like `xfce4-terminal`, this can be remedied by remounting `/dev/pts` to set the
appropriate group number.

First, look at `/etc/group` and find the number corresponding with the group
"tty".  Next, add the following to `/etc/rc.local`:

	mount -o remount,gid=~(tty-gid-number~) /dev/pts

and the "grantpt failed" error no longer persist in the next reboot.  You can
also apply that command to fix the issue for the current session.

### {id="time"} time issues

Some distros unmount filesystems before writing the system clock to the
hardware clock.  This means the global adjtime file is not available, which in
turn means information such as clock drift and whether the hardware clock is in
local vs UTC time is not being utilized properly.

To resolve this, you need to manually write to the hardware clock after
updating the system clock, or configure your init to do so at very early
shutdown before filesystems are unmounted.

For example, after updating the system clock with

- {class="rcmd"}
- ntpdate ~(ntp-domain~)

Write to the hardware clock with

- {class="rcmd"}
- hwclock -w

## {id="stratum-specific"} stratum specific issues

### {id="debian-based"} Debian-based Linux distributions

#### {id="upstart-fix"} Ubuntu/Upstart fix

Older releases of Ubuntu uses Upstart for its init system. Many services in
Ubuntu have been modified to depend on `init` to be specific to Upstart and
refuse to operate otherwise. This means they do not work in chroots out of the
box. See the
[here](https://bugs.launchpad.net/ubuntu/+source/upstart/+bug/430224) for more
information. One way to alleviate this is to run the following two commands as
root (within the Ubuntu ~{stratum~}, via using `brc` for each command or `brc`
to open a shell in the ~{stratum~} and run it from the shell):

- {class="rcmd"}
- dpkg-divert --local --rename --add /sbin/initctl
- ln -s /bin/true /sbin/initctl

#### {id="locale"} Locale packages

In Debian, if you get errors about locale, try installing the `locales-all`
package.

In Ubuntu, if you get errors about locale, try installing the appropriate
`language-pack-~(LANGUAGE~)` (such as `language-pack-en`) package.

#### {id="statoverride"} Statoverride

If you get an error about statoverride when using apt/dpkg, it can most likely
be resolved by deleting the contents of `/var/lib/dpkg/statoverride` in the
relevant ~{stratum~}.  For example:

- {class="rcmd"}
- printf "" > /bedrock/strata/jessie/var/lib/dpkg/statoverride

#### {id="ubuntu-resolvconf"} Ubuntu resolv.conf

If you have difficulty sharing `/etc/resolv.conf` in Ubuntu, note that it creates
a symlink for that file directing elsewhere. It should be safe to remove the
symlink and just create an empty file in its place

### {id="arch"} Arch Linux

#### {id="pacman-filesystem-errors"} Pacman Filesystem Errors

If you get errors about `could not get filesystem information for ~(PATH~)`
when using `pacman`, this is normal and mostly harmless so long as you have
sufficient free disk space for the operation you are attempting. This seems to
be caused by `pacman` assuming that the mount points it sees are the same as the
ones init sees (which would be a fair assumption in almost every case except
Bedrock Linux). You can configure `pacman` to not check for free disk space by
commenting out `CheckSpace` from `/bedrock/strata/~(arch~)/etc/pacman.conf`

### {id="fedora"} Fedora

#### {id="yum-problems"} Problems with using yum.

Febootstrap does not seem to always include the `fedora-release` package. This is
troublesome, as the package is utilized to access the Fedora repositories. If you
find difficulties using `yum`, you might be able to resolve this by downloading
the `fedora-release` package for the given release (e.g.:
`fedora-release-17.noarch.rpm`), and install it thusly (from within the Fedora
~{stratum~}, via `brc`):

- {class="rcmd"}
- rpm -i fedora-~(VERSION~).noarch.rpm

You should then be able to use `yum` to access Fedora's repositories as one
normally would.

### {id="crux"} CRUX

#### {id="crux-slow-boot"} Slow boot

CRUX runs `depmod` on boot which can take a while.  It is not strictly needed
every boot.  To disable this and speed up boot time a bit, edit
`/bedrock/strata/~(crux~)/etc/rc.conf` and change

	if [ -x /etc/rc.modules ]; then
		/etc/rc.modules
	fi

to

	#if [ -x /etc/rc.modules ]; then
	#	 /etc/rc.modules
	#fi

#### {id="crux-shutdown-freeze"} CRUX shutdown freeze

CRUX runs `/etc/rc.shutdown` on shutdown/reboot.  When the `bru` filesystem
mounted at `/etc` - and thus under `/etc/rc.shutdown` - is killed during
shutdown, the shutdown procedure freezes.  A work-around for this issue is to
move `rc.shutdown` elsewhere so `/etc/` is not ripped out from under it.  As
root:

- {class="rcmd"}
- mv /bedrock/strata/~(crux~)/etc/rc.shutdown /bedrock/strata/~(crux~)/rc.local
- ln -s ../rc.local /bedrock/strata/~(crux~)/etc/rc.local
