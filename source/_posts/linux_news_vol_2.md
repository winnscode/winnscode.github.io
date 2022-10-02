---
title: 一周Linux资讯 Vol. 2
date: 2022-09-29 10:00:00
tags: [linux, 一周linux资讯]
categories: blog
---

### 一周Linux资讯 Vol. 2

文韬
2022.9.29

#### linux动向

1. debian12发布了测试版本，命名为bookworm，当前内核版本为5.19。
2. linux v6.0已经到了rc7，预计很快就会正式发布，然后v6.1的merge-window就会开放。
3. MGLRU已经进入mm-stable了，预计会在v6.1合入。
4. rust很有可能也会在v6.1进入内核，现在已经有很多人在做前置的驱动开发，比如最近有人用rust写了一个mac显示驱动，并完成了最基础的渲染……


***
#### CHERI

CHERI： capability hardware enhanced RISC instructions

剑桥和arm等一起搞的一种技术，目的是解决内存安全问题：给内存访问/指针加上“capability”，而不仅仅是一个地址。——类似于原来的addr是data，另外加上了metadata，描述其capability。（具体原理我没细看，感兴趣的朋友可以仔细研究研究）

基于这种技术，就可以搞出一个层次关系： 系统刚启动时是root，拥有所有内存的访问权；在其之下可以进行capability的分配，一层一层嵌套，最终实现内存的隔离与保护。——最终这种保护是落到cpu硬件支持的，所以更可靠。

最终就是形成了一个树：子节点不能突破父节点的域的限制，但是可以自我限制。

目前有一个实验性的架构，arm的Morello。

（毫无疑问，这种探索还是为了解决C/C++的指针问题……）

LWN的介绍：
[Supporting CHERI capabilities in GCC and glibc - lwn](https://lwn.net/Articles/909265/)

官方PAPER（TL;DR）：
[An Introduction to CHERI](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-941.pdf)



***
#### KASAN/fuzzing

首先，最近有LSS（Linux Security Summit） 2022： [LSS 2022](https://events.linuxfoundation.org/linux-security-summit-europe/)，这是Andrey Konovalov(内核KASAN的核心开发者，reviewer)连续第三年在LSS上讨论KASAN了……（可以组成一个完整的系列哈哈哈）


KASAN，最初是用户空间的Address Sanitizer，Kernel版本就是KASAN，是内核dynamic bug detectors的其中一个，主要是检测内存越界访问，use-after-free，double-free等问题。

KASAN的基本原来不展开讨论了，可以参考内核文档。从基本机制而言，可以有几个部分：
1. 通过config开启
2. 需要编译器支持（涉及到堆栈操作时的red-zone setup）
3. shadow memory（1/8的内存用来存放trace数据，所以不适合布置在生产环境）
4. hook（比如alloc/free路径之类的）
5. 支持部分allocator（比如slab之类的，暂时不支持per-cpu）


KASAN有两种常用场景：
1. 特定问题排查（比如已经发现了内存double-free问题，开启KASAN检测）
2. KASAN+fuzzing    ： 用fuzzing来发现潜藏的内存使用问题（核心使用方式）


tag-based：shadow-memory based方案要占用大量内存，现在有新的实现方式：tag-based，其中又能分为software-tag和hardware-tag。貌似需要硬件支持，目前好像只有arm支持（arm有个MTE，arm memory tagging extensions），目的可能是之后能直接应用在生产环境。（我没细看，感兴趣的朋友可以自行深入了解一下；同时安全/虚拟化等都有这样的趋势：从软件实现到硬件实现，以提高性能/特性）


一些其他的Kernel Dynamic Bug Detector介绍：
1. KMSAN，Kernel Memory Sanitizer，核心是用来检测未初始化内存的使用（这会有数据泄露风险，和安全问题高度相关）
2. KCSAN，Kernel Concurrency Sanitizer，并发检测，也就是并发/加锁的检测。
3. KFENCE，好像和KASAN的目的差不多，但目标是能布置在生产环境，其原理好像是概率/采样……



lwn介绍文章：
[Finding bugs with sanitizers - lwn](https://lwn.net/Articles/909245/)


LSS系列：
[2020, Android Security Symposium: Memory Tagging for the Kernel: Tag-Based KASAN - Andrey Konovalov](https://docs.google.com/presentation/d/10V_msbtEap9dNerKvTrRAzvfzYdrQFC8e2NYHCZYJDE/edit#slide=id.g1925acbbf3_0_0)

[2021, Linux Security Summit: Mitigating Linux kernel memory corruptions with Arm Memory Tagging - Andrey Konovalov](https://docs.google.com/presentation/d/1IpICtHR1T3oHka858cx1dSNRu2XcT79-RCRPgzCuiRk/edit#slide=id.gee201659dd_0_97)

[2022, LSS Europe: Sanitizing the Linux kernel - Andrey Konovalov](https://docs.google.com/presentation/d/1qA8fqRDHKX_WM_ZdDN37EQQZwSTNJ4FFws82tbUSKxY/edit#slide=id.g1523988ae10_0_16)


内核文档：
[KASAN - kernel doc](https://www.kernel.org/doc/html/latest/dev-tools/kasan.html#software-tag-based-kasan)



***
#### flexiable arrays

这是GNU Tools上的一个讨论，因为和gcc/clang相关。作者是Gustavo A. R. Silva, Google工程师，KSPP项目（Kernel Self Protection Project）

（我不太了解编译器细节，所以瞎聊一下，大家凑合着看个大概）

flex-arrays，我们在内核代码中可能看到过：
```
struct blob_holder {
    ...
    size_t count;
    unsigned char block[];
}
```

在使用上，有三种形式（历史包袱）：
1. \[1] : 单元素表示，典型的hack，计算外部struct的sizeof时会算一个元素进去
2. \[0] : 0元素表示，GNU extension（非c语言标准）,必须放在尾部，但编译器不能检测其使用是否正确……
3. \[]  : 以后的规范使用方式


在0元素方式中，一种典型的容易出错的场景是，不小心将包含了trailing-array的struct，嵌套到另外一个结构体内：
```
struct foo {
    ...
    struct blob_holder;
    ...
}
```

而在编译器中添加真正的flex-arrays支持之前，这种错误编译器是无法检测的。（可以认为在0元素hack中，编译器直接忽略了flex-arrays，算sizeof、layout的时候都不考虑它。）


编译器选项：
gcc13/clang16: -fstrict-flex-arrays\[=n] (-fsfa)
（clang社区暂时表示拒绝，因为这是非c语言标准……emmm）


[Safer flexible arrays for the kernel - lwn](https://lwn.net/Articles/908817/)


***
#### intel： codeplay & oneAPI


intel的oneAPI项目已经做了好几年了，2022年6月收购了CodePlay，助力这个项目。最近宣布由CodePlay监督（oversee）该项目。（oneAPI还挺有意思的，所以简单聊一聊）

oneAPI的目的是为了在各种硬件架构之上，提供统一的编程框架，最后的目标是：“No transistor left behind”……

比如Intel的硬件架构大概可以分为SVMS四大阵营：
* 标量Scalar： CPU
* 矢量Vector： GPU
* 矩阵Matrix： AI
* 空间Special： FPGA


从功能上，有点类似于HLS（High level Synthesis，高层综合），能将高层语言转换为硬件语言，比如从c++到FPGA能读懂的RTL级别的语言。但是HLS更多的是面向硬件工程师的，做纯软的人很难用，而oneAPI更多的是面向软件工程师/算法工程师的，能更好地遮蔽硬件相关的细节。

（oneAPI应该是Intel花大力气的一个项目，如果能做出来，做起来生态，那还是很有想象空间的）

https://www.phoronix.com/news/Intel-oneAPI-Innovation-2022


***
#### 文件系统相关


**acl**

https://lore.kernel.org/linux-fsdevel/20220928160843.382601-1-brauner@kernel.org/T/#t


acl上最近有一个大的改动：Christian Brauner（idmapped mounts等子系统的维护者）最近发布了一个很大的patchset，在vfs中增加了posix-acl相关的api。——之前的实现是基于xattr的hack，有很多问题，这次是打算完全重构了。


简单介绍一下acl（我也不太懂，没深入研究过）： access control list，就是在常规的自主访问控制之外，可以基于文件/目录，手动构建一些访问控制属性。



**FUSE BPF**

https://lore.kernel.org/linux-fsdevel/YzQ+ke3JIx69Plld@bfoster/T/#t

上周曾提过的fuse-passthrough/bpf的后续，在LPC讨论之后，google工程师这周将其patch推到了上游。——这是一个超大的patch，超过一万行，目的是将fuse扩展为支持“stacked filesystem”。

（我还没来得及看这个patchset，可能是我后面一段时间的重点工作……）




**io_uring/async-buffered-write**

最近io_uring的团队（facebook）给xfs和btrfs都增加了async buffered-write：原来的buffered-write在io_uring中都是直接走slow-path，现在会先尝试fast-path（也就是NOWAIT模式下，可以成功写入page-cache）。

在某些workload下，能显著提高io_uring下buffered-write的效率。——slow-path下需要好几次的上下文切换（底层原理就是将其交给io-worker去submit），而fast-path没有任何的上下文切换。


xfs：
https://lore.kernel.org/io-uring/20220601210141.3773402-1-shr@fb.com/

btrfs：
https://lore.kernel.org/io-uring/CAL3q7H6AH+1Uc08y3XtgwN5nngoQGyxHLq18jaa+y+QqpK=B5g@mail.gmail.com/T/#t


***
#### net

最近看了一个非常好的net系列博客文章，推荐给大家：（网络和存储在架构上非常相似……）


[Linux网络 - 数据包的接收过程 - public0821](https://segmentfault.com/a/1190000008836467)

[Linux网络 - 数据包的发送过程 - public0821](https://segmentfault.com/a/1190000008926093)

[Linux虚拟网络设备之veth - public0821](https://segmentfault.com/a/1190000009251098)

[Linux虚拟网络设备之tun/tap - public0821](https://segmentfault.com/a/1190000009249039)

[Linux虚拟网络设备之bridge - public0821](https://segmentfault.com/a/1190000009491002)


***
#### libvirt

我最近尝试了一下用libvirt（virt-manager / virsh）管理虚拟机，挺好用的。（尤其是virsh的命令行交互非常舒适）

关于其fs直通和gdb调试可以参考单独的文档。


fs直通也可以用nfs： 用9p做直通时，因为write是passthrough，所以效率很低（在虚拟机里编译内核简直是一种折磨）；我换成nfs之后就快多了，搭配上ccache，再也不用在host里cross-compile了……

（如果有人感兴趣的话我可以在之后简单介绍一下我习惯的开发环境搭建……）


***