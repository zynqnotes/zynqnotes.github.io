---
layout: post
title:  "中断（二）：PL 需要产生超过 16 个中断怎么处理？"
date:   2018-04-23 12:00:00
categories: Experiences
tags:
- PetaLinux
- MPSoC
- Interrupt
---

在`中断（一）：让 Linux 接收来自 PL 的自定义中断信号`中，介绍了怎样让 PS 收到来自 PL 的中断。MPSoC 中 PL 到 PS 的中断通道有 16 个。只要 PL 产生的中断数量在 16 个之内，都可以通过前面的方法连接和使用。如果 PL 需要产生超过 16 个中断，该怎么处理呢？

主要的想法就是扩展多层的中断控制器。GIC 可以作为 A53 的第一层中断控制器，所有中断都通过 GIC 上报。MPSoC 中，在 PL 侧可以使用 AXI INTC 或直接使用 PS GPIO Controller，两者都可以作为第二层中断控制器。用户逻辑产生的中断信号，先连接到二层中断控制器的入口，二层中断 IP 再产生新中断信号，传递到 GIC，上报给 A53。

Xilinx Wiki 有两种方式的使用说明，具体步骤不再赘述。
-	PS EMIO GPIO (http://www.wiki.xilinx.com/Device+Tree+Tips: 7 Interrupt Inputs Using GPIO)
-	PL Interrupt Controller (http://www.wiki.xilinx.com/Cascade+Interrupt+Controller+support+in+DTG )

编写 device tree 的几点说明：
- AXI INTC 或 PS GPIO Controller 的节点中添加 `interrupt-controller;` 属性，用来声明这个模块可以作为中断控制器，接受来自下级的中断信号。
- 产生中断的模块，添加类似 `interrupt-parent = <&gpio0>;` 的说明，指向上层中断控制器；`interrupts = < 0 4 >;` 这里的 `0` 是二层中断控制器的输入中断号，`4`表示中断类型（边沿/电平、上升/下降/高/低）。