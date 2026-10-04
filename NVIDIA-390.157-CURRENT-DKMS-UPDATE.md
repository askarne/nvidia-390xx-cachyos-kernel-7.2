# NVIDIA 390.157 — Current CachyOS 7.2 DKMS State and Update Fix

> Current addendum for the repository. This file supersedes older kernel-version-specific examples where they conflict with the current verified state.

## 1. Current verified state

Tested hardware:

- Dell Latitude E5440
- NVIDIA GeForce GT 720M (PCI ID 10de:1140)
- Intel HD Graphics 4400

Current software state after the kernel update and reboot:

- CachyOS
- NVIDIA 390.157
- Kernel: `7.2.8-2-cachyos`
- LTS kernel: `6.18.52-1-cachyos-lts`

Verified DKMS state:

```text
nvidia/390.157, 6.18.52-1-cachyos-lts, x86_64: installed
nvidia/390.157, 7.2.8-2-cachyos, x86_64: installed
```

This means NVIDIA 390.157 is installed for both the known-working LTS kernel and the new CachyOS 7.2.8-2 kernel.

A stale DKMS registration was also observed:

```text
nvidia-390.157/patched: added
```

The `patched` entry is not the working installed driver. The working driver is the normal:

```text
nvidia/390.157
```

## 2. What happened during the kernel update

Before the update, NVIDIA 390.157 was installed for:

```text
7.2.8-1-cachyos
```

The update replaced that kernel with:

```text
7.2.8-2-cachyos
```

Pacman removed the old 7.2.8-1 DKMS registration and then attempted to build NVIDIA for the new kernel.

During the transaction, two DKMS operations appeared:

```text
dkms install --no-depmod nvidia/390.157 -k 7.2.8-2-cachyos
dkms install --no-depmod nvidia-390.157/patched -k 7.2.8-2-cachyos
```

The normal NVIDIA registration was eventually installed successfully.

The separate stale `nvidia-390.157/patched` registration failed because DKMS expected:

```text
/var/lib/dkms/nvidia-390.157/patched/build/patches/0107-cachyos.patch
```

but that directory did not exist.

The actual repository patches were present at the correct system location:

```text
/etc/dkms/nvidia/patches/0107-cachyos.patch
/etc/dkms/nvidia/patches/0108-cachyos.patch
```

Therefore the failed `patched` registration was a stale/duplicate DKMS state problem, not evidence that the compatibility patches themselves were invalid.

## 3. Correct DKMS architecture

There should be one working NVIDIA DKMS module registration:

```text
nvidia/390.157
```

The compatibility patches belong under:

```text
/etc/dkms/nvidia/patches/
```

with:

```text
0107-cachyos.patch
0108-cachyos.patch
```

The DKMS override must apply those patches only to the CachyOS 7.2 kernel series.

The relevant configuration is conceptually:

```text
PATCH[0]="0107-cachyos.patch"
PATCH_MATCH[0]="^7\\.2\\."

PATCH[1]="0108-cachyos.patch"
PATCH_MATCH[1]="^7\\.2\\."

MAKE[1]="'make' -j`nproc` LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=\${kernelver} modules"
MAKE_MATCH[1]="^7\\.2\\."
```

The important point is that the patches are attached to `nvidia/390.157`; a separate DKMS module version named `nvidia-390.157/patched` is not required.

## 4. Patches

The two compatibility patches are:

```text
patches/0107-cachyos.patch
patches/0108-cachyos.patch
```

Their purposes are unchanged:

### 0107-cachyos.patch

Adapts NVIDIA 390xx DRM/KMS code to the Linux 7.2 DRM atomic commit API changes.

### 0108-cachyos.patch

Adapts NVIDIA 390xx code to the Linux 7.2 removal of the kernel `strncpy()` API.

Both are required for the tested NVIDIA 390.157 / CachyOS 7.2 environment.

## 5. LLVM/LLD build requirement

The CachyOS 7.2 kernel used in this investigation is built with LLVM/Clang and LLD/ThinLTO.

The successful build therefore uses the full LLVM toolchain rather than GCC or only `CC=clang`:

```fish
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
  KERNEL_UNAME=7.2.8-2-cachyos \
  modules
```

For DKMS, the same LLVM/LLD settings are supplied through the kernel-specific `MAKE[]/MAKE_MATCH[]` override.

## 6. Safe manual recovery if a future kernel update fails

If a future kernel update produces a build failure, do not remove all NVIDIA DKMS versions blindly.

First identify the kernel:

```fish
uname -r
```

Then check:

```fish
dkms status
```

For a new kernel such as `7.2.8-3-cachyos`, build only that kernel:

```fish
sudo dkms build -m nvidia -v 390.157 -k 7.2.8-3-cachyos
```

If the build succeeds:

```fish
sudo dkms install -m nvidia -v 390.157 -k 7.2.8-3-cachyos
```

Then verify:

```fish
dkms status
```

The desired result is:

```text
nvidia/390.157, 6.18.52-1-cachyos, x86_64: installed
nvidia/390.157, 7.2.8-3-cachyos, x86_64: installed
```

Do not use:

```fish
sudo dkms remove nvidia/390.157 --all
```

when the LTS installation must remain available.

## 7. Cleaning the stale `patched` registration

The stale registration:

```text
nvidia-390.157/patched: added
```

is separate from the working:

```text
nvidia/390.157
```

It should be removed when cleaning up the old DKMS state, but the working `nvidia/390.157` registrations must not be deleted.

A cleanup should target only the stale registration, for example:

```fish
sudo dkms remove -m nvidia-390.157 -v patched --all
```

If DKMS reports that the stale entry does not exist, that is harmless; the important rule is not to remove `nvidia/390.157`.

## 8. Runtime NVIDIA verification

After booting a new kernel, the simple verification sequence is:

```fish
echo "=== KERNEL ==="
uname -r

echo "=== NVIDIA ==="
nvidia-smi

echo "=== DKMS ==="
dkms status
```

For the current verified system, the expected kernel is:

```text
7.2.8-2-cachyos
```

and `nvidia-smi` should report:

```text
NVIDIA-SMI 390.157
GeForce GT 720M
```

## 9. Runtime KMS/PRIME result

The build problem and the runtime DRM/KMS problem are separate.

The tested NVIDIA 390.157 installation initially loaded `nvidia_drm` with:

```text
modeset=N
```

Reloading it with:

```fish
sudo rmmod nvidia_drm
sudo modprobe nvidia_drm modeset=1
```

changed the parameter to:

```text
Y
```

and the kernel registered the NVIDIA DRM hotplug helper.

After this, both NVIDIA and Intel DRM devices appeared.

The PRIME GLX test:

```fish
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
```

worked in the tested KDE Plasma X11/Xorg session.

The same GLX test still failed in the tested Wayland session. This repository does not generalize that result to every NVIDIA 390.x/Wayland configuration; it records only the tested system.

## 10. What is proven now

The following is proven on the tested machine:

1. NVIDIA 390.157 can be built for CachyOS kernel 7.2 with the two compatibility patches.
2. The required LLVM/Clang/LLD build path works.
3. DKMS can install NVIDIA 390.157 for kernel 7.2.8-2.
4. The LTS NVIDIA registration remains installed.
5. After reboot, the new kernel and NVIDIA registration are installed.
6. The previous update failure was caused by a stale/duplicate `nvidia-390.157/patched` DKMS registration attempting to find patches in the wrong build tree.
7. The working registration is `nvidia/390.157`.
8. NVIDIA DRM/KMS requires the tested `modeset=1` runtime configuration for the successful DRM initialization observed here.
9. PRIME GLX rendering was successfully verified in the tested KDE Plasma X11/Xorg session.

## 11. What is not yet proven

Automatic recovery on a future kernel has **not yet been tested with a later kernel release**.

The next real test is simply:

```text
new CachyOS kernel
        ↓
Pacman/DKMS automatic rebuild
        ↓
0107 + 0108 applied automatically
        ↓
nvidia/390.157 installed
        ↓
reboot
        ↓
nvidia-smi works
```

Until that happens, the repository should describe future automatic updates as **expected from the corrected DKMS configuration, not as experimentally proven**.

## 12. Historical commands and lessons

During the investigation, the following diagnostic commands were useful:

### Detect NVIDIA driver

```fish
lspci -nnk | grep -A4 -i "NVIDIA"
```

### Check driver/GPU communication

```fish
nvidia-smi
```

### Check NVIDIA DRM modeset

```fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

### Check DRM devices

```fish
ls -l /dev/dri/
readlink -f /sys/class/drm/card\\*/device/driver
```

### Check the kernel log

```fish
sudo journalctl -k | grep -iE 'nvidia-drm|hotplug'
```

### Test PRIME in X11/Xorg

```fish
echo $XDG_SESSION_TYPE
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
```

The `prime-run` command was not available on the tested system, so the environment variables were used directly.

## 13. Important distinction

There are three different questions:

### A. Does the NVIDIA driver build?

Yes, with patches 0107/0108 and the LLVM/LLD DKMS configuration.

### B. Is the NVIDIA driver installed and communicating with the GPU?

Yes. `nvidia-smi` and `dkms status` establish this.

### C. Does NVIDIA PRIME rendering work?

Yes in the tested X11/Xorg session after NVIDIA DRM modesetting was enabled.

Wayland GLX offload remained unsuccessful in the tested environment and should not be silently presented as working.

## 14. Current repository structure

```text
nvidia-390xx-cachyos-kernel-7.2/
├── README.md
├── FULL-REPORT.md
├── NVIDIA-390.157-COMPLETE-INVESTIGATION-REFERENCE.md
├── NVIDIA-390.157-KMS-PRIME-X11-COMPLETE-SOLUTION.md
├── NVIDIA-390.157-CURRENT-DKMS-UPDATE.md
├── patches/
│   ├── 0107-cachyos.patch
│   └── 0108-cachyos.patch
└── logs/
    └── build-7.2.8-success.log
```

This file is the current update/DKMS addendum. Older reports remain as historical investigation records.
