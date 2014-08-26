---
layout:    post
title:     硬件高速缓存和TLB
category:  内存寻址
description: 硬件高速缓存...
tags: 硬件高速缓存 缓存 TLB
---
RAM有动态RAM和静态RAM，静态RAM用触发器存储信息，只要不断电，信息就不会丢失，不需要刷新。但是，静态RAM集成度相对较低，功耗大。
 
动态RAM用电容存储信息，为了保持信息必须每隔1~2ms就要对高电平电容重新充电，称为刷新，因此必须含有刷新电路，在电路上较复杂，但动态RAM集成度高，且价格便宜。

RAM相对于硬盘来说，已经相当的快了。但和CPU相比，性能还是不够。当今的CPU时钟频率接近几个GHz，而动态RAM芯片的存取时间是时钟周期的数百倍，这就意味着从RAM中取操作数或向RAM存放结果这样的指令执行时，CPU可能需要等待过长的时间。

为了缩小CPU和RAM之间的速度不匹配，引入了硬件高速缓存（*hardware cache memory*）。硬件高速缓存基于局部性原理（*locality principle*）。该原理既适用程序结构也适用数据结构。这表面由于程序循环结构及相关数组可以组成线性数组，最近最常用的相邻地址在最近的将来又被用到的可能性很大。因此，引入小而快的内存来存放最近最常用的代码和数据变得很有意义。

为此，80x86体系结构中引入了一个叫行（*line*）的单位。行由几十个连续的字节组成，它们以脉冲突发模式在慢速DRAM和快速的用来实现高速缓存的片上静态RAM（SRAM）之间传送，用来实现高速缓存。

高速缓存再被细分为行的子集，在一种极端情况下，高速缓存可以是直接映射的，这时主存中的一个行总是存放在高速缓存中完全相同的位置。在另一种极端的情况下，高速缓存是充分关联的，这意味着主存中的任意一个行可以存放在高速缓存中的任意位置。但大多数高速缓存在某种成都上是N-路关联的，意味着主存中的任意一个行可以存放在高速缓存N行中的任意一行中。

高速缓存单元插在分页单元和主内存之间，它包含一个硬件高速缓存内存和一个高速缓存控制器。高速缓存内存存放内存中真正的行。高速缓存控制器存放一个表项数组，每个表项对应高速缓存内存中的一个行。每个表项有一个标签和描述高速缓存状态的几个标志[^1]。

[^1]: 这些标志由一些位组成，这些位让高速缓存控制器能够辨别由这个行当前所映射的内存单元。

{:.center}
![system](/linux-kernel-architecture/images/dram_cache.png){:style="max-width:500px"}

{:.center}
硬件高速缓存

当访问一个RAM存储单元时，CPU从物理地址中提取出子集的索引号并把自己中所有行的标签与物理地址的高几位比较，如果发现某个行标签与这个物理地址的高位相同，则CPU命中一个高速缓存。

当命中高速缓存时，高速缓存控制器进行不同的操作，具体根据存取类型相关。控制器从高速缓存中选择数据并从到CPU寄存器，省去了访问DRAM的时间。当高速缓存没有命中，高速缓存行被写回内存，如果有必要的话，把正确的行从RAM中取出放到高速缓存的表项中。实际上和web开发中常用的memcached或redis、mysql类似。

多处理系统的每一个处理器都有一个单独的硬件高速缓存，因此它们需要额唉的硬件电路用于保持高速缓存内容的同步，每个CPU都有自己的本地硬件高速缓存，但是，现在更新变得更耗时，只要一个CPU修改了它的硬件高速缓存，它就必须检查同样的数据是否包含在其他硬件高速缓存中，如果是，就必须通知其他CPU修改高速缓存中的值。通常把这种活动称作高速缓存侦听。这些实现和内核无关，都由硬件实现。

Pentium处理器高速缓存的一个特点是让操作系统把不同的高速缓存管理策略与没一个页框关联，因此，每一个页目录项和每一个页表项都包含PCD（*Page Cache Disable*）标志和PWD（*Page Write-Through*）标志。PCD指明当访问包含在这个页框的数据时，高速缓存功能必须被启用还是禁用。PWD标志指明数据写到页框时，必须使用策略是回写策略还是通写策略。

Linux清除了所有页目录项和页表项中的PCD和PWT标志，导致的结果是，对于所有的页框都启用高速缓存，对于写操作总是采用回写策略。

### 转换后援缓冲器（TLB） ###

除了通用硬件高速缓存之外，80x86处理器还包含了另一个称为转换后援缓冲期或TLB（*Translation Lookaside Buffer*）的高速缓存用于存储物理地址，从而加快了线性地址的转换。当一个地址被第一次使用时，通过慢速访问RAM中的页表计算出相应的物理地址，同时物理地址被存放在一个TLB表项中。以便以后使用。

在多处理器系统中，每个CPU都有自己的本地TLB，TLB中的对应项不必同步，因为运行在现有的CPU上的进程可以使用同一线性地址与不同的物理地址发生联系。当CPU的*cr3*控制寄存器被修改时，硬件自动使本地的TLB中的所有项都无效，这是因为新的一组页表被启用而TLB指向的是旧数据。