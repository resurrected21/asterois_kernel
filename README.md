# Nothing Phone (3a) Pro — Kernel build (KSU-Next / SukiSU + SusFS)

GitHub Actions workflow that builds a **GKI android14-6.1** kernel for the
Nothing Phone (3a) Pro (`asteroids` / `sm7635`) with a **selectable root solution**
(KernelSU-Next or SukiSU-Ultra) + **SusFS**, and outputs both a flashable
**AnyKernel3 zip** and (optionally) a ready-to-`fastboot` **patched boot.img**.

## How it works

The device boots a GKI kernel, so the workflow rebuilds only the generic GKI
`Image` with a standalone **Neutron Clang** toolchain and packages it. The heavy
Bazel/kleaf manifest build is intentionally avoided — it needs the full
kernel-platform repo manifest and is fragile in CI. Vendor partitions
(`vendor_boot`, `vendor_dlkm`, `init_boot`) stay stock.

Source tree: this repo (a fork of `NothingOSS/android_kernel_msm-6.1_nothing_sm7635`).
The workflow checks out whatever branch you launch it from.

Versions are **pinned** to matched pairings to avoid upstream drift / SusFS mismatch:

| Variant | SusFS | Root |
|---------|-------|------|
| KernelSU-Next | simonpunk `v1.5.11` | KSU-Next `next-susfs-a14-6.1-dev` (kprobe hooks) |
| SukiSU-Ultra | ShirkNeko (current) | SukiSU `builtin` + `scope_min_manual_hooks` |

To bump a version later, edit the commit SHAs in the workflow `env:` block.

## Run it

1. Push this folder to your GitHub repo (it can be the kernel fork itself).
2. On GitHub: **Actions → Build Kernel (Nothing 3a Pro …) → Run workflow**,
   selecting the branch that holds your kernel source.
3. Pick your options (see below) and press **Run workflow**.
4. First run ~25–40 min (it seeds the ccache + toolchain caches); later runs are
   much faster. Download outputs from the run's **Artifacts** (or the Release).

## Workflow inputs

| Input | Default | Notes |
|-------|---------|-------|
| `root_variant` | `KernelSU-Next` | `KernelSU-Next` or `SukiSU-Ultra` |
| `enable_kpm` | `false` | SukiSU only — enable KPM (Kernel Patch Module) |
| `stock_boot_url` | _(empty)_ | Direct URL to a stock `boot.img` → produces a patched boot.img |
| `stock_firmware_url` | _(empty)_ | URL to a firmware/OTA `.7z`/`.zip`/`payload.bin` → boot.img auto-extracted |
| `make_release` | `false` | Publish the outputs as a GitHub Release |

Provide a stock source (`stock_boot_url` **or** `stock_firmware_url`) to get a
flashable `boot.img`. Without one, you still get the AnyKernel3 zip.

## Flashing

You have two routes. **The boot.img + fastboot route is the reliable one** — the
on-device AnyKernel3 repack (e.g. the Kernel Flasher app) frequently fails on
Nothing/Qualcomm GKI devices with *"Repacking image failed"* due to an lz4
kernel-compression quirk in the app's bundled magiskboot.

### Recommended — patched boot.img via fastboot

```bash
adb reboot bootloader
# TEST first — temporary, nothing is written to the phone:
fastboot boot Xeon-NP3aPro-KSU-Next-SusFS-boot.img
# If it boots and the KSU-Next / SukiSU app shows root, flash it for real:
fastboot flash boot Xeon-NP3aPro-KSU-Next-SusFS-boot.img
fastboot reboot
```

### Alternative — AnyKernel3 zip

Flash the zip from a custom recovery, or via the root manager app's built-in
kernel flasher. If it errors at *"Repacking image failed"*, use the boot.img
route above instead.

### After first boot

Install the matching manager app — **KernelSU-Next** or **SukiSU-Ultra** — to
confirm root, then add the **SusFS module** for hiding.

> ⚠️ Requires an unlocked bootloader. The first transition to a rooted kernel may
> require a data wipe (`fastboot -w`) on some setups. **Back up first.**

## ⚠️ Firmware must match

A `boot.img` carries a header tied to your firmware build (OS / security-patch
level). **Use the stock boot.img from the exact Nothing OS build you are currently
on** — otherwise AVB/version checks can reject it or cause a bootloop.

Stock images per build are available from the
[spike0en/nothing_archive](https://github.com/spike0en/nothing_archive) releases,
named `<build>-image-boot.7z` (each contains `boot.img`). Confirm the build against
**Settings → About phone** before flashing.

## Notes & caveats

- **KernelSU-Next is the proven path.** SukiSU's `scope_min_manual_hooks` patch is
  written against AOSP-common `fs/`; Nothing's tree has vendor changes, so that
  patch step *may* need a hunk adjustment on first try. Build KSU-Next first.
- Toolchain (Neutron Clang) and compiled objects (ccache) are cached, so reruns
  skip the toolchain download and recompile only what changed.
- This produces a generic GKI Image and does **not** rebuild vendor kernel modules,
  which is correct for a root-only kernel.
- This is an **init_boot** device (Android 13+ launch): `boot.img` is kernel-only,
  so only `boot` is patched; `init_boot` stays stock.
