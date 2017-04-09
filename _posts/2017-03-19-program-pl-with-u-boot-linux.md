---
layout: post
title:  "2016.3: 在 U-Boot 和 Linux 中下载 MPSoC PL Bit 文件"
date:   2017-03-19 14:00:00
categories: Experiences
tags:
- 2016.3
- Linux
- U-Boot
- PL
- MPSoC
---
# 背景介绍

ZYNQ 和 MPSoC 的 PL Bit 可以在多个软件启动阶段被加载，它们分别是 FSBL、U-Boot、Linux。但是由于 MPSoC 更高级的安全机制，运行在 EL2 的 U-Boot 和 EL1 的 Linux 并没有足够权限对 PL 部分做配置级别的修改。因此，如果要在 U-Boot 和 Linux 中配置 PL，需要借助 ATF(EL3) 和 PMU 来完成。这也是为什么这样一个常用功能需要等到 2016.3 才被支持的原因。

# 主要参考文档	

[http://www.wiki.xilinx.com/Solution+ZynqMP+PL+Programming](http://www.wiki.xilinx.com/Solution+ZynqMP+PL+Programming)

文档中的各个细节步骤都需要遵守，否则就可能配置不成功。因此一下罗列一些操作注意要点。

# 流程描述和细节注意点

## 准备
将 bit 文件转换为 bin 文件（U-Boot 支持 Vivado 生成的原始 bit 格式，Linux 只支持转换后的 bin 格式）

## U-Boot
- 先将 bin(或 bit)存到内存中。通过SD卡（fatload mmc）、以太网(tftpboot)等模式都可以。注意加载 bit 的时候不要将 U-Boot 的代码覆盖掉，U-Boot 默认运行位置在`0x08000000`地址。
- 用`fpga loadb 0 <bitstream address> <bitstream file size>`将 bit 写到 PL。
- PetaLinux 自带的 U-Boot 默认打开了 FPGA 命令。如果以上命令没有响应，请确认 U-Boot 的编译选项。


## Linux
- 更改 Kernel 编译选项： `FPGA Configuration Framework` 和 `Xilinx Zynqmp FPGA` 都要按 Y，使它变成`*`状态。不能在`M`状态，详情查看限制第三条
- 将 bin 文件拷贝到 `/lib/firmware`目录中。**注意不支持 bit**。
- 使用`echo`命令将 bin 的文件名发送到`/sys/class/fpga_manager/fpga0/firmware`。**注意是文件名，而不是文件**。

# 注意功能限制
- 不支持 Partial, Encrypted, Authenticated Bitstream
- Linux 只支持 bin 格式，不支持其他格式
- 必须将驱动编译到内核中，不能作为External Module
- 需要使用[AR68246](https://www.xilinx.com/support/answers/68246.html) 提供的 xilfpga 补丁