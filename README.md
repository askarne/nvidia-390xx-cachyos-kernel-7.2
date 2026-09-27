# NVIDIA 390xx DKMS on CachyOS Kernel 7.2

Patches and build documentation for NVIDIA 390.157 on CachyOS kernel 7.2.x.

## Why this project exists

NVIDIA 390.157 is a legacy proprietary driver and its original kernel interface code predates several Linux 7.2 API changes.

On CachyOS kernel `7.2.8-1-cachyos`, the unmodified 390.157 source fails to build. The two relevant compatibility problems are:

1. **Linux 7.2 removed the kernel `strncpy()` API.**  
   NVIDIA 390.157 still calls `strncpy()` in several places. The compatibility patch changes those calls to the appropriate newer string helpers while retaining compatibility with older kernels.

2. **Linux 7.2 renamed the DRM atomic state interface.**  
   `struct drm_atomic_state` and related APIs were renamed to `struct drm_atomic_commit`. The DRM patch updates NVIDIA's 390xx DRM/KMS integration and its conftest checks to handle the new interface, while retaining compatibility with older kernels.

The goal is therefore **not to modify NVIDIA's driver design**, but to provide the minimum kernel-compatibility layer required for the old 390.157 source to compile against the newer CachyOS kernel.

## Tested environment

- Hardware: Dell Latitude E5440
- Distribution: CachyOS
- Desktop: KDE Plasma 6.7.4
- Session: Wayland
- Target kernel: `7.2.8-1-cachyos`
- Known-working LTS kernel: `6.18.52-1-cachyos`
- NVIDIA 390xx DKMS: `390.157-25`
- DKMS: `3.4.3`
- Kernel compiler: Clang 22.1.8
- Kernel linker: LLD
- Kernel configuration: Clang + ThinLTO

## Patch sources and provenance

The patches in this repository are **adapted from RPM Fusion's NVIDIA 390xx kmod patches** released in `nvidia-390xx-kmod-390.157-27.fc43`.

RPM Fusion's 390.157-27 changelog explicitly records both changes:
- adaptation to the new `struct drm_atomic_commit`
- a Linux >= 7.2 patch removing use of the kernel `strncpy` function

Source package:

`nvidia-390xx-kmod-390.157-27.fc43.src.rpm`

The upstream patch files used as the basis for this repository were:

- `nvidia-390xx-kmod-0107-adaptation-to-new-struct-drm-atomic-commit.patch`
- `nvidia-390xx-kmod-0108-kernel-7.2-remove-strncpy-kernel-function.patch`

The original RPM Fusion patch headers and authorship are preserved in the patch files. For CachyOS, the adaptation made for this repository was limited to the source-tree path layout: the `kernel/` path component used by the RPM Fusion source package was stripped so the patches apply with `patch -p1` to the CachyOS NVIDIA 390.157 source tree.

### Primary sources

- [RPM Fusion source RPM directory](https://archive.rpmfusion.org/Mirrors/rpmfusion.org/nonfree/fedora/updates/43/SRPMS/n/)
- [RPM Fusion 390.157-27 package information and changelog](https://www.rpmfind.net/linux/RPM/rpmfusion/nonfree/fedora/updates/testing/43/x86_64/k/kmod-nvidia-390xx-390.157-27.fc43.x86_64.html)
- [Linux: removal of `strncpy()`](https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux/+/079a028d6327e68cfa5d38b36123637b321c19a7)
- [Linux DRM rename: `drm_atomic_state` → `drm_atomic_commit`](https://git.zx2c4.com/wireguard-linux/commit/drivers/gpu/drm/nouveau?h=davem%2Fnet&id=5164f7e7ff8ec7d41065d3862630c2ba09854328)

## What each patch does

### 0107-cachyos.patch — DRM atomic commit API

This patch updates the 390xx DRM/KMS layer for the Linux 7.2 rename from:

`struct drm_atomic_state`

to:

`struct drm_atomic_commit`

It updates:
- DRM conftest detection
- atomic check callbacks
- atomic state allocation/cleanup wrappers
- reference-counting detection
- NVIDIA DRM KMS structures and function signatures
- compatibility aliases for kernels older than 7.2

The underlying Linux change was made by Maxime Ripard. The motivation was to distinguish the device-level atomic commit object from the per-object state structures.

### 0108-cachyos.patch — removal of kernel `strncpy()`

Linux 7.2 removed the generic kernel `strncpy()` API. The NVIDIA 390xx source still used it.

This patch changes the affected code paths to newer kernel string helpers:
- `strscpy()` where NUL-terminated copying is required
- `strscpy_pad()` where the original padding semantics matter

The affected NVIDIA source areas are:
- `nvidia/nv-gpu-numa.c`
- `nvidia/os-interface.c`
- `nvidia-uvm/uvm8_pmm_gpu.c`
- `nvidia-modeset/nvidia-modeset-linux.c`

The underlying Linux change was part of Kees Cook's `strncpy` removal work.

## Validation

Both patches were tested with:

```
patch --dry-run -p1
```

and applied successfully to a clean NVIDIA 390.157 test source tree.

## Successful build

The CachyOS 7.2.8 kernel uses Clang/LLVM, LLD and ThinLTO. A GCC build failed because of LLVM-specific kernel flags and LTO objects. `CC=clang` alone also failed because an intermediate object was LLVM IR bitcode.

The successful build therefore used the complete LLVM toolchain:

```
make -j"$(nproc)" \
  LLVM=1 \
  CC=clang \
  LD=ld.lld \
  AR=llvm-ar \
  NM=llvm-nm \
  HOSTCC=clang \
  HOSTLD=ld.lld \
  IGNORE_CC_MISMATCH=1 \
  IGNORE_PREEMPT_RT_PRESENCE=1 \
  KERNEL_UNAME=7.2.8-1-cachyos \
  modules
```

Generated successfully:

- `nvidia.ko`
- `nvidia-modeset.ko`
- `nvidia-uvm.ko`
- `nvidia-drm.ko`

See the **raw build log** captured from the successful build: [logs/build-7.2.8-success.log](logs/build-7.2.8-success.log). It contains the original build output rather than a reconstructed summary.

## Important status

The **source patches and manual LLVM/LLD build are verified**.

Permanent DKMS automation is **not yet claimed as complete**. DKMS 3.4.3 patch/override semantics still need to be verified before changing the installed DKMS configuration.

The known-working LTS kernel `6.18.52-1-cachyos` must remain preserved during subsequent DKMS testing.

## Repository contents

```
nvidia-390xx-cachyos-kernel-7.2/
├── README.md
├── FULL-REPORT.md
├── patches/
│   ├── 0107-cachyos.patch
│   └── 0108-cachyos.patch
└── logs/
    └── build-7.2.8-success.log
```

## Licensing / redistribution

This repository does **not** redistribute NVIDIA's proprietary source archive, binary modules, object files, or generated build artifacts.

It contains compatibility patches derived from publicly available RPM Fusion packaging work, plus documentation and build evidence for the CachyOS kernel 7.2.x environment.
