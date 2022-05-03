# nvidia-tweaks

## Description

nvidia-tweaks is a set of tweaks and workarounds for the Linux version of the closed NVIDIA driver. It will allow you to fix some problems related to the driver and can potentially improve performance (Read about the driver module options).

**DISCLAIMER: Don't expect much from this project. The driver is still closed and distributed under the NVIDIA license. There is no way to make your own changes directly to the driver code, therefore all our ability to improve it is severely limited.**

In other words, this project doesn't contain anything you can't do yourself, but just simplifies the process and stores a lot of information.

## Installation

First of all, you must have the NVIDIA driver itself installed and loaded in order for this tweak kit to work. Without it, no further action is meaningful.

And yes, **please ONLY install the NVIDIA driver using the package manager of your distribution!** Installing it manually from the official NVIDIA website may make it more difficult to maintain and impossible to update.

Specific instructions on how to install the driver can be found here:

https://github.com/lutris/docs/blob/master/InstallingDrivers.md

### Arch-based systems (AUR)

Arch Linux systems can simply install the nvidia-tweaks package from the AUR. This package is independent of the specific driver version and can be used for any driver package. It is only necessary that the driver is installed (NVIDIA MODULE is provided).

```
git clone https://aur.archlinux.org/nvidia-tweaks.git
cd nvidia-tweaks
makepkg -sric
```

You can also edit the PKGBUILD before installing it to include some additional settings, which we will talk about later.

### Other distros

There are no packages for other distributions yet. However, you can manually copy all necessary configuration files:

```
git clone https://www.github.com/ventureoo/nvidia-tweaks.git
cd nvidia-tweaks
sudo cp -r nvidia.conf /etc/modprobe.d/nvidia-tweaks.conf
sudo cp -r nvidia-uvm.conf /etc/modules-load.d/
sudo cp -r 60-nvidia.rules /etc/udev/rules.d/
```

The keylase nvidia-patch requires a separate installation. You can read about it on the project site:
https://github.com/keylase/nvidia-patch

## So, how does it work?

First, we configure the kernel modules of the NVIDIA driver (`nvidia.conf`). The thing is that by default, not all parameters are enabled in the driver module. Including the following things we need to change:

- NVreg_EnablePCIeGen3=1 (Default 0) - Enabling full PCIe 3.0 support. If the system supports this 8GT high speed bus then enable it with this module option flag. **If this feature is enabled but the system does not support Gen 3.0, system behavior can become unstable and unstable.** However, almost the vast majority already have motherboards with PCIe 3 or higher. So this parameter can be considered relatively safe. However, if you want to protect yourself and check your own PCIe version run the command: `nvidia-settings --query=PCIEGen`. If the output of the command is the number "3", then everything is fine.
- NVreg_UsePageAttributeTable=1 (Default 0) - Activating the better memory management method (PAT).
The PAT method creates a partition type table at a specific address mapped inside the register and utilizes the memory architecture and instruction set more efficiently and faster.
If your system can support this feature, it should improve CPU performance. To check if your processor is supported by PAT, use the command: `grep -E '^flags.+ pat( |$)' /proc/cpuinfo`.
If the output of the command was not empty, then all is OK (98% of modern processors today have PAT support).

  See also:
  
  https://bbs.archlinux.org/viewtopic.php?id=242007,
  https://old.reddit.com/r/linux_gaming/comments/9wjpkz/poor_gaming_performance_is_this_a_cpu_or_gpu/
- NVreg_InitializeSystemMemoryAllocations (Default 1) - Disables clearing system memory allocation before using it for the GPU.
Potentially improves performance, but at the cost of increased security risks.
Write `options nvidia NVreg_InitializeSystemMemoryAllocations=1` in `/etc/modprobe.d/nvidia.conf`,
if you want to return the default value.

  **Note**: It is possible to use more VRAM (?)

Then we install udev rules from negative17 (https://github.com/negativo17/nvidia-kmod-common/blob/master/60-nvidia.rules). These udev rules are required for node presence and runtime PM and fixes problems with raytracing in vkd3d (https://github.com/HansKristian-Work/vkd3d-proton/issues/711, https://github.com/HansKristian-Work/vkd3d-proton/issues/902).

**Note**: The driver packages in Arch Linux already have these rules by default (See: https://github.com/archlinux/svntogit-packages/commit/3c0985a523c8795a483f3c63d177508717603b65). But for other distributions this may still be necessary. Also, the packages for Arch appear to have an incomplete set of rules, so installing them may still be useful.

Installing `nvidia-uvm.conf` in `/etc/modules-load.d/` together with udev rules also fixes a common issue with nvidia-uvm module loading, which can break CUDA, including NVENC.

There are also tweaks as an option:

- Installing the nvidia-patch from keylase
  - This patch removes restriction on maximum number of simultaneous NVENC video encoding sessions imposed by Nvidia to consumer-grade GPUs. You can read more about it here: https://github.com/keylase/nvidia-patch
  - In the AUR package, you can get it by enabling the option in PKGBUILD: `_nvidia_patch=y`. No additional actions are required, the patch is applied automatically by the pacman hook.
  - As an alternative, there is [nvlax](https://github.com/illnyang/nvlax). Unlike nvidia-patch, it can be used for any driver version.
- Power Setup (PowerMizer)
  - Another kernel module parameter of the `RegistryDwords` driver allows you to configure the GPU power supply. To be more precise, configure the driver power manager - PowerMizer.
  - PowerMizer has three levels of performance:
    - 1 (0x1 bit) - Maximum performance
    - 2 (0x2 bit) - Balance between performance and power saving
    - 3 (0x3 bit) - Powersaving
  - PowerMizer also has two frequency management strategies:
    - Adaptive strategy (Frequencies change depending on the load on the GPU)
    - Static strategy (Frequencies are fixed at strictly one of the performance levels described above)
  - Power saving is set separately for AC and battery (if you have one). That is, you can select separately one frequency management strategy for the battery, and separately for AC. Also if a static strategy has been selected for any of the power supplies, you can set its specific performance level through their respective flags `PowerMizerDefault` (battery) and `PowerMizerDefaultAC` (AC).
  - All PowerMizer settings are specified with a semicolon as part of the `NVreg_RegistryDwords` kernel module parameter. For example: `NVreg_RegistryDwords="PowerMizerEnable=0x1;PerfLevelSrc=0x2222;PowerMizerDefault=0x3;PowerMizerDefaultAC=0x1"` - is set Static strategy with maximum performance for AC operation and maximum power savings for the battery.
  - For convenience, the AUR package has a ready-made set of PowerMizer tuning diagrams. You can apply them by setting the desired one in the PKGBUILD `_powermizer_scheme=value` parameter. See PKGBUILD for a description of possible values.
  - For other distributions you have to manually edit `/etc/modprobe.d/nvidia.conf` and the `NVreg_RegistryDwords` parameter. 
  - See also: https://wins911.blogspot.com/2012/06/etcx11xorg.html
- Force maximum performance (For laptops only)
  - `OverrideMaxPerf` flag for `NVreg_RegistryDwords` parameter enforces applying a certain performance level while activating some GPU overclocking possibilities, such as controlling GPU fan speed via nvidia-settings.
  - The flag takes values in bits only. For example: `NVreg_RegistryDwords="OverrideMaxPerf=0x1"` - Forces of maximum performance on any power source.
  - **Works only for laptops.**
  
All of the optional tweaks mentioned above should be applied according to your preferences. For a package from the AUR, they are applied through the corresponding options in PKGBUILD. For other distributions this will probably require editing `/etc/modprobe.d/nvidia-tweaks.conf` and adding the appropriate parameters.
