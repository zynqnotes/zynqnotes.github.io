---
layout: post
title:  "PetaLinux BSP 包含一些什么内容"
date:   2018-09-10 12:00:00
categories: Experiences
tags:
- PetaLinux
---

## 从 PetaLinux BSP 新建 PetaLinux 工程
```
petalinux-create -t project -s <xx.bsp>
```

## 查看 PetaLinux BSP 中包含了一些什么内容
使用`petalinux-create`创建工程后，就能看到 BSP 里包含的内容。对于 Xilinx 官方发布的 PetaLinux BSP，新建工程的根目录中会有这个 BSP 的 README。 README 文件中描述①文件内容的简介和②怎样一步一步从一个空的 PetaLinux 工程配置出 BSP 所提供的状态的流程说明。

创建出 BSP 的原始硬件信息保存在`project-spec/hw-description/system.hdf`，如果希望自己一步一步重建一遍，或者做一下对照参考，可以使用这个 HDF 再创建出一个 PetaLinux 工程，与 BSP 建立的工程做对比。
```
petalinux-create -t project --template zynqMP -n <project name>
petalinux-config --get-hw-description=<hdf directory>
```

对比过程中，主要比较 `project_spec` 目录。
- `configs/config` 文件是工程配置，可以用 `petalinux-config` 生成。
- `configs/rootfs_config` 是文件系统配置，可以用 `petalinux-config -c rootfs` 生成。
- `meta-user/recipe-core/images/petalinux-user-image.bb` 是用户后添加的软件包，需要手工编辑后，再到 `petalinux-config -c rootfs` 中去使能。
- `meta-user/recipe-bsp/device-tree/` 保存了用户的 device tree，如需修改需要手工编辑。
- `meta-user` 中如果有多出来 `recipe-*` 命名的文件夹，就是 BSP 中额外添加的功能，它们可能是修改 Kernel 选项、为已有的软件打 Patch，或额外添加新的软件。


除此之外，BSP 建立的工程一般还有 pre-built 目录，这是为了方便用户测试 BSP，减少编译时间而准备的预先编译好的二进制包，一般可以直接装载到 SD 卡启动系统。



## 自己打包一个 PetaLinux BSP

如上所述，PetaLinux BSP 中可以包含工程的硬件信息、软件配置、额外的软件包，以及预编译二进制包，因此使用 BSP 来发布 PetaLinux 工程非常方便。如果希望自己创建一个 PetaLinux BSP，可以使用以下步骤。

1. (Optional) 在 PetaLinux 工程的根目录创建 `pre-built` 目录，将可以直接运行的二进制文件拷贝到这里
2. cd 到 PetaLinux 工程外
3. `petalinux-package --bsp -p <plnx-proj-root> --output <my_board.bsp>`

另外，BSP 文件其实只是一个包含了特定所需文件的 zip 包，也可以用任何解压缩软件打开。

## 参考资料
- UG1144: http://www.xilinx.com/support/documentation/sw_manuals/xilinx2018_2/ug1144-petalinux-tools-reference-guide.pdf