---
layout: post
title:  "PetaLinux工具系列(1) - Yocto 的菜谱: bb 文件"
date:   2018-10-20 12:00:00
categories: Experiences
tags:
- PetaLinux
- BitBake
---

## Background

对于 PetaLinux 中的每个组件，大到 kernel，小到一个工具组件的编译和安装，都是通过“菜谱” bb 文件描述，经过 bitbake 这个工具来“烧制”，最后才打包生成最后能启动的镜像。 Bitbake 是 Yocto 的重要组件，PetaLinux 只是因为底层使用了 Yocto，所以一起继承过来了这套工具，他们并不是 Xilinx 或 PetaLinux 自创的。

## Recipe Files 菜谱文件

bb 和 bbappend 文件是 bitbake 的 recipe 描述文件。bb是描述文件的开头，后面可以追加多个 bbappend 文件的内容。使用这种结构主要是因为 Yocto 的组织结构是基于 Layer 层次的。为了能够复用别人的设计，每个人只改动自己的 layer，如果多个 layer 都需要修改同一个软件包，那么就可以在这个软件包的 bb 描述文件上追加 bbappend 以达到自己修改的目的。

### 举个例子

PetaLinux 安装目录中包含了所有软件包的菜谱文件，`components/yocto/source/aarch64/layers`目录中是所有 MPSoC 的菜谱。 如果我们搜索 `gstreamer1.0-omx*.bb*` 会有以下结果：

```
find . -name "gstreamer1.0-omx*.bb*"
./meta-petalinux/recipes-multimedia/gstreamer/gstreamer1.0-omx_%.bbappend
./core/meta/recipes-multimedia/gstreamer/gstreamer1.0-omx_1.12.2.bb
```

这里 `gstreamer1.0-omx_1.12.2.bb` 是基础菜谱，第一个下划线之前的 `gstreamer1.0-omx` 是这个菜谱的名字，`1.12.2` 是它的版本号。`gstreamer1.0-omx_%.bbappend`的第一个下划线之前的菜谱名和它一样，下划线之后的 `%` 表示匹配所有版本，所以在编译时 bitbake 就会把 `meta-petalinux` layer 的 `gstreamer1.0-omx_%.bbappend` 追加到 `core` layer 之后。

如果我们在做 Xilinx ZCU106 VCU TRD 的实验，VCU TRD 的软件包中也有一个对应的 bbappend:
```
xilinx-vcu-trd-zcu106-zu7-v2018.2 $ find . -name "gstreamer1.0-omx*.bb*"
./project-spec/meta-user/recipes-multimedia/gstreamer/gstreamer1.0-omx_%.bbappend
```

这个 `meta-user` layer 中的 `gstreamer1.0-omx_%.bbappend` 又会被追加到到上文已经被追加一次的菜谱之后。注意这个 `meta-user` layer 是在项目工程里，而不是在 PetaLinux 安装目录里。 也就是说，我们是可以在每个项目中为软件包增加一些特定的 Patch 和设置的。

## 菜谱的描述语言

Bitbake 这部炒菜机只能根据它认识的菜谱来炒菜，怎么写菜谱就得看说明书。 Bitbake 说明书 https://www.yoctoproject.org/docs/1.6/bitbake-user-manual/bitbake-user-manual.html 描述了菜谱该怎么写。好在菜谱语言的基础是 Python，就算不会写 Python，仅靠想象力也能读懂一些，不懂的地方再去查说明书即可。

### 配料表
读菜谱的第一步就是读配料表。`SRC_URI`就是 bb 的配料表。配料表描述很强大，可以用家里的菜，也可以去别的地方买菜。用本地的文件就是`files://` ，去远程下载就是 `http://` 或者 `git://`，另外还支持 csv, svn 等等。买来还能拆包装，看到 zip 或 tar.gz 压缩包 bitbake 会自动解压。

一般来说 bb 文件的 SRC_URI 是软件包的一个 release 版本的压缩包或 git tag，bbappend 里要增加一些 patch 会倾向于把 patch 放在 bbappend 同一个层次的单独文件夹内。由于 git 仓库里有软件包的所有版本，因此知道菜谱指定了哪个版本也很重要。在 bb/bbappend 文件中会用 `SRCREV` 关键字指定版本，对于 git 他就是 commit id。

了解菜谱配料表的写法会很有帮助，因为常用的软件包菜谱都是已有的，可能它们的烧法很复杂，但我们不需要修改；我们最经常做的定制就是做一些源代码的修改，生成 patch，添加到原来的编译流程中，只要知道 bbappend 怎么增加 SRC_URI，就能做到这一点了。

另外，有时我们需要把 Bitbake 的菜谱在别的环境中编译一遍，比如放到 Ubuntu RootFS 中。找到这个软件包所有的源文件，是完成后续人工编译的第一步。

说明书中跟配料表相关的描述在这里 https://www.yoctoproject.org/docs/1.6/bitbake-user-manual/bitbake-user-manual.html#idm45023932077824

## 菜谱的更多高级功能

Bitbake 很强大，但要灵活使用并不是一两天就能做到。如果有兴趣，可以通过多读说明书、多读别人的菜谱来学习。

## 总结

这是一篇简单的介绍性的说明，希望你在阅读后能知道

- 怎么找到描述一个软件包的完整菜谱
- 怎么找到一个软件包所有的配料

后续我们再研究怎么炒一些简单的小菜吧。