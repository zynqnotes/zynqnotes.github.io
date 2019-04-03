---
layout: post
title:  "MPSoC Secure Boot 使用安全启动加量不加价"
categories: Tutorial
ta:
- MPSoC
- Secure Boot
---


在电影里，黑客远程控制一个城市中所有的汽车，让它们追逐指定的目标，这样的场景让人感觉不寒而栗。在现实中，某个大网站的用书数据泄露的新闻也已经屡见不鲜。保护系统安全，之前是网络安全工程师的职责，似乎大多数工程师不太关心这个领域。但其实在产品越来越网络化、智能化的现在，安全的意义已经越来越重要，毕竟我们不希望电影里黑客控制汽车的场景真的发生。除了汽车，任何联网的设备，都会给黑客远程访问的可能性；除了放在保险柜里，任何设备只要能被黑客物理接触到，篡改系统或者复制系统也不是没有。

在我之前接触的客户中，也只有少量公司在使用 Xilinx FPGA 和 SoC 芯片所提供的安全功能。其他的情况，可能是有物理环境能保证产品不被别人接触到；但更多可能是工程师觉得安全功能不是必须的，或者使用安全功能太复杂。

攥写本文，我希望能让大家看到
1. 在产品中加入安全功能通常都是有需求的、有价值的，而且需求会越来越大。
2. 为产品添加安全功能，达到一定的安全级别，流程步骤和复杂程度是可控的。

就我了解到的情况，大多数使用 ZYNQ-7000 或 ZYNQ UltraScale+ MPSoC 的用户并没有打开 Secure Boot 功能。如果你能看完全文，照着步骤做一下，打开安全启动功能，就能“免费”提升产品安全等级，就是“**加量不加价**”啦。

上面讲到“达到一定的安全级别”，是因为安全保护是有代价的，黑客破解也是有代价的，我们需要保证以尽可能低的安全保护的代价，使黑客破解系统的代价比他攻破系统能获取到的价值更高，这样黑客就没有攻击系统的动力，达到了自我保护的目的。

因此本文只描述最容易理解、用做小的代价能把 MPSoC 提供的安全启动的功能用起来的部分。MPSoC 的安全启动还可以有很多定制化的空间，更多高级功能比如 PPK Revoke 等，都可以参考本文最后列出的参考文档，Xilinx 官方文档中有更多详细解释。

全文将从以下几个方面讨论安全启动

一、常见芯片攻击类型与防护思路
二、MPSoC 采用的防御方法
三、实战使用指南
四、常见问题和错误
五、参考文档

注：本人并不是安全专家。本文仅是对安全启动的操作记录以及对安全保护的个人理解。对于使用本文描述的方法进行设计生产的后果请恕本人无法对其负责。

# 一、常见芯片攻击类型与防护思路

## 产品克隆

竞争对手购买一份样机，找人复制 PCB （俗称抄板），购买到所有的元器件，复制 Flash 中的内容，如果这样就能让系统启动并且完成所有预定功能，这样就轻易完成了产品克隆。由于没有研发成本，可以使用低价倾销，损害产品设计者的利益。

现在的主芯片一般都有一些机制，在授权的主芯片上特制一些信息，让Flash上的程序和芯片上的信息互相验证。这样只要对方不能复制主芯片上的特制信息，就不能做到克隆。

## 破解软件

当产品无法直接克隆，竞争对手/黑客可以通过查询软件具体实现来了解系统的运行机制，寻找可能的漏洞，或者获取受知识产权保护的信息。

为了不让对方看到软件原始信息，可以对软件进行加密。加密信息存在 Flash 上，芯片上电后解密后运行。黑客仅仅读取 Flash 上的内容无法了解软件原始信息。

## 增加恶意代码

如果整体或部分替换 Flash 上的软件，或让操作系统运行自己开发的恶意代码，有可能可以获取到系统的关键信息。在联网的设备中，恶意代码能获取到的信息可能会更多。在带人工智能功能的设备上，恶意代码能实现的破坏力也可能比传统的攻击更严重。因此控制软件运行权限，确保只有授权应用能够运行，不让非授权的应用在系统中运行，对系统安全也非常重要。

认证，或者称为授权 （Authentication) ，是防止恶意代码运行的重要步骤。

## 打磨芯片获取密钥

加密和认证一般的实现方法都是通过在芯片内部布置只可以烧写一次的反熔丝，用这些熔丝位来非易失地存储密钥。反熔丝相对普通逻辑来说是区别比较明显的，理论上来说，打开芯片封装后是有可能通过显微镜是有可能查找出反熔丝的密钥位，然后用它解密的。

要防护通过这种方式获取密钥，主要有两种方法：

1. 采用非对称方式的公钥-私钥对，在芯片中只存储公钥，或公钥的Hash。对方获得公钥是没有意义的。
2. 对于必须采用对称加密的方式，可以将密钥再次加密。只要再次加密的密钥只能通过某种不能被窥探的方法解密成原始密钥，就可以防护这种方式。

## 旁路攻击

所谓[旁路攻击](https://zh.wikipedia.org/wiki/%E6%97%81%E8%B7%AF%E6%94%BB%E5%87%BB)，就是攻击者并不真正“看”到存储在芯片中的密钥，而是通过其他方式计算出来。一个比较容易理解的例子就是最近在网络上流行一个应用，通过电脑的麦克风采集音频，算出敲击键盘的字符。采集的信息不是直接从键盘的输入的来，而是从麦克风得来，就是一种旁路攻击。

对于获取芯片密钥的旁路攻击，比较常见的是通过监测电源功耗的[简单功耗分析](https://zh.wikipedia.org/wiki/%E8%83%BD%E9%87%8F%E5%88%86%E6%9E%90)、差分功耗分析。

要防止类似的旁路攻击，防护思路主要是让一个密钥所对应的解密数据尽可能少，这样就能避免攻击者采集到足够多的用于统计的信息。

# 二、MPSoC 采用的防御方法

| 攻击方法       | 防御方式    |  MPSoC 上的对应硬件和措施  |
| ---------      | -------- | -----: |
| 产品克隆        | 反熔丝  | 只能烧写一次的反熔丝 eFUSE |
| 破解软件        | 对镜像加密     | 硬件加速的 AES-256 |
| 增加恶意代码    | 认证 | 硬件加速的 RSA-4096 |
| 打磨芯片获取密钥| 非对称加密；对对称加密密钥再次加密    | Black Key in eFUSE  | 
| 旁路攻击       | 用多个密钥，减少每个密钥对应的密文长度  | Key Rolling       |

注意：这里提到的防御方式和 MPSoC 上的对应硬件措施并不是完备的列表，只是实际应用中最常用的方式。

比如 MPSoC 上储存密钥还可以存在 BBRAM (Battery Backed RAM) 中，系统掉电可以继续保存密钥，而且由于它基于 RAM 结构，不是 eFUSE，所以即使打开芯片也不能获取到密钥。但大多数系统都不能容忍产品发布后，支持 BBRAM 的电池耗尽后，系统无法启动的限制，因此暂不介绍。

## 什么是 AES Black Key

AES 是一种对称加密技术。所谓对称加密，就是加密和解密用的是同一个密钥。这样的话，保护密钥就变得尤为重要。把 AES Key 直接烧录到 eFUSE 中被称为 Red Key，如果有人能够打开芯片探测到 eFUSE 的烧录情况，就可以获得密钥。为了防止这种情况发生，Xilinx 提供了 Black Key 方案。

每片 MPSoC 芯片中都有一个硬件模块叫做 PUF: Physical Uncloneable Function。它可以根据每片芯片物理上本身工艺尺度上的微小差异，对 Red Key 进行加密生成 Black Key。当 Black Key 烧录在 eFUSE 后，这片芯片的 PUF 模块通过一些输入信息还可以将 Red Key 还原回来，再用得到的 Red Key 解密安全启动的镜像。因为烧录在 eFUSE 中的是 Black Key 而不是 Red Key，它不能通过简单的方法还原出黑客希望获得的密钥来解密镜像，整个解密过程只能在芯片内部进行，使用 Black Key 技术相当于为产品的安全再增加了一堵墙。

## 什么是 Key Rolling

像功耗分析这样的旁路攻击，是通过统计学原理从密文推算出密钥，再还原成原文。为了不给攻击者足够的信息推算密钥，Key Rolling 的主要想法是一个密钥只用于有线长度的原文。原文切成很多小份，每一小份原文加一个新的密钥打包后用密钥加密，新的密钥用于加密后面的一份原文和密钥，这样滚动前进。

解密时就反向操作，eFUSE 里的密钥只用于解密 section 1，解密出原文1 和密钥1，再用密钥1 解密 section 2，得到原文2 和密钥2，依此类推。

了解了基本的攻击路数和防御方法，会感觉到这些方法其实就是魔高一尺道高一丈。很多防御功能都已经内置在芯片里了，使用成本很低，只要把他们用起来，就能把自己的知识产权保护起来。下面我们就了解一下怎么做实战操作。

# 三、全局观：做一个适合你的安全启动的方案

做一个安全启动方案，可能涉及到做许多选择和决定。每一个决定，都有一定的选择依据。

## 第一个选择：要保护哪些内容

要知道需要保护哪些内容，先要知道有哪些内容可以保护。在一个通常的 MPSoC 设计中，软件逻辑组件有这些：

- FSBL
- PMU Firmware (PMUFW)
- ARM Trusted Firmware (ATF)
- FPGA Bit File
- U-boot
- Linux Kernel + device tree + rootfs
- User Data (Used by u-boot or Linux)

启动加载流程有些可以调换，但主要是通过以上的顺序加载的。FPGA Bit 可以在 FSBL、 U-boot、 Linux 阶段被加载。

![General Boot Flow](ug1137_ch7_boot_flow.png)

每一个组件都可以选择 1) 同时做认证和加密 2) 只做认证 3) 只做加密 4) 什么都不做 (就是 Non-secure boot)。

上面的四种选项意味着，加密和认证是两个独立的变量。但实际使用中“只做加密”这种情况比较少用到。由于 MPSoC 中有硬件加速的 RSA 认证模块，认证对启动时间的影响比较低；由于认证的机制就是在镜像最后加上checksum，对存储空间的需求也很低，所以即使只需要加密功能，顺便也把认证功能加上了。

对于安全启动有一个概念叫做 Root of Trust，信任是要从根部就开始的，或者叫 Chain of Trust，信任链，都是一个意思。要保证某个步骤是安全可靠的，必须前面所有的步骤都是安全可靠的。比如某个应用场景需要保证 FPGA Bit 文件是安全可靠的，Bit 文件是 U-boot 加载的，那么 Bit 文件和从启动到加载 Bit 文件之前的所有步骤都需要经过认证，之后的可以不做，在这个例子中， FSBL，PMUFW，ATF，U-boot 和 Bit 都需要经过认证，Linux 不一定需要。

有一种说法是，如果代码完全是开源的，不包含用户的知识产权，那么这个部分可以只做认证，不做加密，以尽可能少地用到 eFUSE 中的 AES Key；但另一种想法认为，没修改的就不加密，就等于暴露了自己没有修改开源代码这件事，而这个信息本身也是有价值的。两种说法都有道理，怎么选择看自己取舍吧。

## 第二个选择：怎样打包组件

打包的时候，可以把所有内容整个一起打包成 Boot.bin，UG1209 中的例子就是这样做的，这样做完整的安全启动最容易实现，但也意味着这些内容都不可以随意动态变更，每次重启系统恢复到原样。一般特定功能的嵌入式系统倾向这种设计。

但开放式系统不允许这中完整打包，每次复原的设计。用户可能希望把在系统中做的操作保存在 Flash 中，重启后仍然能够访问到。于是另一种常见方案是从 FSBL 到 u-boot 用认证加密的方式打包成一个 Boot.bin，Linux 部分存在独立的 Flash 分区中。怎么保证可读写的 Linux 分区安全是另一个复杂的话题，用户场景和解决方法也多种多样，就像医生得看到病人才能开药，具体怎么做没办法简单一句概括，有机会后续再聊了。

如果将所有内容都打包进 Boot.bin，这样 Boot.bin 可能很大，一个典型的PetaLinux Kernel + Rootfs 有大约60MB，加上 MPSoC 的 Bit 文件大约 10-20 MB，如果需要再添加一些应用程序，可能一套 Boot.bin 有100MB。如果需要一个功能完整的 Linux，甚至可能到 1GB。这就对 Flash 容量有一些挑战，现在最大的 QSPI 芯片只有 128MB，如果 Boot.bin 超过了 QSPI Flash 容量，就只能选其他启动方式，比如 eMMC。由于使用 NOR Flash 技术的 QSPI 比使用 NAND 技术的 eMMC和 SD 更可靠一些，如果考虑到可靠性一定要用 QSPI Flash 启动，就可以将除了 Linux 以外的部分打包成 Boot.bin，放在 QSPI Flash，Linux 部分放在其他介质，比如 eMMC 或 SD 卡。

## 第三个选择：在哪一步做安全认证和解密

前面讲到一个例子，FPGA Bit 可以由 FSBL、U-boot 或 Linux 加载。如果有用户自定义数据需要在安全启动过程中加载，也可以用同样的流程。

最终做认证和解密的都是 PMU。选择在哪一步加载，就需要在这步调用安全启动的 API，与 PMU 通信，由 PMU 进行认证、解密后再加载。

- 由 FSBL 加载的组件需要都打包在 Boot.bin 中，打包过程中指定每个组件的加密、认证方式，FSBL 会根据这些配置在运行时做认证和解密。
- U-boot 定义了一套 API 与 PMU 通信，将认证和解密的任务交给 PMU 执行。如果需要 U-boot 过程中认证、解密，需要使用 Xilinx 特殊的 U-boot 命令 `zynqmp secure`。打包 U-boot 能识别的安全组件也是通过 bootgen，与上面 Boot.bin 的打包方式类似。普通文件和 FPGA Bit 都可以打包成 Boot.bin 供 U-boot 加载。具体方法说明见参考文档5。
- Linux 和 U-boot 类似，都是通过驱动与 PMU 通信。但是截至到目前最新版本 (2018.3)，Linux 只支持对 FPGA Bit 文件进行认证、解密后直接加载，不支持用户数据的认证和解密。具体操作方法见参考文档6。

三个选择全都做完，用户可以自己整理出一个表格，比如这样：

| 组件      | 认证    |  加密 | 打包 |
| --------- | -------- | ----- | -- |
| FSBL              | Y  | Y | Boot.bin |
| PMUFW             | Y  | Y | Boot.bin |
| ATF               | Y  | Y | Boot.bin |
| U-Boot            | Y  | Y | Boot.bin |
| Linux Kernel      | Y  | Y | image.ub.bin |
| Linux Device Tree | Y  | Y | image.ub.bin |
| Linux RootFS      | Y  | Y | image.ub.bin |

FSBL、PMUFW、ATF、U-boot 打包成 Boot.bin，所有组件全都进行认证和加密。Linux 的各组件由 PetaLinux 打包成 image.ub，用 bootgen 整体进行认证和加密，生成 image.ub.bin。

## 没有后悔药：安全为上的测试流程

由于 eFUSE 只可以烧写一次，如果由于不熟悉流程，设置出错的话会导致芯片无法正常使用，因此烧写 eFUSE 需要比较谨慎，尽量先把需要验证的部分验证通过，最后才真正在 eFUSE 中烧写密钥和对应的设置位。

为了让用户能够再不烧写 eFUSE 的情况下提前能验证流程以及密钥的组合，Xilinx 提供了 Boot Header 方法，简称 BH。在生成镜像的时候，将密钥暂存于镜像的头部。启动时，软件从镜像头部寻找密钥，跳过从 eFUSE 硬件读取的过程。BH 流程会使用镜像头部中暂存的密钥与镜像匹配进行认证和解密操作。用户可以在熟悉BH流程后，最后再将密钥烧录到 eFUSE 中。

注意：
1. Boot Header 流程是可选的，不是必须的。它只是给用户检验流程用的。既做 BH 流程又做 eFUSE 流程可能会把人弄晕，如果真是这样的话建议不要做 BH 流程直接做 eFUSE 流程。
2. Boot Header 流程不可以用在真实产品中，因为密钥在镜像头部，仍然可以被提取。

总体测试流程可以梳理为以下步骤：

1. 确认自己的安全启动模式：每个组件用那种加密模式和认证模式，可以列成一个表格。AES 使用 Black Key 还是 Red Key；AES Only 还是 AES + RSA；Key 存在 eFUSE 还是 BBRAM 等。
1. 生成所需的密钥
    - RSA Private Key 和 Public Key
    - AES Key 0 （用于烧录 eFUSE 的 Key）
1. （可选）用 BH 模式生成 BOOT.BIN，进行 RSA 和 AES 在 Boot Header 中的测试
1. 用 eFUSE 模式生成 BOOT.BIN
1. 将 BOOT.BIN 写入 SD 卡或 Flash
1. 从 JTAG 模式启动，将 RSA PPK Hash 和 AES Key (red or black) 写入 eFUSE。用 Flash/SD 模式重启后验证安全启动流程。
  a) RSA PPK 和 AES Red Key 可以同时写入
  b) 如果使用 AES Black Key，分别使用两个程序，先烧录 Blackkey （这个过程称为 PUF Registration），再烧录 RSA PPK Hash

注意：如果启动模式是 SD 卡，万一镜像和 eFUSE 中的密钥不匹配，由于更换 SD 卡中的启动镜像比较容易，那么多次尝试也不会遇到什么困难。与之相对，如果启动设备只有 Flash，则需要先烧录 Flash 后烧录密钥，因为烧录密钥后 JTAG 将不能使用，不能再使用 SDK 自带的 Flash Programmer 工具烧录 Flash。如果烧录 Flash 的镜像错误，无法完成安全启动，但是 eFUSE 又已经烧录，只能通过焊接更换 Flash，离线烧录正确的镜像，再焊接回板上，才能继续使用这个系统。

# 四、实战使用指南

以下几种典型使用模式中，生成密钥和生成加密镜像的流程都是一样的，不同的是需要使用不同的 BIF 配置文件。生成工具是 Vivado 或 SDK 中的命令行小工具 bootgen。烧录 eFUSE 使用 SDK 中自带的例子程序，只需要对头文件进行很少的修改，就能执行。如果使用 Blackkey，额外需要一个注册的操作，输入 red key，运行程序时 MPSoC 产生 Black Key。

## 让 FSBL 能打印出详细启动信息

在调试阶段，主要的安全启动状态信息是由 FSBL 打印出来的。默认 FSBL 并不打印详细调试信息，需要在 FSBL 编译的时候定义 `FSBL_DEBUG_INFO`，才会有详细信息打印。

## 生成密钥和镜像

这一节主要描述这几件事：Bootgen 工具、BIF 描述文件和密钥文件。

### BOOTGEN 命令

Bootgen 可以实现很多功能，最主要的功能是产生用于启动的 Boot.bin。它可以用来生成带安全启动功能的镜像，也可以是不带安全启动功能的。以下是我们在制作安全启动功能 Boot.bin 时常用的一步到位命令：

```
bootgen -arch zynqmp -image secureboot.bif -efuseppkbits ppk0_digest.txt -w -o BOOT.bin -encryption_dump
```

- 指定 ZYNQ UltraScale+ MPSoC 架构。具体哪个型号的芯片不用关心。
- `image` 选项指定输入文件。BIF 是 Boot.bin 结构的描述文件，后面章节马上就会介绍。
- `efuseppkbits` 选项将为 RSA Primary Public Key  产生 Hash，并保存到指定文件。这个 Hash 是将要写入 eFUSE 在 RSA 认证时使用的。
- `o` 指定输出的 Boot.bin 文件名，`w`表示可以覆盖文件。
- `encryption_dump` 指定将每个分区 AES 加密的详细过程输出到 `aes_log.txt` 中。
- 所谓“一步到位”命令，就是它能只能识别 BIF 中所指定的密钥文件是否已经存在，以及互相之间是否匹配。如果
    - 如果密钥已经存在，bootgen 会使用现有密钥
    - 如果密钥不存在，或部分不存在，bootgen 会生成所有需要的密钥
    - 如果发现已有密钥不符合规则，比如所有 AES Key nky 文件的的 Key 0 不一致，bootgen 会报告错误，因为 Key 0 指的是将要写入 eFUSE 的密钥，所有分区的 Key 0 必须一致。

如果只需要其中某一步操作结果，这些参数也可以分开使用。
```
bootgen -p zcu9eg -arch zynqmp -image secureboot.bif // 生成 Key
bootgen -p zcu9eg -arch zynqmp -efuseppkbits ppk0_digest.txt -image secureboot.bif // 生成 PPK Hash
bootgen -p zcu9eg -arch zynqmp -image secureboot.bif -w -o BOOT.bin -encryption_dump // 生成 BOOT.BIN 和 aes_log.txt
```

### BIF 结构

BIF 是 bootgen 的输入描述文件。它描述了需要生成 boot.bin 的结构，比如 boot.bin 中有哪些 partition，每个 partition 的源来自哪些文件，这些 partition 由谁引导，会在哪个 target device 上运行，怎样认证，以及怎样加密。基本它就是上述第三章总结的表格的文字描述。

BIF 的结构大致是这样

```
the_ROM_image:
{
[文件属性]文件名
[参数类型]参数值
}
```

具体的文件属性、参数类型、参数值的可选项，可以参考 UG1137 第 16 章。

### 参考的 BIF: 使用 RSA + AES (Red Key)

```
the_ROM_image:
{
[pskfile]psk0.pem
[sskfile]ssk0.pem
[auth_params]spk_id = 0; ppk_select = 0
[keysrc_encryption]efuse_red_key
[fsbl_config]a53_x64
[aeskeyfile]red.nky
[bootloader, authentication = rsa, encryption = aes]FSBL_1112.elf
[aeskeyfile]pmu_fw.nky
[destination_cpu = pmu, authentication = rsa, encryption = aes]pmu_fw.elf
[aeskeyfile]fpga.nky
[destination_device = pl, authentication = rsa, encryption = aes]top.bit
[destination_cpu = a53-0, exception_level = el-3, trustzone, authentication = rsa]bl31.elf
[aeskeyfile]uboot.nky
[destination_cpu = a53-0, exception_level = el-2, authentication = rsa, encryption = aes]u-boot.elf
}
```

用本章第一节的 Bootgen 命令就能生成 Boot.bin。

从上面的BIF的例子中可以看到：

- 所有 Partition 都加密了，每个 Partition 都对应了一个 nky 文件，这些 nky 文件的 Key 0 一致， Key 1 各不相同。Key 0 解出 Key 1，然后 Key 1 解密对应的 Partition。
- 当 PMUFW 和 FSBL 都加密的情况下，PMUFW 需要由 FSBL 来加载，而不是由 CSU BootROM 来加载，所以使用了 `[destination_cpu = pmu]`，而不是`[pmufw_image]`。参考 [AR 70622](https://www.xilinx.com/support/answers/70622.html)
- Bit 文件需要加密的情况下，Bit 要写在 ATF (bl31.elf) 的前面，因为解密 Bit 需要用的 OCM 空间与存放 ATF 的空间是同一个。参考 UG1137 v8.0 P114

### 参考的 BIF: 使用 RSA + AES (Black Key)


```
the_ROM_image:
{
[pskfile]psk0.pem
[sskfile]ssk0.pem
[auth_params]spk_id = 0; ppk_select = 0
[keysrc_encryption]efuse_blk_key
[fsbl_config]a53_x64
[aeskeyfile]red.nky
[bootloader, authentication = rsa, encryption = aes]FSBL_1112.elf
[aeskeyfile]pmu_fw.nky
[destination_cpu = pmu, authentication = rsa, encryption = aes]pmu_fw.elf
[aeskeyfile]fpga.nky
[destination_device = pl, authentication = rsa, encryption = aes]top.bit
[destination_cpu = a53-0, exception_level = el-3, trustzone, authentication = rsa]bl31.elf
[aeskeyfile]uboot.nky
[destination_cpu = a53-0, exception_level = el-2, authentication = rsa, encryption = aes]u-boot.elf
[bh_key_iv] black_iv.txt
}
```

Black Key 的 BIF 与 Red Key 相比只有微小的改动：
- `[keysrc_encryption]`改为了`efuse_blk_key`
- 最后增加了`[bh_key_iv]`，一个 Initialization Vector。

如果只想做非破坏性的实验，请用附3中著名“Boot Header”的例子BIF。

## 烧录镜像

接下来我们先烧录镜像到 Flash，再烧录密钥到 eFUSE。如果使用 QSPI Flash, NAND Flash 或 eMMC 启动，由于烧录密钥后 JTAG 将无法方便地使用，先烧镜像会方便很多。如果使用 SD 卡启动，先后没有关系。

这一步和非安全启动的操作一样，只是将烧录的内容替换成上一节产生的带有安全启动功能的 Boot.bin。这个 Boot.bin 至少带有 u-boot，因此后续如果还需要烧写更多内容，也会比较方便。

烧录 Flash 可以用 SDK 自带的工具 `SDK -> Xilinx Tools -> Program Flash`，也可以使用 u-boot。U-boot 可以事先存放在 Flash 或 SD 卡中，或者使用[这个脚本](https://gist.github.com/imrickysu/b911be34cf7fffc1b9259610095973fd)从 JTAG 加载 u-boot。

## 烧录密钥

### 生成密钥烧录程序

烧录密钥不需要使用特殊的软硬件工具，只需要在 MPSoC 上运行一个 Standalone 程序。这个程序可以用 Xilinx Software Development Kit (SDK) 生成。

Xapp1333 的 RSA eFUSE Configuration 章节很详细地描述了怎样在 SDK 中生成 `xilskey_efuseps_zynqmp_example` 以及配置头文件的过程。这个例子程序可以烧录 RSA Key 和/或 AES Red Key。Black Key 需要用下面一节讲的 `xilskey_puf_registration` 程序。

简要描述流程就是：
1. 创建 SDK Workspace, 导入 Vivado HDF 文件
2. 在 BSP Settings 中使能 Library: XilSKey 和 XilSecure
3. 双击 BSP 的 MSS 文件，找到 Library - XilSKey, 在 `Import Examples` 弹出的对话框中选择需要导入的例子程序
    - RSA 和 AES Red Key 使用 `xilskey_efuse_zynqmp_example`
    - AES Black Key 使用 `xilskey_puf_registration` 
4. 修改例子程序中的头文件中的参数，以符合自己的需求
5. 编译工程，在板上运行，观察串口输出
    - 如果使用 Black Key，先运行 `xilskey_puf_registration`，再运行`xilskey_efuse_zynqmp_example`

头文件中每个参数的意义在顶部都有详细解释。某些参数仅在使能了其他参数时才有效，具体可以参考C代码，并不是很难读。在这里对必要的选项做个概述。

使用 RSA + AES Red Key 的方案，要修改 `xilskey_efuse_zynqmp_example` 中 RSA 的部分参数和 AES 部分参数；使用 RSA + AES Black Key 的方案，要修改 `xilskey_efuse_zynqmp_example` 中 RSA 的部分参数和 `xilskey_puf_registration` 中的部分参数。

`xilskey_efuse_zynqmp_example` 中 RSA 的部分参数。

| 参数                   | 目标值    |  说明  |
| ---------              | -------- | -----: |
| XSK_EFUSEPS_RSA_ENABLE | TRUE  | 烧写 RSA_ENABLE 位。烧录后没有 RSA 功能的 Boot.bin 将不能启动系统；JTAG 也只有在启动成功后才能使用 |
| XSK_EFUSEPS_WRITE_PPK0_HASH | TRUE| 烧录 RSA PPK0 Hash |
| XSK_EFUSEPS_PPK0_HASH | 上文 ppk0_digest.txt 中的内容 |   由 Bootgen 产生的 PPK0 的 SHA3 Hash |
| XSK_EFUSEPS_PPK0_IS_SHA3| TRUE    | bootgen 默认生成 SHA3 Hash |

`xilskey_efuse_zynqmp_example` 中 AES Red Key 的部分参数：

| 参数                   | 目标值    |  说明  |
| ---------              | -------- | ----- |
| XSK_EFUSEPS_RSA_ENABLE | TRUE  | 烧写 RSA_ENABLE 位。烧录后没有 RSA 功能的 Boot.bin 将不能启动系统；JTAG 也只有在启动成功后才能使用 |
| XSK_EFUSEPS_WRITE_AES_KEY | TRUE| 烧录 AES Key |
| XSK_EFUSEPS_AES_KEY       | red.nky 中的 Key 0 | |

`xilskey_puf_registration` 中需要修改的部分参数：

| 参数                   | 目标值    |  说明  |
| ---------              | -------- | -----: |
| XSK_PUF_INFO_ON_UART | TRUE  | 输出打印到串口 |
| XSK_PUF_PROGRAM_EFUSE | TRUE| 烧录 RSA PPK0 Hash |
| XSK_PUF_PROGRAM_SECUREBITS | TRUE |   由 Bootgen 产生的 PPK0 的 SHA3 Hash |
| XSK_PUF_AES_KEY | red.nky 中的 Key 0| bootgen 默认生成 SHA3 Hash |
| XSK_PUF_IV | 任意         | 完全任意，与其他值没有关系       |

在真实量产环境中，最好也将以下的锁锁上，因为 eFUSE 的烧录是用反熔丝技术将 0 改写成 1，没有烧录过的比特仍然能继续修改，知道所有比特都是1为止。将 LOCK 设置为 1 后，对应的 eFUSE 比特就无法再改写，达到最高的安全性。

`xilskey_efuse_zynqmp_example` 中防止 eFUSE 再次写入的锁

| 参数                     | 目标值    |  说明  |
| ---------                | -------- | ----- |
| XSK_EFUSEPS_PPK0_WR_LOCK | TRUE     | 防止再次写入 PPK0 |
| XSK_EFUSEPS_AES_WR_LOCK  | TRUE     | 防止再次写入 AES Key  |

`xilskey_puf_registration`中的锁

| 参数                     | 目标值    |  说明  |
| ---------                | -------- | ----- |
| XSK_PUF_PROGRAM_SECUREBITS | TRUE     | 后续几个锁将被写入 |
| XSK_PUF_SYN_WRLK | TRUE     | 防止再次写入 syndrome data |

`xilskey_efuse_zynqmp_example` 默认的烧录流程中如果只烧录 RSA Key（`XSK_EFUSEPS_RSA_ENABLE` == TRUE）而没有烧录 AES Key，AES Key CRC Check 仍然会执行，如果此时 AES Key 并没有烧录过，会报告错误而退出程序。手工修改代码，注释掉这部分即可。2018.2 中仍然有以上问题，后续版本中会修复。

eFUSE 烧录完成后，断电重启，就能看到带安全启动功能的Boot.bin中FSBL打印出的内容了。

## 让 U-boot 加载安全启动文件

下面将尝试只在 Boot.bin 中启动到 U-boot，让 U-boot 在继续保持安全的模式下，获取 Bit 和 Linux 文件，进一步进行启动。这样做的好处是 Linux 和 FPGA Bit 不需要打包到 Boot.bin 中，有利于升级时的文件管理，也可以将不同的部分存放在不同的 Flash 中，降低对启动 Flash 的容量要求。

U-boot 可以调用 PMU 对安全启动文件进行认证和解密，可以用来获取签名且加密的 FPGA Bit 文件、Linux 镜像或用户自定义数据。使用步骤分为两步：制作文件和加载。具体过程在 [Xilinx Wiki - Authentication and decryption at u-boot](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842432/Authentication+and+decryption+at+u-boot) 中有详细解释。

### 制作能让 U-boot 加载的安全启动文件

像上文一样通过写一个BIF文件描述需要签名和加密的源文件，经过 bootgen 就能产生安全启动文件。至于 BIF 的写法，基本和上文上电启动时一致，只是我们一般不需要 FSBL, PMUFW 和 ATF，去掉那些段即可。

bootgen 命令的使用方法也一致：
```
bootgen -image Data.bif -w -o Output.bin -arch zynqmp
```

### U-boot 加载安全启动文件

对于数据文件，比如 Linux 镜像，使用 zynqmp 命令加载：
```
zynqmp secure <DDRaddrees> <size>
```

对于 Bit 文件，使用 fpga load 命令加载:
```
fpga loads [dev] [address] [size] [auth] [enc] [Userkey address]
```
关于每个参数的意义，请参考 Wiki 中的详细解释。

## 附录

### 附1： RSA Key 样例

Bootgen 可以根据类似 `[pskfile]psk0.pem`的 BIF 描述生成 RSA Key psk0.pem。如果密钥文件已经存在，bootgen 就会使用这个密钥，而不会生成新密钥。

RSA Private Key psk0.pem 大致内容如下
```
-----BEGIN RSA PRIVATE KEY-----
bi4TvfJuikXSAjJ3XoHDCwGZ8SerGMKSJ8gPWFbN/Ay0x27qBGaoAUNjySyWRPQC
...
tWnmtk/l6E1kGvZXvGRr1BJSDh3IV4k/LehTj+zSHD/vZ0GNFUjqAC4RbEkj7t17
-----END RSA PRIVATE KEY-----
```


### 附2： AES Key 样例

Bootgen 可以根据 `[aeskeyfile]key.nky`生成 AES Key 文件: key.nky。

AES Key 大致内容如下：

```
Device       zcu9eg;

Key 0        A98B3BC6BBA48E2F992C20DB28CB498EB17F3B8D9219AAB6689B726A107F8A70;
IV 0         F18CAED66C8BDA74F8A2DC65;

Key 1        630A30B2815E2518792B3C0CAD84824D7B61586030EE0648FCE363DBC0C03234;
IV 1         43BE7AAAD6631BDB692EB64F;
```

其中 Key 0 是将要烧录到 eFUSE 中的 AES Key。

启动时用 eFUSE 中的 AES Key 解密存于 boot header 中的已经加密了的 Key 1，得到 Key 1 后再用 Key 1 解密这个 Partition。这样保证尽可能少地使用 eFUSE 中的 AES Key，防止 DPA 攻击。

### 附3：常见用法的 BIF 例子

- BIF File with eFUSE Red Key: UG1137 (v8.0) P103
- BIF File with an Operational Key: UG1137 (v8.0) P103
- BIF File for Black Key Stored in eFUSE: UG1137 (v8.0) P106
- BIF File for Black Key Stored in Boot Header: UG1137 (v8.0) P107
- Bif File for Obfuscated Form (Gray) key stored in eFUSE: UG1137 (v8.0) P107
- BIF File with SHA-3 eFUSE RSA Authentication and PPK0: UG1137 (v8.0) P112

# 五、常见问题和错误

## 只烧 RSA Key，不烧 RSA_EN，导致 FSBL 无法启动后续 partition

要让带 RSA 认证的 Image 成功启动，必须将 RSA Key 和  RSA_EN 的 eFUSE 位全部烧录。只烧录 RSA Key 不烧录 RSA_EN会导致 FSBL 无法对后续 Partition 做认证。

对应 Error Log

```
======= In Stage 3, Partition No:1 =======   
UnEncrypted data Length: 0x49E1   
Data word offset: 0x49E1   
Total Data word length: 0x4DA0   
Destination Load Address: 0xFFDC0000   
Execution Address: 0xFFDC8C68   
Data word offset: 0x7A70   
Partition Attributes: 0x883E   
QSPI Reading Src 0x31180, Dest FFFDC490, Length EC0 
.QSPI Reading Src 0x1E9C0, Dest FFDC0000, Length 127C0 
.Authentication Enabled 
Auth: Partition Offset FFDC0000, PartitionLen 13680, AcOffset FFFDC490, HashLen 30 
XFsbl_SpkVer:  XFSBL_ERROR_SPK_RSA_DECRYPT 
Partition 1 Load Failed, 0x2F
```

## RSA 认证成功后 JTAG 无法使用，无法用 SDK 烧录Flash

当系统从经过RSA认证的镜像启动成功，JTAG 是可以使用的，但是默认并没有打开，这导致了一些不方便，比如使用 SDK 烧录其他镜像，或者用 SDK 进行程序调试等。

通过在 FSBL 中增加 [AR 68391](https://www.xilinx.com/support/answers/68391.html) 中提示的代码，就可以打开 JTAG。一般将它放在 hook before handoff 就可以。

```C
Xil_Out32(0xffca0038,0x3F);
Xil_Out32(0xffca003C,0xFF);
Xil_Out32(0xffca0030,0x3);
Xil_Out32(0xFF5E00B0,0x01002002);
Xil_Out32(0xFF5E0240,0x0);
Xil_Out32(0xFFCA3000,0x1);
```

注意：由于这些代码修改 CSU 寄存器，需要运行在 EL3 等级，所以如果放在 U-boot 或者 Linux 代码中是不工作的。

## 用烧写 RSA Key 的程序不能烧录 Blackkey

libSKey 用于烧写 MPSoC 的 eFUSE key，有两个常用程序，分别是 `xilskey_efuseps_zynqmpe_example` 和 `xilskey_puf_registration`。后者能烧写 Blackkey 和 Helperdata。前者能烧写 RSA PPK 和 Redkey 等除了 Blackkey 以外的情况，因为它不处理 helperdata。

## 启动后什么也没显示，改怎么调试？

如果已经烧录了 RSA_EN，但是镜像认证出错，芯片会关闭大部分 JTAG 通道，XSCT 无法扫描 JTAG 上的器件，也就无法进一步debug。在这种情况下只有一个通道被留了下来协助调试，他就是 JTAG Error Status 寄存器。

### 1. 怎样读取 JTAG Error Status
Xilinx Internal Answer Record 68515 描述了这个过程。请联系 FAE 了解怎样获取 JTAG Error Status 信息。

### 2. 获取 CSU Error Code
JTAG Error Status 是一个 121 位的状态信息。根据 AR68515 找出 CSU BootROM Errors (CSU_BR_ERR.ERR_TYPE)。

### 3. 分析错误原因
UG1085 Table 11-8 和 11-9 描述了它的意义。找到对应的错误模式，再修正错误。

## 不同的安全启动模式能不能混用？

安全启动模式有很多中，比如只用 RSA，只用 AES，或同时使用 RSA 和
AES；RSA 和 AES 的情况下，还有使用 eFUSE key, BBRAM Key, Red Key, Gray Key, Black Key 等情况。

以上各种组合，在一次 POR 的情况下，只能保持使用同一种组合。POR 是 Power on Reset，相当于系统完全断电，或 `PS_POR_b` 信号拉低，不包含内部通过寄存器重启的情况。

- 由于 Multiboot 使用内部寄存器重启机制，所以如果使用 Multiboot，需要保证所有的镜像使用相同的安全启动模式。
- 如果 U-boot 需要加载一些经过 RSA 或 AES 加密的镜像，也需要保证加载的镜像与启动镜像使用同样的安全启动模式。

## 一个完备的安全启动还需要考虑哪些因素

###  怎样保证从头到尾的安全启动
本文描述的只是安全启动的最简单的几种方式。如果需要考虑其他方式，比如 Linux 文件系统需要直接存在 Flash 中，要保证黑客无法攻击、修改 Flash 上的文件保证系统安全就会更难一些。一只鸡蛋只要有一条裂缝，它就会变臭的。

### 怎样安全升级
远程系统升级是一个很常见的需求，即使没有安全启动的需求，也可能考虑远程升级和升级失败的恢复。在需要安全启动的状况下，需要保证传输通道可靠安全、传输内容不能包含私钥，用带安全启动功能的 Boot.bin 烧录到 Flash 上，做好烧录失败的恢复机制即可。

### 怎样安全生产

生产主要需要考虑两方面内容，一是生产厂商是否可以信任，哪些内容可以交给厂商，密钥泄露的风险有多大；二是生产效率，怎样以更快的速度、更方便的流程，将密钥烧入 eFUSE，将加密镜像烧如 Flash。

# 六、总结

​本文从讲解常用攻击方法入手，随后介绍了 MPSoC 如何防御以上讲到的常用攻击手段，怎样规划安全启动，加上实际的操作指南，并附上常见问题及解释。希望你看完后能了解到，堵上系统中的安全隐患是有必要的，以及在 MPSoC 上使用安全启动功能，使用成本是很低的，只需要通过一些简单的学习实验，就可以让你的系统更加安全可靠。

# 七、参考文档

1. UG1085: MPSoC Technical Reference Manual
2. UG1209: Zynq UltraScale+ MPSoC: Embedded Design Tutorial
    - Ch.5 -> Secure Boot Sequence
3. UG1137: Zynq UltraScale+ MPSoC Software Developer Guide
    - Ch.7 -> Boot Flow -> Secure Boot Flow
    - Ch.8 -> Boot Time Security
    - Ch.16 Bootgen Image Creation
    - Appx. I: XilSecure Library
    - Appx. J: XilSKey Library
4. Xapp1333: External Secure Storage Using the PUF
5. [Wiki - Authentication and decryption at u-boot](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842432/Authentication+and+decryption+at+u-boot)
6. [Wiki - Solution ZynqMP PL Programming](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841847/Solution+ZynqMP+PL+Programming)
7.  XAPP1319: Programming BBRAM and eFUSEs