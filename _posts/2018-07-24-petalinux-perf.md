---
layout: post
title:  "在 PetaLinux 中进行系统性能测试"
date:   2018-07-24 12:00:00
categories: Experiences
tags:
- PetaLinux
---

# 在 PetaLinux 中安装性能测试工具

- `petalinux-config -c rootfs` 增加 `packagegroup-petalinux-benchmarks`。 benchmark 软件包中包含了多项性能测试组件。具体包含内容可以在它的描述中看到

```
# <petalinux_install_dir>/components/yocto/source/aarch64/layers/meta-petalinux/recipes-core/packagegroups/packagegroup-petalinux-benchmarks.bb

BENCHMARKS_EXTRAS = " \
   hdparm \
   iotop \
   nicstat \
   lmbench \
   iptraf \
   net-snmp \
   lsof \
   babeltrace \
   sysstat \
   dstat \
   dhrystone \
   linpack \
   whetstone \
   iperf3 \
   "
```
- 如果还需要安装其他工具，可以使用 `petalinux-config -c rootfs`，然后用 `/` 搜索你所需要的包，增加它然后保存。
- 如果以上方法找不到所需要的包，但是 Yocto 提供了这个包，可以编辑 `project-spec/meta-user/recipes-core/images/petalinux-image.bbappend`, 增加 `IMAGE_INSTALL_append = " package_name"`，然后再 `petalinux-config -c rootfs` 找到添加的包名，选择然后保存。注意以上命令 `package_name` 前有空格。



# 常用功能

## 查看 CPU 利用率
- `top` 实时显示，几秒钟后刷新一次，或者按空格键刷新；默认显示多核CPU总体占用率，按`1`可以显示每个CPU的占用率
- `sar -u 1` 每秒打印一次

## 查看中断数量

- `cat /proc/interrupts`
- `sar -I <中断号> 1` 每秒钟打印一次<中断号>的新中断数


待续