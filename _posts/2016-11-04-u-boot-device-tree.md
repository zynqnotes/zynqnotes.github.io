---
layout: post
title:  "PetaLinux 2016.3 中 U-Boot 使用的 Device Tree 文件是从哪里来的？"
date:   2016-11-04 12:00:00
categories: Experiences
tags:
- PetaLinux
- U-Boot
- device-tree
---

在 PetaLinux 2016.3 中用 U-Boot 的时候，遇到一个问题感觉和 device tree 有关，于是追踪了一下 device tree 文件的来源。

## 结论 ##

先说结论：至少在 PetaLinux 2016.3 中，U-Boot 是和 Linux 共用 device tree 文件的。其他版本未查验。

## 现象 ##

1. 如果一个新 PetaLinux 工程导入 HDF 后，没有 build device tree，而直接`petalinux-build -c u-boot`，会报错 `No rule to make target 'build/linux/device-tree/system.dtb', needed by U-Boot. Stop.`

2. `build/u-boot/u-boot-plnx/dts`目录中有`dt.dtb`，这是最后合成到 u-boot.elf 中的 device-tree。这个目录中有许多隐藏文件，比如`.dt.dtb.cmd`，这个文件是生成 dt.dtb 的命令，它的内容是

```
cmd_dts/dt.dtb := cat build/linux/device-tree/system.dtb > dts/dt.dtb
```

目录中还有 link script 文件，指定将 dtb 的内容放入某个 section，以便 dtb 作为一个数据段存入最终的 elf 中。

## 误导人的现象 ##

1. 在 PetaLinux 安装目录中，arch/arm/dts子目录，有很多和 zynq/zynqMP 相关的 demo 板的 dts 文件。这些 dts 在 build 过程中是会被编译成 dtb 的，但其实最后并没有被使用。

2. 用`petalinux-config -c u-boot`进行U-Boot的配置，进入 device-tree 配置，里面有地方填入 device tree 的名字，这里默认是`zynqmp-zcu102`，但其实最后并没有被使用。
