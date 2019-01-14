---
layout: post
title: PetaLinux工具系列(2) - 看 BitBake 怎么烧菜
categories: 
  - Experiences
tags:
  - PetaLinux
  - BitBake
---

坊间流言，在 PetaLinux（嗯，我们的餐厅） 中的 BitBake 是一个好厨子。每次烧完菜，厨房已经清理得干干静静，以至于当做好的菜送到我手里，我们再跑去厨房看，简直就觉得这里不曾烧过菜，灶台一尘不染，垃圾桶也被清理得干干净净，可能唯一留下的线索，就是之前我点过的菜单。

作为想偷师厨艺的我，跑去餐厅点这道菜，点完去厨房偷看，一眨眼的功夫，菜烧完灶台就被收拾干净了，再点一遍，再看一遍，还没学会个零头呢，这师傅又已经整理好了，让我甚是苦恼。于是就寻思着怎么能偷偷在厨房装一个摄像头，把 BitBake 的一举一动全都录下来，后来发现原来厨房里本来就又摄像头，大多数细节都能找回来。只是有时候我还想亲自观摩一下厨师切配的菜，只看录像不太够，溜进厨房时碰见他的学徒正好要收拾灶台倒垃圾，我赶紧过去送上一支烟，来来我帮你，你去旁边休息一下。得到这些垃圾就如获至宝，赶紧翻起垃圾来。通过不断地反复看大师的烧菜录像，以及翻看垃圾桶，我终于理解了大师的厨艺精妙之道，逐渐也走向通往大师之路。

现在，我向你透露一下我的偷师过程吧。

首先，我得走进餐厅
```bash
// 配置 PetaLinux 环境
source <petalinux install dir>/settings64.sh
```

点一道我感兴趣的菜（菜谱列表上一次已经讲过怎么看了，不知道的话点[这里](petalinux-recipe-files)）。比如这里我想看 `kernel-module-vcu` 这道菜是怎么烧的。而且除了简单的餐厅默认菜谱，我还想告诉大厨为了私人定制这道菜，于是我把 AR 71798 中的额外菜谱也拿了出来，通过放到 `<petalinux project>/project-spec/meta-user` 中。

一切就绪，下单烧菜。`petalinux-build -c kernel-module-vcu`

不愧为 BitBake 厨师，分分钟就帮我把这道菜烧好了。顾不上享用，我赶紧去看摄像头有没有把烧菜的过程都录下来。

先溜到厨房。厨房一般在 `<petalinux project>/build/work`。但根据每家餐厅的情况有可能稍有不同，就比如一家 PetaLinux 餐厅如果想开在 NFS 分区上，但 PetaLinux 餐厅的规划主管又觉得在 NFS 里煤气太小，烧菜太慢，他们就觉得情愿在能开大火的中央厨房里烧菜，烧完再送过去。不管怎么样，`<petalinux project>/project-spec/configs/config` 这个文件中 `CONFIG_TMP_DIR_LOCATION` 记录了厨房的地址，我们跟着地址去一探究竟就可以了。`cd` 到这个目录。

找一下这道菜时在哪个灶台做的：`find . -name "kernel-module-vcu"`。走到灶台附近一探究竟

```
cd zynqmp-xilinx-linux/kernel-module-vcu
ls
```

原来厨房是必须配备摄像头的，摄像头的记录都存在 `temp` 目录中，分为 log.xx 和 run.xx 文件。 run.xx 是每个步骤的操作；log.xx 还会记录炉灶的温度等当时的环境，要学做菜，其实看这些录像就能学个大概。买洗切配，每个步骤都有单独的文件记录，文件比较多，顺序记录在 `log.task_order` 中。

看一眼烧菜的主过程吧，在 log.do_compile 里
```
make -j 56 KERNEL_SRC=/tmp/petalinux_2018.2_vcu_zcu106_v2_gstpatch_20190102-2019.01.02-05.29.31/work-shared/plnx-zynqmp/kernel-source O=/tmp/petalinux_2018.2_vcu_zcu106_v2_gstpatch_20190102-2019.01.02-05.29.31/work-shared/plnx-zynqmp/kernel-build-artifacts KERNEL_PATH=/tmp/petalinux_2018.2_vcu_zcu106_v2_gstpatch_20190102-2019.01.02-05.29.31/work-shared/plnx-zynqmp/kernel-source KERNEL_VERSION=4.14.0-xilinx-v2018.2 CC=aarch64-xilinx-linux-gcc   -fuse-ld=bfd LD=aarch64-xilinx-linux-ld.bfd   AR=aarch64-xilinx-linux-ar  O=/tmp/petalinux_2018.2_vcu_zcu106_v2_gstpatch_20190102-2019.01.02-05.29.31/work-shared/plnx-zynqmp/kernel-build-artifacts KBUILD_EXTRA_SYMBOLS=
```

可以看出来，编译这个 kernel module，主要是用 make 命令，指定 KERNEL 位置等一系列参数。

只是看录像还是不够满足，因为他们还是不够精细，比如我要看看厨师是不是按照我的定制菜单来烧菜，检查一下他的配料吧。于是我准备实施我的 Plan B，调走收拾垃圾的学徒，亲自上前查看。

- 编辑 `<petalinux project>/project-spec/meta-user/conf/petalinuxbsp.conf`
- 增加一行 `RM_WORK_EXCLUDE += " kernel-module-vcu`
  - 注意引号中的内容以空格开头

打点好了学徒，就可以假模假样地再下一次单让厨师做菜。但是因为经常后厨先把菜做好放冰箱里，有人点单他们热一下就拿出来上菜，根本不经过厨师现场烧制，为了让厨师再亲自做一次，我得把冰箱里的这道菜偷偷扔掉，然后再让厨师亲手烧制（这客户太难伺候了）。

```bash
petalinux-build -c kernel-module-vcu -x distclean
petalinux-build -c kernel-module-vcu
```

再次走进厨房，果然做菜留下的配料什么的都留下来了。`git`里是买来的菜，patch 文件是额外添加的配料，全都在这里，果然没偷懒。

