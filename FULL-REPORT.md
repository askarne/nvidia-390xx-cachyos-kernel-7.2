# NVIDIA 390xx DKMS on CachyOS Kernel 7.2.8 — Full Technical Report

## 1. Objective

Document the investigation and reproducible build work required to make NVIDIA 390xx (390.157) build against CachyOS kernel 7.2.8 while preserving the known-working CachyOS LTS kernel.

This report records the observed failure, environment, source/patch provenance, compiler/toolchain findings, successful build procedure, warnings, and the remaining DKMS integration step. It does **not** claim permanent DKMS integration is complete.

## 2. System under test

- Laptop: Dell Latitude E5440
- Distribution: CachyOS
- Desktop: KDE Plasma 6.7.4
- Session: Wayland
- Target kernel: `7.2.8-1-cachyos`
- Known-working LTS kernel: `6.18.52-1-cachyos`
- NVIDIA legacy package: `nvidia-390xx-dkms 390.157-25`
- NVIDIA utils: `390.157-25`
- DKMS: `3.4.3`

The LTS kernel is intentionally preserved because NVIDIA 390xx DKMS is known to work there.

## 3. Kernel build characteristics

Kernel 7.2.8 was built with LLVM/Clang rather than GCC:

- `CONFIG_CC_IS_CLANG=y`
- `CONFIG_LD_IS_LLD=y`
- `CONFIG_LTO=y`
- `CONFIG_LTO_CLANG=y`
- `CONFIG_LTO_CLANG_THIN=y`
- Kernel compiler: Clang 22.1.8
- Linker: LLD

The compiler check used was:

    /usr/lib/modules/7.2.8-1-cachyos/build/scripts/cc-version.sh clang

which reported Clang 22.1.8.

## 4. Original DKMS failure

The stock NVIDIA 390.157 source failed to compile against kernel 7.2.8. The first decisive compiler error was:

    nvidia/nv-gpu-numa.c:161:5: error: call to undeclared library function 'strncpy'
    ISO C99 and later do not support implicit function declarations

A simple attempt to add `#include <linux/string.h>` did not resolve the failure. The package source was restored clean before applying the proper adaptation patch.

Other source files containing `strncpy` were identified:

- `nvidia/nv-gpu-numa.c`
- `nvidia/os-interface.c`
- `nvidia-uvm/uvm8_pmm_gpu.c`
- `nvidia-modeset/nvidia-modeset-linux.c`

## 5. Upstream patch source

RPM Fusion's NVIDIA 390xx kmod source RPM was inspected:

    /tmp/nvidia-390xx-kmod-390.157-27.fc43.src.rpm

It was extracted into:

    /tmp/nvidia-390xx-27

Two relevant patches were identified:

1. `nvidia-390xx-kmod-0107-adaptation-to-new-struct-drm-atomic-commit.patch`
2. `nvidia-390xx-kmod-0108-kernel-7.2-remove-strncpy-kernel-function.patch`

For the CachyOS NVIDIA source tree, only the `kernel/` path component was stripped from patch paths. The resulting patches are stored here as:

- `patches/0107-cachyos.patch`
- `patches/0108-cachyos.patch`

No NVIDIA proprietary source or binary is redistributed.

## 6. Patch validation

Both adapted patches were tested with `patch --dry-run -p1` and applied successfully to a clean test source tree:

    /tmp/nvidia-390.157-test

## 7. Compiler/toolchain investigation

### 7.1 GCC attempt

A direct GCC build failed because the target kernel was configured and built with Clang/LLVM/LLD/ThinLTO. GCC rejected LLVM/kernel-specific flags including:

- `-mstack-alignment=8`
- `-mretpoline-external-thunk`
- `-fexperimental-late-parse-attributes`
- `-fsplit-lto-unit`
- `-mllvm`
- `-fdebug-info-for-profiling`
- `-improved-fs-discriminator=true`

### 7.2 Clang-only attempt

Using `CC=clang` alone was insufficient. Linking failed with:

    nvidia/nv-frontend.o: file not recognized: file format not recognized

Inspection showed that `nvidia/nv-frontend.o` was LLVM IR bitcode. This demonstrated that the kernel's LTO configuration requires the LLVM/LLD toolchain consistently.

## 8. Successful build

The complete module build succeeded with:

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

The build completed successfully in approximately 4 minutes 27 seconds.

Final stages included:

    MODPOST Module.symvers
    LD [M] nvidia.ko
    LD [M] nvidia-modeset.ko
    LD [M] nvidia-uvm.ko
    LD [M] nvidia-drm.ko
    BTF [M] nvidia-modeset.ko
    BTF [M] nvidia.ko
    BTF [M] nvidia-uvm.ko
    BTF [M] nvidia-drm.ko

## 9. Raw build log

The complete raw output of the successful `7.2.8-1-cachyos` module build is preserved in:

    logs/build-7.2.8-success.log

This file is the original captured build output, not a manually reconstructed summary. It includes the compiler output, warnings, module linking, and final build stages.

## 10. Build warnings

The successful build still emitted:

- multiple `objtool: data relocation to !ENDBR` warnings
- `WARNING: modpost: missing MODULE_DESCRIPTION() in nvidia-uvm.o`

These warnings did not prevent module generation and are retained in the build log.

## 11. Existing DKMS configuration

The installed NVIDIA DKMS configuration contains the equivalent of:

    PACKAGE_NAME="nvidia"
    PACKAGE_VERSION="390.157"
    MAKE[0]="'make' -j`nproc` IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=${kernelver} modules"

Declared modules:

    BUILT_MODULE_NAME[0]="nvidia"
    BUILT_MODULE_NAME[1]="nvidia-uvm"
    BUILT_MODULE_NAME[2]="nvidia-modeset"
    BUILT_MODULE_NAME[3]="nvidia-drm"

## 12. DKMS integration status

DKMS version is `3.4.3`.

The successful manual source build proves that the source adaptations and LLVM/LLD toolchain are sufficient to compile NVIDIA 390.157 for kernel 7.2.8. It does **not** yet prove that DKMS can automatically apply these patches and select the required LLVM/LLD toolchain through a supported DKMS configuration mechanism.

No permanent DKMS override was installed before verifying the exact DKMS 3.4.3 patch/override semantics. This avoids modifying the working LTS installation or relying on an unverified directive.

The next technical step is to verify the installed DKMS 3.4.3 patch/override syntax and then perform a kernel-specific DKMS build/install for `7.2.8-1-cachyos`, without removing the working `6.18.52-1-cachyos` registration.

## 13. Preservation constraint

Do not use:

    sudo dkms remove nvidia/390.157 --all

because the LTS kernel must remain intact.

Subsequent DKMS tests should be scoped specifically to:

    7.2.8-1-cachyos

## 14. Project contents

    nvidia-390xx-cachyos-kernel-7.2/
    ├── README.md
    ├── FULL-REPORT.md
    ├── patches/
    │   ├── 0107-cachyos.patch
    │   └── 0108-cachyos.patch
    └── logs/
        └── build-7.2.8-success.log

The project intentionally excludes NVIDIA proprietary source archives, binary modules, object files, kernel build trees, extracted RPM/source trees, and generated build artifacts.

## 15. Reproducibility statement

At the current stage we have reproduced:

1. The kernel 7.2 compatibility failures.
2. Identification of the relevant upstream RPM Fusion adaptations.
3. Adaptation of their patch paths to the CachyOS NVIDIA source layout.
4. Dry-run validation of both patches.
5. Application of both patches to clean NVIDIA 390.157 source.
6. Successful construction of all four NVIDIA modules using the LLVM/LLD toolchain matching the CachyOS kernel's Clang + ThinLTO configuration.

Permanent DKMS automation remains a separate, unverified step and is deliberately not represented as finished.
