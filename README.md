# NVIDIA 390xx DKMS on CachyOS Kernel 7.2

Patches and DKMS configuration for NVIDIA 390.157 on CachyOS kernel 7.2.x.

## Why this project exists

NVIDIA 390.157 is a legacy proprietary driver and its original kernel interface code predates several Linux 7.2 API changes.

On CachyOS kernel `7.2.8-1-cachyos`, the unmodified 390.157 source fails to build. The two relevant compatibility problems are:

1. **Linux 7.2 removed the kernel `strncpy()` API.**  
   NVIDIA 390.157 still calls `strncpy()` in several places. The compatibility patch changes those calls to the appropriate newer string helpers while retaining compatibility with older kernels.

2. **Linux 7.2 renamed the DRM atomic state interface.**  
   `struct drm_atomic_state` and related APIs were renamed to `struct drm_atomic_commit`. The DRM patch updates NVIDIA's 390xx DRM/KMS integration and its conftest checks to handle the new interface, while retaining compatibility with older kernels.

The goal is therefore **not to modify NVIDIA's driver design**, but to provide the kernel-compatibility layer required for the old 390.157 source to build against newer CachyOS kernels.

## Tested environment

- Hardware: Dell Latitude E5440
- GPU: NVIDIA GeForce GT 720M
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

The upstream patch files used as the basis for this repository were:

- `nvidia-390xx-kmod-0107-adaptation-to-new-struct-drm-atomic-commit.patch`
- `nvidia-390xx-kmod-0108-kernel-7.2-remove-strncpy-kernel-function.patch`

For CachyOS, the adaptation made for this repository was limited to the source-tree path layout: the `kernel/` path component used by the RPM Fusion source package was stripped so the patches apply with `patch -p1` to the CachyOS NVIDIA 390.157 source tree.

The repository patch files were byte-for-byte verified against the corresponding local RPM Fusion patch files.

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

### 0108-cachyos.patch — removal of kernel `strncpy()`

Linux 7.2 removed the generic kernel `strncpy()` API. The NVIDIA 390xx source still used it.

This patch changes the affected code paths to:
- `strscpy()` where NUL-terminated copying is required
- `strscpy_pad()` where the original padding semantics matter

Affected source areas:
- `nvidia/nv-gpu-numa.c`
- `nvidia/os-interface.c`
- `nvidia-uvm/uvm8_pmm_gpu.c`
- `nvidia-modeset/nvidia-modeset-linux.c`

## DKMS configuration

DKMS 3.4.3 supports kernel-specific `PATCH[]/PATCH_MATCH[]` and `MAKE[]/MAKE_MATCH[]` overrides.

This project uses:

`/etc/dkms/nvidia-390.157.conf`

with the two patches stored in:

`/etc/dkms/nvidia/patches/`

The override applies both patches only to kernels matching `^7\.2\.` and selects the full LLVM/LLD build command for those kernels. The original DKMS `MAKE[0]` remains the default for kernels that do not match, preserving the existing LTS configuration.

Example override:

```
PATCH[0]="0107-cachyos.patch"
PATCH_MATCH[0]="^7\.2\."

PATCH[1]="0108-cachyos.patch"
PATCH_MATCH[1]="^7\.2\."

MAKE[1]="'make' -j`nproc` LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=\${kernelver} modules"
MAKE_MATCH[1]="^7\.2\."
```

## Applying the fix

These steps are scoped to NVIDIA 390.157 and kernel 7.2.x. They do **not** remove the working LTS registration.

### 1. Install the patches

```
sudo mkdir -p /etc/dkms/nvidia/patches
sudo cp 0107-cachyos.patch 0108-cachyos.patch /etc/dkms/nvidia/patches/
```

If applying from a clone of this repository, run the commands from the repository directory.

### 2. Install the DKMS override

Create:

`/etc/dkms/nvidia-390.157.conf`

with:

```
PATCH[0]="0107-cachyos.patch"
PATCH_MATCH[0]="^7\.2\."

PATCH[1]="0108-cachyos.patch"
PATCH_MATCH[1]="^7\.2\."

MAKE[1]="'make' -j`nproc` LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=\${kernelver} modules"
MAKE_MATCH[1]="^7\.2\."
```

Verify the configuration can be read:

```
dkms status
```

### 3. Build only for the target kernel

```
sudo dkms build -m nvidia -v 390.157 -k 7.2.8-1-cachyos
```

Expected relevant output:

```
Applying patch 0107-cachyos.patch... done.
Applying patch 0108-cachyos.patch... done.
Building module(s)... done.
```

### 4. Install only for the target kernel

```
sudo dkms install -m nvidia -v 390.157 -k 7.2.8-1-cachyos
```

Verify:

```
dkms status
sudo modinfo -F filename nvidia
```

The target should show:

```
nvidia/390.157, 7.2.8-1-cachyos, x86_64: installed
```

and the module path should be under:

``/lib/modules/7.2.8-1-cachyos/updates/dkms/``

### 5. Reboot and verify actual runtime

Boot the target kernel and run:

```
uname -r
nvidia-smi
lsmod | grep '^nvidia'
```

A successful runtime test should show:
- `uname -r` → `7.2.8-1-cachyos`
- `nvidia-smi` → NVIDIA driver `390.157` and the GPU
- `lsmod` → `nvidia`, `nvidia_uvm`, `nvidia_modeset`, and `nvidia_drm`

## Verified result

The procedure above was successfully tested on the Dell Latitude E5440.

After rebooting into `7.2.8-1-cachyos`:

```
uname -r
7.2.8-1-cachyos
```

```
nvidia-smi
NVIDIA-SMI 390.157
Driver Version: 390.157
GPU: GeForce GT 720M
Memory: 0MiB / 1985MiB
```

Loaded modules:

```
nvidia_drm
nvidia_modeset
nvidia_uvm
nvidia
```

DKMS status after installation:

```
nvidia/390.157, 6.18.52-1-cachyos-lts, x86_64: installed
nvidia/390.157, 7.2.8-1-cachyos, x86_64: installed
```

The installed module was confirmed with:

```
sudo modinfo -F filename nvidia
/lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia.ko.zst
```

and:

```
sudo modinfo nvidia | grep -E '^(version|filename|signer):'
```

reported NVIDIA `390.157` and the DKMS module signing key.

This is a **real post-reboot runtime test**, not only a successful compilation test.

## Build warnings

The successful build emitted:
- multiple `objtool: data relocation to !ENDBR` warnings
- `WARNING: modpost: missing MODULE_DESCRIPTION() in nvidia-uvm.o`

These warnings did not prevent module generation or runtime loading on the tested system.

The complete original build output is preserved in [`logs/build-7.2.8-success.log`](logs/build-7.2.8-success.log).

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

## Important limitations

- This project targets NVIDIA 390.157 and the tested CachyOS 7.2.x environment.
- The runtime result is verified on kernel `7.2.8-1-cachyos`; it should not be interpreted as proof that every future 7.2.x kernel will remain compatible.
- The patches are compatibility adaptations; they do not make the legacy NVIDIA driver a modern driver.
- The repository does not redistribute NVIDIA proprietary source archives or binary modules.

## License / redistribution

This repository does not redistribute NVIDIA's proprietary source archive, binary modules, object files, or generated build artifacts.

It contains compatibility patches derived from publicly available RPM Fusion packaging work, plus documentation and build evidence for the CachyOS kernel 7.2.x environment.
