# NVIDIA 390xx on Arch Linux — Linux 7.2.x

Arch-specific development branch for adapting NVIDIA 390.157 to Linux 7.2.x.

> **Status: WIP / not runtime-verified on Arch Linux yet.**
>
> The working CachyOS result remains on the `main` branch. This branch exists to separate Arch Linux support from CachyOS-specific build assumptions.

## Why this branch exists

The original NVIDIA 390.157 source predates Linux 7.2 API changes. Two compatibility fixes are currently relevant:

1. Linux 7.2 removed the kernel `strncpy()` API.
2. Linux 7.2 renamed the DRM atomic interface from `drm_atomic_state` to `drm_atomic_commit`.

The compatibility patches already validated during the CachyOS investigation are included here as the starting point for Arch testing.

## Important distinction from CachyOS

CachyOS-specific assumptions must not automatically be treated as Arch Linux requirements.

In particular, the CachyOS test used:

- Clang
- LLD
- ThinLTO
- CachyOS-specific kernel configuration
- a CachyOS-specific kernel release

Therefore this branch does **not** currently hard-code the CachyOS LLVM/LLD build command.

The first Arch test should use the Arch kernel's normal DKMS build environment. If an Arch kernel requires a special compiler or linker configuration, that requirement should be added only after it is demonstrated by an actual Arch build failure.

## Patches

### 0107-cachyos.patch

Despite the historical filename, this patch addresses a Linux 7.2 DRM API change rather than a fundamentally CachyOS-only change.

It adapts NVIDIA 390xx DRM/KMS code for:

`drm_atomic_state` → `drm_atomic_commit`

### 0108-cachyos.patch

This patch adapts NVIDIA 390xx code to the Linux 7.2 removal of kernel `strncpy()`, using the appropriate newer string helpers while retaining compatibility with older kernels.

## Arch DKMS approach

The intended Arch configuration is to apply the patches conditionally to Linux 7.2.x:

```
PATCH[0]="0107-cachyos.patch"
PATCH_MATCH[0]="^7\\.2\\."

PATCH[1]="0108-cachyos.patch"
PATCH_MATCH[1]="^7\\.2\\."
```

The patches should be installed under:

```
/etc/dkms/nvidia/patches/
```

and the kernel-specific override under:

```
/etc/dkms/nvidia-390.157.conf
```

The existing default DKMS build command should remain untouched unless Arch testing demonstrates that it cannot build against the target Arch kernel.

## First Arch test

On an actual Arch Linux installation, record:

```
uname -r
pacman -Q linux nvidia-390xx-dkms dkms
cc --version
ld --version
```

Then install the patches and build specifically for the running kernel:

```
sudo mkdir -p /etc/dkms/nvidia/patches
sudo cp patches/0107-cachyos.patch patches/0108-cachyos.patch /etc/dkms/nvidia/patches/

sudo dkms build -m nvidia -v 390.157 -k "$(uname -r)"
```

The important evidence is whether DKMS reports:

```
Applying patch 0107-cachyos.patch... done.
Applying patch 0108-cachyos.patch... done.
Building module(s)... done.
```

If the build succeeds:

```
sudo dkms install -m nvidia -v 390.157 -k "$(uname -r)"
dkms status
sudo modinfo -F filename nvidia
```

Then reboot into the tested kernel and verify:

```
uname -r
nvidia-smi
lsmod | grep '^nvidia'
```

## What counts as Arch verification?

This branch should only be marked **verified** after an actual Arch Linux test demonstrates:

1. The target Arch kernel version.
2. Both patches apply successfully.
3. DKMS builds 390.157 without a fatal error.
4. The modules install successfully.
5. The NVIDIA modules load after reboot.
6. `nvidia-smi` detects the NVIDIA GPU.

A successful build alone is not enough to claim runtime support.

## Current evidence

The patches and their compatibility logic were validated on CachyOS kernel `7.2.8-1-cachyos`, where NVIDIA 390.157 was successfully built, installed through DKMS, loaded after reboot, and verified with `nvidia-smi`.

That evidence establishes the Linux 7.2 compatibility work, but **does not constitute Arch Linux verification**.

## Relationship to main

- `main` — verified CachyOS implementation.
- `arch` — Arch Linux adaptation/testing branch.

Do not use the CachyOS runtime result as evidence that Arch Linux is already supported.

## Redistribution

This repository does not redistribute NVIDIA proprietary source archives, binary modules, or generated proprietary build artifacts. The repository contains compatibility patches, configuration, documentation, and build evidence.
