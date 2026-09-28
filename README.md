# NVIDIA 390xx DKMS on CachyOS Kernel 7.2

Patches and build documentation for NVIDIA 390.157 on CachyOS kernel 7.2.x.
Patches and DKMS configuration for NVIDIA 390.157 on CachyOS kernel 7.2.x.

## Why this project exists

@@ -14,11 +14,12 @@ On CachyOS kernel `7.2.8-1-cachyos`, the unmodified 390.157 source fails to buil
2. **Linux 7.2 renamed the DRM atomic state interface.**  
   `struct drm_atomic_state` and related APIs were renamed to `struct drm_atomic_commit`. The DRM patch updates NVIDIA's 390xx DRM/KMS integration and its conftest checks to handle the new interface, while retaining compatibility with older kernels.

The goal is therefore **not to modify NVIDIA's driver design**, but to provide the minimum kernel-compatibility layer required for the old 390.157 source to compile against the newer CachyOS kernel.
The goal is therefore **not to modify NVIDIA's driver design**, but to provide the kernel-compatibility layer required for the old 390.157 source to build against newer CachyOS kernels.

## Tested environment

- Hardware: Dell Latitude E5440
- GPU: NVIDIA GeForce GT 720M
- Distribution: CachyOS
- Desktop: KDE Plasma 6.7.4
- Session: Wayland
@@ -34,20 +35,14 @@ The goal is therefore **not to modify NVIDIA's driver design**, but to provide t

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
For CachyOS, the adaptation made for this repository was limited to the source-tree path layout: the `kernel/` path component used by the RPM Fusion source package was stripped so the patches apply with `patch -p1` to the CachyOS NVIDIA 390.157 source tree.

The repository patch files were byte-for-byte verified against the corresponding local RPM Fusion patch files.

### Primary sources

@@ -76,71 +71,198 @@ It updates:
- NVIDIA DRM KMS structures and function signatures
- compatibility aliases for kernels older than 7.2

The underlying Linux change was made by Maxime Ripard. The motivation was to distinguish the device-level atomic commit object from the per-object state structures.

### 0108-cachyos.patch — removal of kernel `strncpy()`

Linux 7.2 removed the generic kernel `strncpy()` API. The NVIDIA 390xx source still used it.

This patch changes the affected code paths to newer kernel string helpers:
This patch changes the affected code paths to:
- `strscpy()` where NUL-terminated copying is required
- `strscpy_pad()` where the original padding semantics matter

The affected NVIDIA source areas are:
Affected source areas:
- `nvidia/nv-gpu-numa.c`
- `nvidia/os-interface.c`
- `nvidia-uvm/uvm8_pmm_gpu.c`
- `nvidia-modeset/nvidia-modeset-linux.c`

The underlying Linux change was part of Kees Cook's `strncpy` removal work.
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

## Validation
```
sudo mkdir -p /etc/dkms/nvidia/patches
sudo cp 0107-cachyos.patch 0108-cachyos.patch /etc/dkms/nvidia/patches/
```

If applying from a clone of this repository, run the commands from the repository directory.

### 2. Install the DKMS override

Create:

Both patches were tested with:
`/etc/dkms/nvidia-390.157.conf`

with:

```
patch --dry-run -p1
PATCH[0]="0107-cachyos.patch"
PATCH_MATCH[0]="^7\.2\."

PATCH[1]="0108-cachyos.patch"
PATCH_MATCH[1]="^7\.2\."

MAKE[1]="'make' -j`nproc` LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=\${kernelver} modules"
MAKE_MATCH[1]="^7\.2\."
```

and applied successfully to a clean NVIDIA 390.157 test source tree.
Verify the configuration can be read:

```
dkms status
```

## Successful build
### 3. Build only for the target kernel

The CachyOS 7.2.8 kernel uses Clang/LLVM, LLD and ThinLTO. A GCC build failed because of LLVM-specific kernel flags and LTO objects. `CC=clang` alone also failed because an intermediate object was LLVM IR bitcode.
```
sudo dkms build -m nvidia -v 390.157 -k 7.2.8-1-cachyos
```

The successful build therefore used the complete LLVM toolchain:
Expected relevant output:

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
Applying patch 0107-cachyos.patch... done.
Applying patch 0108-cachyos.patch... done.
Building module(s)... done.
```

Generated successfully:
### 4. Install only for the target kernel

```
sudo dkms install -m nvidia -v 390.157 -k 7.2.8-1-cachyos
```

- `nvidia.ko`
- `nvidia-modeset.ko`
- `nvidia-uvm.ko`
- `nvidia-drm.ko`
Verify:

See the **raw build log** captured from the successful build: [logs/build-7.2.8-success.log](logs/build-7.2.8-success.log). It contains the original build output rather than a reconstructed summary.
```
dkms status
sudo modinfo -F filename nvidia
```

## Important status
The target should show:

The **source patches and manual LLVM/LLD build are verified**.
```
nvidia/390.157, 7.2.8-1-cachyos, x86_64: installed
```

Permanent DKMS automation is **not yet claimed as complete**. DKMS 3.4.3 patch/override semantics still need to be verified before changing the installed DKMS configuration.
and the module path should be under:

The known-working LTS kernel `6.18.52-1-cachyos` must remain preserved during subsequent DKMS testing.
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

@@ -155,8 +277,15 @@ nvidia-390xx-cachyos-kernel-7.2/
    └── build-7.2.8-success.log
```

## Licensing / redistribution
## Important limitations

- This project targets NVIDIA 390.157 and the tested CachyOS 7.2.x environment.
- The runtime result is verified on kernel `7.2.8-1-cachyos`; it should not be interpreted as proof that every future 7.2.x kernel will remain compatible.
- The patches are compatibility adaptations; they do not make the legacy NVIDIA driver a modern driver.
- The repository does not redistribute NVIDIA proprietary source archives or binary modules.

## License / redistribution

This repository does **not** redistribute NVIDIA's proprietary source archive, binary modules, object files, or generated build artifacts.
This repository does not redistribute NVIDIA's proprietary source archive, binary modules, object files, or generated build artifacts.

It contains compatibility patches derived from publicly available RPM Fusion packaging work, plus documentation and build evidence for the CachyOS kernel 7.2.x environment.
