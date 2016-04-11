---
layout: post
title:  "给 Vivado 打 Patch 的方法"
date:   2016-04-11 08:00:00
categories: Experiences
tags:
- Vivado
- Patch
---

Vivado 的 Patch 有很多种，因为说到底 Patch 就只是对安装文件做部分覆盖更新，所以通过 Patch 可以实现的功能有很多，
比如修复 IP bug，修复工具 bug，增加对 demo 板的支持等等。

可以通过很多种方法给 Vivado 打 Patch。因为实际使用的系统可能有各种限制，最终挑选一种最合适的方式使用。

## 最野蛮方法 - 直接覆盖安装目录 ##

- 要求：对 Vivado 安装目录有写权限
- 限制：使用 Patch 后恢复原始状态比较麻烦

## 设置 MYVIVADO 环境变量 ##

- 出处：[AR53821](http://www.xilinx.com/support/answers/53821.html)
- 好处：灵活
    - 没有写权限的用户也可以自己使用 patch
    - 多用户公用一个 Vivado 安装镜像的时候，对每个用户可以使用自己需要的 Patch，不影响别的用户
    - 如果有多个 patch 要使用，在 MYVIVADO 环境变量中添加多个 patch 地址。方便增删。
- 缺点：不适合统一部署
- 具体用法：
    - 解压 patch 到一个目录，设置 MYVIVADO 环境变量内容为这个 patch 的路径
    - 如果有多个 patch 要使用，可以在 MYVIVADO 环境变量中添加多个路径，也可以把不同的 patch 解压到同一个目录

## 使用 patch 目录 ##

- 出处：[AR53821](http://www.xilinx.com/support/answers/53821.html)
- 好处：全局管理方便，适合统一部署；不需要设置环境变量；配置多个 patch 很方便
- 限制：
    - 全局使用后对 patch 目录没有写权限的用户不能取消 patch
    - 需要 Vivado 安装目录的写权限
