---
layout: post
title:  "ZYNQ to MPSoC: Reset Signal from PS to PL"
date:   2016-08-14 08:00:00
categories: Experiences
tags:
- MPSoC
- Vivado
- SDK
- ZYNQ_2_MPSoC
---

ZYNQ 系统中，PS 到 PL 的 Reset 信号是一个特殊信号，通过 SLCR.FPGA_RST_CTRL 寄存器控制。

ZYNQ MPSoC 系统中，PS 到 PL 的 Reset 信号不再像 ZYNQ 一样使用专用走线，通过 SLCR 的特殊控制，而是利用了 96 个 EMIO GPIO 的最高位走线，在软件层也就是通过 GPIO 的控制寄存器来控制这些 Reset 信号线。所以在 Vivado IPI 中，PS 到 PL 的 reset 信号虽然叫做 pl_reset0，是一个属性为 reset signal 的信号，但它其实是一个 EMIO GPIO 信号。 

SDK 2016.4 中产生的 psu_init.c/tcl 中有一个函数 `psu_ps_pl_reset_config` 描述了进行 PL Reset 的流程。

- 设置方向
- 设置使能
- Toggle 1-0-1 （低有效）