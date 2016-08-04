---
layout: post
title:  "MPSoC: APU L2 cache is held in reset"
date:   2016-08-04 08:00:00
categories: Experiences
tags:
- SDK
- XSDB
---
用 JTAG 调试 ZYNQ MPSoC 的时候，可以使用 SDK 进行功能完整的程序调试，也可以用 XSDB 进行底层命令操作。SDK 其实也是通过 XSDB 这个命令接口和芯片通信的。

根据使用 ZYNQ-7000 的经验，在 XSDB 手动操作的时候，一般经过这几个步骤：

0. (Optional) 下载bit文件 `fpga -f xx.bit`
1. 导入初始化代码 `source ps7_init.tcl`
2. 执行初始化流程 `ps7_init`
3. (Optional) 如果下载了bit文件，就要打开 PS-PL 之间的接口通道 `ps7_post_config`
4. 下载软件代码或进行其他后续操作 `dow xx.elf` 或 `mrd <memory address>`

对于 ZYNQ UltraScale+ MPSoC，如果使用以上流程，会在第二步出现一个错误信息 `APU L2 cache is held in reset`，导致后续的 `dow` `mrd` 等命令都无法执行。

仔细在 SDK Log 窗口对比 SDK 的运行流程后发现，`psu_init` 后对于 MPSoC 还需要一个步骤：`rst -processor`，这样就解除了 L2 Cache 的reset，可以对内存地址进行操作了
