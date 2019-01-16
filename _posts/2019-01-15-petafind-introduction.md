---
layout: post
title: PetaLinux探秘(3) - 自制小工具，助你快速查找菜谱
categories: 
  - Experiences
tags:
  - PetaLinux
---

Yocto 是面向大型工程设计的，它期望能让多个厂商的合作更方便，比如 meta-layer 的设计，每个厂商都能通过 bbappend 文件在自己的 meta-layer 中为大家共同协作的软件包添加新的 patch，在这种便利性的情况下其实也增加了整个项目的复杂度。也就是说，读菜谱几乎变成了一个寻宝游戏，菜谱散落在世界各地，要得到最终完整的菜谱，就要把

比如，你可能很难知道某个改变是哪个 bbappend 引入的，哪个 bbappend 里的属性是最终被应用的，或者这个软件包，明明没有使能，但它是怎么被包含到最终镜像里去的。

由于经常需要解决找菜谱的问题，我写了一个小工具，来帮我迅速找到一个软件包所有相关的菜谱。


## PetaFind
https://github.com/imrickysu/petafind

它可以做什么呢？最重要的就是找菜谱。

找所有和gstreamer有关的菜谱，不管在工程里，还是在 PetaLinux 安装目录里：
```
$ petafind r "gstreamer*.bb*"
--------------------------
petafind v0.1
--------------------------

Project Base: /xxx/petalinux_2018.2_proj

------ Project Metadata ------
* Project Version: 2018.2
* Project Family: zynqMP
* Project TMP: /tmp/petalinux_2018.2_proj-2019.01.02-05.29.31


------ Search Recipes ------

Searching gstreamer*.bb* in project meta-user layer

/xxx/petalinux_2018.2_proj/project-spec/meta-user/recipes-multimedia/gstreamer/gstreamer1.0-plugins-bad_%.bbappend
/xxx/petalinux_2018.2_proj/project-spec/meta-user/recipes-multimedia/gstreamer/gstreamer1.0-plugins-base_%.bbappend
/xxx/petalinux_2018.2_proj/project-spec/meta-user/recipes-multimedia/gstreamer/gstreamer1.0-plugins-good_%.bbappend
/xxx/petalinux_2018.2_proj/project-spec/meta-user/recipes-multimedia/gstreamer/gstreamer1.0_%.bbappend
/xxx/petalinux_2018.2_proj/project-spec/meta-user/recipes-multimedia/gstreamer/gstreamer1.0-omx_%.bbappend

Searching gstreamer*.bb* in PetaLinux installation directory for aarch64

/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-petalinux/recipes-multimedia/gstreamer/gstreamer-vcu-examples_0.1.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-petalinux/recipes-multimedia/gstreamer/gstreamer1.0-omx_%.bbappend
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-petalinux/recipes-multimedia/gstreamer/gstreamer1.0-plugins-good_%.bbappend
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-petalinux/recipes-multimedia/gstreamer/gstreamer1.0-plugins-bad_%.bbappend
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-petalinux/recipes-multimedia/gstreamer/gstreamer1.0-plugins-base_%.bbappend
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-petalinux/recipes-multimedia/gstreamer/gstreamer1.0_%.bbappend
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-openembedded/meta-multimedia/recipes-multimedia/gstreamer-0.10/gstreamer_0.10.36.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-plugins-bad_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-meta-base.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-libav_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-rtsp-server_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-plugins-good_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-omx_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-plugins-base_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-plugins-ugly_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-python_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/core/meta/recipes-multimedia/gstreamer/gstreamer1.0-vaapi_1.12.2.bb
/opt/petalinux/2018.2/components/yocto/source/aarch64/layers/meta-qt5/recipes-multimedia/gstreamer/gstreamer1.0-plugins-bad_%.bbappend
```

找配置文件
```
$ petafind config
--------------------------
petafind v0.1
--------------------------

Project Base: /xxx/petalinux_2018.2_proj

------ Project Metadata ------
* Project Version: 2018.2
* Project Family: zynqMP
* Project TMP: /tmp/petalinux_2018.2_proj-2019.01.02-05.29.31


------ Project Configuration Files ------
* Project Config File: 
/xxx/petalinux_2018.2_proj/project-spec/configs/config

* Project Rootfs Config File: 
/xxx/petalinux_2018.2_proj/project-spec/configs/rootfs_config

* Project Rootfs Additional Config: 
/xxx/petalinux_2018.2_proj/project-spec/meta-user/recipes-core/images/petalinux-image.bbappend
- Note: Add recipes supported by Yocto but not listed in PetaLinux rootfs to IMAGE_INSTALL_append

* Project Final Kernel Config: 
/tmp/petalinux_2018.2_proj-2019.01.02-05.29.31/work-shared/plnx-zynqmp/kernel-build-artifacts/.config
```

快速进入 build/tmp 目录（因为我的服务器使用NFS，系统会自动给 PetaLinux 工程的tmp指定一个本地文件夹，而不在工程目录中）
```
$ petatmp
```

还有很多功能准备添加。当然，也欢迎你在 Github 上 Fork 或者提 Issue。 