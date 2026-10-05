# mkdebinitrd

Builds a Debian initramfs from scratch. Busybox is PID 1, and the image is a
real Debian system: `dpkg` and `apt-get` work, so you can install packages at
runtime. The whole system lives in RAM.

```sh
sudo ./mkdebinitrd amd64                      # -> initramfs-trixie-amd64.img + .vmlinuz
sudo ./mkdebinitrd aarch64 -o out/arm64.img   # uname -m spellings work too
sudo ./mkdebinitrd arm64 -i openssh-server,vim-tiny --root-password secret
```

Boot the two outputs together, for example
`qemu-system-x86_64 -m 2G -kernel initramfs-trixie-amd64.vmlinuz -initrd initramfs-trixie-amd64.img -append console=ttyS0 -nographic -nic user`,
or with PXE or iPXE. You don't need a `root=` option.

## What it does

1. Downloads `InRelease` and the `Packages` index, then checks the index's SHA256.
   If a keyring is available (`--keyring`, or the host's
   `/usr/share/keyrings/debian-archive-keyring.gpg`), it also checks the
   signature with `gpgv`. Without a keyring it relies on the HTTPS mirror.
2. Resolves `Priority: required` + `Essential` + `apt busybox-static kmod
   ca-certificates debian-archive-keyring` + `--include` over Depends and
   Pre-Depends. This is about what debootstrap's minbase installs.
3. Downloads the `.deb` files (checking SHA256, cached in `~/.cache/mkdebinitrd`)
   and unpacks them into a merged-/usr tree with the host's GNU tar.
4. Configures the packages:
   * **If the host can run target binaries**, it runs
     `dpkg --install/--unpack/--configure` in a chroot, the way debootstrap
     does. For a foreign arch, the host can run them if a qemu-user binfmt
     handler is already set up. If none is, mkdebinitrd downloads Debian's
     static `qemu-user` for the host arch from the same mirror (cached in
     `<cache-dir>/qemu/`). It then registers a temporary handler named
     `mkdebinitrd-<pid>-<arch>`, using the `F` flag so the emulator doesn't
     need to be inside the chroot, and removes it when the build exits.
     binfmt_misc applies to the whole host, so while a build runs, other
     processes on the host can also execute that arch through it. An
     emulated build takes a few minutes; arm64 took about 4.5 min here.
   * **With `--no-qemu-download`, `--no-exec`, or no usable emulation**, the
     dpkg database is written in the "unpacked" state, and
     `/etc/busybox-init/firstboot` runs `dpkg --configure -a` during boot. The
     image lives in RAM, so this happens on *every* boot.
5. Unpacks the kernel package outside dpkg, so it doesn't pull in
   initramfs-tools. It copies `vmlinuz` next to the image, keeps the modules,
   and runs `depmod`.
6. Strips docs, man pages and locales, and writes a dpkg `path-exclude`
   config so later `apt-get install`s stay small too. It also configures apt to
   skip Recommends and delete downloaded `.deb`s.
7. Packs a newc cpio, compressed with zstd by default (`-C xz|gzip|none`). The
   build fails if the image is larger than `--max-size` (default 2G).

A default amd64 trixie image is about 111 MiB compressed and 175 MiB unpacked.
About 105 MiB of the unpacked size is kernel modules, and
`--exclude-modules sound,drivers/gpu,...` trims them.

## Boot sequence (busybox init)

* `/init` mounts proc, sys, devtmpfs and /run. It writes `/etc/inittab` with
  one console entry per active kernel console (`console=`), then
  `exec busybox init`.
* `/etc/busybox-init/rcS` runs as sysinit. It mounts devpts, shm and cgroup2,
  starts syslogd and klogd (read the log with `logread`), and loads modules
  for present devices. `mdev` handles hotplug module loading. It then runs
  `firstboot` if present, runs DHCP (`udhcpc`) on every physical NIC, and
  finally runs `/etc/busybox-init/start.d/*` and `/etc/rc.local`.
* Consoles get a root shell (`login -f root`). With `--root-password`, they
  get a getty login prompt instead.
* `/etc/busybox-init/stop.d/*` runs at shutdown.

Kernel command line options: `bbinit.autologin=yes|no`, `bbinit.network=no`,
and `bbinit.shell` (drops to a busybox shell before init starts).

There is no service manager. Packages that ship services won't start them on
their own, so start daemons from `start.d` scripts.

## Options worth knowing

| Option | |
|---|---|
| `-s/--suite` | Debian suite (default `trixie`) |
| `-m/--mirror` | mirror (default `https://deb.debian.org/debian`) |
| `-c/--components` | e.g. `main,non-free-firmware` (needed for `firmware-*`) |
| `-i/--include`, `-x/--exclude` | extra packages / packages never chosen |
| `--no-kernel`, `--kernel-package` | bring your own kernel / choose a flavour |
| `--overlay DIR` | copy a directory tree over the image last |
| `--rootfs DIR --keep-rootfs` | keep the staging tree for inspection |
| `--apt-recommends`, `--keep-docs` | turn off the size savings |
| `--no-qemu-download` | don't fetch qemu for foreign arches; configure at boot |
| `--qemu PATH` | use your own static qemu (copied into the chroot) |

Host requirements: Linux, root, Python ≥ 3.6, GNU tar, cpio, find, sort, chroot,
the compressor, and `depmod` for foreign-arch kernels. The builder doesn't
need Debian tools on the host.
