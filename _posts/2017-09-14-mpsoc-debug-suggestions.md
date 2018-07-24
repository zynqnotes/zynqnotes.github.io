---
layout: post
title:  "一些关于 MPSoC 的调试建议"
date:   2017-09-14 12:00:00
categories: Experiences
tags:
- PetaLinux
- Yocto
- ZYNQ
- MPSoC
---

## 准备

### 准备好联网环境或下载 SSTATE CACHE

标准的 Yocto 编译过程中会从网络下载所有软件包的源文件。对于PetaLinux来说，Xilinx除了提供源文件，还提供了预编译的中间文件，或称为 SSTATE CACHE。在有 SSTATE-CACHE 的情况下，编译速度可以加快不少。另外，由于 SSTATE CACHE 可以离线下载，并设置到 `petalinux-config` 中，它也为没有网络的工作环境提供了编译支持。

SSTATE CACHE 可以从 [http://www.xilinx.com/download] -> [Embedded Development](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools.html) 下载。下载后的安装方法可以在它的 README 中找到。


### 确认编译环境安装了PetaLinux所需的32位工具库

编译完整的 PetaLinux Image 必须编译 PMUFW 这个组件。由于 PMU 其实是一个 MicroBlaze，它是个32位的处理器（固化在PS中，不占用PL逻辑资源）。32位的编译器需要32位的软件环境库来支持。

UG1144 中列举了 PetaLinux 运行所需要的库的列表。要保证 host 系统能够运行32位的软件程序。

### 准备足够大的磁盘

Yocto 的编译需要占用巨大的磁盘空间，通常一个工程编译一次就可能占用 50GB。因此有一个足够大的硬盘是必须的。

### 不要用root安装SDK和PetaLinux

其实 Xilinx 全线软件工具都不需要用 root 权限安装。对于 SDK 和 PetaLinux，不是"不需要"，而是"不能"用 root 安装，不然的话即使安装成功也会在运行时出现各种莫名的错误。

## Bring Up

### 确保内存稳定

硬件系统的任何不稳定情况都可能导致 Linux 系统崩溃，而且崩溃的时候可能找不出有用的线索。因此，一定要尽量保证硬件系统的稳定。

当运行 Linux 操作系统的时候，由于所有的操作都需要读写内存，而内存又是一个比较容易出现不稳定情况的外设，所以在开始移植 Linux 操作系统之前，请使用 SDK 建立一个 DRAM Test example 程序，做完整的 DDR Test，甚至跑多次以测试稳定性。

### 所有组件使用匹配的版本

有的时候做一个产品会跨越很多代 Vivado/SDK/PetaLinux 版本的发布，会需要升级版本。在升级的时候需要注意不要混用各种中间文件。

中间文件包括但不限于：
- HDF (由Vivado产生)
- FSBL
- PMUFW
- ATF
- U-boot
- Kernel

由于 PetaLinux 在测试的时候并没有测试过混合使用的情况，如果自行混用各种版本的组件，可能导致不兼容的情况，出现莫名的问题。


### 保持 Vivado 中的外设设置 和 PetaLinux 中的一致 

曾经出现过一些情况，由于 PetaLinux 的 u-boot 默认会编译 mmc 这条指令，而这条指令又是需要 SD 卡的外设驱动。外设驱动所需要的 SD 卡头文件，是由 Vivado 导出的 HDF 中的信息描述，如果在 Vivado 中没有打开 SD，就不会生成对应的头文件信息，PetaLinux 编译时就会出错。所以在 PetaLinux 报告此类错误的时候，请注意查看 Vivado 中是否打开了相应的外设。

## PetaLinux

### 关闭 CPUIDLE
CPUIDLE 特性可以将没有处在运行状态的 Core 设置成 WFE (Wait for Event) 状态来省电。但是这种状态有一个弊端：如果 JTAG 在 CPU 省电状态的时候向 CPU 查询某些信息，没有事先产生中断唤醒 CPU，就会将系统卡死。在调试阶段经常需要用 JTAG (XSCT, hw_server, 查询 ILA, 控制 VIO 等)，所以最好在调试时关闭 CPUIDLE 属性。

关闭方法参考： https://www.xilinx.com/support/answers/69143.html

- 方法一：在 bootargs 中添加 cpuidle.off=1
    - 要获取现在的 bootargs，可以在Linux Console 中输入 `cat /proc/cmdline`
    - 一般在 u-boot 中输入 `setenv bootargs "earlycon clk_ignore_unused mem=2G cpuidle.off=1"` 可以关闭 cpuidle，并启动 Linux
- 方法二：在 kernel config 中直接去掉 kernel 对 CPUIDLE 特性的支持。
