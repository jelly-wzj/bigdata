---
layout: post
title:  "flink checkpoint 原理与应用"
date:   2022-05-01 11:22
categories: bigdata
permalink: /archivers/bigdata-flink-checkpoint-02
---



> Checkpoint 与 state 的关系

Checkpoint 是从 source 触发到下游所有节点完成的一次全局操作。下图可以有一个对 Checkpoint 的直观感受，红框里面可以看到一共触发了 569K 次 Checkpoint，然后全部都成功完成，没有 fail 的。

![img](https://pics0.baidu.com/feed/b8014a90f603738d97c200c3a2d1ce54fa19ec92.jpeg?token=d4bc9c598f6f7cd087674a4e590f1976&s=3AAA782291B85C214A7400D4000050B3)

state 其实就是 Checkpoint 所做的主要持久化备份的主要数据，看下图的具体数据统计，其 state 也就 9kb 大小 。

![img](https://pics2.baidu.com/feed/b3b7d0a20cf431addf09bb7f58fcd2aa2cdd98e8.jpeg?token=c34f7d104ce3c48923c9ce5eaead598c&s=5AA834628BA84C0310C40DCB000050B2)

> 什么是 state

我们接下来看什么是 state。先看一个非常经典的 word count 代码，这段代码会去监控本地的 9000 端口的数据并对网络端口输入进行词频统计，我们本地行动 netcat，然后在终端输入 hello world，执行程序会输出什么？

![img](https://pics0.baidu.com/feed/0b7b02087bf40ad15f74a50945e66fdaabecce93.jpeg?token=616c4eab9efb1ad6b0bad5672cb1dea7&s=B2D015CACBE49F704EC524030000E0C0)

答案很明显，

(hello, 1)

和

(word,1)

那么问题来了，如果再次在终端输入 hello world，程序会输入什么？

答案其实也很明显，

(hello, 2)

和

(world, 2)

。为什么 Flink 知道之前已经处理过一次 hello world，这就是 state 发挥作用了，这里是被称为 keyed state 存储了之前需要统计的数据，所以帮助 Flink 知道 hello 和 world 分别出现过一次。

回顾一下刚才这段 word count 代码。keyby 接口的调用会创建 keyed stream 对 key 进行划分，这是使用 keyed state 的前提。在此之后，sum 方法会调用内置的 StreamGroupedReduce 实现。

![img](https://pics1.baidu.com/feed/f703738da977391241ef3605ead3f81d347ae2dd.jpeg?token=7e8a5b8cf5ce7a66631a6acb9090cfd6&s=86ACF5028CF81F8C76B7F14E0000C0F1)

> 什么是 keyed state

对于 keyed state，有两个特点：

只能应用于 KeyedStream 的函数与操作中，例如 Keyed UDF, window statekeyed state 是已经分区/划分好的，每一个 key 只能属于某一个 keyed state对于如何理解已经分区的概念，我们需要看一下 keyby 的语义，大家可以看到下图左边有三个并发，右边也是三个并发，左边的词进来之后，通过 keyby 会进行相应的分发。例如对于 hello word，hello 这个词通过 hash 运算永远只会到右下方并发的 task 上面去。

![img](https://pics2.baidu.com/feed/f636afc379310a55ca4d1b50a48f3dac8326107c.jpeg?token=6e4db6a0f1f90416fa4868c1755b7bb4&s=823651829CB80B8E7E9E354F0300D0B0)

> 什么是operator state

又称为 non-keyed state，每一个 operator state 都仅与一个 operator 的实例绑定。常见的 operator state 是 source state，例如记录当前 source 的 offset再看一段使用 operator state 的 word count 代码：

![img](https://pics3.baidu.com/feed/dc54564e9258d1095b83b083c292b2ba6d814dea.jpeg?token=e42e87a6b1d29b834c16ab6efc66a763&s=82BC51821FE81E05738465090100B0C1)

这里的

fromElements

会调用

FromElementsFunction

的类，其中就使用了类型为 list state 的 operator state。根据 state 类型做一个分类如下图：

![img](https://pics4.baidu.com/feed/2934349b033b5bb519ebe1462219ab3cb700bc9e.jpeg?token=ee9ee15daa9b6b767f9c0e53b313ddc3&s=8BE6FC121D30708A5766C9C80200E0B2)

除了从这种分类的角度，还有一种分类的角度是从 Flink 是否直接接管：

Managed State：由 Flink 管理的 state，刚才举例的所有 state 均是 managed stateRaw State：Flink 仅提供 stream 可以进行存储数据，对 Flink 而言 raw state 只是一些 bytes在实际生产中，都只推荐使用 managed state，本文将围绕该话题进行讨论。

> 如何在 Flink 中使用 state

下图就前文 word count 的 sum 所使用的

StreamGroupedReduce

类为例讲解了如何在代码中使用 keyed state：

![img](https://pics3.baidu.com/feed/b3fb43166d224f4a8ac3b84b1a3dee579922d164.jpeg?token=05611cf43294f67cc3155d52ab937625&s=BA9403CA13F287CE0471E41F020010C1)

下图则对 word count 示例中的

FromElementsFunction

类进行详解并分享如何在代码中使用 operator state：

![img](https://pics7.baidu.com/feed/a9d3fd1f4134970a084ec5aa8a00afcda6865d41.jpeg?token=2a4c8608cf656f62eec5d6cc446b5af8&s=9A8401C253BAB1CA0461201E0200C0C3)

> Checkpoint 的执行机制

在介绍 Checkpoint 的执行机制前，我们需要了解一下 state 的存储，因为 state 是 Checkpoint 进行持久化备份的主要角色。

> Statebackend 的分类

下图阐释了目前 Flink 内置的三类 state backend，其中

MemoryStateBackend

和

FsStateBackend

在运行时都是存储在 java heap 中的，只有在执行 Checkpoint 时，

FsStateBackend

才会将数据以文件格式持久化到远程存储上。而

RocksDBStateBackend

则借用了 RocksDB（内存磁盘混合的 LSM DB）对 state 进行存储。

![img](https://pics4.baidu.com/feed/4bed2e738bd4b31c12d54cda951c597a9f2ff832.jpeg?token=c763827e53f3d3f554c7454159a1c056&s=29BAEC1312E8450126615A640300A074)

对于

HeapKeyedStateBackend

，有两种实现：

支持异步 Checkpoint（默认）：存储格式 CopyOnWriteStateMap仅支持同步 Checkpoint：存储格式 NestedStateMap特别在 MemoryStateBackend 内使用

HeapKeyedStateBackend

时，Checkpoint 序列化数据阶段默认有最大 5 MB数据的限制

对于

RocksDBKeyedStateBackend

，每个 state 都存储在一个单独的 column family 内，其中 keyGroup，Key 和 Namespace 进行序列化存储在 DB 作为 key。

![img](https://pics3.baidu.com/feed/aa64034f78f0f736e1506038189fcd1ceac413af.jpeg?token=2d5a412a72dfa5b438d215aad4ea11a2&s=4C90CC12A4B0798256580CC9030090BD)

> Checkpoint 执行机制详解

本小节将对 Checkpoint 的执行流程逐步拆解进行讲解，下图左侧是 Checkpoint Coordinator，是整个 Checkpoint 的发起者，中间是由两个 source，一个 sink 组成的 Flink 作业，最右侧的是持久化存储，在大部分用户场景中对应 HDFS。

第一步，Checkpoint Coordinator 向所有 source 节点 trigger Checkpoint；。

![img](https://pics5.baidu.com/feed/cf1b9d16fdfaaf51ec5e55cf9e9eeaebf11f7a54.jpeg?token=bf313b1c194d4d7897fcf02959f0a11a&s=3294E52285B64C331CFDECEA02001032)

第二步，source 节点向下游广播 barrier，这个 barrier 就是实现 Chandy-Lamport 分布式快照算法的核心，下游的 task 只有收到所有 input 的 barrier 才会执行相应的 Checkpoint。

![img](https://pics4.baidu.com/feed/5fdf8db1cb13495475e348a64784ec5dd3094a46.jpeg?token=1818b000e4fce1678f2924a9050aee81&s=329465229F2048131CDDECEA02005032)

第三步，当 task 完成 state 备份后，会将备份数据的地址（state handle）通知给 Checkpoint coordinator。

![img](https://pics2.baidu.com/feed/060828381f30e9244aa730065ec210031c95f7d2.jpeg?token=beea289feafcb53b4f7cb2995c16ae45&s=139C6522DF626C035C5DECEA0000A032)

第四步，下游的 sink 节点收集齐上游两个 input 的 barrier 之后，会执行本地快照，这里特地展示了 RocksDB incremental Checkpoint 的流程，首先 RocksDB 会全量刷数据到磁盘上（红色大三角表示），然后 Flink 框架会从中选择没有上传的文件进行持久化备份（紫色小三角）。

![img](https://pics4.baidu.com/feed/bf096b63f6246b60c43a772af9326449500fa28c.jpeg?token=5947ca442badcadaffe03522c02d98e0&s=32B47522DFB46C035C5DAC6A02007032)

同样的，sink 节点在完成自己的 Checkpoint 之后，会将 state handle 返回通知 Coordinator。

![img](https://pics1.baidu.com/feed/3b87e950352ac65cbfe87318e838cc1492138a3f.jpeg?token=5c35d6eef48600094c24e0dbdf60fd9f&s=33B47522C7F64C235C5DEC6A0200F032)

最后，当 Checkpoint coordinator 收集齐所有 task 的 state handle，就认为这一次的 Checkpoint 全局完成了，向持久化存储中再备份一个 Checkpoint meta 文件。

![img](https://pics0.baidu.com/feed/7acb0a46f21fbe09292882ea7aaa72368644ad2d.jpeg?token=b20c471f745e5f93ca1777c3bacd292c&s=329475228D364C111C5DEC6A02007032)

> Checkpoint 的 EXACTLY_ONCE 语义

为了实现 EXACTLY ONCE 语义，Flink 通过一个 input buffer 将在对齐阶段收到的数据缓存起来，等对齐完成之后再进行处理。而对于 AT LEAST ONCE 语义，无需缓存收集到的数据，会对后续直接处理，所以导致 restore 时，数据可能会被多次处理。下图是官网文档里面就 Checkpoint align 的示意图：

![img](https://pics0.baidu.com/feed/8ad4b31c8701a18b99a340828fe5790d2a38fe86.jpeg?token=1fa4250d6440e06c0989e5f8d5501592&s=1136CC324914CC13087444C40000F032)

需要特别注意的是，Flink 的 Checkpoint 机制只能保证 Flink 的计算过程可以做到 EXACTLY ONCE，端到端的 EXACTLY ONCE 需要 source 和 sink 支持。

Savepoint 与 Checkpoint 的区别

作业恢复时，二者均可以使用，主要区别如下：

![img](https://pics1.baidu.com/feed/83025aafa40f4bfbed50ec63148506f5f6361857.png?token=2884a7dda32f2177143cc62cb111c5db&s=1AAA70238DE44C011CFD90CE0300E0B1)

转载于：唐云(茶干)