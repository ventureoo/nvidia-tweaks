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


> [!WARNING]
> The AUR package was removed for the following reasons:
>
> I am discontinuing maintenance of this package and recommend that everyone remove it from their system. Most of the “tweaks” that are provided within this package are already part of the nvidia-utils package upstream, so you no longer need the following things because you get them with the nvidia-utils package:
>
> - Enabling nvidia-drm.modeset=1 and nvidia-drm.fbdev=1 in nvidia.conf. This is now enabled by default in nvidia-utils as a patch and ensures Wayland/PRIME configurations work properly. For users of “older” driver branches like nvidia-390xx-dkms and nvidia-470xx-dkms this is still relevant, but since these versions do not fully support Wayland sessions, enabling nvidia-drm.modeset=1 for them is only relevant for running PRIME configurations. Contact the maintainers of these packages to have them enable this by default, as it is quite useful for laptops.
> - Using udev 60-nvidia.rules. I've been doing the work of upstreaming some of these rules in nvidia-utils: https://gitlab.archlinux.org/archlinux/packaging/packages/nvidia-utils/-/merge_requests/9, so you don't need to worry about them anymore, only if you're not using the nvidia-390xx-dkms and nvidia-470xx-dkms packages. Again, contact the maintainers of those packages if you want them to sync with changes in the nvidia-utils package upstream. The rest of the rules regarding power management remain relevant only for Turing generation mobile GPUs. Be warned that RTD3 does NOT work with open source versions of driver modules (nvidia-open-dkms) on Turing generation.
> -  The power schemes provided by _powermizer_scheme no longer work with newer driver versions because NVIDIA broke them starting with the 530 driver: https://forums.developer.nvidia.com/t/kernel-module-option-nvreg-registrydwords-for-powermizerenable-doesnt-work-on-530-41-03/247610. Not sure about _override_max_perf.
> -  Hooks for mkinitcpio are no longer needed, as they are now automatically run by mkinitcpio itself when installing any kernel modules:
>
> https://gitlab.archlinux.org/archlinux/mkinitcpio/mkinitcpio/-/merge_requests/392
>
> https://gitlab.archlinux.org/archlinux/mkinitcpio/mkinitcpio/-/merge_requests/256
>
> For those who installed this package for the benefit of nvidia-patch, please refer to the nvidia-patch package (https://aur.archlinux.org/packages/nvidia-patch). The remaining Nvidia module parameters such as NVreg_UsePageAttributeTable=1 and NVreg_InitializeSystemMemoryAllocations=0 are still valid and you can set them yourself in `/etc/modprobe.d/nvidia.conf. I will also still maintain the rest of the theoretical material in the GitHub repository.


### Other distros

There are no packages for other distributions yet. However, you can manually
copy all necessary configuration files:

```
git clone https://www.github.com/ventureoo/nvidia-tweaks.git
cd nvidia-tweaks
sudo cp nvidia-tweaks.conf /etc/modprobe.d/nvidia-tweaks.conf
sudo cp 60-nvidia.rules /etc/udev/rules.d/
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

- ``nvidia_drm.modeset=1`` - enables modesetting support for the NVIDIA driver.
  Critical for Wayland support and proper PRIME Offload operation. **Be sure to
  enable**.

- ``nvidia_drm.fbdev=1`` - enables hardware framebuffer support. Allows to use
  native display resolution in tty. This option has no effect on PRIME laptops,
  as the framebuffer is handled by the integrated graphics. ~~This parameter is
  marked as experimental, so bugs may occur.~~ After 570.xx, this parameter is
  enabled by default and is no longer experimental.

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

There are also tweaks as an option:

- Installing the nvidia-patch from keylase
  - This patch removes restriction on maximum number of simultaneous NVENC
    video encoding sessions imposed by Nvidia to consumer-grade GPUs. You can
    read more about it here: https://github.com/keylase/nvidia-patch
  - As an alternative, there is [nvlax](https://github.com/illnyang/nvlax).
    Unlike nvidia-patch, it can be used for any driver version.
- Power Setup (PowerMizer)
> [!WARNING]
> PowerMizer options doesn't work anymore after 530.41.03 update. You can only use these parameters on driver versions lower than 530, that is, on the 470.xx and 390.xx legacy branches.
> Read more here: https://forums.developer.nvidia.com/t/kernel-module-option-nvreg-registrydwords-for-powermizerenable-doesnt-work-on-530-41-03/247610
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
https://forums.developer.nvidia.com/t/if-you-have-gpu-clock-boost-problems-please-try-gl-experimentalperfstrategy-1/71762/14).
Another issue in the latest driver versions is that
ForceFullCompositionPipeline breaks VK_KHR_present_wait, which affects many
games (See: https://github.com/ValveSoftware/Proton/issues/6869).

So I recommend everyone to use the first way whenever possible. At the moment,
all modern DEs and consequently the composers attached to them are good at
solving this issue. Particularly GNOME and since version 5.21 of Plasma no
longer have any problems (the main thing is to switch the rendering backend to
OpenGL). However, for all less modern environments such as Xfce and Mate, and
of course all window managers, the problem is still relevant. In this case it
is necessary to perform an installation of a separate compositor like Picom.
Now it is good enough, the most important thing is to add it to the autostart
with the following options to start it:

``picom --backend glx --vsync``

For Xfce, you must get rid of the problematic default compositor before using
Picom:

``xfce-query -c xfwm4 -p /general/use_compositing -s false``

P.S. All of the above only makes sense for the X11 sessions. Wayland implies
the use of compositing as a built-in feature.

## Do not use the nvidia-settings or nvidia-xconfig utilities to generate xorg.conf

Note, the author strongly discourages setting up your monitors and saving the
``xorg.conf`` config via nvidia-settings or nvidia-xconfig. This is primarily
because it is simply not necessary. Modern versions of the X server perform
autoconfiguration and working monitors detection themselves, besides most DEs
in their settings already allow you to set the required display refresh rate
and layout for external monitors, overriding all changes made in the xorg.conf
file, which is static and cannot be adjusted to changes in your hardware
configuration (for example, connecting a second monitor on the fly will cause
problems, as it isn't specified in xorg.conf, and autodetection in the presence
of a configuration file. The nvidia-settings options are also limited in
configurations with hybrid graphics (PRIME) or Wayland sessions.

More details on the issues that can arise when using nvidia-settings as a
configurator for Xorg can be found here:

https://unix.stackexchange.com/questions/697517/how-to-correlate-xorg-conf-config-for-nvidia-gpu-with-xrandr-detected-screens/697553#697553

If, however, you are a user of tiling window managers (WM), where there are no
convenient out-of-the-box customization tools, the author recommends that you
use tools such as xrandr and [picom](https://github.com/yshui/picom).

## Wayland + NVIDIA

Currently, starting with driver branch 515 and higher, NVIDIA has support for
GBM API, which made it possible to run Sway and wlroots-like window managers,
and also significantly improved Wayland sessions in GNOME/KDE Plasma.

Most of the flickering and artifact issues on Nvidia have been fixed with the
introduction of Explicit sync [1], which has been supported in the Nvidia
driver since branch 555, so it is strictly recommended to use the latest
version of driver on Wayland. Note that support for explicit sync on the
compositor side was added in KDE Plasma 6.1 and GNOME 46.2, sway 1.10 or anoter
wlroots-based compositor that using at least wlroots 0.18.

I would also like to point out that GNOME Wayland and Plasma Wayland are now
working at a pretty good level, it's not perfect, but progress is significant.
For gamers I would also recommend using the native Wayland driver in Wine, this
achieves less lag input and avoids Xwayland issues. So, here's a set of
environment variables to help you on NVIDIA using Wayland:

```
SDL_VIDEODRIVER=wayland # Can break some native games
QT_QPA_PLATFORM="wayland;xcb" # Only for Qt 5 apps
MOZ_DBUS_REMOTE=1 # For shared clipboard with Xwayland apps
GBM_BACKEND=nvidia-drm
WLR_NO_HARDWARE_CURSORS=1
_JAVA_AWT_WM_NONREPARENTING=1
```

It is best to place these variables in ``/etc/environment`` for greater
reliability. This should also work regardless of the shell or distribution
used. Not all of them are useful if you are not using the wlroots-based window
manager. I would also recommend you to avoid Chromium (without
Ozon)/Electron-based applications, as they can all be very unstable in Wayland
on NVIDIA. ~~Browsers on QtWebEngine such as qutebrowser unfortunately do not
work either.~~ QtWebEngine works if you use ``QT_QUICK_BACKEND=software``
environment variable.

If you do not have a Wayland session to choose from in GDM, then include the
``NVreg_PreserveVideoMemoryAllocations=1`` parameter to the ones we already
described above.

```
  sudo systemctl enable nvidia-resume.service nvidia-suspend.service nvidia-hibernate.service
```

This will also allow you to avoid some sleeping issues on Wayland.

> [!CAUTION]
>  On laptops with PRIME enabling ``NVreg_PreserveVideoMemoryAllocations=1``
>  may cause issues with sleep. Therefore it is recommended to use it only on
>  desktop PCs.

[1] - https://github.com/NVIDIA/egl-wayland/pull/104

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

> [!WARNING]
> Gamescope crashes when this variable is used:
> https://github.com/ValveSoftware/gamescope/issues/526#issuecomment-1733739097

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

> [!WARNING]
> I strongly advise against specifying the above environment variables for the
> whole system. Please specify them for specific applications/games with
> nvidia-settings or using Lutris/Steam.

## GSP firmware

**NOTE:** Starting with version 555.42.02, the use of GSP firmware is enabled
by default on GPUs where it is supported. In open source versions of the NVIDIA
driver modules, GSP is always used and you cannot disable it.

GSP (GPU System Processor) - this is a special chip which is present on NVIDIA
video cards starting from Turing and above, which offloads GPU initialization
and control tasks, which are usually performed on CPU. This should improve
performance and reduce the load on the CPU.

Some users report issues when using GSP, such as minor performance drops. You
can disable GSP using the ``NVreg_EnableGpuFirmware=0`` option by adding it to
the ``/etc/modprobe.d/nvidia.conf`` config.  Note that this parameter works
only for closed driver modules.

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

## Partial unlocking of TDP limit on laptops with Ampere GPUs and higher

Unfortunately on new driver versions it is not possible to set the TDP limit
manually via nvidia-smi. But for users of laptops with Ampere generation GPUs
(RTX 30xx) and higher, there is a workaround that partially solves the issue by
slightly increasing the TDP limit. For this purpose, you just need to enable
the ``nvidia-powerd`` service, which enables Dynamic Boost technology:

```
 sudo systemctl enable nvidia-powerd
```

For example on a laptop with a 3050 Mobile this allows you to raise dynamically
(i.e. depending on system load) the TDP limit from 35W to 40W, without
significant temperature changes.

Note that Dynamic Boost technology only works when the laptop is running on AC
power and also affects CPU performance by changing the maximum frequency.

See more:

https://download.nvidia.com/XFree86/Linux-x86_64/565.57.01/README/dynamicboost.html

## Firejail + NVIDIA

For 3D acceleration to work properly via NVIDIA GPU inside Firejail starting
with the 550.xx driver and above, you need to add the following lines to your
``~/.config/firejail/globals.local``:

```
noblacklist /sys/module
whitelist /sys/module/nvidia*
read-only /sys/module/nvidia*
```

## Low-latency frame presentation mode

Starting with the 570.xx driver, the NVIDIA driver has a so-called low-latency
display mode, which allows you to get latency close to the vblank value. You
can enable this mode by adding ``NVreg_RegistryDwords=RMIntrLockingMode=1`` to
your modprobe configuration for the nvidia module. Note that this is
experimental and may have potential issues.

## Hybrid graphics (NVIDIA Optimus)

All sorts of fun :)

First, let's understand what hybrid graphics are (Of course, technical details
are not something everyone wants to go into, but it is necessary for further
understanding).  Hybrid graphics is a hardware configuration in which you have
two graphics cards that can work in tandem with each other. This approach is
mainly found in laptops where you have integrated graphics (iGPU) of your CPU,
and discrete graphics (dGPU). The main advantage is that integrated graphics
should (but not necessarily) only be used for low-profile tasks, such as
surfing the Internet, watching videos, etc. And discrete graphics are used for
high-performance things like gaming, editing, 3D modeling, and so on.
Consequently, if two GPUs share "big" and "small" tasks, then if we have only
"small" tasks running at the moment, we don't need to use our dGPU, so it can
simply be disabled (as if asleep), thereby significantly reducing power
consumption. This way when our dGPU is needed again (we run an application
using it), it will wake up and start working.

It sounds good on paper, as it allows modern laptops to balance low power
consumption while leaving the ability to engage high-performance graphics, but
in practice there are a lot of problems. Obvious problem No. 1, which lies in
concept: Which GPU will do the output to screen? There are several variations
here:

1. (MUXless) The most common case is when the integrated graphics completely
   control your (at least) internal display. This leads to the fact that
   firstly: you can not completly turn off iGPU, because it is responsible for
   output on internal display. Secondly, and more terribly, dGPU can't output
   anything directly to the screen, forcing the iGPU to be used as a buffer to
   which rendered frames for displaying on the screen go. As it turned out in
   practice, this is the source of most problems. Because there is an overhead
   of copying frames from dGPU to iGPU, and also iGPU and dGPU can be
   controlled by different drivers, which may not want to be friends with each
   other at all (hello Linux).

2. (MUXed) The best case (which of course is less common) is when you have a
   special multiplexer - MUX. The MUX, is controlling for which GPU will
   display the image on the screen. We will not go into the details of its
   work, but simply say that with it both GPUs can now directly output an
   image. Unfortunately, laptops with MUX are much rarer and usually more
   expensive.

3. (MUXed) There is also another option in which you already have two MUX. One
   MUX controls the output on the built-in screen, the second MUX controls the
   image output on the external display (ports).

For a better understanding, take a look at this picture.

![hyrbid-graphics](https://github.com/ventureoo/nvidia-tweaks/assets/92667539/2332379d-56c8-4771-b6d4-a1fa0060efaf)

Now that we have understood the concept, we can look at how it works in Linux
now.

PRIME is a unifying technology for working with different sets of hybrid
graphics in Linux, like NVIDIA Optimus/AMD Dynamic Switchable Graphics. PRIME
Offload is an implementation of the idea of moving the execution of render from
one GPU to another in Linux. PRIME support in a closed NVIDIA driver actually
started only with the 435.17 driver. So if you are a user of the outdated 390xx
or even 340xx driver branches, PRIME will not work for you. Note that I also
strongly discourage you from using outdated ways to handle hybrid graphics,
such as nvidia-xrun or Bumblebee. They are obsolete and unsupported (Bumblebee
has not been updated for over 8 years), run solely on hacks and have low
performance. At the same time the Nouveau driver supports PRIME Offload, which
can be an alternative for older dGPUs.

So, the good news about PRIME Offload is that you should already have it
working out of the box (yes, manual configuration of xorg.conf is not required
on the latest driver versions, since many options are already enabled by
default), under two conditions:

1. All NVIDIA driver modules are properly installed and loaded. Most of the
   problems with PRIME occur here because some people don't know that their
   dGPU is not properly serviced. On a desktop PC, the modules leak to a black
   screen at boot time, but on laptops it will just cause your system to rely
   entirely on the iGPU. So trying to use PRIME Offload will result in an
   error. You can check that all modules are loaded with ``lsmod | grep
   nvidia`` and also with ``lspci -k``. One of the common reasons why NVIDIA
   modules may not boot is that Secure Boot is enabled in your UEFI, so it is
   recommended to disable it (or to sign modules and kernel, but this is beyond
   the scope of this guide).

2. You have the option ``nvidia-drm.modeset=1`` enabled. This is also **very
   important** because even if you have the driver properly installed and
   working, without this option PRIME and some other features (such as Wayland,
   DMA-BUFs) simply will not work. See previous instructions to enable this
   option.

3. Don't use DDX drivers like ``xf86-video-intel`` and ``xf86-video-amdgpu``.
   ``xf86-video-intel`` is an obsolete driver that can cause many issues and
   glitches in rendering your desktop. ``xf86-video-amdgpu``, although it's
   actively maintained, doesn't support PRIME Synchronization technology
   (https://gitlab.freedesktop.org/xorg/driver/xf86-video-amdgpu/-/issues/11),
   which can cause tearing on laptops. Instead it is better to use the built-in
   DDX driver modesetting which requires no further setup, you just need to
   uninstall the old DDX drivers from system then modesetting started using.

To check that PRIME Offload is working properly, you need to use the following
set of environment variables:

```
# If the glxinfo command is not found, install mesa-utils
__NV_PRIME_RENDER_OFFLOAD=1 __VK_LAYER_NV_optimus=NVIDIA_only __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo | grep vendor
```

In the last two, the dGPU driver is set as the OpenGL/Vulkan provider used,
respectively.

If the output of the command indicates to you mention of your NVIDIA card,
and without specifying the environment variables an integrated graphics, then
everything is fine. This means you have the Reserve PRIME mode set up and
working correctly.

Note that the above environment variables can be replaced by the ``prime-run``
command in Arch-based distributions. Essentially ``prime-run`` is just a wrapper
script over these variables, but it is much more convenient to write prime-run
than a large set of environment variables.

Also note that on most systems running Vulkan applications and games that use
Vulkan via DXVK/vkd3d-proton by default will cause your dGPU to be used. You
can check this by running ``vkcube`` without any variables.

If not, you will probably have to specify the use of PRIME Offload either in
the Lutris settings (in the "Global options" section) or specify above
variables in the Steam game "Properties", remembering to add the ``%command%``
delimiter at the end, for example:

```
prime-run %command%
```

Or (if you're not on the Arch-based system):

```
__NV_PRIME_RENDER_OFFLOAD=1 __VK_LAYER_NV_optimus=NVIDIA_only __GLX_VENDOR_LIBRARY_NAME=nvidia %command%
```

Please note that some features of discrete graphics in this mode are somewhat
reduced. So, you will not be able to configure your monitors via
nvidia-settings as it was indicated in the previous section, because iGPU are
responsible for connecting and maintaining internal display (only if you have a
MUXless laptop) and external monitors. The possibility of overclocking and
manual power management of a dGPU is also excluded.

But the main issue with PRIME is the performance of external monitors. The
thing is, as stated in the beginning, main display of laptop is controlled by
iGPU. But external ports can be controlled by either dGPU or iGPU depending on
your laptop. In case they are controlled by the iGPU, you may be fine. But if
dGPU is used to control the external display, then problems begin.

Your external monitor may be artfect, or very slow. It's worth noting that this
isn't really an Nvidia driver issue, but also in compositors, as they aren't
very good at handling these situations, since the implication is that you only
have one GPU to manage all the displays. When multiple displays are handled by
different GPUs the compositor doesn't always know what to do.

Some people recommend using dGPU as the default GPU in this case, but I'm not
really in favor of this solution as it kills your laptop's power savings and on
MUXless configurations still causes frames to be copied to the iGPU.

My advice is to just wait for modern compositors to fix these issues. Some work
is already being done on this in GNOME and KDE:

https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/3304

https://bugs.kde.org/show_bug.cgi?id=452219

In Plasma 6, this issue will probably already be fixed by using the latest 550
driver and EGL_ANDROID_native_fence_fd extension in KWin (its support added by
Nvidia since 545).

Gnome users should switch to using version 46.1+ where this has been fixed.

### System hangs when using 550 driver

If you notice system hangs when using driver 550 and above, try switching to
using open NVIDIA modules. This is a known issue, see here:
https://forums.developer.nvidia.com/t/series-550-freezes-laptop/284772/168
