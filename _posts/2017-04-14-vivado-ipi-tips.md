---
layout: post
title:  "用 IPI 搭建 ZYNQ 工程常见问题、误区和技巧"
date:   2017-04-14 12:00:00
categories: Experiences
tags:
- Vivado
- IPI
- ZYNQ
- MPSoC
---

# Port vs. Interface

在 IPI Diagram 中，右键单击空白区域，会出现`Create Port`和`Create Interface Port`的选项。 Port 和 Interface 有什么区别呢？

![](/images/2017/ipi_create_interface.png "IPI Create Interface Port")

- Interface 是带协议的一组接口，比如 AXI, AXI Lite, AXI Stream 都属于 Interface。 Interface 是不同意义信号的组合。
- Port 是单根信号线，或者同一用途信号线的组合 (vector)，比如数据总线`data[31:0]`只包含数据。 在`Create Port`对话框可以定义信号的 type，比如时钟、复位，或者其他。

AXI 对外是一个 Interface；AXI 的时钟和复位信号对外都是 Port。

在实际的操作中，我们很少手工创建一个 Port 或者 Interface，而是选择已有 IP 的某个 Port，右键后选择 `Make External` 或 `Ctrl+T`，这样就会自动创建与它相同类型的 Port/Interface，然后连接。

# Interface vs. Bus

- Interface 翻译为接口，只规定信号组合的规范，而不定义连接的拓扑结构。
- Bus 翻译为总线，稍微有点含糊，根据上下文不同，有时可以指 Interface, 有时候又可以指某种拓扑结构。

AXI 是 Interface，怎么连接并没有在 AXI 协议中规定。点到点直连，还是主到多个从做选通；谁做仲裁，谁控制优先级等等关系到总线实现的部分，都没有在 AXI 协议中规定，于是 Vivado 中的 AXI Interconnect 就以自己的方式做了一套实现。

# AXI 及其相关信号

在 IPI 中，AXI Interface 中上百根信号线用一根粗线来简化，于是很容易让人误以为这根粗线就是 AXI 的全部。

其实，一组完整的 AXI Interface，不仅包含这根粗线，还包含一根随路时钟线 ACLK，以及一根复位线 resetn。连接一个 AXI 总线的时候，需要总体考虑这三根线应该怎样连接。甚至扩展开来说，在考虑一组任意总线连接的时候，我们都要有一个意识，同时考虑它的时钟方案和复位方案。具体时钟方案和复位方案就不在这里展开讲了。

![](/images/2017/ipi_axi_signals.png)

# Bus Associated Clock

对于一个与 IPI Diagram 外部相连的 Interface，IPI 无法得知这组总线在 IPI 外部采用了哪个时钟和复位连接，但是它需要时钟的信息来做内部连接和结构的 Infer。比如，当一组 AXI 是从 AXI Interconnect 的 Master 端连接到外部，IPI 就可以根据外部 Interface 关联的时钟设置，来计算得到在这个 AXI Interconnect 模块中是否需要实现一组跨时钟域逻辑。

实际操作上，在 IPI Block Design 中每一个 External Interface 都需要设置 Associated Clock. 单击 Interface 图标，然后在它的属性对话框中从下拉菜单中选择时钟信号。

![](/images/2017/ipi_3.png)

![](/images/2017/ipi_4.png)

这个时钟信号当然也需要是一个 External Port，才会出现在下拉菜单列表里。否则，只显示 `There are no clock ports in this design`。

![](/images/2017/ipi_2.png)
