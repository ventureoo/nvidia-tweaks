# nvidia-tweaks

## Description

nvidia-tweaks is a set of tweaks and workarounds for the Linux version of the
closed NVIDIA driver. It will allow you to fix some problems related to the
driver and can potentially improve performance (Read about the driver module
options).

In other words, this project doesn't contain anything you can't do yourself,
but just simplifies the process and stores a lot of information.

## Installation

First of all, you must have the NVIDIA driver itself installed and loaded in
order for this tweak kit to work. Without it, no further action is meaningful.

And yes, **please ONLY install the NVIDIA driver using the package manager of
your distribution!** Installing it manually from the official NVIDIA website
may make it more difficult to maintain and impossible to update.

Specific instructions on how to install the driver can be found here:

https://github.com/lutris/docs/blob/master/InstallingDrivers.md

### Arch-based systems (AUR)

Arch Linux systems can simply install the nvidia-tweaks package from the AUR.
This package is independent of the specific driver version and can be used for
any driver package. It is only necessary that the driver is installed (NVIDIA
MODULE is provided).

```
git clone https://aur.archlinux.org/nvidia-tweaks.git
cd nvidia-tweaks
makepkg -sric
```

You can also edit the PKGBUILD before installing it to include some additional
settings, which we will talk about later.

### Other distros

There are no packages for other distributions yet. However, you can manually
copy all necessary configuration files:

```
git clone https://www.github.com/ventureoo/nvidia-tweaks.git
cd nvidia-tweaks
sudo cp -r nvidia.conf /etc/modprobe.d/nvidia-tweaks.conf
sudo cp -r nvidia-uvm.conf /etc/modules-load.d/
sudo cp -r 60-nvidia.rules /etc/udev/rules.d/
```

The keylase nvidia-patch requires a separate installation. You can read about
it on the project site: https://github.com/keylase/nvidia-patch

## So, how does it work?

First, we configure the kernel modules of the NVIDIA driver (`nvidia.conf`).
The thing is that by default, not all parameters are enabled in the driver
module. Including the following things we need to change:

- NVreg_UsePageAttributeTable=1 (Default 0) - Activating the better memory
  management method (PAT). The PAT method creates a partition type table at a
  specific address mapped inside the register and utilizes the memory
  architecture and instruction set more efficiently and faster. If your system
  can support this feature, it should improve CPU performance. To check if your
  processor is supported by PAT, use the command: `grep -E '^flags.+ pat( |$)'
  /proc/cpuinfo`. If the output of the command was not empty, then all is OK
  (98% of modern processors today have PAT support).

  See also:
  
  https://bbs.archlinux.org/viewtopic.php?id=242007,
  https://old.reddit.com/r/linux_gaming/comments/9wjpkz/poor_gaming_performance_is_this_a_cpu_or_gpu/

- NVreg_InitializeSystemMemoryAllocations (Default 1) - Disables clearing
  system memory allocation before using it for the GPU.

  Potentially improves performance, but at the cost of increased security
  risks. Write `options nvidia NVreg_InitializeSystemMemoryAllocations=1` in
  `/etc/modprobe.d/nvidia.conf`, if you want to return the default value.

  **Note**: It is possible to use more VRAM (?)

Then we install udev rules from negative17
(https://github.com/negativo17/nvidia-kmod-common/blob/master/60-nvidia.rules).
These udev rules are required for node presence and runtime PM and fixes
problems with raytracing in vkd3d
(https://github.com/HansKristian-Work/vkd3d-proton/issues/711,
https://github.com/HansKristian-Work/vkd3d-proton/issues/902).

**Note**: The driver packages in Arch Linux already have these rules by default
(See:
https://github.com/archlinux/svntogit-packages/commit/3c0985a523c8795a483f3c63d177508717603b65).
But for other distributions this may still be necessary. Also, the packages for
Arch appear to have an incomplete set of rules, so installing them may still be
useful.

Installing `nvidia-uvm.conf` in `/etc/modules-load.d/` together with udev rules
also fixes a common issue with nvidia-uvm module loading, which can break CUDA,
including NVENC.

There are also tweaks as an option:

- Installing the nvidia-patch from keylase
  - This patch removes restriction on maximum number of simultaneous NVENC
    video encoding sessions imposed by Nvidia to consumer-grade GPUs. You can
    read more about it here: https://github.com/keylase/nvidia-patch
  - In the AUR package, you can get it by enabling the option in PKGBUILD:
    `_nvidia_patch=y`. No additional actions are required, the patch is applied
    automatically by the pacman hook.
  - As an alternative, there is [nvlax](https://github.com/illnyang/nvlax).
    Unlike nvidia-patch, it can be used for any driver version.
- Power Setup (PowerMizer)
  - Another kernel module parameter of the `RegistryDwords` driver allows you
    to configure the GPU power supply. To be more precise, configure the driver
    power manager - PowerMizer.
  - PowerMizer has three levels of performance:
    - 1 (0x1 bit) - Maximum performance
    - 2 (0x2 bit) - Balance between performance and power saving
    - 3 (0x3 bit) - Powersaving
  - PowerMizer also has two frequency management strategies:
    - Adaptive strategy (Frequencies change depending on the load on the GPU)
    - Static strategy (Frequencies are fixed at strictly one of the performance levels described above)
  - Power saving is set separately for AC and battery (if you have one). That
    is, you can select separately one frequency management strategy for the
    battery, and separately for AC. Also if a static strategy has been selected
    for any of the power supplies, you can set its specific performance level
    through their respective flags `PowerMizerDefault` (battery) and
    `PowerMizerDefaultAC` (AC).

  - All PowerMizer settings are specified with a semicolon as part of the
    `NVreg_RegistryDwords` kernel module parameter. For example:
    `NVreg_RegistryDwords="PowerMizerEnable=0x1;PerfLevelSrc=0x2222;PowerMizerDefault=0x3;PowerMizerDefaultAC=0x1"`
    - is set Static strategy with maximum performance for AC operation and
    maximum power savings for the battery.
  - For convenience, the AUR package has a ready-made set of PowerMizer tuning
    diagrams. You can apply them by setting the desired one in the PKGBUILD
    `_powermizer_scheme=value` parameter. See PKGBUILD for a description of
    possible values.
  - For other distributions you have to manually edit
    `/etc/modprobe.d/nvidia.conf` and the `NVreg_RegistryDwords` parameter. 
  - See also: https://wins911.blogspot.com/2012/06/etcx11xorg.html

- Force maximum performance (For laptops only)
  - `OverrideMaxPerf` flag for `NVreg_RegistryDwords` parameter enforces
    applying a certain performance level while activating some GPU overclocking
    possibilities, such as controlling GPU fan speed via nvidia-settings.
  - The flag takes values in bits only. For example:
    `NVreg_RegistryDwords="OverrideMaxPerf=0x1"` - Forces of maximum
    performance on any power source.
  - **Works only for laptops.**
  
All of the optional tweaks mentioned above should be applied according to your
preferences. For a package from the AUR, they are applied through the
corresponding options in PKGBUILD. For other distributions this will probably
require editing `/etc/modprobe.d/nvidia-tweaks.conf` and adding the appropriate
parameters.

## Some tips from me

### Do not use the Force Composition Pipeline to eliminate with tearing

There are two ways to deal with tearing in Linux: 

1) Use a compositor with properly working Vsync frame synchronization
(something all working environments are trying to do at the moment)
2) Use driver options that allow you to fix it (like the Force (Full)
Composition Pipeline).

Some people advise doing the second way. Personally, I don't agree with that.
Yes, the Force Composition Pipeline does fix the tearing, but you do pay some
overhead. Namely, you get the input lag problem, since this option contributes
to a massive increase in latency everywhere you can. This is especially
sensitive for places like games, because that is where the difference is best
felt. In addition, it also causes problems with resetting the frequencies of
your GPU. It is verified that the Force Composition Pipeline increases the idle
time for the driver to change its level of performance (See
https://forums.developer.nvidia.com/t/if-you-have-gpu-clock-boost-problems-please-try-gl-experimentalperfstrategy-1/71762/14)

So I recommend everyone to use the first way whenever possible. At the moment,
all modern DEs and consequently the composers attached to them are good at
solving this issue. Particularly GNOME and since version 5.21 of Plasma no
longer have any problems (the main thing is to switch the rendering backend to
OpenGL). However, for all less modern environments such as Xfce and Mate, and
of course all window managers, the problem is still relevant. In this case it
is necessary to perform an installation of a separate compositor like Picom.
Now it is good enough, the most important thing is to add it to the autostart
with the following options to start it:

``picom --experimental-backends --backend glx --vsync``

**Note:** ``--experimental-backends`` is not required as of v10 picom. 

For Xfce, you must get rid of the problematic default compositor before using
Picom:

``xfce-query -c xfwm4 -p /general/use_compositing -s false``

P.S. All of the above only makes sense for the X11 sessions. Wayland implies
the use of compositing as a built-in feature.

## Wayland + NVIDIA

Currently, starting with driver branch 515 and higher, NVIDIA has support for
GBM API, which made it possible to run Sway and wlroots-like window managers,
and also significantly improved Wayland sessions in GNOME/KDE Plasma.
~~However, this implementation still has a number of problems, the main one
being flickering when using XWayland/OpenGL acceleration in applications. As
described
[here](https://forums.developer.nvidia.com/t/nvidia-495-on-sway-tutorial-questions-arch-based-distros/192212/78),
this problem is caused by the fact that the NVIDIA driver only supports the new
OpenGL explicit sync instead of implicit sync. So I recommend you to use only
native support in Wayland applications wherever possible, and avoid OpenGL
acceleration. It looks pretty ridiculous, but it's a workable solution. With
the Vulkan backend in Sway I haven't found any obvious problems with
rendering.~~ The information above is out of date. Artifacts and blinks have
been fixed in sway-git/wlroots-git, please build them from AUR and install the
latest driver (525+). There is no need to use Vulkan. I would also like to
point out that GNOME Wayland and Plasma Wayland are now working at a pretty
good level, it's not perfect, but progress is significant. So, here's a set of
environment variables to help you  on NVIDIA using Wayland:

```
export SDL_VIDEODRIVER=wayland # Can break some native games
export XDG_SESSION_TYPE=wayland
export QT_QPA_PLATFORM="wayland;xcb"
export MOZ_ENABLE_WAYLAND=1
export GBM_BACKEND=nvidia-drm
export WLR_NO_HARDWARE_CURSORS=1 
export KITTY_ENABLE_WAYLAND=1
```

These environment variables should be added to your ``~/.bashrc`` or
``~/.zshrc``, depending on the shell you are using. Not all of them are useful
if you are not using the wlroots-based window manager. I would also recommend
you to avoid Chromium (without Ozon)/Electron-based applications, as they can
all be very unstable in Wayland on NVIDIA. ~~Browsers on QtWebEngine such as
qutebrowser unfortunately do not work either.~~ QtWebEngine works if you use
``QT_QUICK_BACKEND=software`` environment variable.

For the wlroots Vulkan backend to work, also make sure that you are using the
NVIDIA Vulkan beta driver (to support the required Vulkan extensions).

If you do not have a Wayland session to choose from in GDM, then include the
``NVreg_PreserveVideoMemoryAllocations=1`` parameter to the ones we already
described above.

```
  sudo systemctl enable nvidia-resume.service nvidia-suspend.service nvidia-hibernate.service
```

This will also allow you to avoid some sleeping issues on Wayland.

Attention! On laptops with PRIME enabling
``NVreg_PreserveVideoMemoryAllocations=1`` may cause issues with sleep.
Therefore it is recommended to use it only on desktop PCs.

## Environment variables

The NVIDIA driver has some specific environment variables. I will not list the
technical specifics of how they work, I will just tell you about the effect
they can have, more details you will learn if you follow the links:

``__GL_THREADED_OPTIMIZATIONS=1`` (Off by default) - Enable multi-threaded
OpenGL processing. The analog of Mesa's ``mesa_glthread`` variable.
Use selectively for native games/applications, because sometimes it can cause
performance regression [1], [2]. Some games may not run with this variable at
all (e.g. some native parts of Metro). Note that for some applications this
environment variable is already used in the default NVIDIA profile supplied
with the driver (``/usr/share/nvidia/nvidia-application-profiles-525.78.01-rc``
for example).

[1] - https://www.phoronix.com/review/nvidia_threaded_opts (a very old benchmark)

[2] - https://www.phoronix.com/review/nvidia-t2015-optimizations

``__GL_MaxFramesAllowed=1`` (Default value is 2?) - Roughly speaking, it sets
the type of frame buffering by the driver. You can specify "3" (Triple
buffering) for more FPS or better performance in applications/games with VSync.
I recommend to try "1". It can noticeably decrease FPS value in games, but in
return you will get better rendering delays and real physical response, because
the frame will be displayed to you immediately on the screen without
unnecessary processing steps. This hack was used in some compositors to improve
latency [1][2][3][4].

[1] - https://github.com/yshui/picom/pull/641

[2] - https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/1269

[3] - https://forums.developer.nvidia.com/t/how-to-turn-on-low-latency-mode-max-pre-render-frames-on-linux/108353

[4] - https://phabricator.kde.org/D19867

``__GL_YIELD="USLEEP"`` (Unset by default) - Pretty specific parameter with
several possible values. Most interestingly, the ``USLEEP`` value can
potentially reduce latency and CPU load [1][2]. More information can be found
in the driver documentation:
https://download.nvidia.com/XFree86/Linux-x86_64/525.78.01/README/openglenvvariables.html

[1] - https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=925528

[2] - https://aweirdimagination.net/2020/05/24/100-cpu-usage-in-games-with-nvidia-linux-drivers/

**Note:** I'm not entirely sure how relevant this variable is right now.

``__GL_SHADER_DISK_CACHE_SKIP_CLEANUP=1`` - disables OpenGL/Vulkan shader cache
limit (``~/.cache/nvidia`` by default).  Recommended for modern games and DXVK
2.0+, where the cache can reach more than a gigabyte. 

**WARNING:** I strongly advise against specifying the above environment
variables for the whole system. Please specify them for specific
applications/games with nvidia-settings or using Lutris/Steam.

## GSP firmware

GSP (GPU System Processor) - this is a special chip which is present on NVIDIA
video cards starting from Turing and above, which offloads GPU initialization
and control tasks, which are usually performed on CPU. This should improve
performance and reduce the load on the CPU.

To check if the GSP firmware is currently in use, use the following command:

```
 nvidia-smi -q | grep GSP
```

By default, GSP firmware is enabled only for a limited number of GPUs. But if
you have GPU of Turing generation and above, you can also force their use via
the parameter for the ``NVreg_EnableGpuFirmware=1`` kernel module. Add this
parameter as we did above with others, after rebooting you should get the
following output:

```
GSP Firmware Version                  : 530.41.03
```

(the output may differ depending on the driver version)

**WARNING:** I strongly suggest forcing the use of GSP firmware only on the
most recent driver, as the first releases with its support may contain certain
problems. Only starting from 530 you will have support for suspend and resume
when using GSP firmware. This feature can also work badly on PRIME
configurations, so please check dmesg logs for errors if you want to use this.

See also:

https://www.techpowerup.com/291088/nvidia-unlocks-gpu-system-processor-gsp-for-improved-system-performance

https://download.nvidia.com/XFree86/Linux-x86_64/530.41.03/README/gsp.html

## Resizable Bar

Starting with version 530.30.02 NVIDIA introduced support for the Resizeble
Bar. Previously, we already had a third-party patch for this (see the patches
directory), but it also required to patch for kernel. Although this change was
not documented for some reason, we now have the ``NVreg_EnableResizableBar``
prameter which is disabled by default. Keep in mind that for Resize Bar to work
properly, you not only need a GPU that supports it (it is officially supported
since Ampere), but also an compatible motherboard.

To make sure that your GPU actually supports this, go to nvidia-settings,
select your GPU, and look in the ``Resizable Bar`` column. If it says "Yes",
that's all right and you can use the kernel parameter specified above:
``NVreg_EnableResizableBar=1``.

(P.S. For some reason, the ``Resizable Bar`` column is not shown in
nvidia-settings in the Wayland session)

