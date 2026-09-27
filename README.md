# NVIDIA 390xx DKMS on CachyOS Kernel 7.2

Patches and build documentation for NVIDIA 390.157 on CachyOS kernel 7.2.x.

## Tested environment
- Dell Latitude E5440
- CachyOS KDE Plasma 6.7.4 / Wayland
- Target kernel: 7.2.8-1-cachyos
- Known-working LTS kernel: 6.18.52-1-cachyos
- NVIDIA 390xx DKMS: 390.157-25
- DKMS: 3.4.3
- Kernel compiler: Clang 22.1.8
- Kernel linker: LLD
- Kernel configuration: Clang + ThinLTO

## Problem
Stock NVIDIA 390.157 fails against kernel 7.2.8. The decisive failure is the removal of the kernel `strncpy()` interface; the kernel 7.2 DRM atomic commit API also requires adaptation.

## Patches
- `patches/0107-cachyos.patch` — adaptation to the newer DRM atomic commit API.
- `patches/0108-cachyos.patch` — kernel 7.2 adaptation removing obsolete `strncpy()` usage.

These are adapted from the corresponding RPM Fusion NVIDIA 390xx kmod patches by changing only the path prefix required by the CachyOS NVIDIA source tree.

## Successful build
Both patches applied cleanly to a clean NVIDIA 390.157 source tree. The complete module set built successfully using the LLVM/LLD toolchain matching the kernel:

```
make -j"$(nproc)" \
  LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm \
  HOSTCC=clang HOSTLD=ld.lld \
  IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 \
  KERNEL_UNAME=7.2.8-1-cachyos modules
```

Generated modules:
- nvidia.ko
- nvidia-modeset.ko
- nvidia-uvm.ko
- nvidia-drm.ko

## Important status
The source patches and manual LLVM/LLD build are verified. Permanent DKMS automation has **not** yet been claimed as complete; DKMS 3.4.3 patch/override semantics still need to be verified before changing the installed DKMS configuration.

The working LTS kernel must remain preserved. Do not use a blanket `dkms remove ... --all` during subsequent testing.

## Scope
This repository contains patches and documentation only. It does not redistribute NVIDIA proprietary source archives or binary kernel modules.
