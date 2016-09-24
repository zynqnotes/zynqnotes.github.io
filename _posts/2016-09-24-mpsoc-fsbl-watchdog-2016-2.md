---
layout: post
title:  "MPSoC FSBL 代码分析之 Watchdog (2016.2)"
date:   2016-09-24 12:00:00
categories: Experiences
tags:
- MPSoC
- SDK
- FSBL
- WatchDog
---

根据FSBL代码，如果系统中使能了Watchdog，FSBL就会将它初始化，在超出定时时间后复位系统。

## 初次调用 ##

在`xfsbl_initialize.c` 中`XFsbl_PrimaryBootDeviceInit()`函数中系统初始化的代码中可以找到如下看门口初始化的部分。

```
#ifdef XFSBL_WDT_PRESENT
	Status = XFsbl_InitWdt();
	if (XFSBL_SUCCESS != Status) {
		XFsbl_Printf(DEBUG_GENERAL,"WDT initialization failed \n\r");
		goto END;
	}
#endif
```

以上代码需要弄清两个问题：

1. 怎么样才算系统中使用了看门口，即`XFSBL_WDT_PRESENT == 1`？
2. `XFsbl_InitWdt()`做了哪些初始化工作？


## 怎样使能看门狗 ##
`XFSBL_WDT_PRESENT`是在`xfsbl_hw.h`中定义

```
#if (!defined(FSBL_WDT_EXCLUDE) && defined(XPAR_XWDTPS_0_DEVICE_ID))
#define XFSBL_WDT_PRESENT
#endif
```

其中`FSBL_WDT_EXCLUDE`是根据用户在`xfsbl_config.h`的头部定义宏`FSBL_WDT_EXCLUDE_VALUE`的值(0/1)，代码自动define出来的。所以如果人为地想要不使能看门狗，就将`FSBL_WDT_EXCLUDE_VALUE`设置成1，否则就保持默认值0。

`XPAR_XWDTPS_0_DEVICE_ID`经过几次#define, 最后映射到`xparameters.h`中定义的`XPAR_PSU_WDT_0_DEVICE_ID`。在Vivado中不论使能SWDT0还是SWDT1，在`xparameters.h`中WDT的Device ID都是从0开始分配，所以不管是LPD还是FPD的SWDT都可以在这段FSBL代码中起作用（但事实不是，请往后看）。

```
/* Definitions for peripheral PSU_WDT_0 */
#define XPAR_PSU_WDT_0_DEVICE_ID 0
#define XPAR_PSU_WDT_0_BASEADDR 0xFF150000
#define XPAR_PSU_WDT_0_HIGHADDR 0xFF15FFFF
#define XPAR_PSU_WDT_0_WDT_CLK_FREQ_HZ 25000000


/* Definitions for peripheral PSU_WDT_1 */
#define XPAR_PSU_WDT_1_DEVICE_ID 1
#define XPAR_PSU_WDT_1_BASEADDR 0xFD4D0000
#define XPAR_PSU_WDT_1_HIGHADDR 0xFD4DFFFF
#define XPAR_PSU_WDT_1_WDT_CLK_FREQ_HZ 25000000
```

由上面的定义可以看出，`WDT_0`的基地址是`0xFF150000`，这其实是Low Power Domain的WDT，也就是和RPU在同一个Power Domain中。对应下面Vivado的设置中，也叫做`SWDT0`.

对应的，`WDT_1`的基地址是`0xFD4D0000`，这是Full Power Domian的WDT，也就是和APU在同一个Power Domain中。对应下面Vivado的设置中，也叫做`SWDT1`。

在Vivado中使能看门狗的位置是`I/O Configuration -> Low Speed -> SWDT`，里面会有`SWDT0`和`SWDT1`的开关，可以将它们的引脚接到MIO或者EMIO。

注意1：以上的选择只能同时改变SWDT的时钟输入管脚和复位输出管脚。

注意1：Vivado中还有一个地方会出现WDT的字眼，它是在`PS-PL Interface`中`Interrupt`的部分，也就是说，在这个部分中使能WDT，只是将WDT的中断信号引出而已，并不是使能SWDT。



## 看门狗初始化 ##
具体的初始化代码在`xfsbl_msic_drivers.c`中。

初始化流程主要设置了这几个参数

- PRESCALE 预分频
- WDT RESET VALUE = `XFSBL_WDT_EXPIRE_TIME` = 100秒
- `SWDT.MODE[RSTEN]`允许SWDT产生RESET信号（没有允许产生IRQ）
- 在`PMU_GLOBAL.ERROR_SRST_EN_1`和`PMU_GLOBAL.ERROR_EN`中使能LPD WDT

根据以上设置，当WDT计数到0，产生RESET信号，PMU接收到信号后进行系统复位。由于`PMU_GLOBAL`寄存器中只使能了LPD WDT，所以在不改FSBL代码的情况下应该为FSBL使能LPD WDT。


## 什么时候操作看门狗 ##

### 每次Load Partition时复位看门狗 ###
`XFsbl_PartitionLoad()`函数中：
```
#ifdef XFSBL_WDT_PRESENT
	/* Restart WDT as partition copy can take more time */
	XFsbl_RestartWdt();
#endif
```

### 进入Fallback前停止看门狗 ###
`XFsbl_FallBack()`函数中：

```
#ifdef XFSBL_WDT_PRESENT
	/* Stop WDT as we are restarting */
	XFsbl_StopWdt();
#endif
```

什么是Fallback？

简单来说就是在执行到特定的错误时，调用Fallback函数，就可以自动重启，并寻找Golden Image进行启动。Fallback功能是通过Multiboot寄存器实现的，它记录了当前Image的起始地址，所以Fallback的流程就是将Multiboot寄存器＋1，然后进行Reset。

     In Fallback,  soft reset is applied to the system after incrementing the multiboot register. A hook is provided to before the fallback so that users can write their own code before soft reset


## 启动时报告WDT引起复位的错误原因 ##

系统启动初始化时调用的`XFsbl_Initialize()`中有`XFsbl_ResetValidation()`。

- 首先从`PMU_GLOBAL.PERS_GLOB_GEN_STORAGE1(0xFFD80054)`读取`FsblErrorStatus`。这是一个用户自定义的寄存器，软件写入而不是硬件自动写入，只有在POR的时候才会清空，在soft reset的时候会保留内容。
- 然后从`CRL_APB.RESET_REASON(0xFF5E0220)`读取`ResetReasonValue`。这个寄存器是`Clock Control Low Power Domain (CRL_APB)`模块的，它可以区分上一次复位来自POR，PMU引发的PS Only Reset，还是PMU引发的System Reset。同样也是只有在POR的时候才会被清空。这个寄存器是由硬件将状态信息写入的。
- 然后从`PMU_GLOBAL.ERROR_STATUS_1(0xFFD80530)`读取`ErrStatusRegValue`。模块报错信息会被集中到这个寄存器。同样是POR时才被清空。
- 采集到以上信息后判断是否上一次复位的原因是WDT。如果是的话，打印一句信息，并设置范围值指示这是WDT Reset；如果不是的话，就保证自己的状态记录正常。然后继续运行，函数返回。
- 返回到`XFsbl_Initialize()`后继续带着状态信息返回到`main()`，在`main()`中的状态机处理返回状态。状态机看到不是`XFSBL_SUCCESS`的Status，就跳到`XFSBL_STAGE_ERROR`状态，然后进行Lock Down。

综合以上情况，可以发现2016.2中MPSoC的FSBL并不支持由WDT引发的错误进入MultiBoot的流程，寻找后面的Golden Boot。它只会报告错误原因，帮助用户查找错误。


## 重点回顾 ##

- 在Vivado中配置MPSoC PS，打开`I/O Configuration`中的`SWDT0`可以让FSBL支持看门狗功能。SWDT0是LPD WDT。
- 默认看门狗超时时间是100秒。可以修改`XFSBL_WDT_EXPIRE_TIME`来修改超时时间。
- 如果FSBL不需要看门狗，可以在`xfsbl_config.h`定义`FSBL_WDT_EXCLUDE_VALUE` = 1
- 这个版本的FSBL在遇到看门狗超时的情况下将Multiboot寄存器＋1，重启，也就是寻找下一个可执行的Image。下一个Image中如果也包含同样的FSBL，会报告上一次重启的错误原因信息，然后死锁，而不是进行后续的Multiboot。

