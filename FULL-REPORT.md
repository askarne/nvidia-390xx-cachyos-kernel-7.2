# NVIDIA 390xx DKMS on CachyOS Kernel 7.2.8 — Full Technical Report

## 1. Objective

Document the investigation and reproducible DKMS integration required to make NVIDIA 390xx (390.157) build and load against CachyOS kernel 7.2.8 while preserving the known-working CachyOS LTS kernel.

The final state is now verified through a complete sequence:

1. identify the Linux 7.2 source incompatibilities;
2. adapt and validate the two RPM Fusion compatibility patches;
3. determine the required LLVM/LLD toolchain;
4. configure DKMS with kernel-specific patches and make variables;
5. build with DKMS;
6. install the resulting modules;
7. reboot into kernel 7.2.8;
8. verify the NVIDIA driver and all four modules at runtime.

## 2. System under test

- Laptop: Dell Latitude E5440
- GPU: NVIDIA GeForce GT 720M
- Distribution: CachyOS
- Desktop: KDE Plasma 6.7.4
- Session: Wayland
- Target kernel: `7.2.8-1-cachyos`
- Known-working LTS kernel: `6.18.52-1-cachyos-lts`
- NVIDIA legacy package: `nvidia-390xx-dkms 390.157-25`
- NVIDIA utils: `390.157-25`
- DKMS: `3.4.3`
- Kernel compiler: Clang 22.1.8
- Linker: LLD

The LTS kernel was intentionally preserved throughout the investigation.

## 3. Kernel build characteristics

Kernel 7.2.8 was built with LLVM/Clang rather than GCC:

- `CONFIG_CC_IS_CLANG=y`
- `CONFIG_LD_IS_LLD=y`
- `CONFIG_LTO=y`
- `CONFIG_LTO_CLANG=y`
- `CONFIG_LTO_CLANG_THIN=y`
- Clang 22.1.8
- LLD

The kernel's LTO configuration is significant because the NVIDIA module build contains LLVM IR objects.

## 4. Original DKMS failure

The stock NVIDIA 390.157 source failed to compile against kernel 7.2.8. The first decisive compiler error was:

    nvidia/nv-gpu-numa.c:161:5: error: call to undeclared library function 'strncpy'
    ISO C99 and later do not support implicit function declarations

Adding `#include <linux/string.h>` did not resolve the failure, so the source was restored before applying the proper compatibility patch.

Other affected source files were identified:

- `nvidia/nv-gpu-numa.c`
- `nvidia/os-interface.c`
- `nvidia-uvm/uvm8_pmm_gpu.c`
- `nvidia-modeset/nvidia-modeset-linux.c`

## 5. Patch provenance

RPM Fusion's NVIDIA 390xx kmod source RPM was inspected:

    /tmp/nvidia-390xx-kmod-390.157-27.fc43.src.rpm

Two relevant patches were identified:

1. `nvidia-390xx-kmod-0107-adaptation-to-new-struct-drm-atomic-commit.patch`
2. `nvidia-390xx-kmod-0108-kernel-7.2-remove-strncpy-kernel-function.patch`

For the CachyOS NVIDIA source tree, only the `kernel/` path component was stripped from patch paths. The resulting repository files are:

- `patches/0107-cachyos.patch`
- `patches/0108-cachyos.patch`

The repository versions were byte-for-byte verified against the corresponding local RPM Fusion patch files.

## 6. Patch 0107 — DRM atomic commit API

Patch 0107 adapts the NVIDIA 390xx DRM/KMS code to the Linux 7.2 rename from `struct drm_atomic_state` to `struct drm_atomic_commit`.

It updates:
- conftest detection;
- atomic check callbacks;
- allocation and cleanup wrappers;
- reference-counting detection;
- DRM/KMS structures and signatures;
- compatibility aliases for older kernels.

The underlying Linux commit is:

    5164f7e7ff8ec7d41065d3862630c2ba09854328

Title:

    drm: Rename struct drm_atomic_state to struct drm_atomic_commit

## 7. Patch 0108 — kernel `strncpy()` removal

Patch 0108 adapts the affected NVIDIA source to the removal of the kernel `strncpy()` API in Linux 7.2.

It uses:
- `strscpy()` for NUL-terminated copying;
- `strscpy_pad()` where padding semantics are required;
- conditional compilation to retain compatibility with older kernels.

Affected files:

- `nvidia/nv-gpu-numa.c`
- `nvidia/os-interface.c`
- `nvidia-uvm/uvm8_pmm_gpu.c`
- `nvidia-modeset/nvidia-modeset-linux.c`

The underlying Linux commit is:

    079a028d6327e68cfa5d38b36123637b321c19a7

## 8. Patch validation

Both patches were tested with:

    patch --dry-run -p1

and then applied successfully to a clean NVIDIA 390.157 test source tree.

The resulting source was inspected to confirm both adaptations were actually present.

## 9. Compiler/toolchain investigation

### 9.1 GCC attempt

A direct GCC build failed because the target CachyOS kernel uses LLVM-specific compiler flags and ThinLTO. Errors involved flags such as:

- `-mstack-alignment=8`
- `-mretpoline-external-thunk`
- `-fexperimental-late-parse-attributes`
- `-fsplit-lto-unit`
- `-mllvm`
- `-fdebug-info-for-profiling`
- `-improved-fs-discriminator=true`

### 9.2 Clang-only attempt

Using `CC=clang` alone was insufficient. Linking failed with:

    nvidia/nv-frontend.o: file not recognized: file format not recognized

Inspection showed that `nvidia/nv-frontend.o` was LLVM IR bitcode. Therefore the full LLVM/LLD toolchain had to be selected.

## 10. Successful manual build

The clean patched source was built with:

    cd /tmp/nvidia-390.157-test
    make clean

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

The build generated all four modules:

- `nvidia.ko`
- `nvidia-modeset.ko`
- `nvidia-uvm.ko`
- `nvidia-drm.ko`

The raw captured build output is preserved in:

    logs/build-7.2.8-success.log

## 11. Build warnings

The successful build emitted:

- multiple `objtool: data relocation to !ENDBR` warnings;
- `WARNING: modpost: missing MODULE_DESCRIPTION() in nvidia-uvm.o`.

The warnings did not stop module generation.

The final build stages included:

    MODPOST Module.symvers
    LD [M] nvidia.ko
    LD [M] nvidia-modeset.ko
    LD [M] nvidia-uvm.ko
    LD [M] nvidia-drm.ko
    BTF [M] nvidia-modeset.ko
    BTF [M] nvidia.ko
    BTF [M] nvidia-uvm.ko
    BTF [M] nvidia-drm.ko

## 12. DKMS integration

DKMS 3.4.3 was verified locally to support:

- `PATCH[]` with `PATCH_MATCH[]`;
- `MAKE[]` with `MAKE_MATCH[]`;
- patch files under `/etc/dkms/<module>/patches/`;
- version-specific override files such as `/etc/dkms/nvidia-390.157.conf`.

The final configuration used:

    /etc/dkms/nvidia/patches/0107-cachyos.patch
    /etc/dkms/nvidia/patches/0108-cachyos.patch
    /etc/dkms/nvidia-390.157.conf

The override contains:

    PATCH[0]="0107-cachyos.patch"
    PATCH_MATCH[0]="^7\.2\."

    PATCH[1]="0108-cachyos.patch"
    PATCH_MATCH[1]="^7\.2\."

    MAKE[1]="'make' -j`nproc` LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=\${kernelver} modules"
    MAKE_MATCH[1]="^7\.2\."

This is intentionally kernel-specific. The original `MAKE[0]` remains available for kernels that do not match `^7\.2\.`.

## 13. DKMS build verification

The following command was run:

    sudo dkms build -m nvidia -v 390.157 -k 7.2.8-1-cachyos

DKMS reported:

    Applying patch 0107-cachyos.patch... done.
    Applying patch 0108-cachyos.patch... done.
    Building module(s)... done.

It then signed all four modules:

    Signing module /var/lib/dkms/nvidia/390.157/build/nvidia.ko
    Signing module /var/lib/dkms/nvidia/390.157/build/nvidia-uvm.ko
    Signing module /var/lib/dkms/nvidia/390.157/build/nvidia-modeset.ko
    Signing module /var/lib/dkms/nvidia/390.157/build/nvidia-drm.ko

This is the critical proof that DKMS itself, rather than a manual build, applied the two patches and built the target kernel modules.

## 14. DKMS installation

The following command was then run:

    sudo dkms install -m nvidia -v 390.157 -k 7.2.8-1-cachyos

DKMS installed:

    /usr/lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia.ko.zst
    /usr/lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia-uvm.ko.zst
    /usr/lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia-modeset.ko.zst
    /usr/lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia-drm.ko.zst

and completed:

    Running depmod.... done.

The existing LTS registration remained installed.

## 15. Post-reboot runtime verification

The system was rebooted into the target kernel and the following tests were performed.

### Kernel

    uname -r

Result:

    7.2.8-1-cachyos

### NVIDIA driver

    nvidia-smi

Reported:

    NVIDIA-SMI 390.157
    Driver Version: 390.157
    GPU: GeForce GT 720M
    Memory: 0MiB / 1985MiB

This confirms that the 390.157 driver is not merely built or installed; it is operational after boot on kernel 7.2.8.

### Loaded modules

    lsmod | grep '^nvidia'

Reported the four expected modules:

    nvidia_drm
    nvidia_modeset
    nvidia_uvm
    nvidia

### DKMS state

After installation:

    nvidia/390.157, 6.18.52-1-cachyos-lts, x86_64: installed
    nvidia/390.157, 7.2.8-1-cachyos, x86_64: installed

### Installed module path

    sudo modinfo -F filename nvidia

Result:

    /lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia.ko.zst

### Module version and signing

    sudo modinfo nvidia | grep -E '^(version|filename|signer):'

Reported:

    filename: /lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia.ko.zst
    version: 390.157
    signer: DKMS module signing key

## 16. Final verified state

The tested machine now has both kernel registrations intact:

    6.18.52-1-cachyos-lts → NVIDIA 390.157 → installed
    7.2.8-1-cachyos       → NVIDIA 390.157 → installed and working

The target kernel successfully boots with the legacy NVIDIA driver loaded.

This establishes four separate levels of verification:

1. source compatibility;
2. manual LLVM/LLD build;
3. DKMS patch application and build;
4. post-reboot runtime operation.

## 17. Applying the fix to another system

The following procedure is intended for a system with NVIDIA 390.157 DKMS and a CachyOS 7.2.x kernel.

### Files

Copy these repository files:

    patches/0107-cachyos.patch
    patches/0108-cachyos.patch

to:

    /etc/dkms/nvidia/patches/

Create:

    /etc/dkms/nvidia-390.157.conf

with:

    PATCH[0]="0107-cachyos.patch"
    PATCH_MATCH[0]="^7\.2\."

    PATCH[1]="0108-cachyos.patch"
    PATCH_MATCH[1]="^7\.2\."

    MAKE[1]="'make' -j`nproc` LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=\${kernelver} modules"
    MAKE_MATCH[1]="^7\.2\."

Then:

    dkms status

Build the target kernel:

    sudo dkms build -m nvidia -v 390.157 -k <kernel-version>

Verify that DKMS reports both patches as applied.

Install:

    sudo dkms install -m nvidia -v 390.157 -k <kernel-version>

Reboot into that kernel and verify:

    uname -r
    nvidia-smi
    lsmod | grep '^nvidia'

### Important

Do not use:

    sudo dkms remove nvidia/390.157 --all

when an existing working kernel must be preserved.

Always scope build/install commands to the intended kernel.

## 18. Reproducibility statement

The following has been reproduced and verified:

1. Linux 7.2 compatibility failures in stock NVIDIA 390.157.
2. Identification of the relevant RPM Fusion adaptations.
3. Patch-path adaptation for the CachyOS NVIDIA source layout.
4. Dry-run and actual application of both patches.
5. Successful manual compilation with the full LLVM/LLD toolchain.
6. Successful DKMS build with both patches applied automatically.
7. Successful DKMS installation and module signing.
8. Successful boot into `7.2.8-1-cachyos`.
9. Successful runtime detection of the GeForce GT 720M by NVIDIA 390.157.
10. Successful loading of `nvidia`, `nvidia_uvm`, `nvidia_modeset`, and `nvidia_drm`.
11. Preservation of the working `6.18.52-1-cachyos-lts` DKMS registration.

The runtime result is verified specifically on `7.2.8-1-cachyos`. Future kernel releases may require additional compatibility changes.

## 19. Project contents

    nvidia-390xx-cachyos-kernel-7.2/
    ├── README.md
    ├── FULL-REPORT.md
    ├── patches/
    │   ├── 0107-cachyos.patch
    │   └── 0108-cachyos.patch
    └── logs/
        └── build-7.2.8-success.log

The project intentionally excludes NVIDIA proprietary source archives, binary modules, object files, extracted RPM/source trees, kernel build trees, and generated build artifacts.


---

# 20. Update after kernel 7.2.8-2

The machine was subsequently updated from `7.2.8-1-cachyos` to `7.2.8-2-cachyos` and rebooted.

The verified DKMS state is:

```text
nvidia/390.157, 6.18.52-1-cachyos-lts, x86_64: installed
nvidia/390.157, 7.2.8-2-cachyos, x86_64: installed
nvidia-390.157/patched: added
```

The first two lines are the working installed registrations. The third line is a stale/duplicate registration left from the earlier DKMS setup and is not required for the working driver.

During the kernel transaction, DKMS attempted to process that stale registration and reported:

```text
Error! Patch 0107-cachyos.patch as specified in dkms.conf cannot be
found in /var/lib/dkms/nvidia-390.157/patched/build/patches/.
```

The compatibility patches themselves were present at:

```text
/etc/dkms/nvidia/patches/0107-cachyos.patch
/etc/dkms/nvidia/patches/0108-cachyos.patch
```

Therefore this update failure belonged to the stale `nvidia-390.157/patched` DKMS registration and its expected patch-tree layout. It did not invalidate the compatibility patches or the normal `nvidia/390.157` registration.

The corrected documentation and recovery procedure are maintained in:

`NVIDIA-390.157-CURRENT-DKMS-UPDATE.md`

### Future-kernel verification status

Automatic success on a kernel newer than `7.2.8-2-cachyos` has not yet been experimentally verified. The next kernel update is the definitive test of whether the corrected DKMS configuration reapplies the patches automatically.
