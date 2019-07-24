---
layout: post
title: 【翻译】一致性哈希：算法权衡
date: 2019-07-23
author: Nicholas Huang
header-img:
categories: DistributedSystem
tags:
    - DistributedSystem
    - ConsistentHash
---

# 【翻译】一致性哈希：算法权衡
[原文](https://medium.com/@dgryski/consistent-hashing-algorithmic-tradeoffs-ef6b8e2fcae8)

译者自诉：
第一次翻译这种技术博客，感觉还是和自己看不一样。自己看只需要了解大意就行了，但是翻译得讲”信达雅“，我感觉我这次一个都没沾边。最开始的动机就是学习，后面才发现dgryski是大神。。。
翻译的不好，欢迎各位批评指正，鞠躬感谢！

[赠人玫瑰，手留余香](https://www.buymeacoffee.com/dgryski?source=post_page)
遇到一件事，我有一个kv集合，还有一些用于存储kv的服务器——也许是memcached、Redis、MySQL或者其他。现在想把这些keys分布到各个服务器上，方便我再次找到它们，但是我又不想维护一个全局字典。

一种解决办法是对N取模的哈希。

首先，选择一个哈希函数将key（字符串）映射成一个整数。这个函数要快，因此就排除了加密类型的函数——比如[SHA-1](https://en.wikipedia.org/wiki/SHA-1?source=post_page)或者[MD5](https://en.wikipedia.org/wiki/MD5?source=post_page)，他们的结果确实分布均匀，但是计算代价太高——有更加低代价的选项，比如[MurmurHash](https://en.wikipedia.org/wiki/MurmurHash?source=post_page)就可以，现在还有稍微好一些的选择。非加密的哈希函数，比如[xxHash](https://github.com/cespare/xxhash?source=post_page)，[MetroHash](https://github.com/dgryski/go-metro?source=post_page)或者[SipHash1-3](https://github.com/dgryski/go-sip13?source=post_page)都是不错的替代品。

如果有N个服务器，用哈希函数处理key，然后再对N取模：
`server := serverList[hash(key)%N]`

这一步优点很多。易懂。它的计算代价很低，取模计算比哈希计算要更简单。如果N是2的幂次方，那么只用取低位的bit就行（这是对一组所或其他内存数据结构进行分片的好方法）。

但是它的缺点呢？最重要的事情是如果你更改了服务器的数量，基于每个key都要映射到其他地方。这太糟糕了！

让我们想想一个“优化”的函数能在这里做什么。

>当增加或者移除服务器，只有1/n的keys需要移动
>不移动任何不需要移动的keys

关于第一点的解释，如果要从9台服务器增加到10台，那么新的服务器要有全部keys的1/10，并且这些keys都是从之前的9台中平等地选出来的。这些keys只能移到新的服务器，而不是在两台之前的服务器中。相似的，如果需要移除机器（比如机器崩了），该机器的keys需要平等的分布到剩余的机器中。

幸运的是，有一篇论文解决了这些问题。在1997年，论文[Consistent Hashing and Random Trees:Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web](https://www.akamai.com/es/es/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf?source=post_page)发表了。这篇论文描述了Akamai在他们的分布式内容分发网络中使用的方法。

等到2007年，这些观点才进入大众视野。那年有两个工作发表了：
    
    1.last.fm的[Ketama memcached client](https://www.last.fm/user/RJ/journal/2007/04/10/rz_libketama_-_a_consistent_hashing_algo_for_memcache_clients?source=post_page)
    2.[Dynamo:Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf?source=post_page)
    
这些巩固了一致性哈希作为标准伸缩性技术的地位。现在，它被用在Cassandra、Riak和其他每个需要处理服务器之间的负载分发的分布式系统。

这个算法就是常用的基于环的一致性哈希。你也许已经看过了“带点的圆圈”的图片。当你搜索[an image search for "consistent hashing"](https://www.google.com/search?q=consistent%20hashing&tbm=isch&source=post_page)的时候，你会得到：
    ![img](https://miro.medium.com/max/1400/1*zPtGTCNOwu1p3kzn_sZFVQ.png)    
你可以认为圆上分布了[0...2^32-1]范围内的所有整数。它的基本思路是，每个服务器被哈希函数映射到圆上一个点。为了寻找一个特定key所对应的服务器，先对key做哈希并且在圆上找到一个点，然后从那个点向前扫描直到你发现第一个服务器的哈希值。    

实际上，每个服务器会在圆上出现多次，那些额外出现的点就是“虚拟节点”。虚拟节点减少了服务器之间的负载不等的问题。如果虚拟节点较少，不同服务器分配的keys的数量会差距很大。

（在术语上做一个简短的提示，原始的一致性哈希算法论文把服务器叫做“节点”。后来很多论文也用“节点”、“服务器”和“分片”。本文会交替使用这三个名称。）

另一个优势是，基于环的哈希是前向的。这里有一个参考[groupcache](https://github.com/golang/groupcache?source=post_page)的简单实现（为了描述清晰，稍微改了下）：

在哈希环上增加每个节点都要使用不同的名字（比如 `0 node1, 1 node1, 2 node1`等）哈希`m.replicas`次。节点的哈希值添加到`m.nodes`，并和节点建立映射后存储到`m.hashMap`中。最后对`m.nodes`数组排序方便在查询中使用二分查找。

```
    func (m *Map) Add(nodes ...string) {
        for _, n := range nodes {
            for i := 0; i < m.replicas; i++ {
                hash := int(m.hash([]byte(strconv.Itoa(i) + " " + n)))
                m.nodes = append(m.nodes, hash)
                m.hashMap[hash] = n
            }
        }
        sort.Ints(m.nodes)
    }
```
为了查找已知的key在哪个节点上存储，需要将它哈希成一个整数。在已排序的`m.nodes`数组中查找比这个key的哈希值大的最小节点（一种特殊的情况是，我们需要考虑环的起点）。然后在map中查找节点的哈希值以找到节点信息。

```
    func (m* Map) Get(key string) string {
        hash := int(m.hash([]byte(key)))
        idx := sort.Search(len(m.keys),
            func(i int) bool {return m.keys[i] >= hash}
        )
        if(idx == len(m.keys) {
            idx = 0
        }
        return m.hashMap[m.keys[idx]]
    }
```
## 关于Ketama的简要介绍
[Ketama](https://github.com/RJ/ketama?source=post_page)是一个使用哈希环对多服务器进行keys分片的内存缓存型客户端。为了适配Go的实现，我遇到了这个问题：
    用Go怎么描述这行C代码？
    
```
    unsigned int k_limit = floorf(pct * 40.0 * ketama->numbuckets);
```
这是一个棘手的问题：你无法单独回答它——需要知道变量的类型和C的提升规则：

```
    float floorf(float x);
    unsigned int numbuckets;
    float pct;
```
答案是：
    
```
    limit := int(float32(float64(pct) * 40.0 * float64(numbuckets)))
```
这么写的原因是根据C的算术提升规则和常量40.0是float64型的。
一旦我整理完成我的[go-ketam](https://github.com/dgryski/go-ketama?source=post_page)实现，我马上完成了我自己的哈希环库（[libchash](https://github.com/dgryski/libchash?source=post_page)），它的正确性不是基于浮点数的四舍五入。我的库会稍微快些，因为没用MD5计算哈希。

经验：如果你构建跨语言项目，避免隐式的浮点数转换和浮点数。
简介到此为止。
## “我们完成了？”还是“为啥还是一个搜索话题？”
哈希环是我们最初的问题的一个解决方案，但是问题解决了么？不尽然，因为哈希环也有一些问题。
首先，在不同节点间的负载分发仍然是不平均的。假设每个服务器有100个副本（虚拟节点），那么负载的标准差大约是10%。桶的大小的99%的置信区间是平均负载的0.76到1.28倍之间（所有的key的数量/服务器数量）。这个可变的范围导致很难计划服务器的容量。将每个服务器上的副本增加到1000个能够把标准差减少到约3.2%，并将99%的置信区间降低到一个更小的范围0.92到1.09。

这样带来了明显的内存消耗。比如1000个节点，4MB的数据，在没有cache竞争的情况下，所有处理器cache都miss的时候，需要O(logn)次搜索（其中n=1e6）。
## Jump Hash
在2014年，Goolge发表了一篇名为[“A fast, Minimal Memory, Consistent Hash Algorithm”](https://arxiv.org/abs/1406.2294?source=post_page)也叫“Jump Hash”的论文。该算法包含在了[2011年的Guava库版本](https://github.com/google/guava/commit/f9318924b71d4ed2c59d3d835fe6a4ce3feefcbf?source=post_page#diff-e6922c1523168e608b83158728d07e63R188)，同时说明了它是从C++代码移植来的。

Jump Hash解决了哈希环的两个缺点：它没有内存开销和几乎完美的key分发（桶的标准差是0.000000746%，99%的置信区间是0.99999998到1.00000002）。

Jump Hash也很快。它的循环执行O(ln n)次，比环形哈希的二分查找O(logn)快，并且通过在寄存器中执行计算，没有cache miss的开销，可以做的更快。

下面是来自于[github.com/dgryski/go-jump](https://github.com/dgryski/go-jump?source=post_page)的代码，从论文中的C++代码翻译而来。该算法使用一个key的哈希值作为随机数生成器的种子，然后使用随机数在桶的列表中“向前跳”直到达到列表末尾，它最后所处的桶就是结果。论文中有更为完整的作用机制解释和关于这个优化过的循环的求导：

```
    func Hash(key int64, numBuckets int) int32 {
        var b int64 = -1
        var j int64
        for j < int64(numBuckets) {
            b = j
            key = key * 2862933555777941757 + 1
            j = int64(float64(b + 1) * (float64(int64(1)<<31) / float64((key >> 33) + 1)))
        }
        return int32(b)
    }
```
Jump Hash确实很棒，它既快速又能平均分配负载。那么使用它的条件是什么？最主要的限制是它只返回0到numBuckets-1范围中的一个整数。它不支持任意的桶的名字（比如环形哈希，即使两个不同的实例收到不同顺序的服务器列表，但是他们的key映射结果是一样的）。因此，最好把Jump Hash当做提供分片编号而不是服务器名字。第二，你只能在范围的大端（upper bound)添加或者移除节点，这意味着不支持任意节点的移除。因此，你不能在可能崩溃的内存缓存实例中使用这个算法来分发key——没有办法将崩溃的节点从可用列表中移除。

这些都反映出Jump Hash更适合于数据存储应用，你可以用复制解决节点失效的问题。它同样不适用于节点权重。

## “我们完成了？”还是“为啥还是一个搜索话题？”（2）
环形哈希以高内存使用为代价，提供了任意桶的增加和减少操作以减少负载变化。Jump Hash以降低改变分片数量的灵活定为代价，提供了非常高效完美的负载分布。

有没有方法，既能有环形大小变化的灵活性，又能在没有内存开销的情况下降低变化量？

## Multi-Probe Consistent Hashing
另一篇Google的论文["Multi-Probe Consistent Hashing"(2015)](https://arxiv.org/abs/1505.00062?source=post_page)试图解决这个问题。MPCH提供了O(n)的空间效率（每个入口一个节点），和O(1)的节点增加和移除操作。使用条件呢？慢慢往下看。

基本的想法就是把节点哈希多次并且增加了内存消耗，替换成节点只哈希一次但是在查找的时候key哈希k，返回距离所有查询最近的节点。k的值取决于期望的方差。为了达到1.05的峰值因数（意味着负载很重的节点比平均值高最多5%），k取21。利用精心设计的数据结构，你能把全部查找的开销从O(k log n)降到只有O(K)。[我的实现](https://github.com/dgryski/go-mpchash?source=post_page)使用了精心设计的数据结构。

一个比较点，为了达到等同于1.05的峰值因数，环形哈希需要每个节点有`700 ln n`个副本。如果有100个节点，这会导致超过1M内存的使用。

## Rendezvous Hashing
另一个较早的解决一致性哈希问题的算法叫做[”rendezvous hashing“](https://www.eecs.umich.edu/techreports/cse/96/CSE-TR-316-96.pdf?source=post_page)或者”高随机权重哈希“。它也是在1997年第一次发表的。

它的想法是用节点和key一起做hash，节点提供高位的哈希值。这个算法的副作用是很难避免O(n)的查询效率，因为要对所有节点进行迭代。

[一个实现](https://github.com/dgryski/go-rendezvous?source=post_page)。我的实现使用了预先哈希节点来优化多次哈希和使用异或随机生成树发生器作为一个低代价的整数哈希函数。

```
    func (r *Rendezvous) Lookup(k string) string {
        khash := r.hash(k)
        
        var midx int
        var mhash = xorshiftMult64(khash ^ r.nhash[0])
        
        for i, nhash := range r.nhash[1:] {
            if h:= xorshiftMult64(khash ^ nhash); h > mhash {
                midx = i + 1
                mhash = h
            }
        }
        return r.nstr[midx]
    }
```
即使rendezvous hashing有O(n)的查询效率，内部循环开销不高。根据节点数，它能轻松的”足够快“，参考最后的基准测试数据。

## Maglev Hashing
在2016年，Google发表了一篇[”Maglev: A Fast adn Reliable Software Network Load Balancer“](https://research.google.com/pubs/pub44824.html?source=post_page)论文。这篇论文中的一节描述了一个后来被称为”maglev hashing“的新一致性哈希算法。

它最主要的目标是，和环形哈希或者rendezvous hashing相比，加速查询和降低内存使用。算法高效的设计了一个能在常数时间找到节点的查询表。它的两个副作用是，节点失败时生成新表较慢（论文假设后端失效是罕见的），限制了后端节点数量最大值。Maglev Hashing还有一个目的是，在添加和移除节点时”最小化中断“，不追求最佳。把maglev定位成软件负载平衡器，这足够了。

查询表是一个所有节点的随机排列，一个查询先哈希key，再在某个位置上检查入口。这是O(1)加一个小常量数值的时间消耗（就是哈希key的时间）。

为了更深入描述查询表的建立过程，请看原始论文或者[the summary at The Morning Paper](https://blog.acolyer.org/2016/03/21/maglev-a-fast-and-reliable-software-network-load-balancer/?source=post_page)。

## 复制（Replication）
复制是对于一个已知的key使用一致性哈希选择第二个（或者更多）的节点，这个既能防范节点失败，也能查询第二个节点减少尾延迟。一些策略使用全节点复制（比如，每个服务器有两个全量备份），其他的策略则在服务器间复制key。

你始终可以在第二次查询的时候，以可预测的方式修改key或者key的哈希函数，并执行一次完整的查询。只是需要注意的是避免复制key（和原始key）落到同一个节点上。

有些算法可以直接选择多个节点进行回退或者复制。比如，环形哈希使用环上的下一个节点；multi-probe使用下一个最近的节点；rendezvous使用下一个高位（或者地位）；jump比较难，但是[也可以做](https://github.com/dgryski/go-shardedkv/blob/master/choosers/jump/jump.go?source=post_page#L27)

就像这篇文章中的其他部分，选择复制策略也需要权衡。[Vladimir Smirnov](https://github.com/civil?source=post_page)在的关于[Graphite Metrics Storage at Booking.com](https://www.youtube.com/watch?v=RzO2tmrPRfo&feature=youtu.be&t=13m50s&source=post_page)的演讲中谈到了复制策略的不同的权衡。

## 节点权重（Weighted Hosts）
一致性哈希算法在怎样简单高效的添加不同权重的服务器上各不相同。这意味着，发送更多（或者更少）的负载到某个服务器和到余下的是一样。在环形哈希中，可以根据期望的负载调整副本的数量，这会相当大地增加内存使用量。Jump Hash和Multi-Probe consistent hashing使用和维护它们已有的性能保证会更加棘手一些。可以通过添加指向原始节点的第二个”影子“节点，但是如果加载倍数不是整数，这个方法会失效。一种方法是把所有节点的数量用同一个数来缩放，但这会同时增加内存和查询时间。

Maglev hashing适配权重是通过修改查询表的构建过程，保证权重更重的节点在查询表上更频繁地选择入口。

[Weighted rendezvous hashing](http://www.snia.org/sites/default/files/SDC15_presentations/dist_sys/Jason_Resch_New_Consistent_Hashings_Rev.pdf?source=post_page)修改了一行代码以在rendezvous算法中增加了权重：通过`-weight/math.Log(h)`的缩放来选择最大联合哈希值。
## 负载平衡
使用一致性哈希完成负载平衡看起来是一个吸引人的想法。但是根据算法可能导致分布不平衡的情况，这个不会比随机分配更好。

幸运的是（再一次来自于Google），除了Maglev，我们还有两种一致性哈希方法来完成负载平衡。

第一个，在2016年，[Consistent Hashing with Bounded Loads](https://ai.googleblog.com/2017/04/consistent-hashing-with-bounded-loads.html?source=post_page)，考虑在服务器中分发key，检查负载，如果一个节点负载太高就跳过它。有一个文章详细描述了怎样添加到[HAProxy at Vimeo](https://medium.com/vimeo-engineering-blog/improving-load-balancing-with-a-new-consistent-hashing-algorithm-9f1bd75709ed)。也能在[standalong package](https://github.com/buraksezer/consistent?source=post_page)中找到。

对于选择哪个后端集合连接的客户端，Google的[SRE Book](https://landing.google.com/sre/books/)描述了一个叫做”deterministic subsetting“的算法，全部细节请看20章”Load Balancing in the Datacenter“。我有一个快速的实现在[github.com/dgryski/go-subset](https://github.com/dgryski/go-subset?source=post_page)。亚马逊的博客["shuffle sharding"](https://aws.amazon.com/cn/blogs/architecture/shuffle-sharding-massive-and-magical-fault-isolation/?source=post_page)描述了一个类似的方法。

负载平衡是一个很大的话题，足以谢一本书了。两个视频（需要翻墙）：
* [Load Balancing Is Impossible](https://www.youtube.com/watch?v=kpvbOzHUakA&source=post_page)by Tyler McMullen
* [Predictive Load-Balancing: Unfair But Faster & More Robust](https://www.youtube.com/watch?v=6NdxUY1La2I&source=post_page)by Steve Gury

## 基准测试
现在是你一直等待的环节，希望你没有调到文章的最下面，忽略了每个一致性哈希哈数的警告和权衡
![benchmarks for a single lookup with different node counts; timings in nanoseconds](https://miro.medium.com/max/1400/1*fl7F4cFSXEcFilGt5-NvFw.png)

## 总结
想你看到的，没有完美的一致性哈希算法，他们都有取舍。还有很多我没有提到的，但就像上面调到的，它们都努力的做到分发、内存使用、查询时间和构建时间（包括节点的增加和移除消耗）之间的平衡。


