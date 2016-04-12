---
layout: post
title:  "XAPP1082 v4 调试心得"
date:   2016-04-05 08:00:00
categories: Experiences
tags:
- 1000Base-X
- SGMII
- Ethernet
---

由于 ZYNQ 的 PS 侧没有集成硬件的 SERDES，要使用带 1000Base－X 或 SGMII 都必须使用 PL 将 PS 输出的 GMII 转换成对应的接口。在 Vivado 中这个转换过程涉及到 IP 调用、配置、信号连接、管脚锁定等等步骤，而且软件方面也涉及到为 PL 侧的 PHY 指定特定的驱动程序，所以要打成这一目的还是有一些工作量要做的。[Xapp1082](http://www.xilinx.com/support/documentation/application_notes/xapp1082-zynq-eth.pdf) 以及它配套的 [Wiki Page](http://www.wiki.xilinx.com/Zynq+PL+Ethernet) 可以是一个使用 1000Base-X 或 SGMII 设计的起点。

Xapp1082 已经发布到第四个版本，曾经需要对内核驱动打补丁流程繁琐，但是现在对 PCS/PMA IP 的支持已经集成到 Kernel Driver 中，不需要再手工打补丁，方便了许多。

## 基本流程 ##
个人感觉 Wiki Page 上的流程说明比较冗长而且排版不突出重点。在此将主要操作流程罗列一遍。由于 Xapp1082 针对 ZC706 开发板，因此如果要移植到自己的板上，可能会需要做一些的改动。

1. 生成 PL 逻辑对应的 bit 文件。步骤都是到 `hardware/vivado/scripts/` 目录中运行 `vivado -source xxx.tcl`。需要什么硬件对应什么 tcl。生成的工程在 `hardware/vivado/runs` 目录中。这个步骤没有什么改动。如果是自己的工程，可以将建出的 block design 与自己的设计整合到一起。
    - 使用 PS EMAC，PL 用 PCS/PMA IP 的，就选 `ps_emio_eth` 开头的 tcl
    - 使用 PL 完成 EMAC 功能的，选用 `pl_eth` 开头的 tcl
    - 使用 SGMII的，选用带 `sgmii` 后缀的
    - 使用 1000Base-X 的，选用_不_带 `sgmii` 的
    - 带 bd 的 tcl 会被其他 tcl 调用，不需要手动 source

2. 建 PetaLinux 工程，导入 PL 定义的 `hdf` 文件 
    - Wiki 上使用 bsp 来建工程，但是真正的工程肯定都导入 Vivado 的设置，所以这个步骤需要微调。
    - `petalinux-create -t project -n <PROJECT_NAME> --template zynq`
    - `cd <PROJECT_NAME>`
    - `petalinux-config --get-hw-descriptions=<HDF DIRECTORY>`

3. 指定使用的 Linux Kernel 和 u-boot 的版本
    - 如果不需要对 Kernel 和 U-boot 做任何改变，可以直接使用 PetaLinux 2015.4 自带的代码，不需要做任何更改，不需要像 Wiki 上指定从 GitHub 上下载一遍 Kernel。
    - 如果需要对 Kernel 或 U-boot 的源代码做修改，建议使用[这篇文章]({% post_url 2016-04-04-setup-petalinux-kernel-source %})中的办法管理源代码。

4. 如果是 ZC706，就打上给 FSBL 的 Patch；如果是自己的目标板，就不用打 FSBL Patch。
    - 因为 ZC706 板上供 SERDES 的125MHz 的参考时钟是由可编程的 Si5324 芯片提供的，因此如果目标板有不需要编程就可以使用的参考时钟，就可以不必使用 Xapp1082 中对于 FSBL 所打的 Patch。

5. 如果需要让 u-boot 也能使用 PL 侧的 PHY，需要给 u-boot 打一个补丁
    - 如果需要打补丁，需要使用步骤3指定的方法指定 u-boot 源文件位置
    - 补丁地址参考[这里](https://github.com/imrickysu/u-boot-xlnx/commit/ea72a0b1eb02cff060c4ea85b73d5a74479efd50)

6. 在 Kernel 中 Enable Xilinx PHY Driver
    - 进行 Kernel 配置 `petalinux-config -c kernel`
    - 打开 `XILINX_PHY` 的 driver。可以用 `/` 搜索，也可以在 `Device Drivers> Network device support > PHY Device support and infrastructure > Drivers for xilinx PHYs` 找到。

7. Build
    - `petalinux-build`

8. 生成 boot.bin
    - `petalinux-package --boot --fsbl=zynq_fsbl.elf --fpga=ps_emio_sfp.bit --u-boot`
    - 这个步骤也可以手工写好自己的bif文件，然后使用 bootgen 工具生成
