---
layout: post
title:  "PetaLinux 和 Yocto 之一：初用 Yocto 就得找找找"
date:   2017-05-01 12:00:00
categories: Experiences
tags:
- PetaLinux
- Yocto
- ZYNQ
- MPSoC
---

## Background

Yocto 是业界流行的嵌入式 Linux 编译框架，很多公司都在使用它。Yocto Project 的 Advisory Board 可以在[这里](https://www.yoctoproject.org/about/governance/admininstrative-leadership)找到，其中主要是 CPU 芯片生产商 (Intel, Xilinx, etc)、操作系统提供商 (Wind River, Linaro, etc) 和系统集成应用公司(Juniper Networks, Dell)。除了 Advisory Board 里的公司，很多公司也都已经或者正准备将开发环境转移到 Yocto 平台上。

## Yocto

Yocto 的好处是它提供了一套完整的框架，不仅可以包含Linux中每个组成原件的编译和打包，还可以编译工具链、生成软件开发SDK、增量打补丁等。它避免了每个厂商自己维护一套流程系统，拼装出一套自己定制的Linux。用 Yocto 的话，流程已经确定好了，只需要在它的框架下根据自己的需要定制即可。熟悉了这套流程，切换平台也很容易。

但是功能强大带来的一个弊端就是对于简单的任务来说，复杂性提高了：配置文件增加了，各种流程都有规范不可以随心所欲了，规范增加需要读的文档也增多了，系统增大编译时间也增加了，等等。

Xilinx PetaLinux 从 2016.4 开始由原先基于 Make 的自定义流程向 Yocto 迁移。PetaLinux 工具仍然保留着，比如 petalinux-create, petalinux-build 等命令仍然可以使用。如果直接使用 Xilinx 提供的所有组件不需要修改，那么迁移到 2016.4 对用户来说的感觉就只是编译时间变了(可能变慢可能变快，根据项目和电脑的配置可能不同)、需要预留的磁盘空间变大了，其它使用流程并没有特别的变化；如果需要做进一步的定制，比如需要在 rootfs 中增加一个库、一个工具，或需要修改 u-boot 源代码等，都会涉及到和 Yocto 打交道的地方。有很多常用操作已经在 UG1144 中有覆盖，但也有很多其它使用中的疑问，最近我会写一些常见问题和解决方法。

## Log 在哪里

之前的 PetaLinux 编译过程只产生一个 log 文件，存放在 `build/build.log` 文件中。现在的 `build/build.log` 中只有有限的信息，每一个组件的具体编译信息都在它自己的目录中。通常 Yocto 也会有提示说，详细的错误信息参加 xxx 文件，但是只看提示的文件却不一定能找到真正的问题根源，因为 Yocto 将同一个 app 不同编译步骤的 log 存在不同文件中，所以有可能前一步出错导致后续步骤错误被报告出来，但问题根源还需要到前一个步骤的 log 文件中去找。

### 例子：pmu-firmware 编译失败

在进行 `petalinux-build` 的时候遇到 pmu-firmware 编译失败是很多人第一次运行基于 Yocto 的 PetaLinux 时遇到的问题。从原理上分析，如果其它组件都能编译通过，唯独 pmu-firmware 失败，比较大的可能性是因为 pmu-firmware 是基于 PMU 硬件的，而 PMU 的本质是 MicroBlaze，其他组件都是基于 Cortex-A53 的。那么编译器工具链的问题可能性很大。

遇到错误的时候，Console 中只是提示""，而 build.log 中也是一样的打印，并没有更多关于错误的细节。要查找错误的原因，我们就需要找到 pmu-firmware 的编译 log。

![错误信息](/images/2017/petalinux_pmufw_error.png)

build.log 中的完整错误信息

    ERROR: pmu-firmware-0.2+xilinx+gitAUTOINC+ef07b552f4-r0 do_deploy: Function failed: do_deploy (log file is located at /home/me/myData/work/zcu102_2016.4/petalinux/build/tmp/work/aarch64-xilinx-linux/pmu-firmware/0.2+xilinx+gitAUTOINC+ef07b552f4-r0/temp/log.do_deploy.5422)
	ERROR: Logfile of failure stored in: /home/me/myData/work/zcu102_2016.4/petalinux/build/tmp/work/aarch64-xilinx-linux/pmu-firmware/0.2+xilinx+gitAUTOINC+ef07b552f4-r0/temp/log.do_deploy.5422
	NOTE: recipe pmu-firmware-0.2+xilinx+gitAUTOINC+ef07b552f4-r0: task do_deploy: Failed
	ERROR: Task 17 (/opt/Xilinx/petalinux-v2016.4/components/yocto/source/aarch64/layers/meta-xilinx-tools/recipes-pmu/pmu/pmu-firmware_git.bb, do_deploy) failed with exit code '1'
	NOTE: Running task 924 of 930 (ID: 3, /opt/Xilinx/petalinux-v2016.4/components/yocto/source/aarch64/layers/meta-xilinx-tools/recipes-pmu/pmu/pmu-firmware_git.bb, do_populate_sysroot)
	NOTE: recipe pmu-firmware-0.2+xilinx+gitAUTOINC+ef07b552f4-r0: task do_populate_sysroot: Started
	NOTE: recipe pmu-firmware-0.2+xilinx+gitAUTOINC+ef07b552f4-r0: task do_populate_sysroot: Succeeded
	NOTE: recipe pmu-firmware-0.2+xilinx+gitAUTOINC+ef07b552f4-r0: task do_package: Started
	NOTE: recipe pmu-firmware-0.2+xilinx+gitAUTOINC+ef07b552f4-r0: task do_package: Succeeded
	NOTE: Tasks Summary: Attempted 924 tasks of which 912 didn't need to be rerun and 1 failed.
	ERROR: Failed to build pmufw

虽然没有讲到具体为什么出错，至少以上错误信息包含了这些内容

- 那个具体流程出错了：`pmu-firmware-0.2+xilinx+gitAUTOINC+ef07b552f4-r0` 的 `do_deploy`
- 错误的 log 在哪里： `/home/me/myData/work/zcu102_2016.4/petalinux/build/tmp/work/aarch64-xilinx-linux/pmu-firmware/0.2+xilinx+gitAUTOINC+ef07b552f4-r0/temp/log.do_deploy.5422`
- 这个流程的 recipt 文件的位置：`/opt/Xilinx/petalinux-v2016.4/components/yocto/source/aarch64/layers/meta-xilinx-tools/recipes-pmu/pmu/pmu-firmware_git.bb`

接下去查看一下 log 文件，其主要信息是

    install: cannot stat '/home/me/myData/work/zcu102_2016.4/petalinux/build/../components/plnx_workspace/pmu-firmware/Release/pmu-firmware.elf': No such file or directory

看上去应该是 pmu-firmware.elf 在前一个步骤没有正确生成。为什么呢？我们就需要查看一下生成 pmu-firmware 的流程的 log。在这个目录中，各个 task 的 log，比如

    log.do_compile
	log.do_configure
	log.do_create_yaml
	log.do_deploy
	log.do_fetch
	log.do_package
	log.do_packagedata
	log.do_package_qa
	log.do_package_write_rpm
	log.do_patch
	log.do_populate_lic
	log.do_populate_sysroot
	log.do_rm_work
	log.do_rm_work_all
	log.do_unpack
	log.do_xsct_setup

用 `ls -t` 命令可以按时间顺序排列，查看 do_deploy 之前产生了什么 log 文件。我们可以找到 log.do_compile。

在这个例子里，我们查看 log.do_compile 就可以知道是系统缺少了某些库，导致 mb-gcc 程序无法正常运行，而产生了错误。安装上相应的库之后，就能顺利运行了。

在这个目录里，除了 log.do_compile 还有一个 run.do_compile 文件，就是运行了 run.do_compile 中的命令，才产生了 log.do_compile。

此外，即使没有仔细阅读 log 文件，仅仅知道 pmu-firmware 编译出错了，我们也可以在 build 目录中运行 `find . -name "pmu-firmware*"` 来查找 pmu-firmware 的编译的临时文件夹，在里面用 `grep -r ERROR`查找错误有关信息。

所以，在看到 PetaLinux 编译出错的时候，仔细阅读错误报告，善用 Linux 环境中的搜索工具，就可以帮我们快速定位问题。




