---
title: Build QLI for UNO Q
description: Prepare a Linux host, build Qualcomm Linux with meta-foundries, and preserve the UNO Q image and OS repository.
type: page
doc-category: instructional
authors: David Griego, Codex:GPT-6
last-edited: 2026-10-28
license: MIT
access: public
references:
  - https://github.com/foundriesio/meta-foundries/tree/ee25a99b3f2ef66af36a51da583f8223bcaf58d1
  - https://github.com/qualcomm-linux/meta-qcom-arduino/tree/8457c3dbb5dc650c7b0ac45c99ccb004a05f9d4f
  - https://github.com/qualcomm-linux/meta-qcom-distro/tree/f0c3c7bd98c36bcb797dc5d8d361dbf90d3cafc0
  - https://github.com/qualcomm-linux/meta-qcom/tree/b094cfccc192d8e02809bf87c6dd9c76bcf31330
  - https://github.com/qualcomm-linux/kernel/tree/8d0ad3756ce7d4febc2a5cbdc4d7ab9781d798ca
  - https://github.com/siemens/kas/tree/5.4
relations:
  prev: README.md
  next: flash.md
  parent: README.md
---

# Build QLI for UNO Q

[Guide overview](README.md) · **Part 1 of 4**

Prepare your computer and build Qualcomm® Linux® (QLI) with meta-foundries for Arduino® UNO Q.
Save the flash bundle and OS repository for the flashing and update exercises.

> **Validation status:** On October 28, 2026, a hardware walkthrough followed this page to build OS versions 1 and 2 with the M-05 overlay.
> See [sources and validation](validation.md) for what has and has not been run on hardware.

- [Prepare the build host](#prepare-the-build-host)
- [Build QLI](#build-qli)

## Prepare the Build Host

You need an x86-64 Linux computer running Docker.
The machine that builds the image is the **build environment**.
The machine that flashes the board and runs the update server is the **local computer**.
They can be the same machine.

Use a case-sensitive Linux filesystem for the build.
As a planning allowance, provide 16 GB RAM and 200 GB free disk space; actual requirements vary with caches and parallelism.
Reuse existing download and shared-state caches when available.

### Prepare Your Linux Host

Do this on the build environment and, if it is a separate machine, on the local computer.
Install Git, Python 3, and curl. For Ubuntu:

```bash
sudo apt update
sudo apt install git python3 curl
```

Install Docker Engine with its Buildx and Compose plugins, using **one** of these routes:

- **Docker's packages:** follow [Docker's Ubuntu instructions](https://docs.docker.com/engine/install/ubuntu/).
  They add Docker's apt repository and install `docker-ce`, `docker-buildx-plugin`, and `docker-compose-plugin`.
- **Ubuntu's packages** (Ubuntu 24.04): `sudo apt install docker.io docker-buildx docker-compose-v2`.

Use only one route; their packages conflict.
kas-container and the packaging steps run Docker as your account.
If `docker` needs `sudo`, follow Docker's [post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/).

Check Docker in a new shell:

```bash
docker run --rm hello-world
docker compose version
docker buildx version
```

### Get the Guide Files

The build reads this guide's `kas/` and `workarounds/` directories, so each machine you use needs a copy of the guide.
Clone standalone-docs and copy the guide directory out of it:

```bash
git clone https://github.com/foundriesio/standalone-docs.git "$HOME/standalone-docs"
cp -a "$HOME/standalone-docs/uno-q-setup-and-updates" "$HOME/unoq-guide"
cd "$HOME/unoq-guide"
export GUIDE_DIR="$PWD"
mkdir -p "$GUIDE_DIR/workspace"
```

If you are reading the guide from a branch or pull request, check out that revision in `$HOME/standalone-docs` before copying.
Do this on the build environment and on the local computer.
In a new shell, run `cd "$HOME/unoq-guide"` and set `GUIDE_DIR` again.

## Build QLI

Run this section in your **build environment**.
The configuration builds meta-foundries' UNO Q configuration on the Yocto Project Wrynose release branches.
It selects `qcom-distro-sota` and machine `uno-q`, supplied by `meta-qcom-arduino`.
It builds `core-image-full-cmdline` with the platform updater, Docker application support, networking, and Android Debug Bridge (ADB).
ADB is enabled in the shared image so either console path works; Bughopper users can complete the guide entirely through serial.

The `uno-q` machine brings the board's stock boot chain: Arduino's boot firmware release 251020,
the Android bootloader (ABL) partition layout, U-Boot in the boot partition, and systemd-boot.
After each boot, `qbootctl` marks the boot slot successful; otherwise the firmware would switch to the other slot.
The kernel is `linux-qcom` 6.18 from `meta-qcom`.
The flash bundle contains no separate UEFI image and no firmware capsule.

### What Is Pinned

The [companion configuration](kas/uno-q-wrynose.yml) includes meta-foundries
[`ci/uno-q.yml`](https://github.com/foundriesio/meta-foundries/blob/ee25a99b3f2ef66af36a51da583f8223bcaf58d1/ci/uno-q.yml),
the configuration that meta-foundries' continuous integration builds for the UNO Q.
That file includes the machine and distro configuration of `meta-qcom-arduino` and `meta-qcom`.
kas applies the lockfiles beside the included files.
meta-foundries [`ci/uno-q.lock.yml`](https://github.com/foundriesio/meta-foundries/blob/ee25a99b3f2ef66af36a51da583f8223bcaf58d1/ci/uno-q.lock.yml)
pins `meta-qcom` at its QLI 2.1.1 tag, `qli-2.1.1`, and pins `meta-qcom-arduino`.
meta-qcom's [`ci/base.lock.yml`](https://github.com/qualcomm-linux/meta-qcom/blob/b094cfccc192d8e02809bf87c6dd9c76bcf31330/ci/base.lock.yml) pins the remaining layers.

| Layer | Branch | Commit | Pinned by |
| --- | --- | --- | --- |
| meta-foundries | `wrynose` | `ee25a99` | Companion configuration |
| `meta-qcom-arduino`, the UNO Q machine | `wrynose` | `8457c3d` | meta-foundries `ci/uno-q.lock.yml` |
| `meta-qcom`, the Qualcomm BSP | `wrynose` | `b094cfc`, tag `qli-2.1.1` | meta-foundries `ci/uno-q.lock.yml` |
| `meta-qcom-distro`, the QLI distro | `wrynose` | `f0c3c7b` | `meta-qcom` `ci/base.lock.yml` |
| OpenEmbedded Core | `wrynose` | `ef022bf` | `meta-qcom` `ci/base.lock.yml` |
| BitBake | `2.18` | `0ad6c1c` | `meta-qcom` `ci/base.lock.yml` |
| `meta-lts-mixins` | `wrynose/linux-firmware` | `71dfb81`, with four `meta-qcom` patches | `meta-qcom` `ci/base.lock.yml` |
| `meta-openembedded` | `wrynose` | `47d7dbd` | `meta-qcom` `ci/base.lock.yml` |
| `meta-selinux` | `wrynose` | `1c3a699` | `meta-qcom` `ci/base.lock.yml` |
| `meta-security` | `wrynose` | `c0d1d62` | `meta-qcom` `ci/base.lock.yml` |
| `meta-virtualization` | `wrynose` | `70a5d5a` | `meta-qcom` `ci/base.lock.yml` |
| `meta-updater` | `wrynose` | `92f4788` | `meta-qcom` `ci/base.lock.yml` |
| `meta-ai` | `main` | `60e38e2` | `meta-qcom` `ci/base.lock.yml` |
| `meta-audioreach` | `master` | `b578fcb` | `meta-qcom` `ci/base.lock.yml` |

Each branch contains its pinned commit, as `kas-container` requires.
The companion configuration also repeats the `meta-qcom` and `meta-qcom-arduino` commits from the lockfile:
kas reads the configuration files those two layers include before the lockfile moves their checkouts.
Use the companion configuration throughout this walkthrough.

The companion configuration adds the guide's update tag and server addresses, and ADB.
It also replaces the `meta-qcom` entry that forces Qualcomm RAM-dump mode after a kernel crash;
the optional overlay `kas/diag-download-mode.yml` restores that setting for diagnosis.

meta-foundries `ee25a99` selects aktualizr-lite version 98 at `57e4f7f` and composectl at `fb6b533`.
The client derives the booted OSTree stateroot, including `nodistro`, when `pacman.os` is unset.
The guide leaves `pacman.os` unset and requires the OS-update exercise to verify finalization after reboot.

### Install kas-container and Build OS Version 1

Install the [kas-container wrapper](https://kas.readthedocs.io/en/latest/userguide.html#kas-container)
and put it on `PATH`.
The example below fixes the wrapper and build-container version at 5.4:

```bash
mkdir -p "$HOME/.local/bin"
curl -fL https://raw.githubusercontent.com/siemens/kas/5.4/kas-container \
  -o "$HOME/.local/bin/kas-container"
chmod +x "$HOME/.local/bin/kas-container"
export PATH="$HOME/.local/bin:$PATH"
export KAS_IMAGE_VERSION=5.4
export KAS_WORK_DIR="$HOME/unoq-build"
export DL_DIR="$HOME/yocto-cache/downloads"
export SSTATE_DIR="$HOME/yocto-cache/sstate"
mkdir -p "$KAS_WORK_DIR" "$DL_DIR" "$SSTATE_DIR"
kas-container --version
```

The last command must print `kas-container 5.4`.
The `PATH` setting places this wrapper ahead of any other installed kas-container, for this shell only.
These settings last for this shell; in a new shell for a build, set `GUIDE_DIR` and repeat the `export` lines above.
If you already have Yocto Project download and shared-state caches, point `DL_DIR` and `SSTATE_DIR` at them; otherwise keep these paths.
Keep the build tree on a case-sensitive Linux filesystem.

### Select the Board's Memory Configuration

Start with the base configuration:

```bash
export GUIDE_KAS='kas/uno-q-wrynose.yml'
```

**4 GB UNO Q boards** need the temporary M-05 memory reservation.
Until a boot-firmware release fixes the memory map, every 4 GB board is affected; 2 GB boards are not.
Record the board's RAM size and, with a Bughopper, its firmware revision in `$GUIDE_DIR/workspace/board-record.txt`.
Keep this record in the build environment's guide directory, next to the build records.
The firmware revision comes from the console on the local computer; copy that line into the record:

- **RAM size:** UNO Q boards ship with different RAM sizes.
  Read your board's variant from its product label or purchase record.
  On a running board, `free -h` is supporting evidence only: reserved memory lowers the total it reports.
- **Firmware revision:** this needs the Bughopper's serial console, which you can open before the first flash.
  Fit the Bughopper as shown in [Connect a Bughopper](flash.md#with-a-bughopper).
  On the local computer, install `picocom` (`sudo apt install picocom`) and open the console:

  ```bash
  ls /dev/serial/by-id/
  export BUGHOPPER_PORT='/dev/serial/by-id/PASTE_THE_BUGHOPPER_PORT'
  picocom -b 115200 "$BUGHOPPER_PORT"
  ```

  If `picocom` reports a permission error, see the serial-permission note in [Connect a Bughopper](flash.md#with-a-bughopper).
  Power-cycle the board: unplug the UNO Q's USB-C cable, wait five seconds, and plug it back in.
  Early in the boot output, the boot firmware prints a line beginning `UEFI Ver`; record that line.
  For example, the boot firmware that this guide's image installs prints `UEFI Ver : 5.0.250619.BOOT.XF.4.1.2-00023-KAMORTALAZ-5`.
  Exit `picocom` with Ctrl+A, then Ctrl+X.
  Without a Bughopper, skip this record; the RAM size decides.

If you cannot determine the RAM size, find it before building.

On 4 GB boards, the boot firmware lists the 17 MiB window `0x7b8ff000` through `0x7c9fefff` as System RAM.
With the edk2 UEFI boot chain of earlier guide revisions, pages in that window raised synchronous external aborts during OS updates.
A hardware test of the stock boot chain on a 4 GB board on October 28, 2026 found the window listed as System RAM too.
The [optional layer](workarounds/meta-unoq-m05/conf/layer.conf) reserves the window in the device tree, with `no-map`,
to keep it out of Linux allocation and the linear map.
Its patch ports the [earlier `linux-arduino` patch](https://github.com/foundriesio/foundries-open-update-tests/blob/10b030947f7fc30d513428129f36aecc4cd957bc/build/recipes/linux-arduino/0001-arm64-dts-qcom-qrb2210-arduino-imola-reserve-m05-boundary.patch)
to the `linux-qcom` 6.18 kernel.
This reduces usable RAM and is a temporary firmware workaround.
A fixed memory map is expected from a future boot-firmware release.
Retain the reservation on affected boards until a released firmware passes validation without it.

For a 4 GB board, copy the supplied layer into the kas work directory and select its overlay.
If `$KAS_WORK_DIR/meta-unoq-m05` exists from an earlier revision of this guide, move it aside first:
its append targets the `linux-arduino` kernel, which this configuration does not build.

```bash
mkdir -p "$KAS_WORK_DIR/meta-unoq-m05"
cp -a "$GUIDE_DIR/workarounds/meta-unoq-m05/." "$KAS_WORK_DIR/meta-unoq-m05/"
export GUIDE_KAS="$GUIDE_KAS:kas/m05-4gb.yml"
```

The overlay's relative repository path resolves under `KAS_WORK_DIR`, including inside kas-container.
It targets `linux-qcom_6.18` on `uno-q` and intentionally fails if that recipe is absent.
The patch applies to the kernel that the pinned `meta-qcom` selects.
Do not suppress a dangling-append error or generalize the patch to another kernel or board without reviewing it.
Use the same selection and layer contents for both OS builds; test firmware changes separately.

### Check the Sources and Build

The configuration and the lockfiles it includes pin every repository, so the guide needs no lockfile of its own:
`kas-container lock kas/uno-q-wrynose.yml` reports `No floating repos found. Nothing to lock.`
If `kas/uno-q-wrynose.lock.yml` exists from an earlier revision of this guide, move it out of `kas/` first.
kas applies a lockfile next to a configuration automatically, and an old one would replace these pins.

Record your selection, write out the resolved configuration, and list its commits:

```bash
cd "$GUIDE_DIR"
printf '%s\n' "$GUIDE_KAS" > "$GUIDE_DIR/workspace/kas-config"
kas-container dump --resolve-refs "$GUIDE_KAS:kas/os-v1.yml" \
  > "$GUIDE_DIR/workspace/os-v1-resolved.yml"
awk '/^repos:/{r=1; next} /^[^ ]/{r=0} r && /^  [a-z]/{name=$1} r && /commit:/{print name, $2}' \
  "$GUIDE_DIR/workspace/os-v1-resolved.yml"
```

Check that the command lists 14 repositories and that each commit begins with the commit in the [table above](#what-is-pinned). Then build:

```bash
kas-container build "$GUIDE_KAS:kas/os-v1.yml"
```

The first build runs about 7,000 BitBake tasks.
On a 16-core host, with the sources downloaded and most shared state already cached, it took about 16 minutes;
a build without shared-state caches takes considerably longer.
Allow two hours or more on a smaller host, or when the sources still need downloading.
If the build stops with an error, see [troubleshooting](troubleshooting.md#troubleshoot-a-failed-checkpoint).
Keep the pinned revisions; changing one to work around an error produces an untested combination.

Keep `workspace/kas-config`, `workspace/os-v1-resolved.yml`, and the optional layer bytes with your build records.
Use the same selection for OS version 2.
Do not update layer revisions between the two builds in this walkthrough.

The deployment directory is:

```bash
export DEPLOY="$KAS_WORK_DIR/build/tmp/deploy/images/uno-q"
ls "$DEPLOY"
```

Before continuing, confirm that it contains an `ostree_repo` directory and
`core-image-full-cmdline-uno-q.rootfs.qcomflash`.
That name is a symbolic link to the build's timestamped flash directory, so list it with `ls -L`:

```bash
ls -L "$DEPLOY/core-image-full-cmdline-uno-q.rootfs.qcomflash" \
  | grep -x -E 'prog_firehose_ddr.elf|rawprogram0.xml|patch0.xml'
```

The command must print all three file names.
The build also writes `core-image-full-cmdline-uno-q.rootfs.qcomflash.tar.gz`, which unpacks into a differently named directory.
This guide creates its own archive in the next step, so you can ignore that file.

Preserve the OS repository before the next build changes it:

```bash
mkdir -p "$GUIDE_DIR/workspace/releases/1"
cp -a "$DEPLOY/ostree_repo" "$GUIDE_DIR/workspace/releases/1/ostree_repo"
tar -chzf "$GUIDE_DIR/workspace/unoq-os1-flash.tar.gz" \
  -C "$DEPLOY" core-image-full-cmdline-uno-q.rootfs.qcomflash
```

The archive follows symlinks so the transferred flash bundle contains its referenced files.
If your builder is remote, transfer the archive and `releases/1/ostree_repo` to the local guide's `workspace/`.
For example, on the local computer, replacing the host and paths:

```bash
scp builder:/absolute/path/to/guide/workspace/unoq-os1-flash.tar.gz "$GUIDE_DIR/workspace/"
mkdir -p "$GUIDE_DIR/workspace/releases/1"
scp -r builder:/absolute/path/to/guide/workspace/releases/1/ostree_repo \
  "$GUIDE_DIR/workspace/releases/1/"
```

**Next:** [Flash UNO Q and connect to the board](flash.md).
Keep the guide directory, OS repository, build records, and Linux build environment for the later update exercises.
