---
layout: post
title:  "加快调试 PetaLinux 速度"
date:   2016-04-27 08:00:00
categories: Experiences
tags:
- PetaLinux
- Debug
---

在调试 PetaLinux 和运行在 PetaLinux 上的软件的时候，经常会涉及到不停地重新编译、加载到板上的流程，有时效率比较低，以下为几个常用的加快调试速度的小方法。

## 善用仿真 ##
使用 QEMU 可以仿真与硬件无关的部分，包括内核相关的代码和应用代码。编译Kernel，启动仿真，比将代码下载到开发板上快不少，而且在没有开发板的时候也能调试。

- `petalinux-boot --qemu --kernel` 可以启动对 Kernel 的仿真
- `petalinux-boot --qemu --u-boot` 可以启动对 u-boot 的仿真

在启动对Kernel的仿真时，会先启动一个 u-boot 的仿真，如果 u-boot 能从以太网加载 kernel，就让 u-boot 加载。如果不想让 u-boot 加载 kernel，可以退出第一个Qemu流程 `Ctrl+A+X`，然后第二个 Qemu 流程会自动启动，直接加载 Kernel。

在2015.4的 PetaLinux 中似乎有个小 Bug：在自己建立的工程中如果用上述命令仿真，PetaLinux会抱怨`pre-built`目录中的文件找不到。我的解决方法是把 `image/linux` 目录软链接到 PetaLinux 所期望的位置。

```
mkdir -p pre-built/linux
ln -s image/linux prebuilt/linux/images
```

## 选择性编译 ##
petalinux-build 默认情况下会将 Kernel, App, device-tree, u-boot, FSBL 都重新编译一遍。
虽然 Make 工具会检查这些部分是否真的需要重新编译，不需要编译的话可以跳过，但至少让 Make 检查一遍也是要花时间的。

如果确切了解自己更改了那一部分，哪些模块需要重新编译，那么可以在 petalinux-build 后用 -c 选项指定编译的内容。

petalinux-build -c 后可以加的选项为：

1. `kernel`: Linux kernel
2. `device-tree`
3. `u-boot`
4. `rootfs`
5. `rootfs/app` 或 `rootfs/module`: 仅重新编译一个application或一个module

选择性编译部分模块后，再用 `petalinux-package --image` 将内容打包成image.ub。经常做这个步骤的话，可以用 `&&` 连接两个步骤，用一行命令完成编译与打包的功能。

如果需要将编译出的 `image.ub` 复制到 `/tftpboot` 目录，需要手动运行 `cp` 命令。当然，这一步也可以和上面两步操作用 `&&` 连接为一行指令。

## 从网络加载 Kernel ##
如果不是在调试与 boot.bin 相关的内容 (FSBL, bit, u-boot)，可以使用 u-boot 的 tftpboot 功能，从局域网加载 kernel image，这样不需要经常来回烧写 SD 卡或QSPI Flash.

如果需要启用 u-boot 的网络加载Kernel的功能，需要在 petalinux-config 中修改以下几个地方

1. `subsystem auto hardware settings -> advanced bootable images storage settings -> kernel image settings = ethernet`
2. `u-boot config -> tftp server ip address` 默认值是 AUTO，如果编译u-boot的host有多个网卡/IP 地址，最好手动指定 tftp server 的 IP 地址

## 自动化烧写 Boot.bin ##
当调试 Boot.bin 相关的部分时，需要不停地重新烧写 boot.bin 到 SD 卡上或 QSPI Flash 中。把这个过程自动化，就能省时省力防止出错。

从 Windows 复制文件到 SD 卡的 bat script:

```
copy BOOT.BIN f:\ /Y
RunDll32.exe shell32.dll,Control_RunDLL hotplug.dll
```

其中 `f:\` 是 SD 卡驱动器位置。
