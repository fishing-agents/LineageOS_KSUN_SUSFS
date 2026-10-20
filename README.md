# OnePlus 11 (salami) LineageOS 24.0 — KernelSU Next + SUSFS

GitHub Actions build of the [LineageOS sm8550 kernel](https://github.com/LineageOS/android_kernel_oneplus_sm8550) (android13-5.15 GKI) with [KernelSU Next](https://github.com/KernelSU-Next/KernelSU-Next) and [SUSFS](https://gitlab.com/simonpunk/susfs4ksu). Patch flow follows [WildKernels/OnePlus_KernelSU_SUSFS](https://github.com/WildKernels/OnePlus_KernelSU_SUSFS).

## Which image

KernelSU Next is built into the kernel, and on salami the kernel lives in **boot.img**. `init_boot.img` only holds the ramdisk, so leave it stock.

Each run downloads the current LineageOS nightly `boot.img`, swaps in the new `Image`, and sets the kernel release string to match stock, so the stock vendor modules still load.

## Outputs

- `boot.img`: flash with `fastboot flash boot boot.img`, or `fastboot boot boot.img` to test once without flashing
- `AK3_*.zip`: flash from recovery or a kernel flasher
- `info.txt`: versions and commits

The `boot.img` matches the LineageOS build date in `info.txt`. Rebuild after each OTA, and install the matching KernelSU Next manager.

## Run

Actions → *Build salami KSUN + SUSFS kernel* → Run workflow. Tick `release` to publish.
