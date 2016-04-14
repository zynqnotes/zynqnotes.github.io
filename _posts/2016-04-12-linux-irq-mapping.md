---
layout: post
title:  "最新 Linux Kernel 中断号的映射"
date:   2016-04-12 08:00:00
categories: Experiences
tags:
- Kernel
- Interrupt
---

当写一个带中断信号的设备的驱动程序的时候，注册中断函数最重要的部分其实就是两点：

- 声明一个返回值类型是`irqreturn_t`的中断响应程序
- 在`request_irq()`函数中提交中断号和中断响应程序的名字、参数，把他们连接起来，以达到产生中断时系统就调用相应中断响应程序的目的

在第二步中，request_irq所提交的中断号曾经是硬件中断号，与硬件芯片有关。但在最新的 Linux Kernel 中，这一点已经改变了 （其实已经改变很久了，只是 PetaLinux 最近才把这个变更吸收进来产生影响）。简单地说，就是现在如果仍然提交这个硬件中断号，则不能得到正确的中断注册。而现在的处理方法对用户来说其实也并没有变得比原先复杂，只是把复杂的映射关系放在后台看不见的地方自动做掉了。

## 为 Custom IP 注册中断的流程 ##
在 ZYNQ 中，PS 相关的 IP 都已经有了驱动程序，因此不需要我们手工再写驱动，做绑定中断等工作。只要 device-tree 写得正确，系统启动的时候自动会调用相关驱动程序。

如果是用户自定义的 IP，这个 IP 需要产生中断信号，让系统做及时响应，那么我们就需要为它写驱动。主要步骤如下：

1. 在 Vivado 中将 IP 的中断信号连接到 ZYNQ 的 IRQF2P
    - 启动 IRQF2P，需要在 ZYNQ 的配置中，找到 Interrupt，启用 IRQF2P。
    - IRQF2P 的意思是 IRQ Fabric to Processsor
    - IRQF2P 是一个总线。如果有多个中断信号要连接到 IRQF2P，需要经过一个 Concat IP 将多根独立信号合并成一个总线

2. 在 Vivado 中 Generate Output Products，然后 Export Hardware 产生 HDF 文件

3. 在 PetaLinux 工程中导入 HDF，让 PetaLinux 自动产生 device-tree
    - `petalinux-config --get-hw-description=<hdf_directory>` 
    - 产生的 device-tree 在 subsystems/linux/configs/device-tree 目录中
    - 如果 PL 中的 IP 是 Xilinx IP, 那么它的参数所对应的 device-tree 会生成在 pl.dtsi 中，它会被 system-top.dts 引用
    - pl.dtsi 中应该包含类似 `interrupt-parent = <&intc>;`，`interrupts = <0 29 4>;`的两句话。其中`interrupt-parent`指定了中断控制器是哪个 IP，`interrupts` 后的三个参数的意思分别是：Shared Peripheral Interrupt(0) 还是 Private Peripheral Interrupt(1), 硬件中断号, 中断采集方式（边沿敏感还是电压敏感）。具体意义参考 Linux Kernel 源文件的 Documentation/devicetree/bindings/arm/gic.txt
    - 如果产生中断的是自定义的 IP，或者是使用 HDL 实现的设计，直接连线到 ZYNQ 的 IRQF2P，device-tree-generator 可能不能识别这个 IP 的名字和参数，需要用户参考 Xilinx IP 的 device-tree，写到 system-top.dts 中。请包含 `interrupt-parent` 和 `interrupts` 参数
    - 注意 PL IP 的 device-tree 描述中应该包含 `compatible` 参数，它的值一般写为 `vendor,name-version`的形式。记住这个参数的值，在软件驱动中需要使用它。

4. 在 PetaLinux 工程中产生驱动模板
    - `petalinux-create -t module -n <module_name>`
    - 驱动作为 Kernel Module，产生的目录是 components/modules/<module_name>

5. 修改 PetaLinux Module 文件
    - 在 components/modules/<module_name>.c 中搜索 `compatible`, 填入步骤3中 device-tree 对应的 compatible 的值。这样，驱动加载的时候就会自动在 device-tree 中搜索相同 compatible 值的模块，并读取这个模块的其他参数值，比如 `interrupt-parent` 和 `interrupt`。
    - Kernel 读取了 `interrupt` 和 `interrupt parent` 参数后，会自动将硬件的 IRQ 号码映射为虚拟的 IRQ 号码，并存在 `r_irq->start` 中。`request_irq` 使用 `r_irq->start` 的 IRQ 号来注册中断。

6. 编译
    - `petalinux-build` 可以做完整编译，生成新的 image.ub

7. 验证
    - 用新的 image.ub 启动 Linux 后，运行 `insmod /lib/modules/4.0.0-xilinx/extra/<module_name>.ko`，就能加载这个 Kernel Module。
    - 正常情况下，console 中应该打印类似信息：`<module_name> at <address> mapped to <address>, irq=<IRQ number>`
    - 运行 `cat /proc/interrupts` 后应该可以看到新注册的中断的虚拟中断号、硬件中断号(device-tree 的中断号 + 32)、中断源模块名称等信息

## 总结 ##
新的 Kernel 中虽然会将硬件中断号映射为虚拟中断号，但是对于使用 device-tree 描述 custom IP，并且使用 PetaLinux 自带 example code 来解析 device tree 的情况，coding 并没有因此变得复杂，底层系统自动处理了这个映射的流程。