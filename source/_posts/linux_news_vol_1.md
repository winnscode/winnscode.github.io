---
title: 一周Linux资讯 Vol. 1
date: 2022-09-23 10:00:00
tags: [linux, kernel]
categories: 一周linux资讯
---

### 一周Linux资讯 Vol. 1

文韬
2022.9.23

#### brightness/backlight

屏幕亮度/背光控制的用户态接口正在大改：

https://lpc.events/event/16/contributions/1390/attachments/990/1916/kernel-recipes-backlight-2022-16x9.pdf

作者Hans de Goede是redhat/fedora的开发者，内核大佬，很多acpi/platform/drm相关子系统的维护者，所以预计大概率会在以后被合入（现在的接口确实有点让人迷糊……）


***
#### MGLRU

MGLRU貌似在业界大受好评，各种benchmark都表现得很不错，甚至被认为可能是2022内核的最大创新。一些发行版已经提前合入了，比如openwrt，谷歌内部的一些os（MGLRU是chromeos团队开发的）等。尤其是对于低内存设备，用户体验效果大大提高了（所以openwrt这么积极……它是给嵌入式设备用的）

其它的一些：
1. 目前已经发布了v15，看起来有机会在6.1合入。
2. 貌似有官方的backport版本？
3. MGLRU和eBPF有结合，开放了一些策略给用户空间，比如generation的赋值。（很有想象空间？）

https://www.phoronix.com/news/MGLRU-LPC-2022
https://lpc.events/event/16/contributions/1269/attachments/1024/1969/The%20Multi-gen%20LRU%20-%20LPC%202022.pdf
（双语能力好的同学可以去看LPC的视频）


***
#### vDSO-getrandom()

getrandom()接口大幅提速：（15x？）

https://www.zx2c4.com/projects/linux-rng-5.17-5.18/inside-linux-kernel-rng-presentation-sept-13-2022.pdf

1. 这是在vDSO接口里，这是syscall的一种优化路径，就是内核把一部分函数接口，作为虚拟动态共享库导出给用户空间直接使用，省去常规的上下文切换开销，常见的还有gettimeofday，一般是一些性能敏感、无安全风险的接口。

2. 这个改动里面我没细看，大概也看不太懂

3. 作者已经做了backport，4.19/5.10都有，所以可以考虑关注一下。


***
#### 128-bit zettalinux

https://lwn.net/Articles/908026/

* zettalinux：
    * 想得很远，128-bits（是否有必要？）
    * 一个原因是，bits还可以给安全功能用……（比如现在的64bits，经常分为48+16，16用来打tag……）
    * HIGHMEM之类的可以永久退役……
    * 如果真的要更改，如何更改？
    * cpu架构支持，128-bits寄存器……
    * 类型定义，尤其是long和指针之间的关系
    * 用户接口定义和兼容性问题：
    * 如何在128-bits机器上跑32/64bits kernel
    * 如何在128-bits机器/内核上 ，跑32/64bits用户程序
    * 如何推动这个问题？——现在还没有这种机器……
    * 用qemu，模拟一个这种机器吧！
    * ……

***
#### Intel-DSA


https://www.phoronix.com/news/Intel-DSA-2.0-Linux-Start
https://01.org/blogs/2019/introducing-intel-data-streaming-accelerator


dsa, data stream accelerator，数据流加速器，和之前的DMA功能类似，貌似是现在Intel高速数据io的方向，搞了好几年时间了，可能差不多上线了（大概率会先在服务器处理器上）。

我稍微瞄了一眼它的说明，大概是利用PCI-E的很多新特性，实现一个集成在处理器内部的data-move加速器，来高效处理各种数据访问/搬运工作：存储、网络、持久性内存之类的。

在实现上，应该是依赖于PCI-E的很多高级特性，最后大概是一个SVM（shared-virtual-memory）的搞法，让设备可以直接访问到各个进程的地址空间，并且不用现在的各种pin（现在用DMA+IOMMU也可以实现一些类似的功能，但貌似比较麻烦？要很小心地处理缓存一致性问题？貌似和直接使用进程的地址空间还是有不小的差别）

如果有对PCI-E体系很感兴趣的同学，大概可以关注一下这个方向，如果能把DMA/IOMMU/PCI-E/ACPI等硬件体系架构相关的内容搞搞懂，带带我们就更好了



***
#### loongarch：acpi patch


[[PATCH V4] LoongArch: Add ACPI-based generic laptop driver @ 2022-09-19  2:15 Huacai Chen](https://lore.kernel.org/linux-acpi/20220919021540.2873061-1-chenhuacai@loongson.cn/t/#u)


* loongarch：
    * （不太懂，转发）
    * drivers/platform/loongarh-laptop?


***
#### 内存管理：Memory Tiering


[Memory Tiering - Jérôme Glisse / Google](https://lpc.events/event/16/contributions/1266/attachments/1023/1968/LPC2022%20Memory%20Tiering.pdf)


作者J. Glisse, 是heterogeneous memory manage的维护者（异构内存管理）。

文章里有一个内存的层级图很好，register/cache/HBM/local-ddr/remote-ddr/CXL-ddr/nvm/over-network……（延迟、带宽、容量）

内存分层的核心是cold/hot的判定策略，以及迁移算法之类的……（当然又会涉及到一些探讨：策略是全局统一的，还是分group的？或者启发式的？或者开放给用户空间，bpf定制？）



***
#### 内存管理：maple-tree

[Introducing maple trees - lwn](https://lwn.net/Articles/845507/)

[The Maple Tree - Liam R. Howlett (LPC 2022)](https://lpc.events/event/16/contributions/1226/attachments/1116/2145/Maple_Tree.pdf)


[The Maple Tree, A Modern Data Structure for a Complex Problem - Liam R. Howlett (blog)](https://blogs.oracle.com/linux/post/the-maple-tree-a-modern-data-structure-for-a-complex-problem)


maple-tree，试图解决vma的管理问题。——vma当前使用的是rbtree+list，存在一些问题，比如顺序遍历效率、rcu-safe、全局mmap_lock等问题。

maple-tree是一种变种的btree，可以认为是range/gap btree。——vma经常要查找一段足够长的gap，如果有gap的记录，搜索起来会更高效。

maple-tree设计的核心要点是rcu-safe，这样才能尽量用rcu保护，而不是大锁。——rcu-safe我也没搞太懂，大意是rbtree旋转时会涉及多个node，而btree在调整时都是单node的。

maple-tree对cache更友好。——里面的细节比较多，可以直接看文章。（现在在内核设计个东西，不考虑cache都不能拿出手……）

maple-tree除了在vma之外，还有潜在的使用场景，比如pid管理，page-cache，extent管理……——目前内核的核心数据结构：list, hlist/buckets, btree, rbtree, radix-tree/xarray, ...


一些补充材料：



[[PATCH 00/94] Introducing the Maple Tree @ 2021-04-28 15:35 Liam Howlett](https://lore.kernel.org/lkml/20210514204440.ofwxmcdm6nrmur6m@revolver/)

[[PATCH v4 00/66] Introducing the Maple Tree @ 2021-12-01 14:29 Liam Howlett](https://lore.kernel.org/lkml/20211201142918.921493-16-Liam.Howlett@oracle.com/t/#u)


***
#### 调度：latency hints for CFS task


[Latency hints for CFS task - Vincent Guittot LPC’22](https://lpc.events/event/16/contributions/1273/attachments/975/1898/latency_LPC_sched_MC_2022.pdf)


作者Vincent Guittot是scheduler的co-maintainer，所以将来合入的可能性很大。latentcy和桌面环境还是比较相关的。

cfs的核心思路是：大家的优先级（nice值）都只是影响vruntime的计算，而在调度决策时一视同仁，简单地根据vruntime比大小。——越小的就越应该优先得到执行机会。但这可能导致不合理的切换：比如进程A刚开始执行1us，就被切换走了；或者A和B，两个人的vruntime差不多，A执行一下就比B多，所以切换到B；B执行一下又超过了A，于是又切换到A，如此循环……

所以，引入了一些参数来进行抢占限制（可以通过sysctl调整）：
* kernel.sched_child_runs_first         ： 默认为1
* kernel.sched_latency_ns               : “一轮”的时间（全部得到执行机会）
* kernel.sched_min_granularity_ns       ： 抢占时应保证的最小执行时间
* kernel.sched_wakeup_granularity_ns    ： wakup时的抢占限制
* ……


现在引入一个新的参数kernel.sched_latency，在preempt中参与判定。比如对于latency敏感的task，可以提前抢占（抢占掉当前运行的lower vruntime进程）之类的。

（注意，latency仅仅影响抢占的优先级，但不影响vruntime的计算，也不影响全局的公平性……）

文中还提到一个测试工具软件包： rc-tests，其中有hackbench（进程负载模拟/压力测试？），cyclictest（实时性测试？）之类的……



一些补充材料：

[About use of kernel parameter 'sched' - Red Hat](https://access.redhat.com/solutions/177953)

[Linux调度器何时需触发抢占？—— 从hackbench谈起 - aliyun](https://developer.aliyun.com/article/789487)


[[PATCH v2 0/7]  Add latency_nice priority @ 2022-05-12 16:35 Vincent Guittot](https://lore.kernel.org/lkml/d4467cd50d884b438dc8c2993669bed0@AcuMS.aculab.com/t/#u)



***
#### 调度：Nest Scheduling



["Nest" Is An Interesting New Take On Linux Kernel Scheduling For Better CPU Performance - phoronix](https://www.phoronix.com/news/Nest-Linux-Scheduling-Warm-Core)

[OS Scheduling with Nest: Keeping Tasks Close Together on Warm Cores - Julia Lawall LPC'22](https://lpc.events/event/16/contributions/1198/attachments/983/1909/plumbers.pdf)


尝试解决的核心问题是： task被调度到一个idle-core时，其电压频率太低，运算速度太慢。如何解决？让刚刚进入idle的core保持“warm”，同时尽可能将task调度到warm-idle-core上去。


如何实现？给core分层（nest）：primary-set/preserve-set/normal-set，根据执行情况，core的位置会动态调整。比如primary-set里有很多core，其中可能有task已经退出的，已经进入idle的，也就是“空穴”。为了保持它的warm，可以将它的idle策略实现为自旋。如果很快有新的task进来，那就可以由CFS调度到该warm-core来，直接以高频高性能运行；如果一段时间没有task，则将该core降级为perserve，同时更改其idle策略……



这应该主要是用在服务器上，因为这种策略，以及后续的实现，应该会增加功耗/发热，同时获得更好的性能表现。同时这个方案还是很有启发性的，涉及到了调度与电源管理的很多主题。


***
#### 调度：RTLA


[RTLA: what is next? - Daniel Bristot de Oliveira LPC'22](https://lpc.events/event/16/contributions/1265/attachments/1091/2093/2022_lpc_rtpa.pdf)



rtla: real-time linxu analysis。（很新，rt相关，不太懂，转发一下）




***
#### Rust相关

1. rust很有可能会在6.1进入内核，刚开始可能只是一个“hello world”玩具，但是会给社区一个信号，rust真正被接受了。

2. rust有很多相关工作正在进行，比如gccrs（rust的gcc前端）等。——这个项目比较缺人，目前还在紧张的开发周期中，如果有人感兴趣可以考虑参与一下。

3. rust与内核之间还有一些问题需要解决，比如inline、宏等。——换而言之，要想完全利用起来rust的安全特性，和内核无缝衔接，还有一些重要的问题需要解决。

4. 整个社区中有很多人在关注，很多人在做前置开发，比如rust-nvme驱动，rust-9p驱动等等。

5. 还有一个有趣的项目叫做Aya： Rust in kernel via eBPF，很炫酷……



[Rust for Linux Status Update - Miguel Ojeda LPC'22](https://lpc.events/event/16/contributions/1256/attachments/1048/2003/Rust%20Status%20-%20LPC%202022.pdf)

(Miguel Ojeda是Rust in kernel的维护者，这个pdf里讲了rust当前的开发状态。——我没细看……)


[Linux (PCI) NVMe driver in Rust - Andreas Hindborg (Western Diginal) LPC'22](https://lpc.events/event/16/contributions/1180/attachments/1017/1961/deck.pdf)


[gccrs - Philip Herron & David Faust LPC'22](https://lpc.events/event/16/contributions/1215/attachments/1032/1980/LPC2022-gccrs.pdf)


[Rust in the Kernel via eBPF - Dave & Michal LPC'22)](https://lpc.events/event/16/contributions/1182/attachments/997/1924/Rust%20in%20the%20Kernel.pdf)


[Async Rust and 9p server - Wedson Almeida Filho](https://kangrejos.com/Async%20Rust%20and%209p%20server.pdf)


[rustc_codegen_gcc: A gcc codegen for the Rust compiler - Antoni Boucher](https://kangrejos.com/rustc_codegen_gcc:%20A%20gcc%20codegen%20for%20the%20Rust%20compiler.pdf)


***
#### 安卓存储方案相关


最近比较关注安卓的存储方案，尤其是fuse-passthrough，以及virtual-A/B方案。

android的数据访问高度依赖于fuse，因为它要通过fuse做动态的文件访问权限管控。所以引入了fuse直通，让内核可以直接基于某个本地文件系统的fd而完成数据传输，以避免数据拷贝和系统调用开销。当前的开发点是引入ebpf，来帮助内核做策略选择。

virtual-A/B是系统存储/OTA更新方案，是从A/B,non-A/B方案演化过来的，内部使用了dm-snapshot，dm-user，dm-linear等各种内核组件。当前的讨论点是尝试使用io_uring-based ublk，替代掉之前google自己维护的dm-user。



fuse:
[FS stacking with FUSE: performance issues and mitigations - Alessio Balsini & Paul Lawrence LPC'21](https://lpc.events/event/11/contributions/1048/attachments/838/1662/2021%20LPC_%20FS%20stacking%20with%20FUSE_%20performance%20issues%20and%20mitigations.pdf)

[fuse-bpf - Daniel Rosenberg & Paul Lawrence LPC'22](https://lpc.events/event/16/contributions/1339/attachments/945/1861/LPC2022%20Fuse-bpf.pdf)

[FUSE Passthrough - Android](https://source.android.com/docs/core/storage/fuse-passthrough)

virtual-A/B:
[dm-snapshot in userspace - Akilesh Kailash & David Anderson LPC'21](https://lpc.events/event/11/contributions/1049/attachments/826/1562/2021%20LPC_%20dm-snapshot%20in%20user%20space.pdf)

[io_uring in Android OTA - Akilesh Kailash LPC'22](https://lpc.events/event/16/contributions/1331/attachments/951/1867/LPC2022%20-%20io_uring%20in%20Android%20OTA.pdf)

[Virtual A/B Overview - Andriod](https://source.android.com/docs/core/ota/virtual_ab)


***