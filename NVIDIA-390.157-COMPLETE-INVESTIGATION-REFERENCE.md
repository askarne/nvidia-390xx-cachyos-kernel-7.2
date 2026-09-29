
# NVIDIA 390.157 on CachyOS Kernel 7.2.8 — Complete Investigation Reference

> هذا الملف سجل تاريخي مستقل للتحقيق من تثبيت NVIDIA 390.157، وحل توافق kernel 7.2، وبناء DKMS، والتحقق بعد الإقلاع، ثم تشخيص DRM/KMS وPRIME حتى آخر نتيجة مؤكدة.
>
> **مهم:** هذا الملف مرجعي ولا يستبدل الملفات الموجودة في المستودع. لم يتم تعديل README.md أو FULL-REPORT.md أو patches أو logs.

---

## 1. الجهاز والبيئة

- Dell Latitude E5440
- Intel Core i5-4310U
- Intel HD Graphics 4400
- NVIDIA GeForce GT 720M
- NVIDIA PCI ID: 10de:1140
- CachyOS
- KDE Plasma 6.7.4
- Target kernel: 7.2.8-1-cachyos
- Known-working LTS: 6.18.52-1-cachyos
- NVIDIA driver: 390.157
- nvidia-390xx-dkms: 390.157-25
- DKMS: 3.4.3
- Clang: 22.1.8
- Linker: LLD
- Kernel configuration: Clang + ThinLTO

الجهاز Hybrid Graphics؛ Intel هي البطاقة المدمجة وGT 720M هي البطاقة المنفصلة.

---

# 2. الهدف

كان الهدف الأول تشغيل NVIDIA 390.157 على CachyOS kernel 7.2.8، ثم التأكد من أن البطاقة تعمل فعليًا وليس فقط أن ملفات التعريف مثبتة.

لذلك انقسم التحقيق إلى مرحلتين:

1. توافق المصدر والبناء مع kernel 7.2.
2. التشغيل الفعلي: kernel driver، DRM/KMS، Xorg، GLX وPRIME.

المرحلة الأولى حُلّت بنجاح. المرحلة الثانية كشفت مشكلة مستقلة في DRM/KMS/PRIME.

---

# 3. فشل البناء الأصلي

NVIDIA 390.157 الأصلي لم يكن يبنى على CachyOS kernel 7.2.8 بسبب تغييرات Linux 7.2.

المشكلتان الأساسيتان:

### 3.1 إزالة strncpy()

Linux 7.2 أزال kernel API المسماة strncpy()، بينما 390.157 ما زال يستخدمها.

المناطق المتأثرة:

~~~
kernel/nvidia/nv-gpu-numa.c
kernel/nvidia/os-interface.c
kernel/nvidia-uvm/uvm8_pmm_gpu.c
kernel/nvidia-modeset/nvidia-modeset-linux.c
~~~

ظهر الفشل بوضوح في nvidia/nv-gpu-numa.c.

### 3.2 تغيير DRM atomic API

تم تغيير:

~~~
struct drm_atomic_state
~~~

إلى:

~~~
struct drm_atomic_commit
~~~

وهذا يتطلب تكييف طبقة NVIDIA DRM/KMS.

---

# 4. patch 0107

تم استخدام:

~~~
nvidia-390xx-kmod-0107-adaptation-to-new-struct-drm-atomic-commit.patch
~~~

وهو مبني على Linux commit:

~~~
5164f7e7ff8ec7d41065d3862630c2ba09854328
~~~

بعنوان:

~~~
drm: Rename struct drm_atomic_state to struct drm_atomic_commit
~~~

الـpatch يحدث توافق NVIDIA 390xx DRM/KMS مع الواجهة الجديدة، مع إبقاء التوافق مع kernels الأقدم.

---

# 5. patch 0108

تم استخدام:

~~~
nvidia-390xx-kmod-0108-kernel-7.2-remove-strncpy-kernel-function.patch
~~~

وهو مبني على:

~~~
079a028d6327e68cfa5d38b36123637b321c19a7
~~~

بعنوان:

~~~
string: Remove strncpy() from the kernel
~~~

ويستبدل الاستخدامات المناسبة بـ:

~~~
strscpy()
strscpy_pad()
~~~

بحسب semantics النسخة الأصلية.

---

# 6. التحقق من الـpatches

تم التحقق من أن الـpatches المستخدمة مطابقة byte-for-byte لملفات RPM Fusion المحلية.

SHA256:

~~~
0107:
e918d44a27ffb6e7a36b5dc249d28114de79e56faa9456c6b389128f2b628d82

0108:
09f09f41e7d240c58aa67e43c58ba348345bf4cc2f199fe40e960712722c1544
~~~

التعديل الخاص بـCachyOS كان في path layout فقط، بحيث تطبق الملفات مع patch -p1.

---

# 7. مشكلة compiler/toolchain

بعد معالجة API incompatibilities، ظهر أن بناء NVIDIA باستخدام GCC الافتراضي لا ينسجم مع LLVM-specific flags في kernel CachyOS.

تم اختبار CC=clang وحده أيضًا، لكنه لم يكن كافيًا؛ ظهر تعارض في التعامل مع nvidia/nv-frontend.o وLLVM IR.

الحل كان استخدام مسار LLVM كاملًا.

---

# 8. البناء اليدوي الناجح

الأمر الذي نجح:

~~~fish
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
~~~

الناتج:

~~~
nvidia.ko
nvidia-modeset.ko
nvidia-uvm.ko
nvidia-drm.ko
~~~

وهذا أثبت أن 390.157 أصبح قابلًا للبناء على kernel 7.2.8.

---

# 9. تحذيرات البناء

ظهرت تحذيرات مثل:

~~~
objtool: data relocation to !ENDBR
~~~

و:

~~~
missing MODULE_DESCRIPTION() in nvidia-uvm.o
~~~

لكنها لم تمنع إنشاء modules أو تحميلها لاحقًا.

السجل الخام الكامل موجود في:

~~~
logs/build-7.2.8-success.log
~~~

---

# 10. دمج الحل مع DKMS

إعداد DKMS استخدم patches فقط لـkernel 7.2.x:

~~~
PATCH[0]="0107-cachyos.patch"
PATCH_MATCH[0]="^7\.2\."

PATCH[1]="0108-cachyos.patch"
PATCH_MATCH[1]="^7\.2\."
~~~

واستخدم مسار LLVM للبناء:

~~~fish
make -j$(nproc) LLVM=1 CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm HOSTCC=clang HOSTLD=ld.lld IGNORE_CC_MISMATCH=1 IGNORE_PREEMPT_RT_PRESENCE=1 NV_EXCLUDE_BUILD_MODULES='__EXCLUDE_MODULES' KERNEL_UNAME=\${kernelver} modules
~~~

مع:

~~~
MAKE_MATCH[1]="^7\.2\."
~~~

وبذلك بقي إعداد LTS منفصلًا.

---

# 11. نتيجة DKMS

بعد البناء والتثبيت:

~~~
nvidia/390.157, 6.18.52-1-cachyos-lts, x86_64: installed
nvidia/390.157, 7.2.8-1-cachyos, x86_64: installed
~~~

وهذا يعني أن DKMS نجح لكل من LTS والـ7.2.8.

---

# 12. التحقق بعد reboot

الأمر:

~~~fish
uname -r
~~~

النتيجة:

~~~
7.2.8-1-cachyos
~~~

إذن kernel المستهدف يعمل فعليًا.

---

# 13. nvidia-smi

الأمر:

~~~fish
nvidia-smi
~~~

النتيجة:

~~~
NVIDIA-SMI 390.157
Driver Version: 390.157
GPU: GeForce GT 720M
Bus-Id: 00000000:03:00.0
Temp: 66C
Perf: P8
Memory: 0MiB / 1985MiB
GPU-Util: N/A
Processes: Not Supported
~~~

هذا يثبت أن kernel driver يستطيع التواصل مع GT 720M فعليًا.

لكنه لا يثبت أن سطح المكتب الحالي يستخدم NVIDIA للرسم.

---

# 14. lspci

الأمر:

~~~fish
lspci -nnk
~~~

Intel:

~~~
00:02.0 VGA compatible controller [0300]: Intel Corporation Haswell-ULT Integrated Graphics Controller [8086:0a16] (rev 0b)
        DeviceName: Onboard IGD
        Subsystem: Dell Device [1028:05de]
        Kernel driver in use: i915
~~~

NVIDIA:

~~~
03:00.0 3D controller [0302]: NVIDIA Corporation GF117M [GeForce 610M/710M/810M/820M / GT 620M/625M/630M/720M] [10de:1140] (rev a1)
        Subsystem: Dell GeForce GT 720M [1028:05de]
        Kernel driver in use: nvidia
        Kernel modules: nouveau, nvidia_drm, nvidia
~~~

إذن:

~~~
Intel  -> i915
NVIDIA -> nvidia
~~~

---

# 15. lsmod

النتيجة:

~~~
nvidia_drm     65536  0
nvidia_modeset 1347584  1 nvidia_drm
nvidia_uvm     1380352  0
nvidia         19787776 19 nvidia_uvm,nvidia_modeset
~~~

كل وحدات NVIDIA الأساسية محملة:

~~~
nvidia
nvidia_modeset
nvidia_uvm
nvidia_drm
~~~

---

# 16. OpenGL على Wayland

الأمر:

~~~fish
glxinfo -B
~~~

النتيجة:

~~~
name of display: :0
direct rendering: Yes
Vendor: Intel
Device: Mesa Intel(R) HD Graphics 4400 (HSW GT2)
Accelerated: yes
OpenGL vendor string: Intel
OpenGL renderer string: Mesa Intel(R) HD Graphics 4400 (HSW GT2)
OpenGL version: Mesa 26.2.3
~~~

إذن renderer الأساسي كان Intel.

هذا ليس فشلًا بحد ذاته في جهاز Hybrid Graphics؛ كان يلزم اختبار PRIME offload.

---

# 17. prime-run

لم يكن prime-run موجودًا:

~~~fish
prime-run
~~~

أعطى:

~~~
fish: Unknown command: prime-run
~~~

ثم:

~~~fish
which prime-run
pacman -Qo prime-run
pacman -Ss prime-run
~~~

ولم توجد حزمة توفره في النظام الحالي.

لكن هذا لم يُعتبر سبب المشكلة، لأن PRIME اختُبر يدويًا.

---

# 18. PRIME offload يدويًا

الأمر:

~~~fish
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
~~~

فشل:

~~~
X Error of failed request: BadValue
(integer parameter out of range for operation)

Major opcode of failed request: 152 (GLX)
Minor opcode: 24 (X_GLXCreateNewContext)
Value in failed request: 0x0
~~~

إذن المشكلة ليست مجرد غياب prime-run.

---

# 19. الانتقال إلى X11

تم تغيير الجلسة لاختبار ما إذا كانت المشكلة خاصة بـWayland.

~~~fish
echo $XDG_SESSION_TYPE
~~~

النتيجة:

~~~
x11
~~~

أي أن الاختبار أصبح KDE Plasma + X11.

---

# 20. OpenGL على X11

تم تشغيل:

~~~fish
glxinfo -B
~~~

وظل renderer الأساسي Intel.

إذن المشكلة ليست مقتصرة على Wayland.

---

# 21. nvidia-settings على Wayland

أعطى:

~~~
ERROR: Unable to find display on any available system
~~~

وهذا لم يُعتبر دليلًا على تعطل GPU لأن الأداة تعتمد على واجهات X/NVIDIA القديمة.

---

# 22. nvidia-settings على X11

أعطى:

~~~
ERROR: Error querying enabled displays on GPU 0 (Missing Extension).
ERROR: Error querying connected displays on GPU 0 (Missing Extension).
~~~

وهذا يشير إلى أن Xorg لا يقدم واجهة NVIDIA display provider المتوقعة للأداة.

---

# 23. xrandr

المحاولة:

~~~fish
xrandr --listproviders
~~~

فشلت لأن xrandr غير مثبت:

~~~
fish: Unknown command: xrandr
~~~

وهذه أداة تشخيص مفقودة وليست دليلًا بحد ذاتها على غياب PRIME providers.

---

# 24. nvidia_drm.modeset

المحاولة الأولى:

~~~fish
cat /sys/module/nvidia_drm/parameters/modeset
~~~

أعطت:

~~~
cat: /sys/module/nvidia_drm/parameters/modeset: Permission denied
~~~

والأمر الصحيح:

~~~fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
~~~

والقيمة التي ظهرت أثناء التحقيق:

~~~
N
~~~

أي:

~~~
nvidia_drm.modeset = 0
~~~

---

# 25. modinfo nvidia_drm

الأمر:

~~~fish
sudo modinfo nvidia_drm | grep -E '^(filename|version|parm):'
~~~

أظهر:

~~~
filename: /lib/modules/7.2.8-1-cachyos/updates/dkms/nvidia-drm.ko.zst
version: 390.157
parm: modeset:Enable atomic kernel modesetting (1 = enable, 0 = disable (default)) (bool)
~~~

إذن module الصحيح موجود ويدعم modeset، لكنه معطل حاليًا.

---

# 26. kernel journal — الدليل الحاسم

السجل أظهر:

~~~
nvidia: loading out-of-tree module taints kernel.
nvidia: module license 'NVIDIA' taints kernel.
nvidia: module verification failed: signature and/or required key missing - tainting kernel
NVRM: loading NVIDIA UNIX x86_64 Kernel Module 390.157
~~~

ثم:

~~~
nvidia-uvm: Loaded the UVM driver
~~~

ثم:

~~~
nvidia-modeset: Loading NVIDIA Kernel Mode Setting Driver
~~~

ثم:

~~~
[drm] [nvidia-drm] [GPU ID 0x00000300] Loading driver
~~~

ثم:

~~~
[drm] Initialized nvidia-drm 0.0.0 for 0000:03:00.0 on minor 0
~~~

حتى هذه النقطة NVIDIA DRM بدأ بشكل طبيعي.

ثم حدث الفشل:

~~~
Failed to initialize the nv-hotplug-helper DRM client
(ensure DRM kernel mode setting is enabled via nvidia-drm.modeset=1).
~~~

ثم:

~~~
[nvidia-drm] [GPU ID 0x00000300] Unloading driver
~~~

هذه أهم نتيجة في التحقيق كله.

التسلسل هو:

~~~
nvidia-drm loads
        ↓
GPU recognized
        ↓
nv-hotplug-helper fails
        ↓
nvidia-drm unloads
~~~

والـkernel نفسه يقترح:

~~~
nvidia-drm.modeset=1
~~~

كشرط يجب اختباره.

---

# 27. /dev/dri

الأمر:

~~~fish
ls -l /dev/dri/
~~~

أظهر:

~~~
card1
renderD128
by-path/
~~~

ثم:

~~~fish
ls -l /dev/dri/by-path/
~~~

أظهر:

~~~
pci-0000:00:02.0-card -> ../card1
pci-0000:00:02.0-render -> ../renderD128
~~~

ثم:

~~~fish
readlink -f /sys/class/drm/card1/device/driver
~~~

النتيجة:

~~~
/sys/bus/pci/drivers/i915
~~~

إذن card1 هو Intel.

لم يظهر NVIDIA DRM card node بالطريقة المتوقعة بعد أن قام nvidia-drm بالتفريغ.

---

# 28. Xorg log

لم يوجد:

~~~
/var/log/Xorg.0.log
~~~

لكن السجل كشف أن Xorg يستخدم:

~~~
/home/roc/.local/share/xorg/Xorg.0.log
~~~

لذلك المسار الصحيح للتحقيق هو:

~~~fish
grep -iE 'nvidia|glx|prime|provider' ~/.local/share/xorg/Xorg.0.log | tail -100
~~~

---

# 29. ما الذي استخدمه Xorg؟

Xorg رأى بطاقتي:

~~~
8086:0a16
10de:1140
~~~

لكنه استخدم:

~~~
/dev/dri/card1
~~~

وهو Intel.

حاول drivers:

~~~
intel
modesetting
fbdev
vesa
~~~

ثم استخدم modesetting.

المسار المهم:

~~~
modeset(0): using drv /dev/dri/card1
~~~

---

# 30. Xorg/OpenGL

Xorg أظهر:

~~~
modeset(0): glamor: Using OpenGL 4.6 context.
~~~

ثم:

~~~
modeset(0): glamor X acceleration enabled on Mesa Intel(R) HD Graphics 4400 (HSW GT2)
~~~

ثم:

~~~
[DRI2] DRI driver: crocus
~~~

وAIGLX باستخدام crocus.

إذن:

~~~
Xorg
 ↓
modesetting
 ↓
/dev/dri/card1
 ↓
i915
 ↓
Intel HD 4400
 ↓
Mesa/crocus
~~~

---

# 31. الشاشة الداخلية

Xorg اكتشف:

~~~
eDP-1 connected
~~~

بدقة:

~~~
1366x768
~~~

أي أن الشاشة تعمل طبيعيًا عبر Intel.

المشكلة الحالية هي NVIDIA offload/provider وليست تشغيل الشاشة الأساسية.

---

# 32. فصل مشكلة touchpad

Xorg اكتشف أيضًا:

~~~
AlpsPS/2 ALPS GlidePoint
~~~

عبر:

~~~
/dev/input/event10
~~~

واستخدم libinput.

هذا يؤكد أن تحقيق NVIDIA مستقل عن مشكلة ALPS touchpad.

---

# 33. NVIDIA GLX libraries

تم التحقق من وجود:

~~~
libGLX_nvidia.so.0 => /usr/lib/libGLX_nvidia.so.0
libGLX_nvidia.so.0 => /usr/lib32/libGLX_nvidia.so.0
libGLX_nvidia.so   => /usr/lib/libGLX_nvidia.so
libGLX_nvidia.so   => /usr/lib32/libGLX_nvidia.so
~~~

إضافة إلى libGLX العامة.

إذن المشكلة ليست ببساطة missing libGLX_nvidia.

---

# 34. module signature warning

ظهر:

~~~
nvidia: module verification failed: signature and/or required key missing - tainting kernel
~~~

وهذا متوقع لوحدة DKMS غير موثقة بالمفتاح المناسب.

لا يمكن اعتباره سبب مشكلة PRIME الحالية، لأن module تم تحميله وnvidia-smi يعمل.

---

# 35. رسائل NVIDIA الأخرى

ظهرت رسائل متعلقة بـBAR mapping وبعض مسارات ioctl/close/shutdown أثناء استخدام أدوات NVIDIA.

هذه الرسائل لا تكفي وحدها لإثبات عطل hardware.

الأدلة الأقوى هي:

~~~
nvidia-smi works
PCI detects GPU
nvidia kernel module loads
nvidia-drm starts
nv-hotplug-helper fails
nvidia-drm unloads
~~~

---

# 36. ما تم حله

تم حل:

- توافق مصدر NVIDIA 390.157 مع kernel 7.2.
- patch 0107.
- patch 0108.
- بناء LLVM/Clang/LLD.
- DKMS build.
- DKMS install.
- تحميل kernel modules.
- اكتشاف GPU.
- nvidia-smi.
- التشغيل الفعلي للتعريف بعد reboot.

---

# 37. ما لم يُحل بعد

المشكلة المتبقية هي:

~~~
NVIDIA DRM/KMS/PRIME graphics integration
~~~

المسار الحالي:

~~~
NVIDIA driver
      ✓
kernel module
      ✓
GPU detection
      ✓
nvidia-smi
      ✓
nvidia-drm initial load
      ✓
modeset = N
      ↓
nv-hotplug-helper failure
      ↓
nvidia-drm unload
      ↓
Xorg uses Intel
      ↓
PRIME GLX offload fails
~~~

---

# 38. الاختبار التالي

الاختبار التالي الذي حدده التحقيق هو:

~~~
nvidia_drm.modeset=1
~~~

ويجب اختباره أولًا مؤقتًا عند الإقلاع، وليس تغيير الإعداد الدائم مباشرة.

السبب أن سجل kernel نفسه قال:

~~~
ensure DRM kernel mode setting is enabled via nvidia-drm.modeset=1
~~~

وهذا يجعل modeset=1 الاختبار التشخيصي التالي المباشر.

**حتى نهاية هذا التقرير لم يتم إثبات نتيجة هذا الاختبار بعد.**

---

# 39. أوامر ما بعد الاختبار

بعد الإقلاع مع nvidia_drm.modeset=1:

~~~fish
sudo cat /sys/module/nvidia_drm/parameters/modeset
~~~

ثم:

~~~fish
ls -l /sys/class/drm/
~~~

ثم:

~~~fish
sudo journalctl -b | grep -iE 'nvidia-drm|hotplug|drm' | tail -100
~~~

ثم:

~~~fish
env __NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo -B
~~~

الهدف هو معرفة هل:

~~~
modeset = Y
~~~

وهل يبقى nvidia-drm محملًا، وهل يصبح NVIDIA renderer متاحًا عبر PRIME.

---

# 40. النتيجة المتوقعة إذا نجح الاختبار

في حالة نجاح PRIME، يجب أن يتحول الاختبار من Intel إلى NVIDIA، مثل:

~~~
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: GeForce GT 720M/PCIe/SSE2
~~~

ولا يشترط تطابق renderer string حرفيًا؛ المهم أن يكون vendor/renderer هو NVIDIA.

---

# 41. إذا لم ينجح modeset=1

إذا أصبح:

~~~
modeset = Y
~~~

لكن بقي:

~~~
BadValue
~~~

فيجب الانتقال إلى Xorg/provider/GLX diagnostics:

~~~fish
grep -iE 'nvidia|glx|prime|provider|drm' ~/.local/share/xorg/Xorg.0.log | tail -100
~~~

و:

~~~fish
sudo journalctl -b | grep -iE 'nvidia|nvidia-drm|drm|glx|prime|provider|hotplug' | tail -150
~~~

ثم يمكن استخدام xrandr بعد تثبيته:

~~~fish
xrandr --listproviders
~~~

---

# 42. إذا بقي nvidia-drm يفرغ نفسه

إذا كان modeset=Y لكن nvidia-drm ما زال يفشل، فالأوامر التالية هي الخطوة التالية:

~~~fish
sudo journalctl -b | grep -iE 'nvidia-drm|nvrm|drm|firmware|pci|bar' | tail -200
~~~

ويجب التركيز على:

- DRM
- KMS
- PCI
- BAR
- NVRM
- hotplug
- firmware

ولا ينبغي اعتبار رسائل BAR الحالية سببًا نهائيًا قبل اكتمال هذا الاختبار.

---

# 43. لماذا لم نستخدم optimus-manager أو nvidia-xrun بعد؟

لم يتم إدخال أدوات مثل:

~~~
optimus-manager
nvidia-xrun
~~~

قبل حل حالة NVIDIA DRM/KMS.

إضافة طبقة إدارة أخرى الآن ستزيد عدد المتغيرات وتخفي السبب الأساسي.

الترتيب الصحيح للتحقيق:

~~~
NVIDIA kernel driver
        ↓
nvidia-drm
        ↓
DRM/KMS
        ↓
Xorg provider
        ↓
PRIME GLX
        ↓
application rendering
~~~

والتحقيق وصل حاليًا إلى الحد الفاصل بين DRM/KMS وXorg/PRIME.

---

# 44. الحالة النهائية المؤكدة

## يعمل

~~~
CachyOS 7.2.8
NVIDIA 390.157 source compatibility
patch 0107
patch 0108
Clang/LLVM/LLD build
DKMS
NVIDIA kernel modules
PCI detection
nvidia-smi
~~~

## غير مثبت نجاحه بعد

~~~
NVIDIA DRM KMS
NVIDIA Xorg provider
PRIME GLX offload
NVIDIA as active OpenGL renderer
~~~

## القيمة الحالية

~~~
nvidia_drm.modeset = N
~~~

## الخطأ الحاسم

~~~
Failed to initialize the nv-hotplug-helper DRM client
(ensure DRM kernel mode setting is enabled via nvidia-drm.modeset=1).
~~~

## آخر فشل PRIME مؤكد

~~~
X Error of failed request: BadValue
Major opcode: 152 (GLX)
Minor opcode: 24 (X_GLXCreateNewContext)
Value in failed request: 0x0
~~~

---

# 45. الخلاصة التقنية

تم حل مشكلة بناء NVIDIA 390.157 وتشغيل module على CachyOS kernel 7.2.8 بنجاح.

تم إثبات:

~~~
NVIDIA 390.157
        ↓
kernel 7.2.8
        ↓
DKMS
        ↓
kernel modules
        ↓
GT 720M
        ↓
nvidia-smi
~~~

لكن مرحلة graphics offload لا تزال غير مكتملة.

الدليل الأقوى هو أن nvidia-drm يبدأ ثم يفشل nv-hotplug-helper بسبب حالة KMS الحالية، وبعد ذلك يفرغ نفسه. Xorg يستخدم Intel عبر i915 وMesa، بينما PRIME GLX يفشل بـBadValue.

لذلك آخر نقطة مؤكدة في التحقيق هي:

~~~
nvidia_drm.modeset = N
        ↓
nv-hotplug-helper failure
        ↓
nvidia-drm unload
        ↓
Intel remains Xorg renderer
        ↓
PRIME GLX fails
~~~

والخطوة التالية الوحيدة التي يجب اختبارها قبل إضافة طبقات أخرى هي:

~~~
nvidia_drm.modeset=1
~~~

ولا ينبغي اعتبارها حلًا مثبتًا قبل تسجيل نتيجة الاختبار بعد الإقلاع.

---

# 46. علاقة هذا الملف بباقي المستودع

هذا الملف مستقل عن الملفات السابقة:

~~~
README.md
FULL-REPORT.md
patches/0107-cachyos.patch
patches/0108-cachyos.patch
logs/build-7.2.8-success.log
~~~

الغرض منه هو الاحتفاظ بالسجل الكامل للتشخيص اللاحق بعد نجاح البناء، خصوصًا انتقال التحقيق من kernel compatibility إلى runtime DRM/KMS/PRIME.

لا يجب اعتبار هذا الملف بديلًا عن README أو FULL-REPORT، ولا تعديل محتوى تلك الملفات بسبب إضافة هذا المرجع.

---

# 47. ملاحظة تاريخية

هذا التقرير يتوقف عمدًا عند **آخر نتيجة مؤكدة** بدل افتراض نتيجة لم يتم اختبارها.

إذا نجح اختبار nvidia_drm.modeset=1 لاحقًا، ينبغي إضافة تقرير/قسم زمني جديد يوضح:

1. قيمة modeset الجديدة.
2. حالة /sys/class/drm.
3. رسائل nvidia-drm بعد الإقلاع.
4. نتيجة PRIME glxinfo.
5. نتيجة Xorg provider.
6. أي إعداد دائم تم اعتماده.

وبذلك يبقى هذا الملف سجلًا تاريخيًا قابلًا للمقارنة مع النتائج اللاحقة.
