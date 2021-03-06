---
title: 分布式理论-CAP理论
date: 2021-03-02 12:00:00
categories:
  - 理论
  - 分布式理论
tags:
  - CAP
author: 长歌
abbrlink: 821096212
---

{% cq %}
CAP原则又称CAP定理，指的是在一个分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）。CAP 原则指的是，这三个要素最多只能同时实现两点，不可能三者兼顾。
{% endcq %}
<!-- more -->


# CAP理论

## CAP定义

### Consistency 一致性
一致性是指“all nodes see the same data at the same time”，即更新操作成功后，所有节点在同一时间的数据完全一致。


### Availability 可用性
可用性是指“reads and writes always succeed”，即用户访问数据时，系统是否能在正常响应时间返回结果。


### Partition Tolerance 分区容错性
分区容错性是指“the system continues to operate despite arbitrary message loss or failure of part of the system”，即分布式系统在遇到某节点或网络分区故障的时候，仍然能够对外提供满足一致性和可用性的服务。



## CAP权衡
{% asset_img cap.png CAP %}
### CA without P
如果不要求 Partition Tolerance，即不允许分区，则强一致性和可用性是可以保证的。其实分区是始终存在的问题，因此 CA 的分布式系统更多的是允许分区后各子系统依然保持 CA。

### CP without A
如果不要求可用性，相当于每个请求都需要在各服务器之间强一致，而分区容错性会导致同步时间无限延长，如此 CP 也是可以保证的。很多传统的数据库分布式事务都属于这种模式。

### AP without C
如果要可用性高并允许分区，则需放弃一致性。一旦分区发生，节点之间可能会失去联系，为了实现高可用，每个节点只能用本地数据提供服务，而这样会导致全局数据的不一致性。

### 结论
通过CAP理论，我们知道无法同时满足CAP三个特性。

现代多数互联网应用场景，部署机器多且分散，节点和网络故障无法避免。要保证服务可用性达N个9的目标，必然要保证AP，舍弃C(放弃强一致性追求最终一致性)。

但当涉及到金钱相关业务时，一致性必须得到保证。这种情况下就需要放弃A。如果发生网络故障时，放弃系统的可用性也要保证数据的一致性。

两种方式各有取舍，应该根据业务场景判断，选择合适的方案。


## Reference
- [分布式系统的CAP理论-HollisChuang's Blog](https://www.hollischuang.com/archives/666)
- [mwhittaker's blog](https://mwhittaker.github.io/blog/an_illustrated_proof_of_the_cap_theorem/)