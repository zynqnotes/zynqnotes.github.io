---
layout: post
title:  "怎样设置 PetaLinux 2016.3 的外部 Kernel Source"
date:   2016-04-04 21:55:56
categories: Experiences
tags:
- PetaLinux
- Version Control
---

在 PetaLinux 的安装包中包含了一个当时的稳定版的 Linux Kernel 和 u-boot 的源代码。如果不需要修改任何 Kernel 和 u-boot 的功能，这份自带的 Linux Kernel 和 u-boot 源代码都能很好地完成任务。如果需要修改这两部分的代码，PetaLinux 系统也提供了灵活的方式使用其他源文件。

在 UG1144 _Configure Out-of-tree Build_ 中说明了这两种方法

- 在 `petalinux-config` 的菜单中选择 Kernel/U-boot 的 Source 为 `remote`，然后指定 remote source 的 git repository 和 tag。以下称为`remote模式`。
- 在 `components` 目录中添加子目录 `linux-kernel` 或 `u-boot`，然后在这些子目录中添加真实 kernel/u-boot 的目录。添加的目录名会显示在 `petalinux-config` 的菜单中，选择你的 Kernel/u-boot 目录名就选择了使用这个目录里的 Kernel/u-boot。以下称这种模式为`component模式`。

在 PetaLinux 编译的时候，对于不同的源文件种类会做不同的处理

- 如果使用 remote 模式，那么第一次 build 的时候 git repository 会被 clone 到 build 目录，checkout 某个指定的 tag，然后再进行build。后续如果不做特殊指定，就不再 clone，毕竟如果从 GitHub 上 clone 一次 Kernel 快则十几分钟，慢则几小时。
- 如果使用 component 模式，则直接在 build 目录中编译

同时，操作系统中也有一些特性，如果需要我们可以加以利用

- git clone 的源文件地址可以是 GitHub 的地址，也可以是一个内网地址，也可以是一个本机的路径，只要这里存着一份 linux-xlnx repo 就可以。
- 使用 components 模式时，PetaLinux 也可以使用 `ln -s` 定义的目录软链接。这个软链接可以指向 git repository。多个 PetaLinux 的 Kernel Source 可以用软链接指向同一份源文件，以达到共享源文件以及节省磁盘空间的目的。

根据以上的种种特性，以及应用时的不同场景产生的不同需求，以下是我的使用方法

- 我的电脑里总是会存一份独立的 Kernel/U-Boot 的 Clone，总是随着 GitHub 上的版本保持到最新，所有后续的操作都利用这份源文件，而不是直接和 GitHub 打交道。
- 对于要做比较多的定制以及比较长时间的开发的项目，为了和别的项目区分开，使用component模式让这个 PetaLinux 工程独占一份带 git history 的 Kernel 源文件，在这个基础上做自己的 branch。
- 很少用 remote 模式，也很少用软链接的component模式，因为他们都将工程与源代码分割开来了。