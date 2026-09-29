# NVIDIA 390.157 on CachyOS Kernel 7.2 — Complete Fix & Recovery Guide

## Purpose

This guide provides a practical procedure for users who have:

- CachyOS / Arch-based Linux
- Linux kernel 7.2.x
- NVIDIA 390.157 legacy driver
- NVIDIA GeForce GT 720M or a similar 390.xx-supported GPU
- KDE Plasma
- Hybrid Intel + NVIDIA graphics

The guide covers:

1. Making NVIDIA 390.157 build on kernel 7.2.x.
2. Enabling NVIDIA DRM KMS with `modeset=1`.
3. Applying the KMS fix safely from a **Wayland session**.
4. Recovering safely through **TTY** when only Xorg is available.
5. Avoiding the known graphical freeze caused by dynamically loading `nvidia_drm modeset=1` from an active Xorg session.
6. Installing a KDE X11 session when the system has Wayland only.
7. Verifying that the NVIDIA GPU and OpenGL actually work.

> **Important:** This guide separates the kernel/driver fix from the Wayland/Xwayland GLX issue. Native X11 NVIDIA OpenGL was verified working with NVIDIA 390.157, while the tested Wayland/Xwayland NVIDIA GLX path remained unresolved.

---

# 1. Before starting

Check the kernel:

```fish
uname -r
```

For the validated setup, the result was:

```
7.2.8-1-cachyos
```

Check the GPU/driver:

```fish
nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
```

Expected:

```
GeForce GT 720M, 390.157
```

Check the current graphical session:

```fish
echo $XDG_SESSION_TYPE
```

Possible results:

```
wayland
```

or:

```
x11
```

---

# 2. IMPORTANT WARNING — Xorg users

## Do NOT do this inside an active Xorg session

Do **not** run:

```bash
sudo rmmod nvidia_drm
sudo modprobe nvidia_drm modeset=1
```

while KDE is actively running on Xorg.

On the tested system, dynamically loading `nvidia_drm` with `modeset=1` from an active Xorg session caused the graphical session to become unresponsive.

The important distinction is:

- `rmmod nvidia_drm` itself did not cause the observed freeze.
- The freeze occurred when running:
  
```bash
sudo modprobe nvidia_drm modeset=1
```

inside the active graphical Xorg session.

Therefore:

> **If you are currently on Xorg, use the TTY procedure in Section 7 instead.**

---

# 3. Recommended path — Wayland session

If:

```fish
echo $XDG_SESSION_TYPE
```

returns:

```
wayland
```

you can perform the runtime KMS test from the current session.

First check the current state:

```fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

If it returns:

```
N
```

continue.

If it already returns:

```
Y
```

the KMS runtime fix is already active; skip to Section 5.

---

# 4. Enable NVIDIA DRM KMS from Wayland

From the Wayland session:

```bash
sudo rmmod nvidia_drm
```

Then:

```bash
sudo modprobe nvidia_drm modeset=1
```

Verify:

```fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

Expected:

```
Y
```

Then verify the NVIDIA DRM device:

```bash
ls -l /dev/dri/
```

You should see an additional NVIDIA DRM device such as:

```
card0
renderD129
```

The exact card/render numbers can differ between systems.

---

# 5. Verify that the NVIDIA DRM fix actually worked

Run:

```bash
nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
```

Expected:

```
GeForce GT 720M, 390.157
```

Then check the kernel log:

```bash
sudo dmesg | grep -iE 'nvidia-drm|hotplug-helper'
```

The successful state should contain:

```
Registered the nv-hotplug-helper DRM client.
```

The original failure looked like:

```
Failed to initialize the nv-hotplug-helper DRM client
(ensure DRM kernel mode setting is enabled via nvidia-drm.modeset=1).
```

Therefore:

```
modeset=0
    ↓
nv-hotplug-helper failure

modeset=1
    ↓
nv-hotplug-helper registered
```

---

# 6. IMPORTANT: runtime fix is not permanent

The command:

```bash
sudo modprobe nvidia_drm modeset=1
```

changes the module parameter for the current boot/session.

It does **not** automatically make the setting permanent.

Check:

```fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

After a reboot, it can return to:

```
N
```

unless `modeset=1` has been configured persistently.

The runtime procedure is therefore intended first as a **safe confirmation of the cause and solution**.

Do not change the bootloader until the runtime behavior has been verified.

---

# 7. Xorg-only users — safe TTY procedure

If you have only Xorg, or you are currently logged into Xorg, do **not** dynamically load `nvidia_drm modeset=1` from the running desktop.

Use a TTY instead.

## 7.1 Enter TTY

Press:

```
Ctrl + Alt + F3
```

If F3 does not work, try:

```
Ctrl + Alt + F2
```

or:

```
Ctrl + Alt + F4
```

You should get a text login screen.

Log in with your normal Linux username and password.

---

# 8. Stop the graphical login manager from TTY

First identify the display/login manager:

```bash
systemctl status display-manager
```

On some KDE systems it may be `sddm`, while newer KDE installations can use `plasmalogin`.

Stop the graphical login manager:

```bash
sudo systemctl stop display-manager
```

This closes the graphical session.

Do not run the NVIDIA module reload while the graphical server is still using it.

---

# 9. Load NVIDIA DRM with modeset=1 from TTY

Now run:

```bash
sudo rmmod nvidia_drm
```

Then:

```bash
sudo modprobe nvidia_drm modeset=1
```

Verify:

```bash
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

Expected:

```
Y
```

Then:

```bash
nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
```

Expected:

```
GeForce GT 720M, 390.157
```

Finally:

```bash
sudo dmesg | tail -n 100 | grep -iE 'nvidia-drm|hotplug-helper'
```

Look for:

```
Registered the nv-hotplug-helper DRM client.
```

---

# 10. Start KDE again

If the module loaded successfully, start the display manager:

```bash
sudo systemctl start display-manager
```

Then return to the graphical session.

The exact KDE session available depends on what is installed.

---

# 11. If the system has Wayland only

If the machine has only a Wayland KDE session and you want the verified native-X11 path, install the KDE X11 session from a terminal/TTY.

On CachyOS/Arch, first refresh package databases:

```bash
sudo pacman -Sy
```

Then search for the available Plasma X11 session package:

```bash
pacman -Ss plasma x11 session
```

On current Plasma installations, the package commonly used for the X11 session is:

```
plasma-x11-session
```

Install it:

```bash
sudo pacman -S plasma-x11-session
```

If the package is already installed, pacman will report that it is up to date.

> Package names can change across Arch/CachyOS generations. If `plasma-x11-session` is not found, use the search command above rather than installing an unrelated package.

After installation, restart the display manager:

```bash
sudo systemctl restart display-manager
```

Or simply log out and select the Plasma X11 session from the login screen.

---

# 12. Verify the X11 session

After logging into Plasma X11:

```fish
echo $XDG_SESSION_TYPE
```

Expected:

```
x11
```

Then test NVIDIA GLX:

```fish
__GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
```

A working NVIDIA result should contain:

```
direct rendering: Yes

OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: GeForce GT 720M/PCIe/SSE2
OpenGL version string: 4.6.0 NVIDIA 390.157
```

This is the strongest practical verification because it proves an actual NVIDIA OpenGL context was created.

---

# 13. Wayland users: do not confuse two different results

On a hybrid Intel/NVIDIA system, running:

```fish
glxinfo -B
```

from Wayland may show:

```
OpenGL vendor string: Intel
OpenGL renderer string: Mesa Intel(R) HD Graphics 4400
```

That does not mean the NVIDIA kernel driver is broken.

The explicit NVIDIA test is:

```fish
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
```

On the investigated system this returned:

```
X_GLXCreateNewContext
BadValue
```

The same happened with:

```fish
__GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
```

Therefore the tested NVIDIA GLX path through Wayland/Xwayland remains unresolved.

---

# 14. What is actually fixed

The following chain is verified:

```
NVIDIA 390.157
      ↓
kernel 7.2.8 compatibility patches
      ↓
LLVM/Clang build
      ↓
DKMS
      ↓
kernel module loading
      ↓
GT 720M detection
      ↓
nvidia-smi
      ↓
nvidia_drm.modeset=1
      ↓
nv-hotplug-helper registration
      ↓
native X11 NVIDIA OpenGL
```

The only unresolved part is:

```
Wayland
   ↓
Xwayland
   ↓
NVIDIA 390.157 GLX/PRIME
```

---

# 15. Kernel 7.2 source compatibility

If NVIDIA 390.157 does not build against kernel 7.2.x, the two compatibility areas investigated here are:

### Patch 0107

```
0107-cachyos.patch
```

DRM atomic API compatibility.

SHA256:

```
e918d44a27ffb6e7a36b5dc249d28114de79e56faa9456c6b389128f2b628d82
```

### Patch 0108

```
0108-cachyos.patch
```

strncpy() removal compatibility.

SHA256:

```
09f09f41e7d240c58aa67e43c58ba348345bf4cc2f199fe40e960712722c1544
```

See the repository's patch files for the exact changes.

---

# 16. Troubleshooting checklist

## Case A — Wayland

Check:

```fish
echo $XDG_SESSION_TYPE
```

If:

```
wayland
```

then the runtime test can be performed:

```bash
sudo rmmod nvidia_drm
sudo modprobe nvidia_drm modeset=1
```

Verify:

```fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

Expected:

```
Y
```

---

## Case B — Xorg

If:

```fish
echo $XDG_SESSION_TYPE
```

returns:

```
x11
```

**Do not** run the dynamic `modprobe nvidia_drm modeset=1` procedure from the active KDE Xorg desktop.

Use:

```
Ctrl + Alt + F3
```

then log in and stop the display manager before changing the NVIDIA DRM module.

---

## Case C — only Wayland installed

Install the KDE X11 session:

```bash
sudo pacman -S plasma-x11-session
```

Then restart the display manager or log out and select Plasma X11.

---

## Case D — NVIDIA GLX works on X11 but not Wayland

This is consistent with the results documented by this investigation.

Do not reinstall the driver merely because:

```
X_GLXCreateNewContext
BadValue
```

appears under Wayland.

First verify:

```fish
nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
```

and:

```fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

Then test native X11.

---

# 17. Recovery if the graphical session becomes frozen

If a user accidentally runs the module reload from Xorg and the desktop freezes:

1. Do not keep clicking the frozen desktop.
2. Press:

```
Ctrl + Alt + F3
```

3. Log in.
4. Check the display manager:

```bash
systemctl status display-manager
```

5. If necessary, restart it:

```bash
sudo systemctl restart display-manager
```

This will terminate the current graphical session, so unsaved graphical work may be lost.

If restarting the display manager is not sufficient, reboot from TTY:

```bash
sudo reboot
```

After reboot, do not repeat the runtime module reload from an active Xorg session.

---

# 18. Permanent configuration — after successful testing

Once the runtime KMS test is confirmed, the next step is to make:

```
nvidia_drm.modeset=1
```

persistent.

The exact method depends on the boot setup and initramfs configuration.

Do not blindly add multiple conflicting configuration files.

The recommended order is:

```
1. Build/patch driver
2. Confirm DKMS
3. Confirm nvidia-smi
4. Test modeset=1 at runtime
5. Confirm nv-hotplug-helper
6. Confirm native X11 OpenGL
7. Only then make modeset=1 persistent
```

---

# 19. Final verified result

For the tested Dell Latitude E5440:

```
GPU:
GeForce GT 720M

Driver:
NVIDIA 390.157

Kernel:
7.2.8-1-cachyos

KMS:
nvidia_drm.modeset=1

Verified OpenGL:
Native KDE X11

Wayland:
NVIDIA GLX/PRIME still unresolved
```

The most important safety rule in this guide is:

> **Wayland: runtime module reload can be tested from the session.**
>
> **Active Xorg: do not dynamically load nvidia_drm with modeset=1; use TTY after stopping the graphical login manager.**

This distinction prevents the Xorg graphical freeze observed during the investigation.
