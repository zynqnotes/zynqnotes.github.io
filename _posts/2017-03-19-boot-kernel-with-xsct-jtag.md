---
layout: post
title:  "使用XSCT通过JTAG将MPSoC启动到Linux Kernel"
date:   2017-03-19 13:00:00
categories: Experiences
tags:
- XSCT
- Linux
- u-boot
---
# 需要加载和运行的组件

- psu_init.tcl: 芯片初始化脚本
- PMU Firmware: 推荐使用但不是必需，不使用会少一些功能)
- ARM Trusted Firmware (bl31.elf): 启动Linux必需的组件
- u-boot.elf: 可以由PetaLinux产生，或自己编译产生
- image.ub: PetaLinux打包的Kernel, rootfs以及device tree多合一启动包加载脚本

# 加载脚本

XSCT 脚本：load_linux.tcl
```
connect
target 9
source psu_init.tcl
psu_init
target 3
dow pmufw.elf
con
target 10
rst -processor
dow u-boot.elf
dow bl31.elf
con
after 8000
stop
dow -data image.ub 0x10000000
con
```

# 如何在XSCT中运行XSCT脚本

source load_linux.tcl

# u-boot 启动命令
```
bootm 0x10000000
```