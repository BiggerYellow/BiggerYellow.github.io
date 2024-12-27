---
layout: default
redis: true
modal-id: 30000008
date: 2024-12-19
img: pexels-adil-dahmani-157456832-10727340.jpg
alt: image-alt
project-date: December 2024
client: Start Bootstrap
category: redis
subtitle: 跳表
description: zskiplist
---


### 跳表 - zskiplist 
***
跳表的性能可以保证在查找、删除、添加等操作的时候在对数期望时间内完成，这个性能是可以和平衡树来相比较的，而且在实现方面比平衡树要优雅，这就是跳表的长处。跳跃表的缺点就是需要的存储空间较大，属于利用空间来换取时间的数据结构。  
跳表结构在 redis 中的运用场景只有一个，就是作为有序列表 （Zset）的使用。  


### 为什么使用跳表
***  
对于一个单链表来讲，即便链表中存储的数据是有序的，如果我们想要在其中查找某个数据，也只能从头到尾遍历链表。这样查找效率就会很低，时间复杂度会很高，是 O(n)。  
例如：查找 12，需要 7 次查找。  
<center>   
<img src="../../img/redis/skiplist/db-redis-ds-x-9.png" class="img-responsive img-centered" alt="image-alt">
</center> 

如果我们增加如下两级索引，那么它搜索次数就变成了 3 次。  
<center>   
<img src="../../img/redis/skiplist/db-redis-ds-x-10.png" class="img-responsive img-centered" alt="image-alt">
</center>

### zskiplist 数据结构  
***   

<center>   
<img src="../../img/redis/skiplist/skiplistStructure.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">跳跃表结构示意图</div>
</center> 

跳表的数据结构定义在 server.h 中，主要由以下几个结构体组成：  
```CPP
// 层高最大值限制
#define ZSKIPLIST_MAXLEVEL 64 /* Should be enough for 2^64 elements */
// 层高是否继续增长的概率
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */


/* ZSETs use a specialized version of Skiplists
 * ZSET 使用特殊版本的快表
 * */
//跳表节点定义
typedef struct zskiplistNode {
    //存储的内容
    sds ele;
    //对应的分值  用于排序
    double score;
    //后退指针
    struct zskiplistNode *backward;
    //变长数组 记录层信息，层级越高跳过的节点越多（因为层高越高概率越低）
    struct zskiplistLevel {
        //执行当前层的下一个节点
        struct zskiplistNode *forward;
        //当前节点 与 forward 所指节点间隔的节点数
        unsigned long span;
    } level[];
} zskiplistNode;

//跳表数据结构
typedef struct zskiplist {
    //头尾节点
    struct zskiplistNode *header, *tail;
    //长度
    unsigned long length;
    //最大层级
    int level;
} zskiplist;


//有序集合数据结构
typedef struct zset {
    //字典 当元素较少时使用字典作为底层结构，适用于直接查询
    dict *dict;
    //跳表 适用于范围查询
    zskiplist *zsl;
} zset;
```

- zskiplist  
header: 跳表头结点指针，头结点是一个特殊的节点，它不保存实际分值和对象，它的 level 数组长度为 32.
tail: 跳表尾结点指针，尾结点是一个真实的节点，和头结点不一样  
length: 跳跃表长度，即跳表当前节点数量，不包括头结点  
level: 跳表当前节点中的最大层数，不包括头结点  

- zskiplistNode  
ele: 节点保存的对象，是一个 sds 字符串对象。老版本存放的是 obj，最后存储的也是 sds 字符串对象。  
score: 节点分值，跳表中所有节点都按照分值从小到大排序  
backward: 指向前驱节点  
level[]: zskiplistLevel 结构体的数组，数组中的每个 zskiplistLevel 元素称为层。每层中保存了后继节点的指针 forward 和 span。  
  - forward: 表示后继节点的指针  
  - span: 表示当前节点到 forward 指向的后继节点之间需要跨越多少个节点  

  

**源码解析**  
有序集合中常用API（包含跳表和压缩列表）
```CPP
//有序集合相关API
zskiplist *zslCreate(void);
void zslFree(zskiplist *zsl);
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele);
unsigned char *zzlInsert(unsigned char *zl, sds ele, double score);
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node);
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range);
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range);
double zzlGetScore(unsigned char *sptr);
void zzlNext(unsigned char *zl, unsigned char **eptr, unsigned char **sptr);
void zzlPrev(unsigned char *zl, unsigned char **eptr, unsigned char **sptr);
unsigned char *zzlFirstInRange(unsigned char *zl, zrangespec *range);
unsigned char *zzlLastInRange(unsigned char *zl, zrangespec *range);
unsigned long zsetLength(const robj *zobj);
void zsetConvert(robj *zobj, int encoding);
void zsetConvertToZiplistIfNeeded(robj *zobj, size_t maxelelen, size_t totelelen);
int zsetScore(robj *zobj, sds member, double *score);
unsigned long zslGetRank(zskiplist *zsl, double score, sds o);
int zsetAdd(robj *zobj, double score, sds ele, int in_flags, int *out_flags, double *newscore);
long zsetRank(robj *zobj, sds ele, int reverse);
int zsetDel(robj *zobj, sds ele);
robj *zsetDup(robj *o);
int zsetZiplistValidateIntegrity(unsigned char *zl, size_t size, int deep);
void genericZpopCommand(client *c, robj **keyv, int keyc, int where, int emitkey, robj *countarg);
sds ziplistGetObject(unsigned char *sptr);
int zslValueGteMin(double value, zrangespec *spec);
int zslValueLteMax(double value, zrangespec *spec);
void zslFreeLexRange(zlexrangespec *spec);
int zslParseLexRange(robj *min, robj *max, zlexrangespec *spec);
unsigned char *zzlFirstInLexRange(unsigned char *zl, zlexrangespec *range);
unsigned char *zzlLastInLexRange(unsigned char *zl, zlexrangespec *range);
zskiplistNode *zslFirstInLexRange(zskiplist *zsl, zlexrangespec *range);
zskiplistNode *zslLastInLexRange(zskiplist *zsl, zlexrangespec *range);
int zzlLexValueGteMin(unsigned char *p, zlexrangespec *spec);
int zzlLexValueLteMax(unsigned char *p, zlexrangespec *spec);
int zslLexValueGteMin(sds value, zlexrangespec *spec);
int zslLexValueLteMax(sds value, zlexrangespec *spec);
```

- zslCreate   

```CPP
/* Create a new skiplist.
 * 创建新的跳表
 * */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    //分配内存空间
    zsl = zmalloc(sizeof(*zsl));
    //初始化层数为1
    zsl->level = 1;
    //初始化长度为0
    zsl->length = 0;
    //创建跳表节点并赋值给头结点  最大level为32
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    //循环初始化层信息
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        //下一个节点为null
        zsl->header->level[j].forward = NULL;
        //间隔数量为0
        zsl->header->level[j].span = 0;
    }
    //头结点的回退指针指向null
    zsl->header->backward = NULL;
    //尾节点指向null
    zsl->tail = NULL;
    return zsl;
}

/* Create a skiplist node with the specified number of levels.
 * The SDS string 'ele' is referenced by the node after the call.
 * 创建具有指定数量层级的跳表。
 * 在调用之后，节点引用SDS字符串 ele
 * */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    //分配内存
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    //初始化分数
    zn->score = score;
    //初始化字符串
    zn->ele = ele;
    return zn;
}
```
创建跳表结构，初始化跳表结构中各个字段，创建跳表节点层级为 32 的头结点，初始化时直接按照最大值申请高度，避免后续高度增加时为头结点重新分配内存。  

- zslInsert  

```CPP
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'.
 * 在跳表中插入一个新节点。假设元素不存在（取决于调用者强制指定）。跳表有传入的sds字符换 ele 的所有权
 * */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    //update 存放需要更新的跳表节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    //rank[i] 表示的是 update[i] 指向的节点的排位，即 update[i] 到头结点的间距
    unsigned int rank[ZSKIPLIST_MAXLEVEL];

    int i, level;

    serverAssert(!isnan(score));
    //x 指向跳表头部
    x = zsl->header;
    //第一步 收集需要更新的节点与步长信息
    //从最大层级开始 从上往下处理层级   收集需要更新的节点与步长信息
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position
         * 为到达插入位置而交叉的存储位置
         * */
        //若从最大开始则为0，否则为之前计算过的值
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];

        //若层级存在下一个节点 且 分值大于下个节点分值 或 分值相等但是 ele 较小  说明需要包含该层级信息
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            //累加当前节点与下一个节点间隔的数量
            rank[i] += x->level[i].span;
            //继续处理下一个节点
            x = x->level[i].forward;
        }
        //待更新节点 x 代表该层已经找到最后一个节点了
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not.
     * 我们假设元素不存在于跳表中，由于我们允许重复的分数，重新插入相同元素不应该发生，由于 zslInsert 的调用者应该在哈希表中测试元素是否已经存在
     * */
    //第二步 获取随机层高，补全需要更新的节点
    //计算随机层级
    level = zslRandomLevel();
    //若新计算层级大于当前跳表节点的最大层级
    if (level > zsl->level) {
        //即头结点中未使用的层都需要记录
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        //更新最大层级
        zsl->level = level;
    }
    //第三步 创建并分层插入节点，同时更新同层前一节点步长信息
    //创建跳表节点
    x = zslCreateNode(level,score,ele);
    //更新同层的节点信息
    for (i = 0; i < level; i++) {
        //更新新节点的下一个跳表节点
        x->level[i].forward = update[i]->level[i].forward;
        //将沿途记录的各个节点的 forward 指针指向新节点
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here
         * 当 x 插入到这里时， update[i] 覆盖的更新范围
         * */
        //计算新节点跨越的节点数量
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        //更新新节点插入之后，沿途节点的 span 值，其中 +1计算的是新节点
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    //第四步，更新新增节点未涉及层节点的步长信息及跳表相关信息
    /* increment span for untouched levels
     * 为未使用的层级新增 span，直接从表头节点指向新节点
     * */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    //设置新节点的后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    //跳跃表技术加1
    zsl->length++;
    return x;
}
```
插入节点可以分为四步，原有跳表结构如下:  
<center>   
<img src="../../img/redis/skiplist/zslInsert0.png" class="img-responsive img-centered" alt="image-alt">
</center>
假设现在需要插入元素 80，且获取到随机的层高为 5 （假设为了所有情况都覆盖到）  
<center>   
<img src="../../img/redis/skiplist/zslInsert1.png" class="img-responsive img-centered" alt="image-alt">
</center>

1. 收集需要更新的节点和步长信息  
  - 将插入新增节点后每层受影响节点存在 update 数组中，update[i] 为第 i+1 层会受影响节点（红框中标出来的节点）  
  - 将会受影响的节点到头结点的距离存在 rank 数组中， rank[i] 为头结点与第 i+1 层会受影响节点中间存在的节点数（rank 为 [6,5,3,3]）  
<center>   
<img src="../../img/redis/skiplist/zslInsert2.png" class="img-responsive img-centered" alt="image-alt">
</center>

2. 获取随机层高，补全需要更新的节点，同时可能更新跳表高度  
  - 通过 zslRandomLevel 函数计算当前插入节点的层高，层高越高出现的几率越小  
  - 因为搜索需要更新节点是从跳跃表当前高度的那一层开始的，如果新插入的节点的层高比当前表高还高，那么高出的这几层的头结点也是需要更新信息的  
  - 如果当前层高高于表高，则更新表高（从4变成5）  
<center>   
<img src="../../img/redis/skiplist/zslInsert3.png" class="img-responsive img-centered" alt="image-alt">
</center>

3. 创建并分层插入节点，同时更新同层前一节点步长信息  
  - 创建节点，根据当前节点的层高，在每一层进行节点插入  
  - 更新每层前一个节点（update[i] 对应节点）与自身节点的步长信息  

4. 更新新增节点未涉及层节点的步长信息，以及跳表相关信息与节点自身的相关信息  
  - 更新 zsl->level 到 level 中的节点，意味着待插入的节点层高大于后面节点的层高，后面的步长信息都需要加 1  
  - 更新跳表长度与当前节点与第一层下一节点的后退指针（后退指针只有底层链表有）  


- zslGetRank、zslGetElementByRank

```CPP
/* Find the rank for an element by both score and key.
 * Returns 0 when the element cannot be found, rank otherwise.
 * Note that the rank is 1-based due to the span of zsl->header to the
 * first element.
 * 根据分数和键值查找元素的排名
 * */
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    //当前排名
    unsigned long rank = 0;
    int i;

    //从头结点开始处理
    x = zsl->header;
    //从最大层级开始处理
    for (i = zsl->level-1; i >= 0; i--) {
        //若当前层级存在下一个节点  且 分数大于下一个节点的分数  且  分值相同但是值小于当前节点
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            //累加当前节点的跨度
            rank += x->level[i].span;
            //继续遍历下一个
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL
         * x可能是跳表头部，所以测试obj 是否为非空
         * */
        if (x->ele && sdscmp(x->ele,ele) == 0) {
            //若值相等直接返回值
            return rank;
        }
    }
    return 0;
}

/* Finds an element by its rank. The rank argument needs to be 1-based.
 * 根据元素的排名查找元素。rank参数需要以1为基础
 * */
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;
    int i;

    //x执行头结点
    x = zsl->header;
    //从最大层级开始往下遍历
    for (i = zsl->level-1; i >= 0; i--) {
        //若当前节点存在下一个节点  且 当前节点跨度小于排名 rank
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank)
        {
            //累增跨度
            traversed += x->level[i].span;
            //继续处理下一个节点
            x = x->level[i].forward;
        }
        //如果跨度相等则返回节点
        if (traversed == rank) {
            return x;
        }
    }
    return NULL;
}
```
zslGetRank: 查询元素在跳表中的排名 、 zslGetElementByRank: 根据排名查询元素的值。  
以上两个查询方法类似，从最大层级从上往下遍历，判断是否大小满足继续遍历的条件，满足则继续处理下一个节点，不满足则直接返回。  

- zslDelete

```CPP
/* Delete an element with matching score/element from the skiplist.
 * The function returns 1 if the node was found and deleted, otherwise
 * 0 is returned.
 * 从跳表中匹配分数或元素删除元素。
 * 如果节点被发现且删除该方法返回1，否则返回0
 *
 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
 * it is not freed (but just unlinked) and *node is set to the node pointer,
 * so that it is possible for the caller to reuse the node (including the
 * referenced SDS string at node->ele).
 * 如果node为null，则被删除的节点将被 zslFreeNode() 释放，否则它不会被释放（只是解除链接），*node 被设置为节点指针，所以调用方可以重用节点（包括引用的SDS字符串在节点的ele）
 * */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    //update 待更新的节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    //x从头节点开始处理
    x = zsl->header;
    //从跳表已有的最大层级开始往下记录需要处理的节点
    for (i = zsl->level-1; i >= 0; i--) {
        //迭代找到最后一个大于待删除的节点
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            //继续处理下一个跳表节点
            x = x->level[i].forward;
        }
        //待更新的跳表节点
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object.
     * 我们可能有多个相同分数的元素，我们需要的是找到具有相同分数和对象的元素
     * */
    // x 当前为要更新的最后一个节点 即离待删除节点最近的节点
    // 此时 第 0 层 forward 指向的可能就是要删除的节点
    x = x->level[0].forward;
    //若存在该节点 且 分数和大小相等
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        //删除该跳表节点及待更新的节点
        zslDeleteNode(zsl, x, update);
        //若无引用则直接释放 x 内存
        if (!node)
            zslFreeNode(x);
        else
            //若有引用则
            *node = x;
        return 1;
    }
    //为 0 说明找到
    return 0; /* not found */
}

/* Internal function used by zslDelete, zslDeleteRangeByScore and
 * zslDeleteRangeByRank.
 * 删除的内部使用方法
 * zsl:当前跳表
 * x:待删除节点
 * update:待更新节点
 * */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    //处理所有层级的节点
    for (i = 0; i < zsl->level; i++) {
        //若待更新节点下一个节点指向待删除节点 被删除的节点在第i层有节点，则update[i]为被删除节点的前一个节点
        if (update[i]->level[i].forward == x) {
            //当前节点跨度 + 下一节点跨度 - 待删除节点
            update[i]->level[i].span += x->level[i].span - 1;
            //将当前节点指向待删除的下一个节点
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            //被删除节点在第 i 层无节点，则 步长=原步长 - 1（被删除节点）
            update[i]->level[i].span -= 1;
        }
    }
    //若待删除节点存在下一个节点
    if (x->level[0].forward) {
        //更新被删除节点下一个节点的后退指针
        x->level[0].forward->backward = x->backward;
    } else {
        //否则指向末尾节点
        zsl->tail = x->backward;
    }
    //更新最大层级
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    //跳表长度减1
    zsl->length--;
}
```
删除节点与添加节点步骤类型，分为三步:  
1. 收集需要更新的节点  
2. 删除节点所在的层移除节点，并更新前一节点的步长信息（update[i] 所存节点）
3. 更新跳表高度与长度  


### 为什么不用平衡树或哈希表
***  
- 为什么不是平衡树，作者回答如下  

> https://news.ycombinator.com/item?id=1171423


```
There are a few reasons:
1) They are not very memory intensive. It's up to you basically. Changing parameters about the probability of a node to have a given number of levels will make then less memory intensive than btrees.

2) A sorted set is often target of many ZRANGE or ZREVRANGE operations, that is, traversing the skip list as a linked list. With this operation the cache locality of skip lists is at least as good as with other kind of balanced trees.

3) They are simpler to implement, debug, and so forth. For instance thanks to the skip list simplicity I received a patch (already in Redis master) with augmented skip lists implementing ZRANK in O(log(N)). It required little changes to the code.

About the Append Only durability & speed, I don't think it is a good idea to optimize Redis at cost of more code and more complexity for a use case that IMHO should be rare for the Redis target (fsync() at every command). Almost no one is using this feature even with ACID SQL databases, as the performance hint is big anyway.

About threads: our experience shows that Redis is mostly I/O bound. I'm using threads to serve things from Virtual Memory. The long term solution to exploit all the cores, assuming your link is so fast that you can saturate a single core, is running multiple instances of Redis (no locks, almost fully scalable linearly with number of cores), and using the "Redis Cluster" solution that I plan to develop in the future.
```

实现节点且达到了类似的效果。  


- 跳表与平衡树、哈希表的比较  

skiplist 和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个 key 的查找，不适宜做范围查找。  

在做范围查找时，平衡树比 skiplist 操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历不容易实现。而在 skiplist 上进行范围查找就非常简单，只需要在找到小值之后，对第 1 层链表进行若干步的遍历就可以实现。  

平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而 skiplist 的插入和删除只需要修改相邻节点的指针，操作简单又快速。  

从内存占用上来说，skiplist 比平衡树更灵活一些。一般来说，平衡树每个节点包含 2 个指针（分别指向左右子树），而 skiplist 每个节点包含的指针数目平均为 1/(1-p)，具体取决于参数 p 的大小。如果像 redis 里的实现一样，取 p=1/4，那么平均每个节点包含 1.33 个指针，比平衡树更有优势。  

查找单个 key，skiplist 和 平衡树的时间复杂度都为 O(log n)，大体相当；而哈希表在保持较低的哈希值冲突概率的前提喜爱，查找时间复杂度接近 O(1)，性能更高一些。  

从算法实现难度上来比较，skiplist 比 平衡树要简单的多。  

> 参考博客  
> https://pdai.tech/md/db/nosql-redis/db-redis-x-redis-ds.html#%E6%95%B4%E6%95%B0%E9%9B%86-intset  
> https://cloud.tencent.com/developer/article/1350965  
> https://juejin.cn/post/7382734569605316627  
> https://juejin.cn/post/6844904004498128903  
> https://nullcc.github.io/2017/11/17/Redis%E4%B8%AD%E7%9A%84%E5%BA%95%E5%B1%82%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84(7)%E2%80%94%E2%80%94%E8%B7%B3%E8%B7%83%E8%A1%A8(zskiplist)/  
> https://fanlv.fun/2019/08/17/reids-source-code-7/  
> https://www.cnblogs.com/yinbiao/p/11238374.html  
