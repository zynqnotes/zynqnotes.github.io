---
layout: post
title:  "U-boot使用的Device Tree是从哪里来的？"
date:   2016-11-04 12:00:00
categories: Experiences
tags:
- PetaLinux
- u-boot
- device-tree
---

在PetaLinux 2016.3中用u-boot的时候，遇到一个问题感觉和device tree有关，于是追踪了一下device tree的来源。

## 结论 ##
先说结论：至少在PetaLinux 2016.3中，u-boot是和Linux共用device tree的。其他版本未查验。

## 现象 ##
1. 如果一个新PetaLinux工程导入HDF后，没有build device tree，而直接`petalinux-build -c u-boot`，会报错 `No rule to make target 'build/linux/device-tree/system.dtb', needed by u-boot. Stop.`

2. `build/u-boot/u-boot-plnx/dts`目录中有`dt.dtb`，这是最后合成到u-boot.elf中的device-tree。这个目录中有许多隐藏文件，比如`.dt.dtb.cmd`，这个文件是生成dt.dtb的命令，它的内容是

```
cmd_dts/dt.dtb := cat build/linux/device-tree/system.dtb > dts/dt.dtb
```

目录中还有link script文件，指定将dtb的内容放入某个section，以便dtb作为一个数据段存入最终的elf中。

## 误导人的现象 ##
1. 在PetaLinux安装目录中，arch/arm/dts子目录，有很多和zynq/zynqMP相关的demo板的dts文件。这些dts在build过程中是会被编译成dtb的。

2. 用`petalinux-config -c u-boot`进行u-boot的配置，进入device-tree配置，里面有地方填入device tree的名字，这里默认是`zynqmp-zcu102`
