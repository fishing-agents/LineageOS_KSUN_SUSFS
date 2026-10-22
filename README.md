# LineageOS KernelSU Next + SUSFS kernels

Kernels for LineageOS devices with [KernelSU Next](https://github.com/KernelSU-Next/KernelSU-Next) and [SUSFS](https://gitlab.com/simonpunk/susfs4ksu) built in. Everything builds on GitHub Actions from the official LineageOS kernel source. Each build is repacked into the official LineageOS nightly `boot.img` for that device.

Patch flow follows [WildKernels/OnePlus_KernelSU_SUSFS](https://github.com/WildKernels/OnePlus_KernelSU_SUSFS), adapted to LineageOS kernel trees.

## Supported devices

- **OnePlus 11 (salami)**
  - LineageOS: 24.0
  - Kernel: [android_kernel_oneplus_sm8550](https://github.com/LineageOS/android_kernel_oneplus_sm8550), 5.15 GKI (android13-5.15)
  - Flash: `boot`
  - Status: builds in CI

More devices can be added if their LineageOS kernel is GKI 5.10, 5.15, 6.1 or 6.6. Open an issue with the device codename.

## Downloads

Grab the latest build from [Releases](../../releases/latest):

- `boot.img`: stock LineageOS `boot.img` with the new kernel swapped in
- `AK3_*.zip`: AnyKernel3 zip that flashes from recovery
- `info.txt`: LineageOS build date, kernel release, KernelSU Next and SUSFS versions
- `SHA256SUMS`: file checksums

Each `boot.img` matches **one** LineageOS build. Check the date in `info.txt` against **Settings → About phone**.

## Install

1. Update LineageOS to the build listed in `info.txt`.
2. Test without flashing:
   ```
   adb reboot bootloader
   fastboot boot boot.img
   ```
3. If it boots, install the [KernelSU Next manager](https://github.com/KernelSU-Next/KernelSU-Next/releases). It should show the kernel as supported, with SUSFS listed. Then flash:
   ```
   fastboot flash boot boot.img
   ```
   You can flash the AK3 zip from recovery instead.
4. Install a SUSFS userspace module if you want to configure hiding.

To roll back, flash the stock `boot.img` for your build from [download.lineageos.org](https://download.lineageos.org).

### boot or init_boot?

Flash **`boot`**. KernelSU Next with SUSFS is built into the kernel, and on these devices the kernel lives in `boot`. `init_boot` only holds the ramdisk. Guides that flash `init_boot` are for Magisk or KernelSU in loadable-module (LKM) mode. That method can't add SUSFS. Leave `init_boot` stock.

## After a LineageOS update

The update overwrites `boot`, so you lose root. Wait for a new release that matches your build, or run the workflow yourself, then flash again.

## What the build does

- Pulls the latest LineageOS nightly `boot.img` from the official API and checks its sha256.
- Clones the LineageOS kernel plus its `-modules` tree, and uses the clang version the kernel specifies.
- Adds KernelSU Next and the SUSFS kernel patches, with WildKernels' fixups for KernelSU Next.
- Sets the kernel version string to the same value as stock, so the stock vendor modules still load. The build fails if the two differ.
- Uses ThinLTO instead of full LTO, which runs out of memory on hosted runners.
- Repacks the stock `boot.img` with magiskboot, builds the AK3 zip, and publishes a release.

Pinned by default:
- KernelSU Next `4cb8ee68` (33388)
- susfs4ksu `2ad2973` (v2.3.0)

Newer KernelSU Next moves SUSFS into a separately loaded module, which these in-kernel patches don't support. You can override both pins under **Actions → Run workflow**.

## Build it yourself

Fork, then go to **Actions → Build salami KSUN + SUSFS kernel → Run workflow**. The output appears as a release in your fork.

## Troubleshooting

Play Integrity failing? See [docs/play-integrity-debug.md](docs/play-integrity-debug.md).

## Disclaimer

Unofficial build, not affiliated with LineageOS or OnePlus. Unlocked bootloader required. Keep the stock `boot.img` so you can roll back.
