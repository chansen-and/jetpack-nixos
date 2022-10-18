# NixOS support for NVIDIA Jetson Devices

This repository packages components from NVIDIA's Jetpack SDK for use with NixOS, including:
 * Platform firmware flashing scripts
 * A 5.10 Linux kernel from NVIDIA, which includes some open-source drivers like nvgpu (built from source)
 * An [EDK2-based UEFI firmware](https://github.com/NVIDIA/edk2-nvidia) (built from source)
 * OP-TEE (can be built from source, but not yet implemented)
 * Additional packages for:
 ** GPU computing: CUDA, CuDNN, TensorRT
 ** Multimedia: hardware accelerated encoding/decoding with V4L2 and gstreamer plugins
 ** Graphics: Wayland, GBM, EGL, Vulkan
 ** Power control: nvpmodel

This package is based on the Jetpack 5 release, and will only work with devices supported by Jetpack 5:
 * Jetson Orin AGX
 * Jetson Xavier AGX
 * Jetson Xavier NX

These devices are _not_ supported:
 * Jetson Nano
 * Jetson TX2
 * Jetson TX1

In the future, when the following devices are released, it should be possible to make them work as well:
 * Jetson Orin NX
 * Jetson Orin Nano

## Getting started

### Flashing platform firmware
This step may be optional if your device already has recent-enough firmware which includes UEFI. (Post-Jetpack 5. Only newly shipped Orin devices might have this)
If you are unsure, I'd recommend flashing the firmware anyway.

Plug in your jetson device, press the "power" button" and ensure the power light turns on.
Then, to enter recovery mode, hold the "recovery" button down while pressing the "reset" button.
Connect via USB and verify the device is in recovery mode.
```shell
$ lsusb | grep -i NVIDIA
Bus 003 Device 013: ID 0955:7023 NVIDIA Corp. APX
```

On an x86_64 machine (some of NVIDIA's precompiled components like `tegrarcm_v2` are only built for x86_64),
build and run (as root) the flashing script which corresponds to your device:
```shell
$ nix build github:danielfullmer/jetpack-nixos#flash-xavier-agx-devkit
$ sudo ./result/bin/flash-xavier-agx-devkit
```

At this point, your device should have a working UEFI firmware accessible either a monitor/keyboard, or via UART.

### Installation ISO

Now, build and write the customized installer ISO to a USB drive:
```shell
$ nix build github:danielfullmer/jetpack-nixos#installer-iso
$ sudo dd if=./result/iso/nixos-22.11pre-git-aarch64-linux.iso of=/dev/sdX bs=1M oflag=sync status=progress
```
(Replace `/dev/sdX` with the correct path for your device)

As an alternative, you could also try the generic ARM64 multiplatform ISO from NixOS. See https://nixos.wiki/wiki/NixOS_on_ARM/UEFI
(Last I tried, this worked on Xavier AGX but not Orin AGX. We should do additional testing to see exactly what is working or not with the vendor kernel vs. mainline kernel)

### Installing NixOS

Insert the USB drive into the Jetson device.
On the AGX devkits, I've had the best luck plugging into the USB-C slot above the power barrel jack.
You may need to try a few USB options until you find one that works with both the UEFI firmware and the Linux kernel.

Press power / reset as needed.
When prompted, press ESC to enter the UEFI firmware menu.
In the "Boot Manager", select the correct USB device and boot directly into it.

Follow the [NixOS manual](https://nixos.org/manual/nixos/stable/index.html#sec-installation) for installation instructions, using the instructions specific to UEFI devices.
Include the following in your `configuration.nix` (or `flake.nix`, or whatever) before installing:
```nix
{
  imports = [
    (builtins.fetchTree "https://github.com/danielfullmer/jetpack-nixos/master/...") + "/module.nix")
  ];

  hardware.nvidia-jetpack.enable = true;
}
```
The Xavier AGX contains some critical firmware paritions on the eMMC.
If you are installing NixOS to the eMMC, be sure to not remove these partitions!
You can remove and replace the "UDA" partition if you want to install NixOS to the eMMC.
Better yet, install to an SSD.

After installing, reboot and pray!

Xavier AGX side note:
On all recent Jetson devices besides the Xavier AGX, the firmware stores the UEFI variables on a flash chip on the QSPI bus.
However, the Xavier AGX stores it on a uefi_variables partition on the eMMC.
This means that it cannot support runtime UEFI variables, since the UEFI runtime drivers would conflict with the Linux kernel's drivers for the eMMC.
Concretely, that means that you cannot modify the EFI variables from Linux, so UEFI bootloaders will not be able to create an EFI boot entry and reorder the boot options.
You may need to enter the firmware menu and reorder it manually so NixOS will boot first.
(See: https://forums.developer.nvidia.com/t/using-uefi-runtime-variables-on-xavier-agx/227970)

## License

TODO

## Additional Links

Much of this is inspired by the great work done by [OpenEmbedded for Tegra](https://github.com/OE4T).
We also use the cleaned-up vendor kernel from OE4T.

https://elinux.org/Jetson

https://developer.nvidia.com/embedded-computing
https://developer.nvidia.com/embedded/jetpack
https://forums.developer.nvidia.com/c/agx-autonomous-machines/jetson-embedded-systems/70

