# NVIDIA 390xx DKMS on CachyOS Kernel 7.2

Patches and build documentation for NVIDIA 390.157 on CachyOS kernel 7.2.x.

## Why this project exists

NVIDIA 390.157 is a legacy proprietary driver. On CachyOS kernel 7.2.8-1-cachyos, the unmodified source fails because Linux 7.2 removed kernel strncpy() and renamed the DRM atomic interface from struct drm_atomic_state to struct drm_atomic_commit.

The project provides the minimum compatibility layer required for the old driver to build against the tested CachyOS 7.2 kernel.

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

The patches are adapted from RPM Fusion's NVIDIA 390xx kmod patches released in nvidia-390xx-kmod-390.157-27.fc43.

Upstream patch files:
- nvidia-390xx-kmod-0107-adaptation-to-new-struct-drm-atomic-commit.patch
- nvidia-390xx-kmod-0108-kernel-7.2-remove-strncpy-kernel-function.patch

For CachyOS, the only source-tree adaptation was stripping the kernel/ path component so the patches apply with patch -p1. The repository patch files were byte-for-byte verified against the corresponding local RPM Fusion patch files.

### Patch checksums

NaN0107-cachyos.patch` — e918d44a27ffb6e7a36b5dc249d28114de79e56faa9456c6b389128f2b628d82

NaN0108-cachyos.patch` — 09f09f41e7d240c58aa67e43c58ba348345bf4cc2f199fe40e960712722c1544

### Primary sources

- RPM Fusion source RPM directory
- RPM Fusion 390.157-27 package information and changelog
- Linux commit 079a028d6327e68cfa5d38b36123637b321c19a7: removal of strncpy()
- Linux commit 5164f7e7ff8ec7d41065d3862630c2ba09854328: drm_atomic_state → drm_atomic_commit

## What each patch does

### 0107-cachyos.patch — DRM atomic commit API

Updates NVIDIA DRM/KMS code, conftest detection, atomic callbacks, allocation/cleanup wrappers, reference-counting detection, structures and function signatures for the Linux 7.2 rename from drm_atomic_state to drm_atomic_commit, while retaining older-kernel compatibility.

### 0108-cachyos.patch — removal of kernel strncpy()

Replaces affected uses with strscpy() or strscpy_pad() while retaining compatibility with older kernels. Affected files:
- nvidia/nv-gpu-numa.c
- nvidia/os-interface.c
- nvidia-uvm/uvm8_pmm_gpu.c
- nvidia-modeset/nvidia-modeset-linux.c

## Original build failure

Stock NVIDIA 390.157 failed in nvidia/nv-gpu-numa.c with:

NaNerror: call to undeclared library function 'strncpy'`
NaNISO C99 and later do not support implicit function declarations`

Adding linux/string.h did not fix the Linux 7.2 API removal, so the source was restored clean before applying the compatibility patch.

## Compiler/toolchain investigation

The CachyOS kernel uses LLVM-specific flags including -mstack-alignment=8, -mretpoline-external-thunk, -fexperimental-late-parse-attributes, -fsplit-lto-unit, -mllvm, -fdebug-info-for-profiling and -improved-fs-discriminator=true.

GCC therefore failed because of LLVM-specific kernel flags. CC=clang alone also failed because nvidia/nv-frontend.o was LLVM IR bitcode. The successful build required the complete LLVM/LLD path.

## Patch validation

Both patches were dry-run and then actually applied to a clean NVIDIA 390.157 source tree with patch --dry-run -p1 and patch -p1. The resulting source was checked for drm_atomic_commit and the expected strscpy/strscpy_pad changes.

## Successful manual build

NaNcd /tmp/nvidia-390.157-test`

NaNmake clean`

NaNmake -j$(nproc) LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 KERNEL_UNAME=7.2.8-1-cachyos modules`

Generated successfully: nvidia.ko, nvidia-modeset.ko, nvidia-uvm.ko, nvidia-drm.ko.

The complete raw build output is preserved in logs/build-7.2.8-success.log.

## DKMS configuration

DKMS 3.4.3 supports kernel-specific PATCH/PATCH_MATCH and MAKE/MAKE_MATCH overrides.

Files:
- /etc/dkms/nvidia/patches/0107-cachyos.patch
- /etc/dkms/nvidia/patches/0108-cachyos.patch
- /etc/dkms/nvidia-390.157.conf

Exact override:

NaNPATCH[0]="0107-cachyos.patch"`
NaNPATCH_MATCH[0]="^7\.2\."`

NaNPATCH[1]="0108-cachyos.patch"`
NaNPATCH_MATCH[1]="^7\.2\."`

NaNMAKE[1]="'make' -j`nproc` LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=${kernelver} modules"`
NaNMAKE_MATCH[1]="^7\.2\."`

The 7.2 regex matches 7.2.8-1-cachyos and does not match 6.18.52-1-cachyos-lts, preserving the working LTS registration.

## DKMS build and installation verification

NaNsudo dkms build -m nvidia -v 390.157 -k 7.2.8-1-cachyos`

Verified: both patches applied, module build completed, and all four modules were signed by DKMS.

NaNsudo dkms install -m nvidia -v 390.157 -k 7.2.8-1-cachyos`

Verified installation placed the compressed modules under /lib/modules/7.2.8-1-cachyos/updates/dkms/ and depmod completed successfully.

Final DKMS status:
NaNnvidia/390.157, 6.18.52-1-cachyos-lts, x86_64: installed`
NaNnvidia/390.157, 7.2.8-1-cachyos, x86_64: installed`

modinfo confirmed NVIDIA 390.157 and the DKMS module signing key.

## Post-reboot runtime verification

After rebooting into the target kernel:

NaNuname -r`
NaN7.2.8-1-cachyos`

NaNnvidia-smi`
NaNNVIDIA-SMI 390.157`
NaNDriver Version: 390.157`
NaNGPU: GeForce GT 720M`
NaNMemory: 0MiB / 1985MiB`

Loaded modules:
NaNnvidia_drm`
NaNnvidia_modeset`
NaNnvidia_uvm`
NaNnvidia`

This is a real post-reboot runtime test, not only a compilation or installation test.

## Build warnings

- multiple objtool: data relocation to !ENDBR warnings
- WARNING: modpost: missing MODULE_DESCRIPTION() in nvidia-uvm.o

These warnings did not prevent module generation, installation, loading, or nvidia-smi verification on the tested system.

## Verified final state

NVIDIA 390.157 successfully builds, installs through DKMS, loads after reboot, and is detected by nvidia-smi on CachyOS kernel 7.2.8-1-cachyos. The known-working 6.18.52-1-cachyos-lts DKMS registration remains installed.

## Applying the fix to another system

For the tested CachyOS configuration: copy both patches to /etc/dkms/nvidia/patches/, create /etc/dkms/nvidia-390.157.conf with the override above, build and install for the target 7.2.x kernel, reboot, then verify uname -r, lsmod and nvidia-smi.

Do not assume the LLVM/LLD command is required by other distributions; it reflects the tested CachyOS kernel toolchain.

## Reproducibility

The result was reproduced from a clean NVIDIA 390.157 source tree by applying both patches, building with the complete LLVM/LLD toolchain, registering and installing through DKMS, rebooting, and verifying the running driver with nvidia-smi. The raw build log is preserved.

## Repository contents

NaNnvidia-390xx-cachyos-kernel-7.2/`
NaN├── README.md`
NaN├── FULL-REPORT.md`
NaN├── patches/`
NaN│   ├── 0107-cachyos.patch`
NaN│   └── 0108-cachyos.patch`
NaN└── logs/`
NaN    └── build-7.2.8-success.log`

## Important limitations

- Targets NVIDIA 390.157 and the tested CachyOS 7.2.x environment.
- Runtime is verified on 7.2.8-1-cachyos; this is not proof for every future 7.2.x kernel.
- The patches are compatibility adaptations, not a modernization of the legacy driver.
- The repository does not redistribute NVIDIA proprietary source archives, binary modules, object files, or generated proprietary build artifacts.

## License / redistribution

This repository contains compatibility patches derived from publicly available RPM Fusion packaging work, documentation, and build evidence. It does not redistribute NVIDIA proprietary source archives or binary modules.