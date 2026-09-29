# NVIDIA 390.157 on CachyOS 7.2 — Complete KMS/PRIME/X11 Solution

## 1. Executive Summary

This report documents the complete investigation and final runtime solution for the NVIDIA GeForce GT 720M using the legacy NVIDIA 390.157 driver on CachyOS kernel 7.2.8.

The project had two separate problems:

1. NVIDIA 390.157 initially required kernel 7.2 compatibility patches to compile successfully.
2. After the driver was successfully built and installed, NVIDIA DRM still failed at runtime because nvidia_drm kernel modesetting was disabled.

The final runtime fix was to reload nvidia_drm with modesetting enabled:

~~~fish
sudo rmmod nvidia_drm
sudo modprobe nvidia_drm modeset=1
~~~

After this:

- nvidia_drm reported modeset = Y.
- NVIDIA DRM initialized successfully.
- nv-hotplug-helper registered successfully.
- An NVIDIA DRM card appeared.
- An NVIDIA render node appeared.
- Both Intel and NVIDIA DRM devices were visible.
- PRIME GLX rendering worked after switching KDE Plasma to a real X11/Xorg session.

This report focuses on the complete runtime investigation and final solution. The kernel compatibility patches and successful DKMS build are documented separately in this repository.

---

## 2. Tested System

### Hardware

- Laptop: Dell Latitude E5440
- CPU: Intel Core i5-4310U
- Integrated GPU: Intel HD Graphics 4400
- NVIDIA GPU: GeForce GT 720M
- NVIDIA PCI ID: 10de:1140
- NVIDIA family reported by PCI: GF117M

### Software

- Distribution: CachyOS
- Desktop: KDE Plasma 6.7.4
- Kernel: 7.2.8-1-cachyos
- Known-working LTS kernel: 6.18.52-1-cachyos
- NVIDIA driver: 390.157
- Package: nvidia-390xx-dkms 390.157-25
- DKMS: 3.4.3

Sessions tested:

- KDE Plasma Wayland
- KDE Plasma X11/Xorg

---

## 3. Driver Installation Was Already Successful

The NVIDIA PCI device was correctly detected:

~~~text
03:00.0 3D controller [0302]: NVIDIA Corporation GF117M
[GeForce 610M/710M/810M/820M / GT 620M/625M/630M/720M] [10de:1140]

Kernel driver in use: nvidia
Kernel modules: nouveau, nvidia_drm, nvidia
~~~

nvidia-smi also communicated successfully with the GPU:

~~~text
NVIDIA-SMI 390.157
Driver Version: 390.157
GPU: GeForce GT 720M
Bus-Id: 00000000:03:00.0
Memory: 0MiB / 1985MiB
~~~

Therefore, the NVIDIA kernel driver was installed, loaded, and communicating with the GPU.

However, this alone did not prove that NVIDIA DRM or PRIME rendering was working.

---

## 4. Initial Runtime Failure

The critical parameter was checked with:

~~~fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
~~~

Result:

~~~text
N
~~~

So nvidia_drm was loaded with kernel modesetting disabled.

The module itself was present:

~~~text
filename: /lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia-drm.ko.zst
version: 390.157
parm: modeset:Enable atomic kernel modesetting (1 = enable, 0 = disable (default)) (bool)
~~~

This showed that the problem was not absence of the module. The important difference was its runtime configuration.

---

## 5. Decisive Kernel Log Evidence

The kernel journal showed:

~~~text
[drm] Initialized nvidia-drm 0.0.0 for 0000:03:00.0 on minor 0
Failed to initialize the nv-hotplug-helper DRM client
(ensure DRM kernel mode setting is enabled via nvidia-drm.modeset=1).
[nvidia-drm] ... Unloading driver
~~~

This was the decisive runtime error.

The NVIDIA DRM component started initialization, but the nv-hotplug-helper DRM client could not initialize while modesetting was disabled.

The result was that the NVIDIA DRM device did not remain available to the graphics stack.

---

## 6. DRM State Before the Fix

Before enabling modesetting:

~~~text
/dev/dri/
├── card1
├── renderD128
└── by-path/
    ├── pci-0000:00:02.0-card
    └── pci-0000:00:02.0-render
~~~

The only DRM card belonged to Intel:

~~~text
/sys/class/drm/card1/device/driver
    -> /sys/bus/pci/drivers/i915
~~~

There was no NVIDIA DRM card and no NVIDIA render node.

This explained why the active graphics stack exposed only Intel as a DRM rendering device.

---

## 7. OpenGL Before the Fix

glxinfo reported Intel rendering:

~~~text
direct rendering: Yes
Vendor: Intel
Device: Mesa Intel(R) HD Graphics 4400 (HSW GT2)
Accelerated: yes
OpenGL vendor string: Intel
OpenGL renderer string: Mesa Intel(R) HD Graphics 4400 (HSW GT2)
OpenGL version: Mesa 26.2.3
~~~

The NVIDIA driver was therefore present at the kernel level while the active OpenGL renderer was Intel.

---

## 8. PRIME Test Before the Fix

The PRIME offload environment was tested:

~~~fish
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
~~~

It failed with:

~~~text
X Error of failed request: BadValue
Major opcode: 152 (GLX)
Minor opcode: 24 (X_GLXCreateNewContext)
Value in failed request: 0x0
~~~

At this point the NVIDIA DRM device was still absent, so PRIME could not be considered functional.

---

# 9. Key Observation: nvidia_drm Could Be Reloaded Live

The module list showed:

~~~text
nvidia_drm     65536  0
nvidia_modeset 1347584  1 nvidia_drm
nvidia_uvm     1380352  0
nvidia         19787776 19 nvidia_uvm,nvidia_modeset
~~~

The nvidia_drm use count was zero.

This meant the module could be removed and reloaded without rebooting the system.

That provided a clean live test of whether modeset=1 was actually the missing runtime configuration.

---

# 10. Final Runtime Fix

The following commands were executed:

~~~fish
sudo rmmod nvidia_drm
sudo modprobe nvidia_drm modeset=1
~~~

The parameter was immediately verified:

~~~fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
~~~

Result:

~~~text
Y
~~~

This proved that the NVIDIA DRM module was now loaded with kernel modesetting enabled.

No reboot was required for this diagnostic/fix test.

---

# 11. Kernel Log After the Fix

The kernel log was checked immediately:

~~~fish
sudo journalctl -k --since "2 minutes ago" | grep -iE 'nvidia-drm|hotplug'
~~~

Result:

~~~text
[drm] [nvidia-drm] [GPU ID 0x00000300] Loading driver
[drm] Initialized nvidia-drm 0.0.0 for 0000:03:00.0 on minor 0
Registered the nv-hotplug-helper DRM client.
~~~

The difference was decisive.

### Before

~~~text
Failed to initialize the nv-hotplug-helper DRM client
[nvidia-drm] ... Unloading driver
~~~

### After

~~~text
Registered the nv-hotplug-helper DRM client.
~~~

The NVIDIA DRM initialization failure was therefore resolved by enabling modesetting.

---

# 12. DRM Devices After the Fix

After reloading nvidia_drm:

~~~text
/dev/dri/
├── card0
├── card1
├── renderD128
└── renderD129
~~~

The driver mappings were:

~~~text
/sys/bus/pci/drivers/nvidia
/sys/bus/pci/drivers/i915
~~~

The resulting device assignment was:

~~~text
card0      -> NVIDIA
card1      -> Intel
renderD128 -> Intel
renderD129 -> NVIDIA
~~~

This was direct evidence that the NVIDIA GPU had successfully registered as a DRM device.

Before the fix, only Intel DRM was exposed.

After the fix, both GPUs were exposed.

---

# 13. Wayland Test After the KMS Fix

The PRIME GLX test was repeated while still running KDE Plasma Wayland:

~~~fish
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
~~~

It still returned:

~~~text
X Error of failed request: BadValue
Major opcode: 150 (GLX)
Minor opcode: 24 (GLXCreateNewContext)
Value in failed request: 0x0
~~~

This did not invalidate the KMS fix.

By this point, the following facts had already been independently established:

1. nvidia_drm modeset changed from N to Y.
2. nv-hotplug-helper registered successfully.
3. NVIDIA DRM card0 appeared.
4. NVIDIA renderD129 appeared.

The remaining failure occurred while the desktop session was still Wayland and the test was using GLX.

Therefore, the KMS/DRM failure and the remaining Wayland GLX behavior were treated as separate issues.

No universal claim about NVIDIA 390.x and Wayland is made here; this report records only the behavior verified on the tested system.

---

# 14. Final X11/Xorg Test

The KDE Plasma session was then changed from Wayland to a real X11/Xorg session.

The session was verified as X11 and the same PRIME command was tested again:

~~~fish
echo $XDG_SESSION_TYPE
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
~~~

The PRIME test worked in the X11/Xorg session.

This was the final runtime confirmation that the NVIDIA 390.157 driver, NVIDIA DRM/KMS, and PRIME GLX rendering were working together on the tested hardware.

---

# 15. Final Working Architecture

~~~text
CachyOS
   |
   +-- Linux 7.2.8
   |
   +-- NVIDIA 390.157 DKMS
   |
   +-- nvidia_modeset
   |
   +-- nvidia_drm
          |
          +-- modeset=1
          |
          +-- nv-hotplug-helper registered
          |
          +-- DRM card0
          |
          +-- renderD129
   |
   +-- KDE Plasma X11/Xorg
          |
          +-- PRIME render offload
                 |
                 +-- GeForce GT 720M
~~~

Intel remains available independently:

~~~text
Intel HD Graphics 4400
    |
    +-- i915
    +-- DRM card1
    +-- renderD128
~~~

---

# 16. Root Cause

The complete runtime failure chain was:

~~~text
nvidia_drm loaded with modeset=0
        |
        v
NVIDIA DRM initialization begins
        |
        v
nv-hotplug-helper initialization fails
        |
        v
NVIDIA DRM functionality unloads
        |
        v
No NVIDIA DRM card/render node
        |
        v
Graphics stack exposes Intel DRM only
        |
        v
PRIME GLX cannot use NVIDIA correctly
~~~

After the fix:

~~~text
nvidia_drm loaded with modeset=1
        |
        v
NVIDIA DRM initialization succeeds
        |
        v
nv-hotplug-helper registers
        |
        v
NVIDIA DRM card/render node appears
        |
        v
NVIDIA becomes available to the graphics stack
        |
        v
PRIME rendering works in the tested X11/Xorg session
~~~

---

# 17. Complete Solution in Minimal Form

For a live diagnostic/fix test:

~~~fish
sudo rmmod nvidia_drm
sudo modprobe nvidia_drm modeset=1
~~~

Verify:

~~~fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
~~~

Expected:

~~~text
Y
~~~

Verify DRM:

~~~fish
ls -l /dev/dri/
readlink -f /sys/class/drm/card\*/device/driver
~~~

Both NVIDIA and Intel should be present.

For PRIME GLX testing, use a KDE Plasma X11/Xorg session:

~~~fish
echo $XDG_SESSION_TYPE
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
~~~

The renderer should report the NVIDIA GPU.

---

# 18. Relationship to the Kernel 7.2 Build Fix

There were two independent stages in this project.

## Stage A — Build compatibility

NVIDIA 390.157 required compatibility patches for the CachyOS 7.2 kernel series.

The repository contains:

- 0107-cachyos.patch
- 0108-cachyos.patch

The driver was successfully built and installed through DKMS using the documented LLVM/Clang configuration.

## Stage B — Runtime DRM/KMS

After the successful build and DKMS installation, the separate runtime problem remained:

~~~text
nvidia_drm.modeset = N
~~~

This caused:

~~~text
nv-hotplug-helper DRM client initialization failure
~~~

The runtime solution was:

~~~text
nvidia_drm.modeset = Y
~~~

followed by a successful NVIDIA DRM initialization and a successful PRIME test in X11/Xorg.

The build problem and the runtime KMS problem must therefore not be treated as the same failure.

---

# 19. Final Verified State

~~~text
Kernel:
    7.2.8-1-cachyos

GPU:
    GeForce GT 720M
    PCI ID 10de:1140

NVIDIA driver:
    390.157

DKMS:
    Successfully installed

nvidia_drm:
    Loaded successfully

nvidia_drm.modeset:
    Y after the live fix

nv-hotplug-helper:
    Registered successfully

NVIDIA DRM card:
    Present

NVIDIA render node:
    Present

Intel DRM:
    Present

Wayland:
    KMS/DRM fixed; the tested GLX offload command still failed in the Wayland session

X11/Xorg:
    PRIME GLX offload successfully tested
~~~

---

# 20. Conclusion

The investigation established that the NVIDIA 390.157 driver was not failing because of the kernel build once the required 7.2 compatibility patches were applied.

The remaining runtime problem was caused by NVIDIA DRM kernel modesetting being disabled.

The decisive live fix was:

~~~fish
sudo rmmod nvidia_drm
sudo modprobe nvidia_drm modeset=1
~~~

This changed the system from:

~~~text
modeset=0
    -> nv-hotplug-helper failure
    -> NVIDIA DRM unavailable
    -> Intel-only DRM exposure
    -> PRIME failure
~~~

to:

~~~text
modeset=1
    -> nv-hotplug-helper registered
    -> NVIDIA DRM available
    -> NVIDIA DRM/render nodes present
    -> PRIME works in X11/Xorg
~~~

The final verified working configuration on the tested Dell Latitude E5440 is therefore:

~~~text
NVIDIA 390.157
+
CachyOS 7.2.8 compatibility patches
+
nvidia_drm.modeset=1
+
KDE Plasma X11/Xorg
=
Working NVIDIA PRIME rendering on GeForce GT 720M
~~~

## 21. Persistence Note

The live modprobe test proves the required runtime setting, but this report does not claim that modeset=1 has already been made persistent across reboot.

Making the setting persistent should be treated as a separate finalization step. The live test intentionally established the exact runtime cause and solution before changing the permanent module or boot configuration.
