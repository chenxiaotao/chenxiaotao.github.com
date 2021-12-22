---
layout:     post
title:      Redis梳理（3）
subtitle:   "Redis数据结构"
date:       2020-11-27
author:     "Sheldon"
header-img: "img/pic/post/redis.jpg"
tags:       
  - redis
---

### 1. String字符串

> 字符串是Redis最基本的数据类型，不仅所有key都是字符串类型，其它几种数据类型构成的元素也是字符串。注意字符串的长度不能超过512M.

#### 编码类型

使用 `object encoding key` 命令来查看对象(键值对)存储的数据类型，当我们使用此命令来查询 String 对象时，发现 String 对象包含了三种不同的数据类型：int、embstr 和 raw。

int 类型很好理解，整数类型对应的就是 int 类型，而字符串则对应是 embstr 类型，当字符串长度大于 44 字节时，会变为 raw 类型存储。

#### 内存布局

raw、int、embstr 内存布局如下图所示：

<img src="/assets/images/redis/redis_sds_mem.png" />

**raw 和 embstr 的区别**

其实 embstr 编码是专门用来保存短字符串的一种优化编码，embstr与raw都使用redisObject和sds保存数据，区别在于：embstr的使用只分配一次内存空间（因此redisObject和sds是连续的），而raw需要分配两次内存空间（分别为redisObject和sds分配空间）。因此与raw相比，embstr的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。而embstr的坏处也很明显，如果字符串的长度增加需要重新分配内存时，整个redisObject和sds都需要重新分配空间，因此redis中的embstr实现为只读。

ps：**Redis中对于浮点数类型也是作为字符串保存的，在需要的时候再将其转换成浮点数类型**

#### 编码的转换

当 int 编码保存的值不再是整数，或大小超过了long的范围时，自动转化为raw。

对于 embstr 编码，由于 Redis 没有对其编写任何的修改程序（embstr 是只读的），在对embstr对象进行修改时，都会先转化为raw再进行修改，因此，只要是修改embstr对象，修改后的对象一定是raw的，无论是否达到了44个字节。

#### 为什么是 44 字节

在 Redis 中，如果 SDS 的存储值大于 64 字节时，Redis 的内存分配器会认为此对象为大字符串，并使用 raw 类型来存储，当数据小于 64 字节时(字符串类型)，会使用 embstr 类型存储。既然内存分配器的判断标准是 64 字节，那为什么 embstr 类型和 raw 类型的存储判断值是 44 字节？

这是因为 Redis 在存储对象时，会创建此对象的关联信息，redisObject 对象头和 SDS 自身属性信息，这些信息都会占用一定的存储空间，因此长度判断标准就从 64 字节变成了 44 字节。

了解了 redisObject 之后，我们再来看 SDS 自身的数据结构，从 SDS 的源码可以看出，SDS 的存储类型一共有 5 种：SDS*TYPE*5、SDS*TYPE*8、SDS*TYPE*16、SDS*TYPE*32、SDS*TYPE*64，在这些类型中最小的存储类型为 SDS*TYPE*５，但 SDS*TYPE*５ 类型会默认转成 SDS*TYPE*8，以下源码可以证明，如下图所示：

<img src="/assets/images/redis/redis_sds_code.png" />

那我们直接来看 SDS*TYPE*8 的源码：

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; // 1 byte
    uint8_t alloc; // 1 byte
    unsigned char flags; // 1 byte
    char buf[];
};
```

可以看出除了内容数组(buf)之外，其他三个属性分别占用了 1 个字节，最终分隔字符等于 64 字节，减去 redisObject 的 16 个字节，再减去 SDS 自身的 3 个字节，再减去结束符 `\0` 结束符占用 1 个字节，最终的结果是 44 字节(64-16-3-1=44)，内存占用如下图所示：

<img src="/assets/images/redis/redis_sds.png" />

### 2. List列表

> list 列表，它是简单的字符串列表，按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边），它的底层实际上是个链表结构。

#### 编码规则

列表对象的编码是quicklist，是Redis 3.2 引入的数据类型。 (之前版本中有双向链表linkedlist和ziplist这两种编码。进一步的, 目前Redis定义的10个对象编码方式宏名中, 有两个被完全闲置了, 分别是: `OBJ_ENCODING_ZIPMAP`与`OBJ_ENCODING_LINKEDLIST`。 从Redis的演进历史上来看, 前者是后续可能会得到支持的编码值（代码还在）, 后者则应该是被彻底淘汰了)

#### 内存布局

<img src="/assets/images/redis/redis_list_mem.png" />

### 3. Hash散列

> 哈希对象的键是一个字符串类型，值是一个键值对集合。

#### 编码规则

哈希对象的编码可以是 ziplist 或者 hashtable；对应的底层实现有两种, 一种是ziplist, 一种是dict。

#### 内存布局

<img src="/assets/images/redis/redis_hash_mem.png" />

需要注意的是: 当采用HT编码, 即使用dict作为哈希对象的底层数据结构时, 键与值均是以sds的形式存储的.

如果使用ziplist 编码时，存储如下：

<img src="/assets/images/redis/redis_hash_ziplist.png" />

当使用 hashtable 编码时，存储如下：

<img src="/assets/images/redis/redis_hash_dict.png" />

hashtable 编码的哈希表对象底层使用字典数据结构，哈希对象中的每个键值对都使用一个字典键值对。

在前面介绍压缩列表时，我们介绍过压缩列表是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，相对于字典数据结构，压缩列表用于元素个数少、元素长度小的场景。其优势在于集中存储，节省空间。

#### 编码转换

和上面列表对象使用 ziplist 编码一样，当同时满足下面两个条件时，使用ziplist（压缩列表）编码：

1、列表保存元素个数小于512个

2、每个元素长度小于64字节

不能满足这两个条件的时候使用 hashtable 编码。第一个条件可以通过配置文件中的 `set-max-intset-entries` 进行修改。

### 4. Set集合

> 集合对象 set 是 string 类型（整数也会转换成string类型进行存储）的无序集合。注意集合和列表的区别：集合中的元素是无序的，因此不能通过索引来操作元素；集合中的元素不能有重复。

#### 编码规则

集合对象的编码可以是 intset 或者 hashtable; 底层实现有两种, 分别是intset和dict。 显然当使用intset作为底层实现的数据结构时, 集合中存储的只能是数值数据, 且必须是整数; 而当使用dict作为集合对象的底层实现时, 是将数据全部存储于dict的键中, 值字段闲置不用.

#### 内存布局

<img src="/assets/images/redis/redis_set_mem.png" />

如果使用intset 编码时，存储如下：

<img src="/assets/images/redis/redis_set_intset.png" />

当使用 hashtable 编码时，存储如下：

<img src="/assets/images/redis/redis_set_dict.png" />

#### 编码转换

当集合同时满足以下两个条件时，使用 intset 编码：

1、集合对象中所有元素都是整数

2、集合对象所有元素数量不超过512

不能满足这两个条件的就使用 hashtable 编码。第二个条件可以通过配置文件的 `set-max-intset-entries` 进行配置。

### 5. Zset有序集合

> 与集合对象相比，有序集合对象是有序的。与列表使用索引下标作为排序依据不同，有序集合为每个元素设置一个分数（score）作为排序依据。

#### 编码规则

有序集合的底层实现依然有两种, 一种是使用ziplist作为底层实现, 另外一种比较特殊, 底层使用了两种数据结构: dict与skiplist. 前者对应的编码值宏为ZIPLIST, 后者对应的编码值宏为SKIPLIST

使用ziplist来实现在序集合很容易理解, 只需要在ziplist这个数据结构的基础上做好排序与去重就可以了. 使用zskiplist来实现有序集合也很容易理解, Redis中实现的这个跳跃表似乎天然就是为了实现有序集合对象而实现的, 那么为什么还要辅助一个dict实例呢? 我们先看来有序集合对象在这两种编码方式下的内存布局, 然后再做解释:

首先是编码为ZIPLIST时, 有序集合的内存布局如下:

<img src="/assets/images/redis/redis_zset_ziplist.png" />

然后是编码为SKIPLIST时, 有序集合的内存布局如下:

<img src="/assets/images/redis/redis_zset_skiplist.png" />

说明：其实有序集合单独使用字典或跳跃表其中一种数据结构都可以实现，但是这里使用两种数据结构组合起来，原因是假如我们单独使用 字典，虽然能以 O(1) 的时间复杂度查找成员的分值，但是因为字典是以无序的方式来保存集合元素，所以每次进行范围操作的时候都要进行排序；假如我们单独使用跳跃表来实现，虽然能执行范围操作，但是查找操作有 O(1)的复杂度变为了O(logN)。因此Redis使用了两种数据结构来共同实现有序集合。

#### 编码转换

当有序集合对象同时满足以下两个条件时，对象使用 ziplist 编码：

1、保存的元素数量小于128；

2、保存的所有元素长度都小于64字节。

不能满足上面两个条件的使用 skiplist 编码。以上两个条件也可以通过Redis配置文件`zset-max-ziplist-entries` 选项和 `zset-max-ziplist-value` 进行修改。

### 6. HyperLogLogs（基数统计）

HyperLogLog 是 Redis 2.8.9 版本添加的数据结构，它用于高性能的基数（去重）统计功能，它的缺点就是存在极低的误差率。

HyperLogLog 具有以下几个特点：

- 能够使用极少的内存来统计巨量的数据，它只需要 12K 空间就能统计 2^64 的数据；
- 统计存在一定的误差，误差率整体较低，标准误差为 0.81%；
- 误差可以被设置辅助计算因子进行降低。

当需要做大量数据统计时，普通的集合类型已经不能满足我们的需求了，这个时候我们可以借助 HyperLogLog 来统计，它的优点是只需要使用 12k 的空间就能统计 2^64 的数据，但它的缺点是存在 0.81% 的误差，HyperLogLog 提供了三个操作方法 pfadd 添加元素、pfcount 统计元素和 pfmerge 合并元素。

#### 算法原理

HyperLogLog 算法来源于论文 [*HyperLogLog the analysis of a near-optimal cardinality estimation algorithm*](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)，想要了解 HLL 的原理，先要从伯努利试验说起，伯努利实验说的是抛硬币的事。一次伯努利实验相当于抛硬币，不管抛多少次只要出现一个正面，就称为一次伯努利实验。

我们用 k 来表示每次抛硬币的次数，n 表示第几次抛的硬币，用 k_max 来表示抛硬币的最高次数，最终根据估算发现 n 和 k_max 存在的关系是 n=2^(k_max)，但同时我们也发现了另一个问题当试验次数很小的时候，这种估算方法的误差会很大，例如我们进行以下 3 次实验：

- 第 1 次试验：抛 3 次出现正面，此时 k=3，n=1；
- 第 2 次试验：抛 2 次出现正面，此时 k=2，n=2；
- 第 3 次试验：抛 6 次出现正面，此时 k=6，n=3。

对于这三组实验来说，k_max=6，n=3，但放入估算公式明显 3≠2^6。为了解决这个问题 HLL 引入了分桶算法和调和平均数来使这个算法更接近真实情况。

分桶算法是指把原来的数据平均分为 m 份，在每段中求平均数在乘以 m，以此来消减因偶然性带来的误差，提高预估的准确性，简单来说就是把一份数据分为多份，把一轮计算，分为多轮计算。

而调和平均数指的是使用平均数的优化算法，而非直接使用平均数。

> 例如小明的月工资是 1000 元，而小王的月工资是 100000 元，如果直接取平均数，那小明的平均工资就变成了 (1000+100000)/2=50500 元，这显然是不准确的，而使用调和平均数算法计算的结果是 2/(1/1000+1/100000)≈1998 元，显然此算法更符合实际平均数。

所以综合以上情况，在 Redis 中使用 HLL 插入数据，相当于把存储的值经过 hash 之后，再将 hash 值转换为二进制，存入到不同的桶中，这样就可以用很小的空间存储很多的数据，统计时再去相应的位置进行对比很快就能得出结论。

### 7. Geospatial (地理位置)

GEO 本质上是基于 ZSet 实现的，这个功能可以推算地理位置的信息: 两地之间的距离, 方圆几里的人。

- geoadd：添加地理位置
- geopos：查询位置信息
- geodist：距离统计
- georadius：查询某位置内的其他成员信息
- georadiusbymember：查询member对应地址内其他成员信息
- geohash：查询位置的哈希值
- zrem：删除地理位置

### 8.参考文档
> * [Redis进阶 - 数据结构：redis对象与编码(底层结构)对应关系详解](https://www.pdai.tech/md/db/nosql-redis/db-redis-data-type-enc.html)
