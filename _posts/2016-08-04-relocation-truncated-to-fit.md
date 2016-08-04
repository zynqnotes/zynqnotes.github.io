---
layout: post
title:  "Linker: relocation truncated to fit"
date:   2016-08-04 18:00:00
categories: Experiences
tags:
- SDK
- GCC
---

在 SDK 中编译一个 Standalone App 的时候，gcc compile 通过，但是到 link 阶段报错。


```
Building target: hello.elf
Invoking: ARM A53 gcc linker
aarch64-none-elf-gcc -Wl,-T -Wl,../src/lscript.ld -L../../hello_bsp/psu_cortexa53_0/lib -o "hello.elf"  ./src/helloworld.o ./src/platform.o   -Wl,--start-group,-lxil,-lgcc,-lc,--end-group
../../hello_bsp/psu_cortexa53_0/lib/libxil.a(xil-crt0.o):(.text+0x0): relocation truncated to fit: R_AARCH64_ABS32 against symbol `__sbss_start' defined in .sbss section in hello.elf
../../hello_bsp/psu_cortexa53_0/lib/libxil.a(xil-crt0.o):(.text+0x4): relocation truncated to fit: R_AARCH64_ABS32 against symbol `__sbss_end' defined in .sbss section in hello.elf
../../hello_bsp/psu_cortexa53_0/lib/libxil.a(xil-crt0.o):(.text+0x8): relocation truncated to fit: R_AARCH64_ABS32 against symbol `__bss_start__' defined in .bss section in hello.elf
../../hello_bsp/psu_cortexa53_0/lib/libxil.a(xil-crt0.o):(.text+0xc): relocation truncated to fit: R_AARCH64_ABS32 against symbol `__bss_end__' defined in .bss section in hello.elf
collect2: error: ld returned 1 exit status
make: *** [hello.elf] Error 1
```


Google 后发现一篇文章解释得很好：[Relocation truncated to fit - WTF?](https://www.technovelty.org/c/relocation-truncated-to-fit-wtf.html)

简要解释一下：

- 在64位的系统中，每条指令是64位的。
- 如果是一条跳转指令，可能前32位表示跳转，后32位表示偏移地址。这样的话，4GB之内的地址跳转一条指令就够了。
- 如果要跳转到4GB以外，但仍然使用了同样的跳转指令，链接的时候就会发现32位空间存不下4GB以外的地址。
- relocation 就是指跳转地址
- truncated to fit 就是说空间不够了，被截去了一部分

知道原因以后去查 linker script，发现自动生成的 linker script 把 .text 段存在 psu_ddr_1_MEM_0 内存区域，而 psu_ddr_1_MEM_0 的地址定义是`psu_ddr_1_MEM_0 : ORIGIN = 0x800000000, LENGTH = 0x80100001`, 这是一个4G之外的地址。解决方法就是重新生成一次Linker Script，将所有段使用2GB以内的内存 `psu_ddr_0_MEM_0`.

