# NVIDIA 390.157 on CachyOS Kernel 7.2.8 — Final Runtime Validation

## Executive summary

This report records the completed investigation of NVIDIA 390.157 on a Dell Latitude E5440 with a GeForce GT 720M (PCI ID 10de:1140), running CachyOS kernel 7.2.8.

The investigation has two distinct results:

1. **Kernel compatibility and NVIDIA driver operation are solved and verified.**
2. **Native X11 NVIDIA OpenGL is solved and verified.**
3. **The remaining failure is specifically the NVIDIA GLX/PRIME path through KDE Wayland/Xwayland.**

The evidence no longer supports describing NVIDIA 390.157 as generally broken on kernel 7.2.8.

---

## 1. Hardware and software

| Component | Confirmed value |
|---|---|
| Laptop | Dell Latitude E5440 |
| CPU | Intel Core i5-4310U |
| Integrated GPU | Intel HD Graphics 4400 |
| Discrete GPU | NVIDIA GeForce GT 720M |
| NVIDIA PCI ID | 10de:1140 |
| NVIDIA PCI address | 03:00.0 |
| OS | CachyOS |
| Desktop | KDE Plasma 6.7.4 |
| Target kernel | 7.2.8-1-cachyos |
| Known-working LTS | 6.18.52-1-cachyos |
| NVIDIA driver | 390.157 |
| DKMS | 3.4.3 |
| Graphics | Hybrid Intel + NVIDIA |

---

## 2. Original build problem

The original 390.157 source did not build against kernel 7.2.8.

Two kernel API changes were responsible:

### A. Removal of kernel strncpy()

The legacy source used strncpy() in several NVIDIA components. Kernel 7.2 removed the kernel API, requiring modern replacements such as strscpy() or strscpy_pad() according to the original semantics.

### B. DRM atomic API change

The kernel changed:

    struct drm_atomic_state

to:

    struct drm_atomic_commit

The legacy NVIDIA DRM code therefore needed an API compatibility patch.

---

## 3. Patches that solved kernel 7.2 compatibility

### Patch 0107

    0107-cachyos.patch

Based on Linux commit:

    5164f7e7ff8ec7d41065d3862630c2ba09854328

Purpose:

    drm: Rename struct drm_atomic_state to struct drm_atomic_commit

SHA256:

    e918d44a27ffb6e7a36b5dc249d28114de79e56faa9456c6b389128f2b628d82

### Patch 0108

    0108-cachyos.patch

Based on Linux commit:

    079a028d6327e68cfa5d38b36123637b321c19a7

Purpose:

    string: Remove strncpy() from the kernel

SHA256:

    09f09f41e7d240c58aa67e43c58ba348345bf4cc2f199fe40e960712722c1544

These two patches solved the source-level kernel 7.2 compatibility problems.

---

## 4. Successful LLVM build

CachyOS kernel 7.2.8 uses a Clang/LLVM-oriented toolchain. The successful NVIDIA build used LLVM/Clang/LLD throughout.

The successful build produced:

    nvidia.ko
    nvidia-modeset.ko
    nvidia-uvm.ko
    nvidia-drm.ko

The build stage is therefore confirmed working.

---

## 5. DKMS result

The patched NVIDIA 390.157 source was integrated into DKMS for kernel 7.2.x.

Confirmed installations:

    nvidia/390.157, 6.18.52-1-cachyos-lts, x86_64
    nvidia/390.157, 7.2.8-1-cachyos, x86_64

DKMS is therefore confirmed working.

---

## 6. Kernel and GPU verification

The running target kernel was:

    7.2.8-1-cachyos

PCI detection identified:

    03:00.0 3D controller
    NVIDIA 10de:1140
    Kernel driver in use: nvidia

Intel remained bound to i915.

The NVIDIA runtime test:

    nvidia-smi --query-gpu=name,driver_version --format=csv,noheader

returned:

    GeForce GT 720M, 390.157

This proves that the NVIDIA kernel driver communicates with the physical GT 720M.

---

## 7. Initial DRM/KMS failure

The initial value was:

    sudo cat /sys/module/nvidia_drm/parameters/modeset

Result:

    N

Therefore:

    nvidia_drm.modeset = 0

The kernel journal then reported:

    Failed to initialize the nv-hotplug-helper DRM client
    (ensure DRM kernel mode setting is enabled via nvidia-drm.modeset=1).

followed by NVIDIA DRM unloading.

The observed sequence was:

    nvidia-drm loads
        ↓
    GPU is recognized
        ↓
    nv-hotplug-helper fails
        ↓
    nvidia-drm unloads

This was the key DRM/KMS failure.

---

## 8. Runtime KMS solution

The module was tested with modeset enabled at runtime rather than immediately modifying the permanent boot configuration.

After loading nvidia_drm with modeset=1:

    sudo cat /sys/module/nvidia_drm/parameters/modeset

returned:

    Y

The NVIDIA DRM nodes appeared:

    /dev/dri/card0
    /dev/dri/renderD129

while Intel remained:

    /dev/dri/card1
    /dev/dri/renderD128

Device identification proved:

    card0 = NVIDIA 10de:1140
    card1 = Intel

The kernel journal changed from the previous failure to:

    Registered the nv-hotplug-helper DRM client.

This is direct evidence that modeset=1 fixes the specific nv-hotplug-helper registration failure observed with modeset=0.

---

## 9. Fresh Wayland verification

After recreating the graphical session:

    echo $XDG_SESSION_TYPE

returned:

    wayland

and:

    sudo cat /sys/module/nvidia_drm/parameters/modeset

returned:

    Y

NVIDIA remained available:

    GeForce GT 720M, 390.157

Therefore the KMS state was active in the fresh session.

---

## 10. Wayland OpenGL result

Default OpenGL in Wayland remained Intel:

    OpenGL vendor string: Intel
    OpenGL renderer string: Mesa Intel(R) HD Graphics 4400 (HSW GT2)

This is not itself an error on a hybrid system.

However, explicit NVIDIA PRIME offload was tested:

    env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B

The result was:

    X_GLXCreateNewContext
    BadValue

The same failure occurred when NVIDIA GLX was explicitly selected without PRIME offload.

Therefore the NVIDIA GLX path through Wayland/Xwayland remained non-functional.

---

## 11. NVIDIA GLX libraries are present

The NVIDIA GLX library chain was verified:

    /usr/lib/libGLX_nvidia.so
        -> libGLX_nvidia.so.0
        -> libGLX_nvidia.so.390.157

The NVIDIA EGL vendor file also exists:

    /usr/share/glvnd/egl_vendor.d/10_nvidia.json

Therefore the Wayland failure is not simply caused by a missing NVIDIA GLX library.

---

## 12. Native X11 isolation test

The same installation was then tested in native KDE X11.

    echo $XDG_SESSION_TYPE

returned:

    x11

No driver rebuild or package replacement was performed for this test.

This isolates the compositor/display-server layer from the underlying NVIDIA installation.

---

## 13. Native X11 NVIDIA OpenGL — VERIFIED

The decisive command was:

    __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B

It succeeded with:

    direct rendering: Yes

    Dedicated video memory: 2048 MB

    OpenGL vendor string: NVIDIA Corporation
    OpenGL renderer string: GeForce GT 720M/PCIe/SSE2

    OpenGL core profile version string: 4.6.0 NVIDIA 390.157
    OpenGL version string: 4.6.0 NVIDIA 390.157
    OpenGL ES profile version string: OpenGL ES 3.2 NVIDIA 390.157

This is direct proof of real hardware-accelerated OpenGL rendering through the GT 720M with NVIDIA 390.157 under native X11.

---

## 14. Final test matrix

| Test | Result |
|---|---|
| CachyOS kernel 7.2.8 | PASS |
| 390.157 source compatibility | PASS |
| Patch 0107 | PASS |
| Patch 0108 | PASS |
| LLVM/Clang/LLD build | PASS |
| DKMS | PASS |
| NVIDIA kernel modules | PASS |
| GT 720M PCI detection | PASS |
| nvidia-smi | PASS |
| DRM modeset=0 | nv-hotplug-helper failure |
| DRM modeset=1 | PASS |
| NVIDIA DRM device node | PASS |
| nv-hotplug-helper with modeset=1 | PASS |
| Fresh KDE Wayland | PASS |
| Wayland default OpenGL | Intel |
| Wayland/Xwayland NVIDIA GLX offload | FAIL |
| Native KDE X11 | PASS |
| Native X11 NVIDIA GLX | PASS |
| Native X11 GT 720M hardware OpenGL | PASS |

---

## 15. What is solved

The following are confirmed:

- NVIDIA 390.157 kernel 7.2 compatibility.
- DRM atomic API compatibility.
- strncpy() compatibility.
- LLVM/Clang/LLD build.
- DKMS build and installation.
- NVIDIA kernel-module loading.
- GT 720M detection.
- nvidia-smi communication.
- NVIDIA DRM initialization with modeset=1.
- nv-hotplug-helper registration with modeset=1.
- Hardware-accelerated NVIDIA OpenGL under native X11.

---

## 16. What remains unresolved

The remaining failure is specifically:

    KDE Wayland
        ↓
    Xwayland
        ↓
    GLX
        ↓
    NVIDIA 390.157
        ↓
    X_GLXCreateNewContext / BadValue

This should be investigated as a separate Wayland/Xwayland compatibility issue.

The evidence does not justify describing the NVIDIA kernel driver itself as broken.

---

## 17. Verified working solution

For this exact Dell Latitude E5440 / GT 720M configuration, the directly validated NVIDIA graphics path is:

    CachyOS 7.2.8
        +
    NVIDIA 390.157
        +
    nvidia_drm.modeset=1
        +
    KDE Plasma X11
        +
    NVIDIA GLX

Verification:

    echo $XDG_SESSION_TYPE

must return:

    x11

Then:

    __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B

should report:

    OpenGL vendor string: NVIDIA Corporation
    OpenGL renderer string: GeForce GT 720M/PCIe/SSE2

---

## 18. Permanent modeset configuration

The investigation deliberately did not modify the permanent boot configuration.

The runtime test established causality:

    modeset=0
        → nv-hotplug-helper failure

    modeset=1
        → nv-hotplug-helper registered successfully

Therefore modeset=1 is the setting that should be made persistent for this configuration when the user is ready to configure the boot/module layer.

The exact permanent method depends on the boot configuration and should be applied only after the runtime behavior has been verified.

---

## 19. Important interpretation of the Wayland errors

During the first runtime module reload, KWin/Xwayland already had open file descriptors referring to the previous DRM state.

Messages such as:

    driver (null)
    Failed to open drm node
    couldn't find dev node for drm device
    Xlib: extension "NV-GLX" missing

appeared immediately around the module reload and therefore should not be used as the primary evidence for a fresh Wayland session.

The later fresh Wayland test is more meaningful: modeset remained Y, NVIDIA DRM remained available, but NVIDIA GLX creation still returned BadValue.

---

## 20. Why native X11 changes the diagnosis

The same:

- GPU
- kernel
- NVIDIA driver
- DKMS installation
- GLVND libraries
- 390.157 userspace

successfully created:

    OpenGL vendor: NVIDIA Corporation
    OpenGL renderer: GeForce GT 720M/PCIe/SSE2
    OpenGL 4.6.0 NVIDIA 390.157

under native X11.

Therefore the failure is not a general inability of NVIDIA 390.157 to render on this GPU.

It is isolated to the Wayland/Xwayland graphics path.

---

## 21. Minimal verification procedure

### Kernel

    uname -r

Expected:

    7.2.8-1-cachyos

### NVIDIA

    nvidia-smi --query-gpu=name,driver_version --format=csv,noheader

Expected:

    GeForce GT 720M, 390.157

### KMS

    sudo cat /sys/module/nvidia_drm/parameters/modeset

Expected for the verified working KMS state:

    Y

### Native X11

Enter Plasma X11:

    echo $XDG_SESSION_TYPE

Expected:

    x11

Then:

    __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B

Expected:

    OpenGL vendor string: NVIDIA Corporation
    OpenGL renderer string: GeForce GT 720M/PCIe/SSE2

---

## 22. Final technical conclusion

The NVIDIA 390.157 compatibility work for CachyOS kernel 7.2.8 is complete and verified.

The driver can be:

    patched
      ↓
    compiled with LLVM
      ↓
    installed through DKMS
      ↓
    loaded into kernel 7.2.8
      ↓
    bound to the GT 720M
      ↓
    accessed through nvidia-smi
      ↓
    initialized through nvidia-drm with modeset=1
      ↓
    used for hardware-accelerated OpenGL under native X11

The remaining problem is:

    Wayland/Xwayland + NVIDIA 390.157 GLX offload

which returns:

    X_GLXCreateNewContext
    BadValue

### Final project status

**NVIDIA 390.157 works on CachyOS kernel 7.2.8. The GT 720M is verified to provide hardware-accelerated OpenGL under native X11. NVIDIA DRM/KMS requires modeset=1 for successful nv-hotplug-helper registration. The remaining unresolved issue is NVIDIA GLX/PRIME integration through KDE Wayland/Xwayland.**

---

## 23. Related repository files

This report should be read alongside the existing historical investigation/reference report:

- NVIDIA-390.157-COMPLETE-INVESTIGATION-REFERENCE.md
- 0107-cachyos.patch
- 0108-cachyos.patch
- logs/build-7.2.8-success.log

This file is intentionally focused on the final runtime validation and the clean separation between:
1. kernel/module compatibility,
2. DRM/KMS,
3. native X11 NVIDIA rendering,
4. Wayland/Xwayland NVIDIA rendering.
