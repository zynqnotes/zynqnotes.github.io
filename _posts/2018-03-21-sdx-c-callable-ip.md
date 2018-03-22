---
layout: post
title:  "SDx 2017.4 的 C Callabl IP 和 Pack RTL Kernel"
date:   2018-03-21 12:00:00
categories: Experiences
tags:
- SDSoC
---

关于 C Callable IP，UG1027上描述的不是很详细，但是在 SDx 安装目录的 sample 中有 RTL 的例子。如果之前有 HLS 基础，又了解 RTL，就会比较容易理解。下面简单讲解一下这个例子。

## 例子

在 SDx 安装目录中 `samples/rtl/` 有三个例子，分别是

- count：最简单的例子。HDL IP 中的寄存器可以通过 AXI 总线读取。
- axis_arraycopy：AXI Stream IP 的例子。有可以通过 AXI 总线控制的寄存器，也有通过 AXI Stream 接口作为数据通道输入输出数据的例子。IP 使用 HDL 写成。
- aximm_arraycopy：类似上面 AXIS 的例子，但是通过 AXI 接口作为数据输入和输出通道。IP 使用 HLS 的 C 代码生成。

三个例子中 count 是基础，因为所有 IP 都有一些控制寄存器。其他两个可以按需参考。这里各种 datamover 都有了，似乎就缺 aximm 的例子，否则就全了。

## 例子结构和工程流程

每个例子的目录下都有这些文件

- Makefile：总的流程控制。只要把例子目录拷贝到工作目录，设置好 SDx 工作环境 (source settings64.sh) 就可以在这个目录中执行 make，完成所有编译工作。
- ip: HDL IP 或 HLS IP 的源文件。在这里面也有 Makefile。这个目录里的文件最终生成一个包含 IP 的 Vivado 工程，并且基于这个工程把工程里的文件打包成一个 IP，供后续 SDx 集成使用。
- src：在 SDx 的环境中将 Vivado IP 打包成 SDx Library (libxxx.a)，供将来 SDx App 调用。
- app: 真正的应用，调用 SDx Library，即 C callable IP，实现最终的可执行应用。


## 未完待续


