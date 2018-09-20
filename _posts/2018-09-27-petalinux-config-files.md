---
layout: post
title:  "PetaLinux 的配置文件"
date:   2018-09-27 12:00:00
categories: Experiences
tags:
- PetaLinux
---

## Background

在 PetaLinux 环境中会遇到很多配置文件，有涉及 PetaLinux 工程的，也有涉及到底层 Yocto 环境的。粗略算下来大致有这几种

- PetaLinux 工程配置文件
    - 工程总体配置文件：`petalinux-config` 产生
    - RootFS 配置文件：`petalinux-config -c rootfs` 产生
    - Kernel 配置文件：`petalinux-config -c kernel` 产生
    - U-boot 配置文件：`petalinux-config -c u-boot` 产生
    - Device tree 配置文件
- Yocto 配置文件
    - 某个 meta layer 的配置文件
    - 等效于 Yocto 工程 local.conf 的环境配置文件
    - image 配置文件

下面细说一下这些配置文件

## PetaLinux 工程配置文件

### 工程总体配置文件

- 位置：<project>/project-spec/configs/config
- 图形配置： `petalinux-config`
- 功能

### RootFS 配置文件

### Kernel 配置文件

### U-boot 配置文件

### Device tree 配置文件

## Yocto 配置文件

### meta layer 的配置文件

### Yocto 环境配置文件

### image 配置文件

