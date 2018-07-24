---
layout: post
title:  "一个简单的 VCU 设计"
date:   2018-07-23 12:00:00
categories: Tutorials
tags:
- VCU
- MPSoC
---

## 现有参考资料

- [VCU TRD 2018.1](http://www.wiki.xilinx.com/Zynq+UltraScale+MPSoC+VCU+TRD+2018.1), UG1250
- UG252
- gstreamer: https://gstreamer.freedesktop.org/

## 逻辑设计

1. ZCU106 project
2. 添加 PS: `ZYNQ UltraScale+ MPSoC`
3. 添加 VCU: `ZYNQ UltraScale+ VCU`
4. 点击上方绿色条形中的 `Run Block Automation`, 先做 MPSoC，后做 VCU，Vivado 会自动进行连接
5. Generate Bitstream
6. Export Design，选择将 Bit 打包进 HDF

### 说明

- VCU 模块在PL侧，一共有五个AXI接口。 Block Automation 会将他们分别接在 PS 的多个 HP 通道上，保证有足够的带宽。
- 通过双击 VCU IP，在界面中可以进行内存带宽的预估。如果进行分辨率比较低的编解码，或者编解码路数比较少，对内存带宽的需求较低，可以将多路 AXI 通过一个 AXI Interconnect 合成一个或两个 AXI，接到 HP 通道上。这样可以节省 HP 通道，以备其他需要使用 PS DDR 的逻辑 IP 使用。
- VCU AXI 通过 AXI Interconnect 合并，最多是 4:1, 因为 VCU 的 AXI ID 宽度是4，通过 AXI Interconnect 合并 AXI 需要增加 AXI ID 位宽。 而 HP 的最大 AXI ID 只支持 6 位。
- VCU 输入时钟尽量使用片外时钟，保证较小的 Jitter。

![](images/2018/vcu_ipi.png)
上图为 VCU Encoder 和 Decoder AXI 合并成一个 AXI 连接到 HP 后的框图

## PetaLinux

1. `petalinux-create -t project --template zynqMP -n petalinux; cd petalinux` 建立工程
2. `petalinux-config --get-hw-description=<hdf directory path>` 导入硬件设计
3. `petalinux-config -c rootfs` 增加 `packagegroup-petalinux-gstreamer`。 gstreamer 是用于驱动 VCU 的软件组件。
4. `petalinux-build` 生成各组件。
5. `cd images/linux; petalinux-package --boot --fsbl zynqmp_fsbl.elf --u-boot --fpga xx.bit`  请将 xx.bit 替换为这个目录下 bit 的文件名。

### 说明
packagegroup-petalinux-gstreamer 具体包含哪些内容，可以在它的描述中看到

```
# <petalinux_install_dir>/components/yocto/source/aarch64/layers/meta-petalinux/recipes-core/packagegroups/packagegroup-petalinux-gstreamer.bb

GSTREAMER_PACKAGES = " \
   gstreamer1.0 \
   gstreamer1.0-meta-base \
   gstreamer1.0-plugins-base \
   gstreamer1.0-plugins-good \
   gstreamer1.0-plugins-bad \
   gstreamer1.0-omx \
   gstreamer1.0-rtsp-server \

```

## 运行

1. 将 `images/linux` 目录下的 `BOOT.BIN` 和 `image.ub` 拷贝到 SD 卡。
2. 将 ZCU106 设置为从 SD 卡启动: SW6[1：4] = ON, OFF, OFF, OFF，上电启动
3. 连接串口，Interface 0
4. Login: root, password: root
5. Mount SD 卡: `mount /dev/mmcblk0p1 /mnt`
6. 尝试从 MP4 文件解码: `gst-launch-1.0 filesrc location=xx.mp4 ! qtdemux ! h264parse ! omxh264dec ! queue max-size-bytes=0 ! filesink location=yy.yuv`
7. 尝试从 RAW YUV Video 文件编码为 MP4: `gst-launch-1.0 filesrc location=xx.yuv ! videoparse format=nv12 width=WW height=HH framerate=20/1 ! omxh264enc ! queue ! h264parse  ! mp4mux ! filesink location=yy.mp4`

### 播放编解码后视频文件

- 测试播放 RAW Video: 在 PC 上安装 [ffmpeg](https://www.ffmpeg.org/download.html)，运行指令 `ffplay -f rawvideo -pixel_format nv12 -video_size WWxHH -i xx.yuv`。WW为宽度，HH为高度。因为 RAW Video 中没有视频信息，这些参数都需要手工输入。
- MP4 视频可以用任意播放器播放。