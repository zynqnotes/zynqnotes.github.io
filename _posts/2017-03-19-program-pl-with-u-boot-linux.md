---
layout: post
title:  "2016.3: 在U-boot和Linux中下载PL Bit文件"
date:   2017-03-19 14:00:00
categories: Experiences
tags:
- 2016.3
- Linux
- u-boot
- PL
---
# 主要参考文档	

[http://www.wiki.xilinx.com/Solution+ZynqMP+PL+Programming](http://www.wiki.xilinx.com/Solution+ZynqMP+PL+Programming)

# 流程描述和细节注意点

## 准备
将 bit 文件转换为 bin 文件（u-boot支持Vivado生成的原始bit格式，但Linux中只支持转换后的bin）

## U-boot
- 先将bin(或bit)存到内存中。通过SD卡（fatload mmc）、以太网(tftpboot)等模式都可以。注意存bit的时候不要将u-boot的代码覆盖掉，u-boot默认运行位置在`0x08000000`。
- 用 `fpga loadb 0 <bitstream address> <bitstream file size>`将bit写到PL。

## Linux
- 更改 Kernel 编译选项： `FPGA Configuration Framework` 和 `Xilinx Zynqmp FPGA` 都要按 Y，使它变成`*`状态。不能在`M`状态，详情查看限制第三条
- 将 bin 文件拷贝到 `/lib/firmware`目录中。注意不支持bit。
- 使用`echo`命令将bin的文件名发送到`/sys/class/fpga_manager/fpga0/firmware`。注意是文件名。

# 注意功能限制
- 不支持 Partial,Encrypted, Authenticated Bit-stream
- Linux 只支持 bin 格式，不支持其他格式
- 必须将驱动编译到内核中，不能作为External Module
- 需要使用[AR68246](https://www.xilinx.com/support/answers/68246.html) 提供的 xilfpga 补丁