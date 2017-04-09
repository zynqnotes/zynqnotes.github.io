---
layout: post
title:  "使用 XSDB/XSCT 通过 JTAG 将 MPSoC 启动到 Linux Kernel"
date:   2017-03-19 13:00:00
categories: Experiences
tags:
- XSCT
- Linux
- U-Boot
---
# 背景介绍

很多时候我们需要用 JTAG 是调试 ZYNQ/MPSoC，比如 SDK Program Flash 工具无法烧写 QSPI Flash 的时候，我们需要用 JTAG 下载一个自己定制的 U-Boot 到内存，进行 Flash 的调试和烧写；比如外部存储器件使用受限（比如某些公司不能使用SD卡），但又需要调试 Linux 上层应用的时候等。因此我们在这里介绍一种自动化的脚本，执行脚本后就能自动将 MPSoC 加载到 U-Boot 阶段。如果需要，还可以进一步启动到 Linux。

XSDB 是 SDK 的调试命令行工具。它通过连接 hw_server 控制 JTAG 所发出的命令，与芯片进行连接，进行 SoC 芯片的调试与控制。

XSCT 是 SDK 的命令行工具，它底层包含了 XSDB 的所有功能，并且还能通过命令行控制 SDK，进行创建工程、编译工程等操作，所以我们一般直接使用 XSCT 来执行 XSDB 的命令。

# 需要加载和运行的组件

- psu_init.tcl: 芯片初始化脚本
- PMU Firmware: 推荐使用但不是必需，不使用会少一些功能)
- ARM Trusted Firmware (bl31.elf): 启动Linux必需的组件
- u-boot.elf: 可以由 PetaLinux 生成，或自己编译生成
- image.ub: PetaLinux 打包的 Kernel, rootfs以及device tree 多合一(FIT)启动包

# 加载脚本

XSCT 脚本：load_linux.tcl

```ruby
connect # 连接 hw_server
target 9 # PSU 主模块
source psu_init.tcl # 导入但不执行 psu_init.tcl
psu_init # 执行初始化动作
target 3 # PMU
dow pmufw.elf # 下载 PMU Firmware
con # 让 PMU 继续运行
target 10 # ARM Cortex-A53 CPU0
rst -processor # 从 L2 Cache Reset 状态中释放出来 
dow u-boot.elf # 下载 U-Boot 代码
dow bl31.elf # 下载 ATF 代码
con # 执行 ATF，ATF 会自动加载 U-Boot
after 8000 # 等待 ATF 和 U-BOOT 加载完成
stop # 暂停 A53_0
dow -data image.ub 0x10000000 # 将 Linux Image 通过 JTAG 加载到此地址，可能耗时很长
con # 继续运行 U-Boot
```

注意，以上代码包含了将 Linux Image 通过 JTAG 加载到内存的指令。如果有更快的加载通道，比如通过以太网、通过 SD 卡等其他加载方式，就不需要通过 JTAG加载。

# 如何在XSCT中运行XSCT脚本

1. 启动XSCT
2. `source load_linux.tcl`

# U-Boot 启动 Linux 命令

假设 Linux Image 已经被加载到内存地址 0x10000000，那么只需要执行`bootm 0x10000000`就可以进行启动 Linux 的动作。