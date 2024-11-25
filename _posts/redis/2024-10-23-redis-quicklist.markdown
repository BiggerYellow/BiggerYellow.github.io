---
layout: default
redis: true
modal-id: 30000005
date: 2024-10-23
img: pexels-mutecevvil-28871575.jpg
alt: image-alt
project-date: Ten 2024
client: Start Bootstrap
category: redis
subtitle: 快表
description: quickList
---

### 快表 - quicklist    

***  

quicklist 是一个 3.2 版本之后新增的基础数据结构，是 redis 自定义的一种复杂数据结构，将 ziplist 和 adlist 结合到一个数据结构中。主要是作为 list 的基础数据结构。  
在 3.2 之前，list 是根据元素数量的多少采用 ziplist 或者 adlist 作为基础数据结构， 3.2 之后统一改用 quicklist，从数据结构的角度来说， quicklist 结合了两种数据结构的优缺点，复杂但是实用:  
- 链表在插入，删除节点的时间复杂度很低；但是内存利用率低，且由于内存不连续容易产生内存碎片  
- 压缩表内存连续，存储效率高；但是插入和删除的成本太高，需要频繁的进行数据搬移、释放或申请内存  
而 quicklist 通过将每个压缩列表用双向链表的方式连接起来，来寻求一种收益最大化。  

**redislist数据结构特点**
- 表 list 是一个能维持数据项先后顺序的双向链表 
- 在表 list 的两端追加和删除数据极为方便，时间复杂度为 O(1)
- 表 list 也支持在任意中间位置的存取操作，时间复杂度为 O(N)
- 表 list 经常被用作队列使用

### 数据结构

***  

内部分布图:  
<center>   
<img src="../../img/redis/quicklist/redis_quicklist_structure.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">quicklist内存分布图</div>
</center>  

需要注意的几点:   
- 两端各有 2 个橙黄色的节点，是没有被压缩的。它们的数据指针 zl 指向真正的 ziplist。中间的其他节点是被压缩过的，它们的数据指针 zl 指向被压缩后的 ziplist 结构，即一个 quicklistLZF 结构。
- 左侧头结点上的 ziplist 里有 2 项数据，右侧尾节点上的 ziplist 里有 1 项数据，中间其他节点上的 ziplist 里都有 3 项数据（包括压缩的节点内部）。这表示在表的两端执行过多次 push和pop 操作后的一个状态。



首先是 quicklist 的节点 quicklistNode ，源码如下:  
```C++
/* quicklistNode is a 32 byte struct describing a ziplist for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max zl bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, NONE=1, ZIPLIST=2.
 * recompress: 1 bit, bool, true if node is temporary decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 10 bits, free for future use; pads out the remainder of 32 bits
 *
 * quicklistNode 是一个32字节的结构体，描述quickList的压缩列表。
 * 我们使用位字段将 quicklistNode 控制在 32 字节
 * count: 16位，最大 65536 （最终字节数为 65k，所以最大计数实际小于 32k）
 * encoding: 2位， 原始为1，使用LZF压缩算法为 2
 * container: 2位， 每个链表节点所持有的数据类型是什么，默认的实现是 ziplist，对应的值为2
 * quicklistNode: 1位，如果节点临时解压缩以供使用，则为true
 * attempted_compress: 1位，用于测试时的验证
 * extra:10位，免费供将来使用，补出32位的剩余部分
 *
 * */
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

/* quicklistLZF is a 4+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->zl is compressed, node->zl points to a quicklistLZF
 * quicklistLZF 是一个 4+N 字节的结构体，包含 sz 后面跟着 compressed。
 * sz 是 compressed 字段的长度。
 * compressed 是总长度为 sz 的LZF数据
 * 请注意：非压缩的长度存储在 quicklistNode 的 sz 字段中。
 * 当 quicklistNode 的 zl 是压缩的， 节点的 zl 字段指向 quicklistLZF
 * */
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

/* Bookmarks are padded with realloc at the end of of the quicklist struct.
 * They should only be used for very big lists if thousands of nodes were the
 * excess memory usage is negligible, and there's a real need to iterate on them
 * in portions.
 * When not used, they don't add any memory overhead, but when used and then
 * deleted, some overhead remains (to avoid resonance).
 * The number of bookmarks used should be kept to minimum since it also adds
 * overhead on node deletion (searching for a bookmark to update).
 * 书签在快速列表结构体的末尾使用 realloc 填充。
 * 它们只应该用于非常大的列表，如果有数千个节点，多余的内存使用可以忽略不计，并且确实需要对它们就行部分迭代。
 * 当不使用它们时，它们不会增加任何内存开销，但是当使用并删除它们时，会保留一些开心。
 * 使用的书签数量应该保持在最低限度，因为它也会增加删除节点时的开销。
 * */
typedef struct quicklistBookmark {
    quicklistNode *node;
    char *name;
} quicklistBookmark;
```
quicklistNode 实际上就是对 ziplist 的进一步封装，其中包括:  
- prev:指向链表前一个节点的指针
- next:指向链表后一个节点的指针
- zl:数据指针，如果当前节点的数据没有压缩，那么它指向一个 ziplist 结构；否则，它指向一个 quicklistLZF 结构   
- sz:表示 zl 指向的 ziplist 的总大小（包括 zlbytes、zltail、zllen、zlend和各个数据项）。需要注意的是：如果 ziplist 被压缩了，那么这个 sz 的值仍然是压缩前的 ziplist 大小。  
- count:表示 ziplist 里面包含的数据项个数，这个字段只有 16bit。
- encoding:表示 ziplist 是否压缩了以及用了哪个压缩算法。目前只有两种取值：2 表示被压缩了（而且用的是 LZF压缩算法），1 表示没有压缩
- container:是一个预留字段。本来设计是用来表明一个 quicklist 节点下面是直接存数据，还是使用 ziplist 存数据，或者用其他的数据结构来存数据。但是，在目前的实现中，这个值是一个固定的值 2，表示使用 ziplist 作为数据容器  
- recompress:当我们使用类似 lindex 这样的命令查看了某一项本来压缩的数据时，需要把数据暂时解压，这时就设置 recompress=1 做一个标记，等有机会再把数据重新压缩
- attempted_compress:这个值只对 Redis 的自动化测试程序有用。不用管它
- extra:其他扩展字段，目前也没用上  

这里从变量 count 开始，都采用了位域的方式进行数据的内存声明，使用 6 个 unsigned int 变量只用到了一个 unsigned int 的内存大小。  
C语言支持位域的方式对结构体中的数据进行声明，也就是可以指定一个类型占用几位: 
1. 如果相邻位域字段的类型相同，且其位宽之和小于类型的 sizeof 大小，则后面的字段将紧邻前一个字段存储，直到不能容纳为止；
2. 如果相邻位域字段的类型相同，且其位宽之和大于类型的 sizeof 大小，则后面的字段将从新的存储单元开始，其偏移量为其类型大小的整数倍
3. 如果相邻的位域字段的类型不同，则各编译器的具体实现有差异，VC6 采取不压缩方式，Dev-C++ 采取压缩方式
4. 如果位域字段之间穿插着非位域字段，则不进行压缩
5. 整个结构体的总大小为最宽基本类型成员大小的整数倍  
sizeof(quicklistNode): //output:32，通过位域的声明方式，quicklistNode 可以节省 24 个字节

quicklistLZF 结构表示一个被压缩过的 ziplist。其中：  
- sz:表示压缩后的 ziplist 大小
- compressed: 是个柔性数组，存放压缩后的 ziplist字节数组 


通过 quicklist 将 quicklistNode 连接起来就是一个完整的数据结构。源码如下:  
```C++
/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: 0 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor.
 * 'bookmakrs are an optional feature that is used by realloc this struct,
 *      so that they don't consume memory when not used.
 *  quicklist 是一个 40 字节的结构体，描述一个快速列表
 *  count: 表示 ziplist 节点的总数量
 *  len: 表示 quicklist 节点的总数量
 *  compress: 如果关闭压缩则为0， 否则用来表示在 quicklist 的末尾保留未压缩的 quicklistNode 的数量  对应配置:list-compress-depth
 *  0-是个特殊值，表示不压缩。是Redis的默认值
 *  1-表示quicklist两端各有 1 个节点不压缩，中间的节点压缩
 *  2-表示quicklist两端各有 2 个节点不压缩，中间的节点压缩
 *  3-表示quicklist两端各有 3 个节点不压缩，中间的节点压缩
 *  以此类推
 *
 *  fill: ziplist 的大小设置  对应配置:list-max-ziplist-size
 *  当取正值的时候，表示按照数据项个数来限定每个 quicklist 节点上的 ziplist 长度。比如，当这个参数设置成 5 时，表示每个 quicklist 节点的 ziplist 最多包含 5 个数据项
 *  当取负值的时候，表示按照占用字节数来限定每个 quicklist 节点上的 ziplist 长度。这时，它只能取 -1 到 -5 这五个值，每个值含义如下：
 *  -1:每个 quicklist 节点上的 ziplist 大小不能超过 4kb
 *  -2:每个 quicklist 节点上的 ziplist 大小不能超过 8kb (Redis 给的默认值)
 *  -3:每个 quicklist 节点上的 ziplist 大小不能超过 16kb
 *  -4:每个 quicklist 节点上的 ziplist 大小不能超过 32kb
 *  -5:每个 quicklist 节点上的 ziplist 大小不能超过 64kb
 *  bookmakrs: 是 realloc 这个结构体使用的一个可选特性，这样他们在不使用时就不会消耗内存
 *      */
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
```
- head:指向头节点（左侧第一个节点）的指针
- tail:指向尾节点（右侧第一个节点）的指针
- count:所有 ziplist 数据项的个数总和
- len:quicklist 节点的个数
- fill:16bit，ziplist 大小设置，默认为 -2，存放 list-max-ziplist-size 参数的值  
list-max-ziplist-size 数值含义: 
  - -1:每个 quicklistNode 节点的 ziplist 所占字节数不能超过 4kb （建议配置）
  - -2:每个 quicklistNode 节点的 ziplist 所占字节数不能超过 8kb （默认配置）
  - -3:每个 quicklistNode 节点的 ziplist 所占字节数不能超过 16kb 
  - -4:每个 quicklistNode 节点的 ziplist 所占字节数不能超过 32kb 
  - -5:每个 quicklistNode 节点的 ziplist 所占字节数不能超过 64kb 
  - 任意正数:表示 ziplist 结构最多包含的 entry 个数，最大为 215215  

list-max-ziplist-size 配置产生产生的原因?  
每个 quicklist 节点上的 ziplist 越短，则内存碎片越多。内存碎片多了，有可能在内存中产品很多无法被利用的小碎片，从而降低存储效率。这种情况的极端是每个 quicklist 节点上的 ziplist 只包含一个数据项，这就退变成一个普通的双向链表。  
每个 quicklist 节点上的 ziplist 越长，则为 ziplist 分配大块连续内存空间的难度就越大。有可能出现内存里有很多小块的空闲空间，但却找不到一块足够大的空闲空间分配给 ziplist 的情况。这同样会降低存储效率。这种情况的极端是整个 quicklist 只有一个节点，所有的数据项都分配在这仅有的一个节点的 ziplist 里面，这其实就退化成一个 ziplist 了。  
可见，一个 quicklist 节点上的 ziplist 要保持一个合理的长度。那么到底多长合理呢？Redis 提供了一个配置参数 list-max-ziplist-size ，就是为了让使用者可以来根据实际应用场景进行调整优化。  

- compress:16bit，节点压缩深度设置，存放 list-compress-depth 参数的值  
list-compress-depth:这个参数表示一个 quicklist 两端不被压缩的节点个数。注意:这里的节点个数是指 quicklist 双向链表的节点个数，而不是指 ziplist 里面的数据项个数。实际上，一个 quicklist 节点上的 ziplist，如果被压缩，就是整体被压缩的。  
list-compress-depth取值含义如下：
    - 0:是个特殊值，表示都不压缩，是默认值
    - 1:表示 quicklist 两端各有 1 个节点不压缩，中间的节点压缩
    - 2:表示 quicklist 两端各有 2 个节点不压缩，中间的节点压缩
    - 3:表示 quicklist 两端各有 3 个节点不压缩，中间的节点压缩
    - 以此类推

list-compress-depth 配置产生原因？  
当列表很长的时候，最容易被访问的很可能是两端的数据，中间的数据被访问的频率较低。如果应用场景符合这个特点，那么 list 还提供了一个选项，能够把中间的数据节点进行压缩，从而进一步节省内存空间。

**数据压缩**  

***  

quicklist 每个节点的实际数据存储结构为 ziplist，这种结构的主要优势在于节省存储空间。  
为了进一步降低 ziplist 所占用的空间，Redis 允许对 ziplist 进一步压缩，Redis 采用的压缩算法是 LZF，压缩过后的数据可以分成多个片段，每个片段有 2 部分：  
- 一部分是解释字段，另一部分是存放具体的数据字段  
- 解释字段可以占用 1~3 个字节，数据字段可能不存在  

示例:
> 解释字段\|数据\|......\|解释字段\|数据

LZF 压缩的数据格式有 3 种，即解释字段有 3 种:  
- 000LLLLL:字面型，解释字段占用 1 个字节，数据字段长度由解释字段后 5 位决定；L 是数据长度字段，数据长度是长度字段组成的字面值加 1。例如：0000 0001 代表数据长度为 2
- LLLooooo oooooooo:简短重复型，解释字段占用 2 个字节，没有数据字段，数据内容与前面数据内容重复，重复长度小于 8；L 是长度字段，数据长度为长度字段的字面值加 2, o 是偏移量字段，位置偏移量是偏移字段组成的字面值加 1。例如： 0010 0000 0000 0100 代表与前面 5 字节处内容重复，重复 3 字节
- 111ooooo LLLLLLLL oooooooo:批量重复型，解释字段占 3 个字节，没有数据字段，数据内容与前面内容重复；L 是长度字段，数据长度为长度字段的字面值加 9，o 是偏移量字段，位置偏移量是偏移字段组成的字面值加 1。例如：1110 0000 0000 0010 0001 0000 代表与前面 17 字节处内容重复，重复 11 个字节  

quicklistLZF结构:
```C++
/* quicklistLZF is a 4+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->zl is compressed, node->zl points to a quicklistLZF
 * quicklistLZF 是一个 4+N 字节的结构体，包含 sz 后面跟着 compressed。
 * sz 是 compressed 字段的长度。
 * compressed 是总长度为 sz 的LZF数据
 * 请注意：非压缩的长度存储在 quicklistNode 的 sz 字段中。
 * 当 quicklistNode 的 zl 是压缩的， 节点的 zl 字段指向 quicklistLZF
 * */
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
```
- sz:表示该压缩节点的总长度
- compressed:压缩后的数据片段（多个），每个数据片段由解释字段和数据字段组成  
- 当前 ziplist 未压缩长度存在于 quicklistNode -> sz 字段中
- 当 ziplist 被压缩后，node -> zl字段将指向 quicklistLZF

压缩  
LZF 数据压缩的基本思想是: 数据与前面重复的，记录重复位置以及重复长度，否则直接记录原始数据内容。  
压缩算法的流程如下: 遍历输入字符串，对当前字符及后面 2 个字符进行散列运算，如果在 Hash 表中找到曾经出现的记录，则计算重复字节的长度以及位置，反之直接输出数据。  
```CPP
/*
 * compressed format
 *
 * 000LLLLL <L+1>    ; literal, L+1=1..33 octets
 * LLLooooo oooooooo ; backref L+1=1..7 octets, o+1=1..4096 offset
 * 111ooooo LLLLLLLL oooooooo ; backref L+8 octets, o+1=1..4096 offset
 */
unsigned int
lzf_compress (const void *const in_data, unsigned int in_len, void *out_data, unsigned int out_len)
```

解压缩  
根据 LZF 压缩后的数据格式，可以较为容易地实现 LZF 的解压缩。需要注意的是，可能存在重复数据与当前位置重叠的情况，例如在当前位置前的 15 个字节处，重复了 20 个字节，此时需要按位逐个复制。  
```cpp
unsigned int
lzf_decompress (const void *const in_data, unsigned int in_len, void *out_data, unsigned int out_len)
```

### 源码解析

*** 

*创建节点*
```CPP
/* Create a new quicklist.
 * Free with quicklistRelease().
 * 创建一个快速列表
 * 通过quicklistRelease 释放
 * */
quicklist *quicklistCreate(void) {
    struct quicklist *quicklist;

    //分配内存
    quicklist = zmalloc(sizeof(*quicklist));
    //初始化头尾节点为null
    quicklist->head = quicklist->tail = NULL;
    //初始化快表节点的长度
    quicklist->len = 0;
    //初始化压缩列表的长度
    quicklist->count = 0;
    //初始化压缩的标识
    quicklist->compress = 0;
    //初始化 压缩列表的大小设置为-2
    quicklist->fill = -2;
    quicklist->bookmark_count = 0;
    return quicklist;
}
```
初始化快表，各个字段初始化默认值，fill 默认为 -2，也就是说每个 quicklistNode 中的 ziplist 最长是 8k 字节。  

*节点插入*

插入流程图：  
<center>   
<img src="../../img/redis/quicklist/quicklistPush.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">quicklistPush</div>
</center>  

源码如下：  
```CPP
/* Wrapper to allow argument-based switching between HEAD/TAIL pop
 * 包装器允许在 头部和尾部 pop 之间基于参数的切换
 * */
void quicklistPush(quicklist *quicklist, void *value, const size_t sz,
                   int where) {
    //若当前位置是头部 则在头部插入
    if (where == QUICKLIST_HEAD) {
        quicklistPushHead(quicklist, value, sz);
    } else if (where == QUICKLIST_TAIL) {//若当前位置是尾部 则在尾部插入
        quicklistPushTail(quicklist, value, sz);
    }
}

/* Add new entry to head node of quicklist.
 * 在快表的头部新增节点
 *
 * Returns 0 if used existing head.
 * Returns 1 if new head created.
 * 如果使用已存在的头部则返回0，如果创建新头部则返回1
 * */
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    //定义原始快表头部
    quicklistNode *orig_head = quicklist->head;
    assert(sz < UINT32_MAX); /* TODO: add support for quicklist nodes that are sds encoded (not zipped) */
    //检查头部是否能够允许插入新元素
    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        //将新元素插入到 压缩列表头部
        quicklist->head->zl =
            ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
        //更新压缩列表大小
        quicklistNodeUpdateSz(quicklist->head);
    } else {//若原头部没有空间插入则新建节点插入在头部
        //新建快表节点
        quicklistNode *node = quicklistCreateNode();
        //为 node 初始化压缩列表，同时将新节点插入压缩列表头部
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        //更新压缩长度
        quicklistNodeUpdateSz(node);
        //将新节点插入到快表头部
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    //节点计数+1
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}

/* Add new entry to tail node of quicklist.
 * 同插入头部
 * Returns 0 if used existing tail.
 * Returns 1 if new tail created. */
int quicklistPushTail(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_tail = quicklist->tail;
    assert(sz < UINT32_MAX); /* TODO: add support for quicklist nodes that are sds encoded (not zipped) */
    if (likely(
            _quicklistNodeAllowInsert(quicklist->tail, quicklist->fill, sz))) {
        quicklist->tail->zl =
            ziplistPush(quicklist->tail->zl, value, sz, ZIPLIST_TAIL);
        quicklistNodeUpdateSz(quicklist->tail);
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_TAIL);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeAfter(quicklist, quicklist->tail, node);
    }
    quicklist->count++;
    quicklist->tail->count++;
    return (orig_tail != quicklist->tail);
}

/**
 * 检查插入位置的 ziplist 是否能容纳该元素
 * 计算新插入元素的大小（new_sz），这个大小等于 quicklistNode 的当前大小（node->sz）、插入元素的大小（）
 */
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                           const int fill, const size_t sz) {
    if (unlikely(!node))
        return 0;

    int ziplist_overhead;
    /* size of previous offset
     * 计算下一个节点的 prev 的偏移量大小
     * */
    if (sz < 254) //小于254时 后一个节点的 pre 只有 1字节，否则为 5字节
        ziplist_overhead = 1;
    else
        ziplist_overhead = 5;

    /* size of forward offset
     * 计算下一个节点的 encoding 大小
     * */
    if (sz < 64) //若小于 64 字节当前节点的 encoding 为 1字节
        ziplist_overhead += 1;
    else if (likely(sz < 16384)) //若小于 16384 encoding 为 2字节
        ziplist_overhead += 2;
    else
        ziplist_overhead += 5; //其余情况都为 5 字节

    /* new_sz overestimates if 'sz' encodes to an integer type
     * new_sz 高估了 sz 可以被编码整数类型的场景
     * */
    //计算新 sz 的值，原sz + 新sz + 新增偏移量大小   忽略连锁更新的情况
    unsigned int new_sz = node->sz + sz + ziplist_overhead;
    //检查 fill 为负数是否超过单存储限制，不超过直接返回1
    if (likely(_quicklistNodeSizeMeetsOptimizationRequirement(new_sz, fill)))
        return 1;
    /* when we return 1 above we know that the limit is a size limit (which is
     * safe, see comments next to optimization_level and SIZE_SAFETY_LIMIT)
     * 当我们返回上面的 1 时，我们知道限制是一个大小限制
     * */
    //校验单个节点是否超过 8 kb，主要防止fill为正数时单个节点内存过大
    else if (!sizeMeetsSafetyLimit(new_sz))
        return 0;
    //fill 为正数 是否超过存储限制
    else if ((int)node->count < fill)
        return 1;
    else
        return 0;
}


unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    unsigned char *p;
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    //压缩列表插入
    return __ziplistInsert(zl,p,s,slen);
}


/* Wrappers for node inserting around existing node. */
REDIS_STATIC void _quicklistInsertNodeBefore(quicklist *quicklist,
                                             quicklistNode *old_node,
                                             quicklistNode *new_node) {
    __quicklistInsertNode(quicklist, old_node, new_node, 0);
}

REDIS_STATIC void _quicklistInsertNodeAfter(quicklist *quicklist,
                                            quicklistNode *old_node,
                                            quicklistNode *new_node) {
    __quicklistInsertNode(quicklist, old_node, new_node, 1);
}


/* Insert 'new_node' after 'old_node' if 'after' is 1.
 * Insert 'new_node' before 'old_node' if 'after' is 0.
 * Note: 'new_node' is *always* uncompressed, so if we assign it to
 *       head or tail, we do not need to uncompress it.
 * 如果 after=1 将 new_node 插入到 old_node 之后。
 * 如果 after=0 将 new_node 插入到 old_node 之前。
 * 注意：new_node 总是未压缩的，所以当我们将它设置到头部或者末尾时不需要解压缩他们
 *       */
REDIS_STATIC void __quicklistInsertNode(quicklist *quicklist,
                                        quicklistNode *old_node,
                                        quicklistNode *new_node, int after) {
    //若 after=1，则将新节点插入到旧节点之后
    if (after) {
        //将新节点的 prev 指针指向旧节点
        new_node->prev = old_node;
        //若存在旧节点
        if (old_node) {
            //将 old_node 的 next 指针赋值给 new_node 的 next，即将新节点的下一个节点指向原来的下一个
            new_node->next = old_node->next;
            //若旧节点存在下一个节点
            if (old_node->next)
                //将旧节点的下一个节点的上一个节点指针指向新节点
                old_node->next->prev = new_node;
            //旧节点的下一个节点指针指向新节点
            old_node->next = new_node;
        }
        //快表最后一个节点为旧节点
        if (quicklist->tail == old_node)
            //更新结尾节点为新节点
            quicklist->tail = new_node;
    } else {//其他情况将新节点插入到旧节点之前
        //新节点的下一个节点指向到旧节点
        new_node->next = old_node;
        //如果存在旧节点
        if (old_node) {
            //将新节点的上一个节点指针指向旧节点的上一个节点指针
            new_node->prev = old_node->prev;
            //如果存在旧节点的上一个节点
            if (old_node->prev)
                //将上一个节点的下一个节点指针指向新节点
                old_node->prev->next = new_node;
            //将旧节点的上一个节点指针指向新节点
            old_node->prev = new_node;
        }
        //如果快表的头结点为旧节点
        if (quicklist->head == old_node)
            //更新头结点为新节点
            quicklist->head = new_node;
    }
    /* If this insert creates the only element so far, initialize head/tail.
     * 如果插入导致创建了只有一个元素，则初始化头结点和尾节点
     * */
    if (quicklist->len == 0) {
        quicklist->head = quicklist->tail = new_node;
    }

    /* Update len first, so in __quicklistCompress we know exactly len
     * 首先更新长度，所以在 __quicklistCompress 中我们知道长度
     * */
    quicklist->len++;

    //如果存在旧节点
    if (old_node)
        //尝试将旧节点压缩
        quicklistCompress(quicklist, old_node);
}
```
对于 list 而言，头部或者尾部插入是最简单的。  
头插和尾插方法调用过程相似，一个处理 head 节点，一个处理 tail 节点。方法内部都调用了 _quicklistNodeAllowInsert 先判断了是否能直接在当前头尾节点插入，如果能就直接插入到对应的 ziplist 里，否则就需要新建一个新节点再操作。  _quicklistNodeAllowInsert 其实就是根据 fill 的值来判断是否已经超过最大容量。  
- 如果现有头部大小允许插入新节点，则直接插入到压缩列表中并更新压缩列表大小
- 如果现有头部大小不允许插入新节点，则新建头部为待插入的节点的压缩列表并将其指向新增的快表节点 quicklistNode 的属性 zl，更新压缩列表大小，最后将新快表节点插入到原快表头部  


*特定位置插入*
头插尾插比较简单，但 quicklist 在非头尾插入就比较繁琐了，因为要考虑到插入位置、前后节点的存储情况。  

特定位置插入流程图如下：  
<center>   
<img src="../../img/redis/quicklist/quicklistInsert.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">quicklistInsert</div>
</center>  

源码如下:  
```CPP
/* Insert a new entry before or after existing entry 'entry'.
 * 在已有节点 entry 之前或之后插入一个新节点。
 *
 * If after==1, the new value is inserted after 'entry', otherwise
 * the new value is inserted before 'entry'.
 * 如果 after=1，新值插入在 entry 之后，否则新值插入在 entry 之前
 * */
REDIS_STATIC void _quicklistInsert(quicklist *quicklist, quicklistEntry *entry,
                                   void *value, const size_t sz, int after) {
    int full = 0, at_tail = 0, at_head = 0, full_next = 0, full_prev = 0;
    int fill = quicklist->fill;
    quicklistNode *node = entry->node;
    quicklistNode *new_node = NULL;
    assert(sz < UINT32_MAX); /* TODO: add support for quicklist nodes that are sds encoded (not zipped) */

    //如果不存在已有节点，则直接新建节点插入
    if (!node) {
        /* we have no reference node, so let's create only node in the list
         * 没有参考节点，所以直接在列表中创建新节点
         * */
        D("No node given!");
        //创建新快表节点
        new_node = quicklistCreateNode();
        //创建压缩列表节点
        new_node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        //向 quicklist 中插入新节点
        __quicklistInsertNode(quicklist, NULL, new_node, after);
        //节点计数加1
        new_node->count++;
        quicklist->count++;
        return;
    }

    /* Populate accounting flags for easier boolean checks later
     * 填充记账标志，以便以后更容易地进行布尔检查
     * */
    //检查要插入的节点是否是满的，若已满 full设置为1，未满则为默认0
    if (!_quicklistNodeAllowInsert(node, fill, sz)) {
        D("Current node is full with count %d with requested fill %lu",
          node->count, fill);
        full = 1;
    }

    //检查要插入的节点是否在当前ziplist尾部  after == 1 且 当前偏移量等于节点数量
    if (after && (entry->offset == node->count)) {
        D("At Tail of current ziplist");
        at_tail = 1;
        //检查下一个节点是否已满
        if (!_quicklistNodeAllowInsert(node->next, fill, sz)) {
            D("Next node is full too.");
            full_next = 1;
        }
    }

    //检查要插入的节点是否在头部 after==0 且 当前偏移量==0
    if (!after && (entry->offset == 0)) {
        D("At Head");
        at_head = 1;
        //检查上一个节点是否已满
        if (!_quicklistNodeAllowInsert(node->prev, fill, sz)) {
            D("Prev node is full too.");
            full_prev = 1;
        }
    }

    /* Now determine where and how to insert the new element
     * 现在确定在哪里以及如何插入新元素
     * */
    //当前节点未满 且 在 entry 之后插入
    if (!full && after) {
        D("Not full, inserting after current position.");
        //将已压缩节点解压
        quicklistDecompressNodeForUse(node);
        //取下一个节点，若在末尾节点则返回 null
        unsigned char *next = ziplistNext(node->zl, entry->zi);
        //若next 为 null，即代表往末尾插入，直接调用 ziplistPush 方法向末尾插入
        if (next == NULL) {
            node->zl = ziplistPush(node->zl, value, sz, ZIPLIST_TAIL);
        } else {
            //若不为null，则走插入逻辑，插入在 next 节点之后
            node->zl = ziplistInsert(node->zl, next, value, sz);
        }
        //节点计数加 1
        node->count++;
        //更新 sz 属性
        quicklistNodeUpdateSz(node);
        //重新压缩节点
        quicklistRecompressOnly(quicklist, node);
    } else if (!full && !after) { //当前节点未满 且 在 entry 之前插入
        D("Not full, inserting before current position.");
        //将已压缩节点解压
        quicklistDecompressNodeForUse(node);
        //直接在当前节点之前插入新增节点
        node->zl = ziplistInsert(node->zl, entry->zi, value, sz);
        //同上
        node->count++;
        quicklistNodeUpdateSz(node);
        quicklistRecompressOnly(quicklist, node);
    } else if (full && at_tail && node->next && !full_next && after) {
        /* If we are: at tail, next has free space, and inserting after:
         *   - insert entry at head of next node.
         *   如果当前节点是满的，要插入的位置是当前节点的尾部，且下一个节点还有空间，那就插入到下一个节点的头部
         *   */
        D("Full and tail, but next isn't full; inserting next node head");
        new_node = node->next;
        quicklistDecompressNodeForUse(new_node);
        //在下一个节点的头部插入
        new_node->zl = ziplistPush(new_node->zl, value, sz, ZIPLIST_HEAD);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        quicklistRecompressOnly(quicklist, new_node);
    } else if (full && at_head && node->prev && !full_prev && !after) {
        /* If we are: at head, previous has free space, and inserting before:
         *   - insert entry at tail of previous node.
         *   如果当前节点是满的，要插入的位置是当前节点的头部，且头一个节点还有空间，那就插入到前一个节点的尾部
         *   */
        D("Full and head, but prev isn't full, inserting prev node tail");
        new_node = node->prev;
        quicklistDecompressNodeForUse(new_node);
        //插入到前一个节点的尾部
        new_node->zl = ziplistPush(new_node->zl, value, sz, ZIPLIST_TAIL);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        quicklistRecompressOnly(quicklist, new_node);
    } else if (full && ((at_tail && node->next && full_next && after) ||
                        (at_head && node->prev && full_prev && !after))) {
        /* If we are: full, and our prev/next is full, then:
         *   - create new node and attach to quicklist
         * 如果当前节点是满的，前后节点也是满的，那就创建一个新的节点进入
         *   */
        D("\tprovisioning new node...");
        //新建一个节点
        new_node = quicklistCreateNode();
        //更新新节点 zl 信息
        new_node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        //将新节点插入到旧节点之后
        __quicklistInsertNode(quicklist, node, new_node, after);
    } else if (full) {
        /* else, node is full we need to split it. */
        /* covers both after and !after cases
         * 否则，节点是满的，我们需要把它分裂成两个新节点，覆盖插入之前和之后两种情况，一般用于插入到当前节点 ziplist 中间某个位置时
         * */
        D("\tsplitting node...");
        quicklistDecompressNodeForUse(node);
        //按照 after 将节点拆分
        new_node = _quicklistSplitNode(node, entry->offset, after);
        //将新节点插入到刚创建的节点中
        new_node->zl = ziplistPush(new_node->zl, value, sz,
                                   after ? ZIPLIST_HEAD : ZIPLIST_TAIL);
        new_node->count++;
        quicklistNodeUpdateSz(new_node);
        __quicklistInsertNode(quicklist, node, new_node, after);
        //旧节点合并
        _quicklistMergeNodes(quicklist, node);
    }

    quicklist->count++;
}


/* Insert 'new_node' after 'old_node' if 'after' is 1.
 * Insert 'new_node' before 'old_node' if 'after' is 0.
 * Note: 'new_node' is *always* uncompressed, so if we assign it to
 *       head or tail, we do not need to uncompress it.
 * 如果 after=1 将 new_node 插入到 old_node 之后。
 * 如果 after=0 将 new_node 插入到 old_node 之前。
 * 注意：new_node 总是未压缩的，所以当我们将它设置到头部或者末尾时不需要解压缩他们
 *       */
REDIS_STATIC void __quicklistInsertNode(quicklist *quicklist,
                                        quicklistNode *old_node,
                                        quicklistNode *new_node, int after) {
    //若 after=1，则将新节点插入到旧节点之后
    if (after) {
        //将新节点的 prev 指针指向旧节点
        new_node->prev = old_node;
        //若存在旧节点
        if (old_node) {
            //将 old_node 的 next 指针赋值给 new_node 的 next，即将新节点的下一个节点指向原来的下一个
            new_node->next = old_node->next;
            //若旧节点存在下一个节点
            if (old_node->next)
                //将旧节点的下一个节点的上一个节点指针指向新节点
                old_node->next->prev = new_node;
            //旧节点的下一个节点指针指向新节点
            old_node->next = new_node;
        }
        //快表最后一个节点为旧节点
        if (quicklist->tail == old_node)
            //更新结尾节点为新节点
            quicklist->tail = new_node;
    } else {//其他情况将新节点插入到旧节点之前
        //新节点的下一个节点指向到旧节点
        new_node->next = old_node;
        //如果存在旧节点
        if (old_node) {
            //将新节点的上一个节点指针指向旧节点的上一个节点指针
            new_node->prev = old_node->prev;
            //如果存在旧节点的上一个节点
            if (old_node->prev)
                //将上一个节点的下一个节点指针指向新节点
                old_node->prev->next = new_node;
            //将旧节点的上一个节点指针指向新节点
            old_node->prev = new_node;
        }
        //如果快表的头结点为旧节点
        if (quicklist->head == old_node)
            //更新头结点为新节点
            quicklist->head = new_node;
    }
    /* If this insert creates the only element so far, initialize head/tail.
     * 如果插入导致创建了只有一个元素，则初始化头结点和尾节点
     * */
    if (quicklist->len == 0) {
        quicklist->head = quicklist->tail = new_node;
    }

    /* Update len first, so in __quicklistCompress we know exactly len
     * 首先更新长度，所以在 __quicklistCompress 中我们知道长度
     * */
    quicklist->len++;

    //如果存在旧节点
    if (old_node)
        //尝试将旧节点压缩
        quicklistCompress(quicklist, old_node);
}
```
代码比较长，总结如下:  
- 如果当前被插入节点不满，直接插入。  
- 如果当前被插入节点是满的，要插入的位置是当前节点的尾部，且后一个节点有空间，那就插到后一个节点的头部。  
- 如果当前被插入节点是满的，要插入的位置是当前节点的头部，且前一个节点有空间，那就插到前一个节点的尾部。  
- 如果当前被插入节点是满的，前后节点也是满的，要插入的位置是当前节点的头部或者尾部，那就创建一个新的节点插进去。  
- 否则，当前节点是满的，且要插入的位置在当前节点的中间位置，我们需要把当前节点分裂成两个新节点，然后再插入。  


*删除节点*

源码分析:  
```CPP
/* Delete one element represented by 'entry'
 * 删除一个由 entry 表示的元素
 *
 * 'entry' stores enough metadata to delete the proper position in
 * the correct ziplist in the correct quicklist node.
 * entry 存储了足够的元数据以删除适当的位置，在正确的压缩列表在正确的快速列表节点
 * */
void quicklistDelEntry(quicklistIter *iter, quicklistEntry *entry) {
    quicklistNode *prev = entry->node->prev;
    quicklistNode *next = entry->node->next;
    //删除当前节点
    int deleted_node = quicklistDelIndex((quicklist *)entry->quicklist,
                                         entry->node, &entry->zi);

    /* after delete, the zi is now invalid for any future usage.
     * 在删除之后， zi 现在对任何将来的使用都是无效的
     * */
    iter->zi = NULL;

    /* If current node is deleted, we must update iterator node and offset.
     * 如果当前节点已经被删除了，我们必须更新迭代节点和偏移
     * */
    if (deleted_node) {
        //若是迭代方向为头部
        if (iter->direction == AL_START_HEAD) {
            //迭代器当前指向下一个节点
            iter->current = next;
            //偏移设置为0
            iter->offset = 0;
        } else if (iter->direction == AL_START_TAIL) { //若是迭代方向为尾部
            //迭代器当前指向上一个节点
            iter->current = prev;
            //偏移设置为-1
            iter->offset = -1;
        }
    }
    /* else if (!deleted_node), no changes needed.
     * we already reset iter->zi above, and the existing iter->offset
     * doesn't move again because:
     *   - [1, 2, 3] => delete offset 1 => [1, 3]: next element still offset 1
     *   - [1, 2, 3] => delete offset 0 => [2, 3]: next element still offset 0
     *  if we deleted the last element at offet N and now
     *  length of this ziplist is N-1, the next call into
     *  quicklistNext() will jump to the next node. */
}

/* Delete one entry from list given the node for the entry and a pointer
 * to the entry in the node.
 * 从给定条目节点的列表中删除一个条目，还有一个指向节点中条目的指针
 *
 * Note: quicklistDelIndex() *requires* uncompressed nodes because you
 *       already had to get *p from an uncompressed node somewhere.
 *
 * Returns 1 if the entire node was deleted, 0 if node still exists.
 * Also updates in/out param 'p' with the next offset in the ziplist.
 * 返回1代表节点被删除了，0代表节点还存在
 * */
REDIS_STATIC int quicklistDelIndex(quicklist *quicklist, quicklistNode *node,
                                   unsigned char **p) {
    int gone = 0;

    //删除压缩列表 zl
    node->zl = ziplistDelete(node->zl, p);
    //压缩列表节点减少
    node->count--;
    //如果节点数量为 0
    if (node->count == 0) {
        gone = 1;
        //删除快表中的节点
        __quicklistDelNode(quicklist, node);
    } else {
        //更新快表大小
        quicklistNodeUpdateSz(node);
    }
    //快表节点减少
    quicklist->count--;
    /* If we deleted the node, the original node is no longer valid
     * 如果我们删除了该节点，原始节点不再有效
     * */
    return gone ? 1 : 0;
}

REDIS_STATIC void __quicklistDelNode(quicklist *quicklist,
                                     quicklistNode *node) {
    /* Update the bookmark if any */
    quicklistBookmark *bm = _quicklistBookmarkFindByNode(quicklist, node);
    if (bm) {
        bm->node = node->next;
        /* if the bookmark was to the last node, delete it. */
        if (!bm->node)
            _quicklistBookmarkDelete(quicklist, bm);
    }

    //如果当前节点存在下一个节点
    if (node->next)
        //将当前节点的下一个节点的prev 指向当前节点的prev
        node->next->prev = node->prev;
    //如果当前节点存在前置节点
    if (node->prev)
        //将当前节点上一个节点的next 指向当前节点的next
        node->prev->next = node->next;

    //如果当前节点是快表的末尾节点
    if (node == quicklist->tail) {
        //更新快表末尾节点为 当前节点的上一个节点
        quicklist->tail = node->prev;
    }

    //如果当前节点是快表的头结点
    if (node == quicklist->head) {
        //更新快表头结点为当前节点的下一个节点
        quicklist->head = node->next;
    }

    /* Update len first, so in __quicklistCompress we know exactly len */
    //更新长度
    quicklist->len--;
    quicklist->count -= node->count;

    /* If we deleted a node within our compress depth, we
     * now have compressed nodes needing to be decompressed. */
    __quicklistCompress(quicklist, NULL);

    //释放内存
    zfree(node->zl);
    zfree(node);
}
```
quicklist 对于元素删除提供了删除单一元素以及删除区间元素 2 种方案。  
1. 对于删除单一元素，可以使用 quicklist 对外的接口 quicklistDelEntry 实现，也可以通过 quicklistPop 将头部或者尾部元素弹出。  
quicklistDelEntry 函数调用底层 quicklistDelIndex 函数，该函数可以删除 quicklistNode 指向的 ziplist 中的某个元素，其中 p 指向 ziplist 中某个entry 的起始位置。  
quicklistPop 可以弹出头部或者尾部元素，具体实现是通过 ziplist 的接口获取元素值，再通过上述的 quicklistDelIndex将数据删除。  
2. 对于删除区间元素，quicklist 提供了 quicklistDelRange 接口，该函数可以从指定位置删除指定数量的元素。  

总体删除逻辑为：不管什么方式删除，最终都会通过 ziplist 来执行元素删除操作。先尝试删除该链表节点所指向的 ziplist 中的元素，如果 ziplist 中的元素已经为空了，就将该链表节点也删除掉。  


> https://pdai.tech/md/db/nosql-redis/db-redis-overview.html  
> http://zhangtielei.com/posts/blog-redis-quicklist.html
> https://czrzchao.github.io/redisSourceQuicklist
> https://fanlv.fun/2019/08/12/reids-source-code-6/
> https://juejin.cn/post/7093145133368999943
