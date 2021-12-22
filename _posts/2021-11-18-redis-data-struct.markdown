---
layout:     post
title:      Redis梳理（2）
subtitle:   "Redis底层数据结构"
date:       2020-11-18
author:     "Sheldon"
header-img: "img/pic/post/redis.jpg"
tags:       
  - redis
---

### 0.总览

<img src="/assets/images/redis/redis_object.png" />

### 1. RedisObject对象

在redis的命令中，用于对键进行处理的命令占了很大一部分，而对于键所保存的值的类型（键的类型），键能执行的命令又各不相同。如： `LPUSH` 和 `LLEN` 只能用于列表键, 而 `SADD` 和 `SRANDMEMBER` 只能用于集合键；另外一些命令，比如 `DEL`、 `TTL` 和 `TYPE`, 可以用于任何类型的键；但是要正确实现这些命令，必须为不同类型的键设置不同的处理方式： 删除一个列表键和删除一个字符串键的操作过程就不太一样。说明，**Redis 必须让每个键都带有类型信息, 使得程序可以检查键的类型, 并为它选择合适的处理方式。**

比如说，集合类型就可以由字典和整数集合两种不同的数据结构实现，但是当用户执行 ZADD 命令时，它应该不必关心集合使用的是什么编码，只要 Redis 能按照 ZADD 命令的指示，将新元素添加到集合就可以了。

这说明，**操作数据类型的命令除了要对键的类型进行检查之外, 还需要根据数据类型的不同编码进行多态处理**.

为了解决以上问题，**Redis 构建了自己的类型系统**，这个系统的主要功能包括:

- redisObject 对象；
- 基于 redisObject 对象的类型检查；
- 基于 redisObject 对象的显式多态函数；
- 对 redisObject 进行分配、共享和销毁的机制；

#### 数据结构

在 Redis 中，所有的对象都会包含 redisObject 对象头。我们先来看 redisObject 对象的源码：

```c
typedef struct redisObject {
    // 类型
    unsigned type:4; // 4 bit

    // 编码方式
    unsigned encoding:4; // 4 bit

    // LRU 记录最末一次访问时间（相对于lru_clock）; 或者 LFU（最少使用的数据：8位频率，16位访问时间）
    unsigned lru:LRU_BITS; // 3 个字节 LRU_BITS: 24

    // 引用计数
    int refcount; // 4 个字节
    
    // 指向底层数据结构
    void *ptr; // 8 个字节
} robj;
```

它的参数说明如下：

- type：对象的数据类型，例如：OBJ_STRING、OBJ_LIST、OBJ_SET 等，占用 4 bits 也就是半个字符的大小；
- encoding：对象数据编码，例如：OBJ_ENCODING_RAW、OBJ_ENCODING_HT 等 占用 4 bits；
- lru：记录对象的 LRU(Least Recently Used 的缩写，即最近最少使用)信息，记录了对象最后一次被命令程序访问的时间，内存回收时会用到此属性，占用 24 bits(3 字节)；
- refcount：引用计数器，占用 32 bits(4 字节)，为0可回收；
- *ptr：对象指针用于指向具体的内容，占用 64 bits(8 字节)。举个例子， 如果一个redisObject 的type 属性为OBJ_LIST ， encoding 属性为OBJ_ENCODING_QUICKLIST ，那么这个对象就是一个Redis 列表（List)，它的值保存在一个QuickList的数据结构内，而ptr 指针就指向quicklist的对象；

redisObject 总共占用 0.5 bytes + 0.5 bytes + 3 bytes + 4 bytes + 8 bytes = 16 bytes(字节)。

#### 对象共享

> redis一般会把一些常见的值放到一个共享对象中，这样可使程序避免了重复分配的麻烦，也节约了一些CPU时间。

**redis预分配的值对象如下**：

- 各种命令的返回值，比如成功时返回的OK，错误时返回的ERROR，命令入队事务时返回的QUEUE，等等
- 包括0 在内，小于REDIS_SHARED_INTEGERS的所有整数（REDIS_SHARED_INTEGERS的默认值是10000）

<img src="/assets/images/redis/redis_object_share.png" />

> 注意：共享对象只能被字典和双向链表这类能带有指针的数据结构使用。像整数集合和压缩列表这些只能保存字符串、整数等自勉之的内存数据结构

**为什么redis不共享列表对象、哈希对象、集合对象、有序集合对象，只共享字符串对象**

- 列表对象、哈希对象、集合对象、有序集合对象，本身可以包含字符串对象，复杂度较高；
- 如果共享对象是保存字符串对象，那么验证操作的复杂度为O(1)；
- 如果共享对象是保存字符串值的字符串对象，那么验证操作的复杂度为O(N)；
- 如果共享对象是包含多个值的对象，其中值本身又是字符串对象，即其它对象中嵌套了字符串对象，比如列表对象、哈希对象，那么验证操作的复杂度将会是O(N的平方)；

如果对复杂度较高的对象创建共享对象，需要消耗很大的CPU，用这种消耗去换取内存空间，是不合适的。

### 2. 简单动态字符串-SDS

> 字符串类型的全称是 Simple Dynamic Strings 简称 SDS，中文意思是：简单动态字符串。它是以键值对 key-value 的形式进行存储的，根据 key 来存储和获取 value 值，它的使用相对来说比较简单，但在实际项目中应用非常广泛。

<img src="/assets/images/redis/redis_data.png" />

#### 源码分析

Redis 3.2 之前 SDS 源码如下：

```c
struct sds{
    int len; // 已占用的字节数
    int free; // 剩余可以字节数
    char buf[]; // 存储字符串的数据空间
}
```

为了更加有效的利用内存，Redis 3.2 优化了 SDS 的存储结构，源码如下：

```c
typedef char *sds;

struct __attribute__ ((__packed__)) sdshdr5 { // 对应的字符串长度小于 1<<5
    unsigned char flags;
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 { // 对应的字符串长度小于 1<<8
    uint8_t len; /* 已使用长度，1 字节存储 */
    uint8_t alloc; /* 总长度 */
    unsigned char flags; 
    char buf[]; // 真正存储字符串的数据空间
};
struct __attribute__ ((__packed__)) sdshdr16 { // 对应的字符串长度小于 1<<16
    uint16_t len; /* 已使用长度，2 字节存储 */
    uint16_t alloc; 
    unsigned char flags; 
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 { // 对应的字符串长度小于 1<<32
    uint32_t len; /* 已使用长度，4 字节存储 */
    uint32_t alloc; 
    unsigned char flags; 
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 { // 对应的字符串长度小于 1<<64
    uint64_t len; /* 已使用长度，8 字节存储 */
    uint64_t alloc; 
    unsigned char flags; 
    char buf[];
};
```

- `len` 保存了SDS保存字符串的长度；
- `alloc` 表示整个SDS, 除过头部与末尾的\0, 剩余的字节数；
- `flags` 始终为一字节, 以低三位标示着头部的类型, 高5位未使用；
- `buf[]` 数组用来保存字符串的每个元素。

这样就可以针对不同长度的字符串申请相应的存储类型，从而有效的节约了内存使用。

#### 为啥使用SDS

**(1)常数复杂度获取字符串长度**

由于 len 属性的存在，我们获取 SDS 字符串的长度只需要读取 len 属性，时间复杂度为 O(1)。而对于 C 语言，获取字符串的长度通常是经过遍历计数来实现的，时间复杂度为 O(n)。通过 `strlen key` 命令可以获取 key 的字符串长度。

**(2)杜绝缓冲区溢出**

我们知道在 C 语言中使用 `strcat`  函数来进行两个字符串的拼接，一旦没有分配足够长度的内存空间，就会造成缓冲区溢出。而对于 SDS 数据类型，在进行字符修改的时候，**会首先根据记录的 len 属性检查内存空间是否满足需求**，如果不满足，会进行相应的空间扩展，然后在进行修改操作，所以不会出现缓冲区溢出。

**(3)减少修改字符串的内存重新分配次数**

C语言由于不记录字符串的长度，所以如果要修改字符串，必须要重新分配内存（先释放再申请），因为如果没有重新分配，字符串长度增大时会造成内存缓冲区溢出，字符串长度减小时会造成内存泄露。

而对于SDS，由于`len`属性和`alloc`属性的存在，对于修改字符串SDS实现了**空间预分配**和**惰性空间释放**两种策略：

1、`空间预分配`：对字符串进行空间扩展的时候，扩展的内存比实际需要的多，这样可以减少连续执行字符串增长操作所需的内存重分配次数。

2、`惰性空间释放`：对字符串进行缩短操作时，程序不立即使用内存重新分配来回收缩短后多余的字节，而是使用 `alloc` 属性将这些字节的数量记录下来，等待后续使用。（当然SDS也提供了相应的API，当我们有需要时，也可以手动释放这些未使用的空间。）

**(4)二进制安全**

因为C字符串以空字符作为字符串结束的标识，而对于一些二进制文件（如图片等），内容可能包括空字符串，因此C字符串无法正确存取；而所有 SDS 的API 都是以处理二进制的方式来处理 `buf` 里面的元素，并且 SDS 不是以空字符串来判断是否结束，而是以 len 属性表示的长度来判断字符串是否结束。

**(5)兼容部分 C 字符串函数**

虽然 SDS 是二进制安全的，但是一样遵从每个字符串都是以空字符串结尾的惯例，这样可以重用 C 语言库`<string.h>` 中的一部分函数。

#### 空间预分配

当执行追加操作时，比如现在给`key=‘Hello World’`的字符串后追加`‘ again!’`则这时的len=18，alloc由0变成了18，此时的`buf='Hello World again!\0....................'`(.表示空格)，也就是buf的内存空间是18+18+1=37个字节，其中‘\0’占1个字节redis给字符串多分配了18个字节的预分配空间，所以下次还有append追加的时候，如果预分配空间足够，就无须在进行空间分配了。在当前版本中，当新字符串的长度小于1M时，redis会分配他们所需大小一倍的空间，当大于1M的时候，就为他们额外多分配1M的空间。

执行过APPEND 命令的字符串会带有额外的预分配空间，这些预分配空间不会被释放，除非该字符串所对应的键被删除，或者等到关闭Redis 之后，再次启动时重新载入的字符串对象将不会有预分配空间。因为执行APPEND 命令的字符串键数量通常并不多，占用内存的体积通常也不大，所以这一般并不算什么问题。另一方面，如果执行APPEND 操作的键很多，而字符串的体积又很大的话，那可能就需要修改 Redis 服务器，让它定时释放一些字符串键的预分配空间，从而更有效地使用内存。

### 3. 压缩列表-ZipList

> ziplist是为了提高存储效率而设计的一种特殊编码的双向链表。它可以存储字符串或者整数，存储整数时是采用整数的二进制而不是字符串形式存储。他能在O(1)的时间复杂度下完成list两端的push和pop操作。但是因为每次操作都需要重新分配ziplist的内存，所以实际复杂度和ziplist的内存使用量相关。

#### 数据结构

<img src="/assets/images/redis/redis_ziplist_mem.png" />

- `zlbytes`字段的类型是uint32_t，这个字段中存储的是整个ziplist所占用的内存的字节数。
- `zltail`字段的类型是uint32_t，它指的是ziplist中最后一个entry的偏移量。用于快速定位最后一个entry，以快速完成pop等操作。
- `zllen`字段的类型是uint16_t，它指的是整个ziplit中entry的数量. 这个值只占2bytes（16位）: 如果ziplist中entry的数目小于65535(2的16次方)，那么该字段中存储的就是实际entry的值。若等于或超过65535，那么该字段的值固定为65535，但实际数量需要一个个entry的去遍历所有entry才能得到。
- `zlend`是一个终止字节, 其值为全F，即0xff。ziplist保证任何情况下，一个entry的首字节都不会是255。

#### Entry结构

第一种情况：一般结构 `<prevlen> <encoding> <entry-data>`

`prevlen`：前一个entry的大小；

`encoding`：不同的情况下值不同，用于表示当前entry的类型和长度；

`entry-data`：真是用于存储entry表示的数据；

第二种情况：`<prevlen> <encoding>`

在entry中存储的是int类型时，encoding和entry-data会合并在encoding中表示，此时没有entry-data字段；
redis中，在存储数据时，会先尝试将string转换成int存储，节省空间；

**prevlen编码**

当前一个元素长度小于254（255用于zlend）的时候，prevlen长度为1个字节，值即为前一个entry的长度，如果长度大于等于254的时候，prevlen用5个字节表示，第一字节设置为254，后面4个字节存储一个小端的无符号整型，表示前一个entry的长度；

```bash
<prevlen from 0 to 253> <encoding> <entry>      //长度小于254结构
0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry>   //长度大于等于254
```

**encoding编码**

encoding的长度和值根据保存的是int还是string，还有数据的长度而定；前两位用来表示类型，当为“11”时，表示entry存储的是int类型，其他表示存储的是string；

存储string时：

`|00pppppp|` ：此时encoding长度为1个字节，该字节的后六位表示entry中存储的string长度，因为是6位，所以entry中存储的string长度不能超过63；

`|01pppppp|qqqqqqqq|` 此时encoding长度为两个字节；此时encoding的后14位用来存储string长度，长度不能超过16383；

`|10000000|qqqqqqqq|rrrrrrrr|ssssssss|ttttttt|` 此时encoding长度为5个字节，后面的4个字节用来表示encoding中存储的字符串长度，长度不能超过2^32 - 1;

存储int时：

`|11000000|` encoding为3个字节，后2个字节表示一个int16；

`|11010000|` encoding为5个字节，后4个字节表示一个int32;

`|11100000|` encoding 为9个字节，后8字节表示一个int64;

`|11110000|` encoding为4个字节，后3个字节表示一个有符号整型；

`|11111110|` encoding为2字节，后1个字节表示一个有符号整型；

`|1111xxxx|` encoding长度就只有1个字节，xxxx表示一个0 - 12的整数值；

`|11111111|` 不会出现，zlen有说明

**源码分析**

```c
/* We use this function to receive information about a ziplist entry.
 * Note that this is not how the data is actually encoded, is just what we
 * get filled by a function in order to operate more easily. */
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;
```

- `prevrawlensize`表示 previous_entry_length字段的长度
- `prevrawlen`表示 previous_entry_length字段存储的内容
- `lensize`表示 encoding字段的长度
- `len`表示数据内容长度
- `headersize` 表示当前元素的首部长度，即previous_entry_length字段长度与encoding字段长度之和
- `encoding`表示数据类型
- `p`表示当前元素首地址

#### 为啥ziplist省内存

- ziplist节省内存是相对于普通的list来说的，如果是普通的数组，那么它每个元素占用的内存是一样的且取决于最大的那个元素（很明显它是需要预留空间的）；
- 所以ziplist在设计时就很容易想到要尽量让每个元素按照实际的内容大小存储，**所以增加encoding字段**，针对不同的encoding来细化存储大小；
- 这时候还需要解决的一个问题是遍历元素时如何定位下一个元素呢？在普通数组中每个元素定长，所以不需要考虑这个问题；但是ziplist中每个data占据的内存不一样，所以为了解决遍历，需要增加记录上一个元素的length，**所以增加了prelen字段**。

#### ziplist缺点

- ziplist不预留内存空间, 并且在移除结点后, 也是立即缩容, 这代表每次写操作都会进行内存分配操作。
- 结点如果扩容，导致结点占用的内存增长，并且超过254字节的话，可能会导致链式反应: 其后一个结点的entry.prevlen需要从一字节扩容至五字节. **最坏情况下, 第一个结点的扩容, 会导致整个ziplist表中的后续所有结点的entry.prevlen字段扩容**。虽然这个内存重分配的操作依然只会发生一次，但代码中的时间复杂度是o(N)级别，因为链式扩容只能一步一步的计算。但这种情况的概率十分的小，一般情况下链式扩容能连锁反映五六次就很不幸了。之所以说这是一个蛋疼问题，是因为这样的坏场景下，其实时间复杂度并不高：依次计算每个entry新的空间占用，也就是o(N)，总体占用计算出来后，只执行一次内存重分配，与对应的memmove操作，就可以了。

### 4. 快表-QuickList

#### 源码分析

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

typedef struct quicklistBookmark {
    quicklistNode *node;
    char *name;
} quicklistBookmark;

typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

typedef struct quicklistIter {
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi;
    long offset; /* offset in current ziplist */
    int direction;
} quicklistIter;

typedef struct quicklistEntry {
    const quicklist *quicklist;
    quicklistNode *node;
    unsigned char *zi;
    unsigned char *value;
    long long longval;
    unsigned int sz;
    int offset;
} quicklistEntry;
```
这里定义了6个结构体:

- `quicklistNode` 宏观上quicklist是一个链表，这个结构描述的就是链表中的结点。它通过zl字段持有底层的ziplist。简单来讲，它描述了一个ziplist实例；
- `quicklistLZF` ziplist是一段连续的内存，用LZ4算法压缩后，就可以包装成一个quicklistLZF结构。是否压缩quicklist中的每个ziplist实例是一个可配置项。若这个配置项是开启的，那么quicklistNode.zl字段指向的就不是一个ziplist实例，而是一个压缩后的quicklistLZF实例
- `quicklistBookmark` 在quicklist尾部增加的一个书签，它只有在大量节点的多余内存使用量可以忽略不计的情况且确实需要分批迭代它们，才会被使用。当不使用它们时，它们不会增加任何内存开销。
- `quicklist` 这就是一个双链表的定义。head，tail分别指向头尾指针，len代表链表中的结点，count指的是整个quicklist中的所有ziplist中的entry的数目。fill字段影响着每个链表结点中ziplist的最大占用空间，compress影响着是否要对每个ziplist以LZ4算法进行进一步压缩以更节省内存空间。
- `quicklistIter` 是一个迭代器
- `quicklistEntry `是对ziplist中的entry概念的封装。 quicklist作为一个封装良好的数据结构，不希望使用者感知到其内部的实现，所以需要把ziplist.entry的概念重新包装一下。

<img src="/assets/images/redis/redis_quicklist_mem.png" />

#### 增删逻辑

**添加**逻辑是使用quicklistPush 然后根据左右分别调用 quicklistPushHead 和 quicklistPushTail 添加元素；以 quicklistPushHead 插入头部为例，先判断 quicklist 的 head 节点是否可以插入数据，如果可以插入则使用 ziplist 的接口进行插入，否则就新建 quicklistNode 节点进行插入。函数的入参是待插入的 quicklist，还有需要插入的值 value 以及他的大小 sz。函数的返回值为 int，0 表示没有新建 head，1 表示新建了 head。

**删除** 分为单一元素删除，单一元素的删除函数是 quicklistDelEntry；区间元素删除的函数是 quicklistDelRange，quicklist 在区间删除时，会先找到 start 所在的 quicklistNode，计算删除的元素是否小于要删除的 count，如果不满足删除的个数，则会移动至下一个 quicklistNode 继续删除，依次循环直到删除完成为止。

#### 相关配置
- `quicklist.fill`的值影响着每个链表结点中, ziplist的长度.
  1. 当数值为负数时, 代表以字节数限制单个ziplist的最大长度. 具体为:
  2. -1 不超过4kb
  3. -2 不超过 8kb
  4. -3 不超过 16kb
  5. -4 不超过 32kb
  6. -5 不超过 64kb
  7. 当数值为正数时, 代表以entry数目限制单个ziplist的长度. 值即为数目. 由于该字段仅占16位, 所以以entry数目限制ziplist的容量时, 最大值为2^15个
- `quicklist.compress` 的值影响着quicklistNode.zl字段指向的是原生的ziplist, 还是经过压缩包装后的quicklistLZF
  1. 0 表示不压缩, zl字段直接指向ziplist
  2. 1 表示quicklist的链表头尾结点不压缩, 其余结点的zl字段指向的是经过压缩后的quicklistLZF
  3. 2 表示quicklist的链表头两个, 与末两个结点不压缩, 其余结点的zl字段指向的是经过压缩后的quicklistLZF
  4. 以此类推, 最大值为2^16
- `quicklistNode.encoding` 字段, 以指示本链表结点所持有的ziplist是否经过了压缩. 1代表未压缩, 持有的是原生的ziplist, 2代表压缩过
- `quicklistNode.container` 字段指示的是每个链表结点所持有的数据类型是什么. 默认的实现是ziplist, 对应的该字段的值是2, 目前Redis没有提供其它实现. 所以实际上, 该字段的值恒为2
- `quicklistNode.recompress `字段指示的是当前结点所持有的ziplist是否经过了解压. 如果该字段为1即代表之前被解压过, 且需要在下一次操作时重新压缩.

quicklist对于使用者来说，其使用体验类似于线性数据结构，list作为最传统的双链表，结点通过指针持有数据，指针字段会耗费大量内存。ziplist解决了耗费内存这个问题，但引入了新的问题: 每次写操作整个ziplist的内存都需要重分配。quicklist在两者之间做了一个平衡. 并且使用者可以通过自定义`quicklist.fill`, 根据实际业务情况调参。

### 5. 字典/哈希表-Dict

字典类型本质上是由数组和链表结构组成的：

```c
typedef struct dictht{
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值
    //总是等于 size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
 
}dictht

typedef struct dictEntry { // dict.h
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; // 下一个 entry
} dictEntry;
```

字典类型的数据结构，如下图所示：

<img src="/assets/images/redis/redis_hash_type.png" />

通常情况下字典类型会使用数组的方式来存储相关的数据，但发生哈希冲突时才会使用链表的结构来存储数据。

#### 哈希冲突

字典类型的存储流程是先将键值进行 Hash 计算，得到存储键值对应的数组索引，再根据数组索引进行数据存储，但在小概率事件下可能会出完全不相同的键值进行 Hash 计算之后，得到相同的 Hash 值，这种情况我们称之为**哈希冲突**。

<img src="/assets/images/redis/redis_dict_mem.png" />

哈希冲突一般通过链表的形式解决，相同的哈希值会对应一个链表结构，每次有哈希冲突时，就把新的元素插入到链表的尾部.

键值查询的流程如下：

- 通过算法 (Hash，计算和取余等) 操作获得数组的索引值，根据索引值找到对应的元素；
- 判断元素和查找的键值是否相等，相等则成功返回数据，否则需要查看 next 指针是否还有对应其他元素，如果没有，则返回 null，如果有的话，重复此步骤。

键值查询流程，如下图所示：

<img src="/assets/images/redis/redis_hash_conflict.png" />

#### 渐进式rehash

Redis 为了保证应用的高性能运行，提供了一个重要的机制——渐进式 rehash。 渐进式 rehash 是用来保证字典缩放效率的，也就是说在字典进行扩容或者缩容是会采取渐进式 rehash 的机制。

##### 1）扩容

当元素数量等于数组长度时就会进行扩容操作，源码在 dict.c 文件中，核心代码如下：

```c
int dictExpand(dict *d, unsigned long size)
{
    /* 需要的容量小于当前容量，则不需要扩容 */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;
    dictht n; 
    unsigned long realsize = _dictNextPower(size); // 重新计算扩容后的值
    /* 计算新的扩容大小等于当前容量，不需要扩容 */
    if (realsize == d->ht[0].size) return DICT_ERR;
    /* 分配一个新的哈希表，并将所有指针初始化为NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;
    if (d->ht[0].table == NULL) {
        // 第一次初始化
        d->ht[0] = n;
        return DICT_OK;
    }
    d->ht[1] = n; // 把增量输入放入新 ht[1] 中
    d->rehashidx = 0; // 非默认值 -1，表示需要进行 rehash
    return DICT_OK;
}
```

从以上源码可以看出，如果需要扩容则会申请一个新的内存地址赋值给 ht[1]，并把字典的 rehashindex 设置为 0，表示之后需要进行 rehash 操作。

##### 2）缩容

当字典的使用容量不足总空间的 10% 时就会触发缩容，Redis 在进行缩容时也会把 rehashindex 设置为 0，表示之后需要进行 rehash 操作。

##### 3）渐进式rehash流程

在进行渐进式 rehash 时，会同时保留两个 hash 结构，删除查找更新等操作可能会在两个哈希表上进行，第一个哈希表没有找到，就会去第二个哈希表上进行查找；新键值对加入时会直接插入到新的 hash 结构中，并会把旧 hash 结构中的元素一点一点的移动到新的 hash 结构中，当移除完最后一个元素时，清空旧 hash 结构，主要的执行流程如下：

- 扩容或者缩容时把字典中的字段 rehashidx 标识为 0；
- 在执行定时任务或者执行客户端的 hset、hdel 等操作指令时，判断是否需要触发 rehash 操作（通过 rehashidx 标识判断），如果需要触发 rehash 操作，也就是调用 dictRehash 函数，dictRehash 函数会把 ht[0] 中的元素依次添加到新的 Hash 表 ht[1] 中；
- rehash 操作完成之后，清空 Hash 表 ht[0]，然后对调 ht[1] 和 ht[0] 的值，把新的数据表 ht[1] 更改为 ht[0]，然后把字典中的 rehashidx 标识为 -1，表示不需要执行 rehash 操作。

### 6.整数集合-InSet

> 整数集合（intset）是集合类型的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis 就会使用整数集合作为集合键的底层实现。

#### 数据结构

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;    
```

`encoding` 表示编码方式，的取值有三个：INTSET_ENC_INT16, INTSET_ENC_INT32, INTSET_ENC_INT64

`length` 代表其中存储的整数的个数

`contents` 指向实际存储数值的连续内存区域, 就是一个数组；整数集合的每个元素都是 contents 数组的一个数组项（item），各个项在数组中按值得大小**从小到大有序排序**，且数组中不包含任何重复项。（虽然 intset 结构将 contents 属性声明为 int8_t 类型的数组，但实际上 contents 数组并不保存任何 int8_t 类型的值，contents 数组的真正类型取决于 encoding 属性的值）

<img src="/assets/images/redis/redis_inset_mem.png" />

#### 整数集合的升级

当在一个int16类型的整数集合中插入一个int32类型的值，整个集合的所有元素都会转换成32类型。 整个过程有三步：

- 根据新元素的类型（比如int32），扩展整数集合底层数组的空间大小，并为新元素分配空间。
- 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变。
- 最后改变encoding的值，length+1。
- 如果删除掉刚加入的int32类型时，基于减少开销的权衡，类型不会降级

### 7.跳跃表-ZSkipList

#### 设计原理

**跳跃表** 是受到这种多层链表结构的启发而设计出来的。上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到 _O(logn)_。

但是，这种方法在插入数据的时候有很大的问题。新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的 2:1 的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点 *（也包括新插入的节点）* 重新进行调整，这会让时间复杂度重新蜕化成 _O(n)_。删除数据也有同样的问题。

**跳表** 为了避免这一问题，它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是 **为每个节点随机出一个层数(level)**。比如，一个节点随机出的层数是 3，那么就把它链入到第 1 层到第 3 层这三层链表中。为了表达清楚，下图展示了如何通过一步步的插入操作从而形成一个 skiplist 的过程：

<img src="/assets/images/redis/redis_skiplist_insert.png" />

从上面的创建和插入的过程中可以看出，每一个节点的层数（level）是随机出来的，而且新插入一个节点并不会影响到其他节点的层数，因此，**插入操作只需要修改节点前后的指针，而不需要对多个节点都进行调整**，这就降低了插入操作的复杂度。

期望的目标是 50% 的概率被分配到 `Level 1`，25% 的概率被分配到 `Level 2`，12.5% 的概率被分配到 `Level 3`，以此类推…有 2-63 的概率被分配到最顶层，因为这里每一层的晋升率都是 50%

**Redis 跳跃表默认允许最大的层数是 32**，被源码中 `ZSKIPLIST_MAXLEVEL` 定义，当 `Level[0]` 有 264 个元素时，才能达到 32 层，所以定义 32 完全够用了。

现在我们假设从我们刚才创建的这个结构中查找 23 这个不存在的数，那么查找路径会如下图：

<img src="/assets/images/redis/redis_skiplist_search.png" />

#### 内存结构

redis跳跃表并没有在单独的类（比如skplist.c)中定义，而是其定义在server.h中, 如下:

```c
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

<img src="/assets/images/redis/redis_skiplist_mem.png" />

#### 设计要点

- **头节点**不持有任何数据, 且其level[]的长度为32
- 每个结点
  - `ele`字段，持有数据，是sds类型
  - `score`字段，其标示着结点的得分， 结点之间凭借得分来判断先后顺序，跳跃表中的结点按结点的得分升序排列
  - `backward`指针，这是原版跳跃表中所没有的，该指针指向结点的前一个紧邻结点
  - `level`字段，用以记录所有结点(除过头节点外)；每个结点中最多持有32个zskiplistLevel结构。实际数量在结点创建时，按幂次定律随机生成(不超过32)。每个zskiplistLevel中有两个字段
    - `forward`字段指向比自己得分高的某个结点(不一定是紧邻的)，并且若当前zskiplistLevel实例在level[]中的索引为X，则其forward字段指向的结点，其level[]字段的容量至少是X+1。这也是上图中，为什么forward指针总是画的水平的原因
    - `span`字段代表forward字段指向的结点，距离当前结点的距离。紧邻的两个结点之间的距离定义为1

#### 跳表、红黑树、哈希表比较

skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个key的查找，不适宜做范围查找。所谓范围查找，指的是查找那些大小在指定的两个值之间的所有节点。

在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。

平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。

从内存占用上来说，skiplist比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而skiplist每个节点包含的指针数目平均为1/(1-p)，具体取决于参数p的大小。如果像Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势。

查找单个key，skiplist和平衡树的时间复杂度都为O(log n)，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的。

从算法实现难度上来比较，skiplist比平衡树要简单得多。

### 6.参考文档
> * [Redis进阶-数据结构](https://www.pdai.tech/md/db/nosql-redis/db-redis-x-redis-ds.html)
