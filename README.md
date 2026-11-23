# OrangeFox recovery for the Xiaomi Mi 11 Pro (`mars`)

Device tree for building an **OrangeFox Recovery** (base: `fox_14.1`, i.e. AOSP 14
+ OrangeFox R12) for the **Xiaomi Mi 11 Pro**, codename **`mars`**, SoC
**Qualcomm sm8350 (Snapdragon 888)**, **A/B**, **no dedicated recovery partition**.

The recovery is built **into `boot.img`** (`BOARD_USES_RECOVERY_AS_BOOT`); nothing
in this repository flashes anything by itself.

> **Current status: v4.9 — usable.**
> `/data` is decrypted with the user's PIN (FBE, hardware-wrapped keys, metadata
> encryption), ADB and MTP work, battery/language/theme defaults are applied, and
> zip flashing + partition backup/restore work.
> Full status, the fix history and the remaining work: **[docs/STATUS.md](docs/STATUS.md)**.
> The latest tested image is the `boot-v4.9` asset of the
> [`recovery-test`](https://github.com/S0SL/twrp_mars/releases/tag/recovery-test)
> pre-release (`sha256 0567218e977f45c51267291a04c20557c12f594cec6d943db5753dc8d127b4b1`).

---

## 1. Try it without flashing

```sh
curl -L -O https://github.com/S0SL/twrp_mars/releases/download/recovery-test/boot-v4.9.img.gz
gunzip boot-v4.9.img.gz
fastboot boot boot-v4.9.img          # loads it into RAM; writes nothing
```

`fastboot flash boot boot-v4.9.img` makes the device boot into recovery **every
time** — that is inherent to "no recovery partition" (see
[limitations](docs/STATUS.md)). To go back to Android,
re-flash the ROM's own `boot.img`.

## 2. Build it

You need your own build server (the AOSP tree does not fit in a container):

* **step by step:** [docs/BUILD-ON-SERVER.md](docs/BUILD-ON-SERVER.md)
* short version, on a machine that already has the tree:

  ```sh
  git clone https://github.com/S0SL/twrp_mars.git ~/twrp-mars
  ~/twrp-mars/scripts/build-local.sh --fox-dir ~/fox_14.1            # first run
  ~/twrp-mars/scripts/build-local.sh --fox-dir ~/fox_14.1 --skip-sync  # later runs
  ```

  The build job is `bootimage`; the artifact is `out/target/product/mars/boot.img`.
  `MARS_JOBS=12` is the sweet spot on a 24-core / 22 GB box (16 threads is slower).

The build script runs two guards before compiling anything:

| hook | script | what it protects against |
| --- | --- | --- |
| `== 0. shell syntax lint ==` | `scripts/check-shell.sh` | bash-isms in the shell scripts that ship in the ramdisk (recovery runs Android **mksh**; a `${ro.boot.slot_suffix}` killed the v4.7 firmware script) |
| `== 0b. build-tree patches ==` | `scripts/apply-patches.sh` | re-applies the AOSP-side fixes (FBE credential flow, GUI drag cadence) that this device needs but that cannot live in a device tree |

Static checks for the device tree itself: `./scripts/check-tree.sh` (required
files, fstab/flags syntax, makefile sanity, codename consistency) and
`./scripts/check-shell.sh`.

## 3. Repository layout

```
BoardConfig.mk device.mk fox_mars.mk twrp_mars.mk AndroidProducts.mk Android.mk
vendorsetup.sh board-info.txt manifest.xml modules.load.recovery
firmware/                       touch firmware staged into the ramdisk
prebuilt/                       prebuilt kernel Image (not committed)
recovery/root/                  files that go into the recovery ramdisk:
  init.recovery.qcom.rc           mounts + property defaults (see STATUS.md)
  init.recovery.usb.rc            QTI configfs gadget (adb + MTP + sideload)
  sbin/mars-fw-mount.sh           mounts the firmware/modem image for the TEE
  system/etc/{recovery.fstab,twrp.flags}
  system/bin/, vendor/lib64/      QSEE/keymaster/gatekeeper blobs (see docs/PROVENANCE.md)
  twres/languages/zh_CN.xml       simplified-Chinese UI translation
  vendor/etc/vintf/manifest.xml   VINTF declarations the vendor HALs need
scripts/                        sync / build / patch / check / publish helpers
patches/                        notes for the build-tree fixes (applied by apply-patches.sh)
docs/                           see below
```

## 4. Documentation

| file | content |
| --- | --- |
| **[docs/STATUS.md](docs/STATUS.md)** | **handover doc** — what works, every fix and why, what is left (v5), known limitations |
| [docs/BUILD-ON-SERVER.md](docs/BUILD-ON-SERVER.md) | full build guide, timings, every failure this project hit |
| [docs/BACKUP-GUIDE.md](docs/BACKUP-GUIDE.md) | how to back up `/data` (rooted Android) and partitions (recovery), and how to restore |
| [docs/PROVENANCE.md](docs/PROVENANCE.md) | where every blob comes from |
| [docs/RAMDISK-MODULES.md](docs/RAMDISK-MODULES.md) | staging kernel modules (touch/display) into the ramdisk |
| [docs/BUILD-STATUS.md](docs/BUILD-STATUS.md) | troubleshooting chronology (the "why" behind every guard) |
| [docs/kernel-prep/](docs/kernel-prep/) | prepared-but-unpushed kernel-side change (touch modules) |
| [docs/history/](docs/history/) | superseded phase reports and plans (v4, v4.6, phase 2, fstab diff, phase-1 known issues) |

## 5. Support / known limitations (short)

* No `recovery` partition: a flashed `boot.img` boots recovery, not Android.
* `/data` decryption needs the phone's TEE **and** the vendor property
  `vendor.gatekeeper.disable_spu=true` (the recovery sets it; the stock ROM ships
  it — see [STATUS.md](docs/STATUS.md)).
* SELinux runs **permissive** in this recovery (enforcing needs a sepolicy set).
* `AB_OTA_UPDATER` is off, so zips that rewrite logical partitions
  (`system`/`vendor`/…) are not supported yet — see the v5 list in STATUS.md.
* The image is signed with AVB test keys — for unlocked bootloaders only.

Everything above is explained, with evidence and the exact source locations, in
**[docs/STATUS.md](docs/STATUS.md)**.
