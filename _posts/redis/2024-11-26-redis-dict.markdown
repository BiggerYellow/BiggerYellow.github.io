---
layout: default
redis: true
modal-id: 30000006
date: 2024-11-26
img: pexels-kaiwalya-limaye-1997844793-29188005.jpg
alt: image-alt
project-date: Nov 2024
client: Start Bootstrap
category: redis
subtitle: 字典/哈希表
description: dict
---

### 字典/哈希表
***
通常的存储结构是 Key-Value 形式的，通过 Hash函数 对 key 求 Hash 值来确定 Value 的位置，因此也叫 Hash表。是一种用来解决算法中的查找问题的数据结构，默认的算法复杂度接近 O(1)。  

特点  
- Redis 字典的底层实现为哈希表。    
- 每个字典使用两个哈希表，一般情况下只使用 0 号哈希表，只有在 rehash 进行时，才会同时使用 0 号和 1 号哈希表。  
- 哈希表使用链地址法来解决键冲突的问题。  
- 自动 Rehash 扩展或收缩哈希表。  
- 对哈希表的 rehash 是分多次、渐进式地进行的。  


### 数据结构
***  
示意图:  
<center>   
<img src="../../img/redis/dict/dict-structure.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">dict内存分布图</div>
</center>  

源码如下:  
```CPP
typedef struct dict {
    //哈希算法
    dictType *type;
    //私有数据，用于不同类型的哈希算法的参数
    void *privdata;
    //大小为2的数组，该数组存储元素类型为 dictht，虽然有两个元素，但一般情况下只会使用 ht[0]，只有当该字典扩容、缩容需要进行rehash时，才会用到 ht[1].
    dictht ht[2];
    // rehash 进行到的索引值，当没有在 rehash 的时候，值为-1
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    //rehash 暂停标志，如果大于 0 代表 rehash 暂停
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;

//dictType 实际上就是哈希算法
typedef struct dictType {
    //hash 方法，根据 key 计算哈希值
    uint64_t (*hashFunction)(const void *key);
    //复制 key
    void *(*keyDup)(void *privdata, const void *key);
    //复制 value
    void *(*valDup)(void *privdata, const void *obj);
    //key 比较
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    //销毁 key
    void (*keyDestructor)(void *privdata, void *key);
    //销毁 value
    void (*valDestructor)(void *privdata, void *obj);
    //允许扩容
    int (*expandAllowed)(size_t moreMem, double usedRatio);
} dictType;
```
其主要作用是对散列表在进行一层封装，当字典需要进行一些特殊操作时要用到里面的辅助字段。  


```CPP
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table.
 * 这是我们的哈希表结构。
 * 当我们实现增量渐进式哈希时，每个字典都有两个这样的表，将旧表放到新表中。
 * */
typedef struct dictht {
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值
    //总是等于 size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
} dictht;
```
哈希表的结构体整体占用 32 字节，是由数组 table 组成，table中每个元素都是指向 dict.h/dictEntry 结构，dictEntry 结构定义如下：  

```CPP
typedef struct dictEntry {
    //对应的键
    void *key;
    //对应的值 可以是一个指针，也可以是数字类型
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    //指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```
整体占用 24 字节，key 用来保存键，val 属性用来保存值，是个联合体，值可以是一个指针，也可以是数值类型（包含整数和小数）。  
注意这里还有一个指向下一个哈希表节点的指针，我们知道哈希表最大的问题是存在哈希冲突，如何解决哈希冲突，有开放地址法和链地址法。这里采用的便是链地址法，通过 next 指针可以将多个哈希值相同的键值对连接在一起，用来解决哈希冲突。  


### 常用API
***  
```CPP
// 创建字典
dict *dictCreate(dictType *type, void *privDataPtr);
// 字典扩容
int dictExpand(dict *d, unsigned long size);
// 添加键值对，已存在则不添加
int dictAdd(dict *d, void *key, void *val);
// 添加或查找
dictEntry *dictAddOrFind(dict *d, void *key);
// 添加键值对，若存在则修改，反之，添加
int dictReplace(dict *d, void *key, void *val);
// 删除节点
int dictDelete(dict *d, const void *key);
// 删除节点，但不释放内存
dictEntry *dictUnlink(dict *ht, const void *key);
// 释放字典
void dictRelease(dict *d);
// 查找
dictEntry * dictFind(dict *d, const void *key);
// 收缩字典
int dictResize(dict *d);
// 开启 ressize
void dictEnableResize(void);
// 渐进式 rehash, n 为执行几步
int dictRehash(dict *d, int n);
// 扫描字典
unsigned long dictScan(dict *d, unsigned long v, dictScanFunction *fn, dictScanBucketFunction *bucketfn, void *privdata);
```

**源码解析**  

- dictCreate  

```CPP
/* Create a new hash table
 * 创建哈希表
 * */
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    //分配初始内存
    dict *d = zmalloc(sizeof(*d));

    //初始化哈希表
    _dictInit(d,type,privDataPtr);
    return d;
}

/* Initialize the hash table
 * 初始化哈希表
 * */
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    //哈希表 1 重置
    _dictReset(&d->ht[0]);
    //哈希表 2 重置
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->pauserehash = 0;
    return DICT_OK;
}

/* Reset a hash table already initialized with ht_init().
 * 重置已经用 ht_init() 初始化的哈希表
 * NOTE: This function should only be called by ht_destroy().
 * 请注意，这个方法应该只被 ht_destory() 调用
 * */
static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```
创建哈希列表，先分配结构对应的初始内存，接着初始化哈希表中各字段对应的值，主要是 dict 对象中的字段

- dictExpand  

```CPP
/* return DICT_ERR if expand was not performed
 * 如果扩容执行失败返回 DICT_ERR
 * */
int dictExpand(dict *d, unsigned long size) {
    return _dictExpand(d, size, NULL);
    

/* Expand or create the hash table,
 * when malloc_failed is non-NULL, it'll avoid panic if malloc fails (in which case it'll be set to 1).
 * Returns DICT_OK if expand was performed, and DICT_ERR if skipped.
 * 扩容或创建哈希表，
 * 当malloc_failed为非null时，它将避免malloc失败时的恐慌（在这些情况下，它将被设置为1）
 *
 * */
int _dictExpand(dict *d, unsigned long size, int* malloc_failed)
{
    //如果已尝试扩容失败，重置失败标识 继续扩容
    if (malloc_failed) *malloc_failed = 0;

    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table
     * 如果该大小 小于哈希表中已存在的元素数量，则该大小无效
     * */
    //不能在重哈希过程中 或 扩容后的数量不能小于等于已使用的数量
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    //扩容后的哈希表
    dictht n; /* the new hash table */
    //扩容后的数量，数量乘2
    unsigned long realsize = _dictNextPower(size);

    /* Detect overflows
     * 检查扩容后是否溢出
     * */
    if (realsize < size || realsize * sizeof(dictEntry*) < realsize)
        return DICT_ERR;

    /* Rehashing to the same table size is not useful.
     * 如果扩容后的数量等于表1的数量 返回失败，无需扩容
     * */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL
     * 给新表分配内存 且 初始化所有指针指向null
     * */
    //给扩容后的哈希表初始化属性
    n.size = realsize;
    n.sizemask = realsize-1;
    if (malloc_failed) {
        n.table = ztrycalloc(realsize*sizeof(dictEntry*));
        *malloc_failed = n.table == NULL;
        if (*malloc_failed)
            return DICT_ERR;
    } else
        n.table = zcalloc(realsize*sizeof(dictEntry*));

    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys.
     * 这是第一次初始化吗？如果是这样，那就不是重新散列了，我们只是设置了第一个哈希表，这样他就可以接受键了
     * */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing
     * 为增量哈希准备第二个哈希表
     * */
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}

/* Our hash table capability is a power of two
 * 我们哈希表的数量都是2的幂数
 * */
static unsigned long _dictNextPower(unsigned long size)
{
    unsigned long i = DICT_HT_INITIAL_SIZE;

    //如果超长则新增后返回
    if (size >= LONG_MAX) return LONG_MAX + 1LU;
    //数量一直累乘2
    while(1) {
        if (i >= size)
            return i;
        i *= 2;
    }
}
```
哈希表扩容场景为：当前哈希表空间不足，需要扩容以支撑新元素，还记得 dict#ht[2] 中有两张哈希表吗，扩容实际就是新增一张空哈希表到 h[1] 中，然后通过渐进式哈希逐渐将数据从 h[0] 移动到 h[1] 中。每次扩容后的数量为当前数量最近的 2次幂。

- dictAdd

<center>   
<img src="../../img/redis/dict/dictAdd.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">字典插入流程图</div>
</center>  

```CPP  
/* Add an element to the target hash table
 * 添加元素到目标哈希表中
 * */
int dictAdd(dict *d, void *key, void *val)
{
    //哈希表新增key
    dictEntry *entry = dictAddRaw(d,key,NULL);

    //若新增失败返回失败
    if (!entry) return DICT_ERR;
    //设置对应的值
    dictSetVal(d, entry, val);
    return DICT_OK;
}

/* Low level add or find:
 * This function adds the entry but instead of setting a value returns the
 * dictEntry structure to the user, that will make sure to fill the value
 * field as they wish.
 * 低级别添加或查询：
 * 此函数添加条目，但不设置值，而是向用户返回 dictEntry 结构，这将确保按用户的意愿填充字段
 *
 * This function is also directly exposed to the user API to be called
 * mainly in order to store non-pointers inside the hash value, example:
 * 这个函数也直接暴露给用户API，主要是为了在哈希值中存储非指针，例如：
 *
 * entry = dictAddRaw(dict,mykey,NULL);
 * if (entry != NULL) dictSetSignedIntegerVal(entry,1000);
 *
 * Return values:
 *
 * If key already exists NULL is returned, and "*existing" is populated
 * with the existing entry if existing is not NULL.
 * 如果key已经存在直接返回null， 如果 existing 非null， *existing 则填充为已有的节点
 *
 * If key was added, the hash entry is returned to be manipulated by the caller.
 * 如果添加了 key，则返回哈希项以供调用者操作
 */
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    //判断哈希表是否在重新哈希中
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists.
     * 获取新元素的索引位置，如果元素已经存在则为-1
     * */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry.
     * Insert the element in top, with the assumption that in a database
     * system it is more likely that recently added entries are accessed
     * more frequently.
     * 分配内存且存储新节点
     * 将新元素插入到顶部，假设在数据库系统中，最近添加的条目更有可能被频繁的访问
     * */
    //判断是否处于渐进式哈希，是则返回表1否则返回表0
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    //分配新节点的内存
    entry = zmalloc(sizeof(*entry));
    //新节点的下一个指针指向 当前哈希表数组对应位置
    entry->next = ht->table[index];
    //将新节点替换原索引
    ht->table[index] = entry;
    //已使用节点++
    ht->used++;

    /* Set the hash entry fields.
     * 设置哈希字段
     * */
    dictSetKey(d, entry, key);
    return entry;
}

/* This function performs just a step of rehashing, and only if hashing has
 * not been paused for our hash table. When we have iterators in the
 * middle of a rehashing we can't mess with the two hash tables otherwise
 * some element can be missed or duplicated.
 * 这个函数只执行重新散列的一个步骤，并且只有在散列没有暂停的情况下才执行。
 * 当我们在重新散列过程中使用迭代器时，我们不能混淆两个哈希表，否则可能会丢失或重复某些元素
 *
 * This function is called by common lookup or update operations in the
 * dictionary so that the hash table automatically migrates from H1 to H2
 * while it is actively used.
 * 该函数由字典中的常见查找或更新操作调用，以便哈希表在活跃使用时自动从 H1迁移到 H2.
 * */
static void _dictRehashStep(dict *d) {
    //当 pauserehash 为0 ，表示在哈希中
    if (d->pauserehash == 0) dictRehash(d,1);
}

/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 * 执行 N 步增量哈希。如果仍然存在键需要从旧表移动到新表则返回1，否则返回0
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table, however
 * since part of the hash table may be composed of empty spaces, it is not
 * guaranteed that this function will rehash even a single bucket, since it
 * will visit at max N*10 empty buckets in total, otherwise the amount of
 * work it does would be unbound and the function may block for a long time.
 * 请注意 一个重哈希步骤包含移动一个桶(正如我们使用的链式包含不止一个kry) 从旧表到新表，
 * 但是由于哈希表的其他部分可能包含空白空间，它不保证这个方法重新哈希甚至是单个桶，因为它总共会访问最大 N*10 个空桶，
 * 否则它所做的工作将被解除绑定，而且这个方法可能会阻塞很长时间
 * */
int dictRehash(dict *d, int n) {
    //空桶的最大访问数量
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    //初始化 两个哈希表的当前大小
    unsigned long s0 = d->ht[0].size;
    unsigned long s1 = d->ht[1].size;
    //判断哈希是否被禁止 返回0
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    //如果允许哈希，但是 当前比率 小于 dict_force_resize_ratio 即不允许调整大小 则返回0
    if (dict_can_resize == DICT_RESIZE_AVOID &&
        ((s1 > s0 && s1 / s0 < dict_force_resize_ratio) ||
         (s1 < s0 && s0 / s1 < dict_force_resize_ratio)))
    {
        return 0;
    }

    //循环 n 直到为 0，且 哈希表 0 已使用的节点不等于0
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0
         * 请注意，rehashidx 不能溢出，因为我们确信有更多的元素，因为 哈希表0 使用的元素不等于 0
         * */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        //若哈希表0 所对应的 rehashidx 索引位置的值为 null
        //说明访问到空桶了，哈希索引值++， 最大空桶访问量empty_visits--，直到等于0，说明 rehash结束返回1
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        //取出当前哈希索引所对应的值
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT
         * 将这个桶中的所有键从旧的哈希表中移动到新的哈希表中
         * */
        //因为 de 是dictEntry，是链表，所以需要将整个链表中的元素都重哈希到 表1 中
        while(de) {
            //初始化重哈希的值
            uint64_t h;
            //保留下一个链表要处理的值，因为当前要处理 de
            nextde = de->next;
            /* Get the index in the new hash table
             * 使用表1的 sizemask 重新计算索引位置
             * */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            //将表1中 h 对应字典值插入到 待插入值 de 的后面
            de->next = d->ht[1].table[h];
            //将de 赋值给新位置
            d->ht[1].table[h] = de;
            //表0可用元素 --
            d->ht[0].used--;
            //表1可用元素++
            d->ht[1].used++;
            //将de设置为链表对应的下一个值
            de = nextde;
        }
        //上述操作遍历完后说明 表0的元素都重哈希结束 将 rehashidx 索引位置的值设置为0
        d->ht[0].table[d->rehashidx] = NULL;
        //rehashidx 索引位置+1
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table...
     * 检查是否已经将整个哈希表都重哈希
     * */
    //若表0 已使用的节点数量为 0 说明都重哈希过了
    if (d->ht[0].used == 0) {
        //释放表0的内存
        zfree(d->ht[0].table);
        //将表1 替换 表0
        d->ht[0] = d->ht[1];
        //重置表1
        _dictReset(&d->ht[1]);
        //重置rehashidx为-1
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash...
     * 还有剩余的值需要哈希
     * */
    return 1;
}

/* Returns the index of a free slot that can be populated with
 * a hash entry for the given 'key'.
 * If the key already exists, -1 is returned
 * and the optional output parameter may be filled.
 * 返回空闲槽位的索引，它可以用给定 key 的散列条目填充
 * 如果 key 已经存在，直接返回-1， 且 可选的输出参数可能被填充。
 *
 * Note that if we are in the process of rehashing the hash table, the
 * index is always returned in the context of the second (new) hash table.
 * 请注意 如果处于正在重新哈希的过程中时，这个索引总是在新哈希表的上下文中返回
 * */
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    /* Expand the hash table if needed
     * 如果需要的话扩容哈希表
     * */
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        //扩容失败返回-1
        return -1;
    //从表0和表1中根据哈希值找到对应的值
    for (table = 0; table <= 1; table++) {
        //计算哈希位置
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key
         * 检查这个槽位是否已经存在给定的key
         * */
        //找到当前表中指定哈希位置的节点
        he = d->ht[table].table[idx];
        //遍历哈希表数组，如果 对应键key 相同 或值相同则返回-1，不存在则返回后计算的哈希值
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}
```
哈希表插入元素核心逻辑就是检查原有哈希表是否需要扩容，需要则直接扩容，然后计算新值哈希后对应的索引位置，从 表0 和 表1 (因为处于渐进式哈希过程中会导致表0和表1都有值) 查找是否已经存在该值，不存在则插入到对应的哈希表中，如果产生哈希冲突，则使用链地址法将新值插入到链表中。  
其中需要注意，插入时如果处于渐进式哈希过程中，需要先处理渐进式哈希再执行插入逻辑。  

- dictDelete  

```CPP
/* Remove an element, returning DICT_OK on success or DICT_ERR if the
 * element was not found.
 * 移除元素，成功返回 DICT_OK ，若元素不存在返回 DICT_ERR
 * */
int dictDelete(dict *ht, const void *key) {
    //删除节点并且释放内存
    return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
}

/* Search and remove an element. This is an helper function for
 * dictDelete() and dictUnlink(), please check the top comment
 * of those functions.
 * 搜索并移除元素。他是dictDelete() and dictUnlink()的辅助函数，请检查这些函数的头部注释
 *
 * d:哈希表
 * key:待删除的key
 * nofree:是否释放内存  0-释放 1-不释放
 * */
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    //初始化变量
    //h: key对应的哈希值
    //idx: key对应的索引位置
    uint64_t h, idx;
    //*he: idx对应的哈希链表节点
    //*prevHe: 遍历*he时对应的前节点
    dictEntry *he, *prevHe;
    //哈希表d对应的表0和表1
    int table;

    //如果哈希表1和表2的已使用元素都为0说明哈希表为空，直接返回null
    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;

    //若哈希表处于重哈希过程中，执行重哈希方法
    if (dictIsRehashing(d)) _dictRehashStep(d);

    //计算新key对应的哈希值
    h = dictHashKey(d, key);

    //遍历表0和表1检查是否需要删除元素
    for (table = 0; table <= 1; table++) {
        //计算对应的索引位置
        //例如：h为5，哈希表的大小初始化为4，sizemask则为size-1
        //故有 h&sizemask = 2，所以该键值对就是存放在索引位置为2的地方
        idx = h & d->ht[table].sizemask;
        //找到对应位置哈希链表节点
        he = d->ht[table].table[idx];

        //前置节点
        prevHe = NULL;
        //若存在哈希链表节点
        while(he) {
            //若key相同，执行删除动作
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                /* Unlink the element from the list
                 * 从链表中移除该元素
                 * */
                //判断是否存在前置节点
                if (prevHe)
                    //将前置节点的下一个节点指针指向当前节点的下一个节点
                    prevHe->next = he->next;
                else
                    //没有前置节点 直接将当前节点的下一个节点覆盖到表中
                    d->ht[table].table[idx] = he->next;
                //是否要释放内存 key和val
                if (!nofree) {
                    //哈希表释放
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                    //空间释放
                    zfree(he);
                }
                //删除成功已使用元素
                d->ht[table].used--;
                return he;
            }
            //赋值前置节点 prevHe的值
            prevHe = he;
            //继续往下遍历
            he = he->next;
        }
        //如果哈希表不在重哈希过程中，说明元素都在一个表中，直接返回
        if (!dictIsRehashing(d)) break;
    }
    //若没有发现相同key的元素直接返回null
    return NULL; /* not found */
}
```
删除节点基本同插入节点。  
1. 首先还是执行渐进式哈希
2. 计算待删除的键对应的哈希值
3. 分别遍历表0和表1，并计算哈希值对应的索引位置
4. 找到对应的哈希链表节点，如果值相同则删除该节点并释放内存，否则继续遍历下一个节点直到结束


- dictFind  

```CPP
//根据key查找元素
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    //如果哈希表为空直接返回null
    if (dictSize(d) == 0) return NULL; /* dict is empty */
    //如果哈希表处于重哈希过程中执行渐进式哈希
    if (dictIsRehashing(d)) _dictRehashStep(d);
    //计算该key对应的哈希值
    h = dictHashKey(d, key);
    //遍历两个哈希表
    for (table = 0; table <= 1; table++) {
        //计算当前表中 哈希对应的索引位置
        idx = h & d->ht[table].sizemask;
        //取出对应索引位置的哈希链表节点
        he = d->ht[table].table[idx];
        //遍历链表节点
        while(he) {
            //若存在哈希值相同的key则返回
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            //继续往下遍历
            he = he->next;
        }
        //若遍历过程中发现重哈希结束，直接返回null  说明元素都在另一个表中
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```
查找逻辑也基本类似，见代码逻辑即可。  

### 重点关注-渐进式hash
*** 

- dictRehash

<center>   
<img src="../../img/redis/dict/dictRehash.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">字典渐进式哈希流程图</div>
</center>  

源码如下：  

```CPP
/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 * 执行 N 步增量哈希。如果仍然存在键需要从旧表移动到新表则返回1，否则返回0
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table, however
 * since part of the hash table may be composed of empty spaces, it is not
 * guaranteed that this function will rehash even a single bucket, since it
 * will visit at max N*10 empty buckets in total, otherwise the amount of
 * work it does would be unbound and the function may block for a long time.
 * 请注意 一个重哈希步骤包含移动一个桶(正如我们使用的链式包含不止一个kry) 从旧表到新表，
 * 但是由于哈希表的其他部分可能包含空白空间，它不保证这个方法重新哈希甚至是单个桶，因为它总共会访问最大 N*10 个空桶，
 * 否则它所做的工作将被解除绑定，而且这个方法可能会阻塞很长时间
 * */
int dictRehash(dict *d, int n) {
    //空桶的最大访问数量
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    //初始化 两个哈希表的当前大小
    unsigned long s0 = d->ht[0].size;
    unsigned long s1 = d->ht[1].size;
    //判断哈希是否被禁止 返回0
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    //如果避免哈希，但是 当前比率 小于 dict_force_resize_ratio 即不允许调整大小 则返回0
    if (dict_can_resize == DICT_RESIZE_AVOID &&
        ((s1 > s0 && s1 / s0 < dict_force_resize_ratio) ||
         (s1 < s0 && s0 / s1 < dict_force_resize_ratio)))
    {
        return 0;
    }

    //循环 n 直到为 0，且 哈希表 0 已使用的节点不等于0
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0
         * 请注意，rehashidx 不能溢出，因为我们确信有更多的元素，因为 哈希表0 使用的元素不等于 0
         * */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        //若哈希表0 所对应的 rehashidx 索引位置的值为 null
        //说明访问到空桶了，哈希索引值++， 最大空桶访问量empty_visits--，直到等于0，说明 rehash结束返回1
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        //取出当前哈希索引所对应的值
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT
         * 将这个桶中的所有键从旧的哈希表中移动到新的哈希表中
         * */
        //因为 de 是dictEntry，是链表，所以需要将整个链表中的元素都重哈希到 表1 中
        while(de) {
            //初始化重哈希的值
            uint64_t h;
            //保留下一个链表要处理的值，因为当前要处理 de
            nextde = de->next;
            /* Get the index in the new hash table
             * 使用表1的 sizemask 重新计算索引位置
             * */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            //将表1中 h 对应字典值插入到 待插入值 de 的后面
            de->next = d->ht[1].table[h];
            //将de 赋值给新位置
            d->ht[1].table[h] = de;
            //表0可用元素 --
            d->ht[0].used--;
            //表1可用元素++
            d->ht[1].used++;
            //将de设置为链表对应的下一个值
            de = nextde;
        }
        //上述操作遍历完后说明 表0的元素都重哈希结束 将 rehashidx 索引位置的值设置为0
        d->ht[0].table[d->rehashidx] = NULL;
        //rehashidx 索引位置+1
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table...
     * 检查是否已经将整个哈希表都重哈希
     * */
    //若表0 已使用的节点数量为 0 说明都重哈希过了
    if (d->ht[0].used == 0) {
        //释放表0的内存
        zfree(d->ht[0].table);
        //将表1 替换 表0
        d->ht[0] = d->ht[1];
        //重置表1
        _dictReset(&d->ht[1]);
        //重置rehashidx为-1
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash...
     * 还有剩余的值需要哈希
     * */
    return 1;
}
```

触发 rehash 的条件有两个：  
1. 总的元素个数 除 dict 桶的个数得到每一个桶平均存储的元素个数（pre_num），假设 pre_num > dict_force_resize_ratio，就会触发 dict 扩容操作  
2. 在总元素 * 10 < 桶的个数，也就是填充率必须 < 10%，dict便会进行收缩，让 total/bk_num 接近 1:1  

随着操作的不断进行，哈希表的长度会不断增减。哈希表的长度太长会造成空间浪费，太短哈希冲突明显导致性能下降，哈希表需要通过扩缩容让哈希表的长度保持在一个合理的范围内。  
redis 通过 ht[0] 和 ht[1] 来完成 rehash 的操作，步骤如下：
1. 为 ht[1] 分配空间，分配的空间长度有两种情况：
    - 扩容：第一个大于等于 ht[0].used*2 的 2^n 的数。例如 ht[0].used = 3，那么分配的是距离 6 最近的 2^3=8
    - 缩容：第一个大于等于 ht[0].used/2 的 2^n 的数。例如 ht[0].used = 6，那么分配的是距离 3 最近的 2^2=4
2. 将 h[0] 上的键值对都迁移到 h[1]，迁移的时候都是重新计算索引值的。由于 h[1] 的长度较长，之前在 h[0] 拉链的元素大概率被分到不同的位置。
3. ht[0] 所有的键值对迁移完之后，h[0] 释放，然后 h[0] 替换为 h[1]，并把 h[1] 清空，为下次 rehash 准备

上面说的第二步，迁移的过程不是一次完成的。如果哈希表的长度比较小，一次完成很快。但是如果哈希表很长，例如百万千万，那这个迁移的过程就没有那么快了，会造成命令阻塞。  
下面来看，redis是如何渐进式地将 h[0] 中的键值对迁移到 h[1] 中的:
1. 为 h[1] 开辟空间，字典同时持有 h[0] 和 h[1]
2. 字典中的 rehashidx 维护了 rehash 的进度，设置为 0 的时候，开始 rehash
3. 字典每次增删改查的时候，除了完成指定操作之外，还会顺带把 rehashidx 上的整个链表迁移到 h[1] 中，迁移完成后 rehashidx++
4. 随着字典的不断读取、操作，最终 h[0] 上的所有键值对都会迁移到 h[1] 中，全部迁移完成之后 rehashidx = -1

这种渐进式 rehash 的方式的好处在于，将庞大的迁移工作，分摊到每次的增删改查中，避免了一次性操作带来的性能的巨大损耗。  
缺点就是迁移过程中 h[0] 和 h[1] 同时存在的时间比较长，空间利用率低

**rehash迁移过程**  

<center>   
<img src="../../img/redis/dict/rehash%231.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">渐进式哈希步骤1</div>
</center>  
<center>   
<img src="../../img/redis/dict/rehash%232.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">渐进式哈希步骤2</div>
</center>  
<center>   
<img src="../../img/redis/dict/rehash%233.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">渐进式哈希步骤3</div>
</center>  
<center>   
<img src="../../img/redis/dict/rehash%234.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">渐进式哈希步骤4</div>
</center>  
<center>   
<img src="../../img/redis/dict/rehash%235.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">渐进式哈希步骤5</div>
</center>  
<center>   
<img src="../../img/redis/dict/rehash%236.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">渐进式哈希步骤6</div>
</center>  
<center>   
<img src="../../img/redis/dict/rehash%237.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">渐进式哈希步骤7</div>
</center>  


### 重点关注-字典扫描
*** 

- dictScan

dict.c 中的 dictScan 函数用来遍历字典，迭代其中的每个元素，该函数使用的算法很巧妙。  

遍历一个稳定的字段很简单，但 redis 中的字典因为有 rehash 的过程，使字典可能扩展，也可能缩小。这就带来了问题，如果在两次遍历中间，字典的结构发生了变化（扩展或者缩小），字典中的元素所在的位置相应的会发生变化，那如何保证字典中原有的元素都可以被遍历？又如何能尽可能少的重复迭代呢？这就是该算法的精妙所在，使用该算法，可以做到下面两点：  
- 开始遍历那一刻的所有元素，只要不被删除，肯定能被遍历到，不管字典扩展还是缩小  
- 该算法可能会返回重复元素，但是已经把返回重复元素的可能性降到了最低  

**游标cursor的演变**  
该算法使用了游标 cursor 来遍历字典，它表示每次要访问的 bucket 的索引。bucket 中保存了一个链表，因此每次迭代都会把该 bucket 的链表中的所有元素都遍历一遍。  
第一次迭代时，cursor 置为 0，dictScan 函数的返回值作为下一个 cursor 再次调用 dictScan，最终，dictScan 函数返回0 表示迭代结束。  
首先看一下 cursor 的演变过程，也是该算法的核心所在。这里 cursor 的演变采用了 reverse binary iteration 方法，也就是每次向 cursor 的最高位加1，并向低位方向进位。  

首先看下 cursor 的演变过程：  
```CPP
void test_dictScan_cursor(int tablesize)
{
    unsigned long v;
    unsigned long m0;

    v = 0;
    m0 = tablesize-1;

    printbits(v, (int)log2(tablesize));
    printf(" --> ");

    do
    {   
        v |= ~m0;
        v = rev(v);
        v++;      
        v = rev(v);

        printbits(v, (int)log2(tablesize));
        printf(" --> ");
    }while (v != 0);    
    
    printf("\b\b\b\b\b     \n");
}
```
以 tablesize 为 8 和 16 分别运行该函数，结果如下： 
> 8: 000 => 100 => 010 => 110 => 001 => 101 => 011 => 111 => 000  
> 16: 0000 => 1000 => 0100 => 1100 => 0010 => 1010 => 1110 => 0001 => 1001 => 0101 => 1101 => 0011 => 1011 => 0111 => 1111 => 0000

所谓的 reverse binary itreation 方法，也就是每次向 v 的最高位加 1，并向低位方向进位。比如 1101 的下一个数是 0011，因为 1101 的前三个数为 110，最高位加 1，并且向低位进位就是 001，所以最终得到 0011.  

在 redis 中，字典的哈希表长度始终为 2^n 。因此 m0 始终是 一个低 n 位全为 1 ，其余位全为 0 的数。
整个计算过程，都是在 v 的低 n 位数中进行的，比如长度为 16 的哈希表，则 n=4，因此 v 是从 0到15 这几个数之间的转换。  
下面解释一下计算过程：  
第一步： v|=~m0  //用于保留 v 的低 n 位数，其余位全置为 1  
<center>   
<img src="../../img/redis/dict/rbi%23step1.png" class="img-responsive img-centered" alt="image-alt">
</center>  
第二步： v = rev(v) //将 v 的二进制位进行翻转，所以 v 的低 n 位数成了 高 n 位数，并且进行了翻转  
<center>   
<img src="../../img/redis/dict/rbi%23step2.png" class="img-responsive img-centered" alt="image-alt">
</center>  
第三步： v++  
<center>   
<img src="../../img/redis/dict/rbi%23step3.png" class="img-responsive img-centered" alt="image-alt">
</center>  
最后一步： v = rev(v) //再次翻转  
<center>   
<img src="../../img/redis/dict/rbi%23step4.png" class="img-responsive img-centered" alt="image-alt">
</center>  
因此，最终得到的新 v，就是向最高位加 1，且向低位方向进位。  

**为什么要这个设计**  
这样设计的原因就在于，字典中的哈希表有可能扩展，也有可能缩小。在字典不稳定的情况下，既要遍历到所有没被删除的元素，又要尽可能较少的重复遍历。  
下面详细解释一下这样设计好处，以及为什么不是按照正常的 0,1,2,3... 这样的顺序迭代？  

计算一个哈希表节点索引的方法是 hashkey&mask ，其中，mask 的值永远是哈希表大小减 1。哈希表长度为 8，则 mask 为 111，因此，节点的索引值就取决于 hashkey 的低三位，假设是 abc，如果哈希表长度为16，则 mask 为 1111，同样的节点计算得到的哈希值不变，而索引值是 ?abc ，其中 ? 既可能是 0，也可能是 1，也就是说，该节点在长度为 16 的哈希表中，索引是 0abc 或者 1abc。以此类推，如何哈希表长度为 32，则该节点的索引是 00abc，01abc，10abc或者11abc 中的一个。  
重新看一下该算法中，哈希表长度分别为 8 和 16时，cursor 变化过程：  
> 8: 000 => 100 => 010 => 110 => 001 => 101 => 011 => 111 => 000  
> 16: 0000 => 1000 => 0100 => 1100 => 0010 => 1010 => 1110 => 0001 => 1001 => 0101 => 1101 => 0011 => 1011 => 0111 => 1111 => 0000

哈希表长度为 8 时，第 i 个 cursor（0<=i<=7），扩展到长度为 16 的哈希表中，对应的 cursor 是 2i和2i+1，它们是相邻的，这点很重要。  

首先是 __字典扩展__ 的情况，假设当前字典哈希表长度为 8，在迭代完索引为 010 的 bucket 之后，下一个 cursor 为 110。假设在下一次迭代前，字典哈希表长度扩展成了 16,110 这个 cursor，在长度为 16 的情况下，就成了 0110，因此开始迭代索引为 0110 的 bucket 中的节点。  
在长度为 8 时，已经迭代过的 cursor 分别是： 000,100,010。哈希表长度扩展到 16 后，在这些索引的 bucket 中的节点，分布到新的 bucket 中，新 bucket 的索引将会是：0000,1000,0100,1100,0010,1010。而这些，正好是将要迭代的 0110 之前的索引，从 0110 开始，按照长度为 16 的哈希表 cursor 变化过程迭代下去，这样既不会漏掉节点，也不会迭代重复的节点。  

再看下 __字典缩小__ 的情况，也就是由 16 缩小为 8。在长度为 16 时，迭代完 0100 的 cursor 之后，下一个 cursor 为 1100，假设此时哈希表长度缩小为 8。1100 这个cursor，在长度为 8 的情况下，就 成了 100. 因此开始迭代索引为 100 的 bucket 中的节点。  
在长度为 16 时，已经迭代过的 cursor 是：0000,1000,0100，哈希表长度缩小后，这些索引的 bucket 中的节点，分布到新的 bucket 中，新 bucket 的索引将会是 000 和 100.现在要从索引为 100 的 bucket 开始迭代，这样不会漏掉节点，但是之前长度为 16 时，索引为 0100 中的节点会被重复迭代，然而，也就仅 0100 这一个 bucket 中的节点会重复而已。  
原哈希表长度为 x，缩小后长度为 y，则最多会有 x/y - 1 个原 bucket 的节点会被重复迭代。比如由 16 缩小为 8，则最多就有 1 个 bucket 节点会重复迭代，要是由 32 缩小为 8，则最多会有 3 个。  
当然也有可能不产生重复迭代，还是从 16 缩小为 8 的情况，如果已经迭代完 1100，下一个 cursor 为 0010，此时长度缩小为 8，cursor 就成了 010。  
长度为 16 时，已经迭代过的 cursor 为 0000,1000,0100,1100，长度缩小后，这些 cursor 对应到新的索引是 000 和 100，正好是 010 之前的索引，从 010 开始，按照长度为 8 的 cursor 走下去，不会漏掉节点，也不会重复节点。  
所以说这种算法，保证了能迭代完所有节点而不会漏掉，又能尽可能少的重复遍历。  

如果按照正常的顺序迭代，下面分别是长度为 8 和 16 对应的 cursor 变化过程：  
> 8: 000 --> 001 --> 010 --> 011 --> 100 --> 101 --> 110 --> 111 --> 000
> 16: 0000 --> 0001 --> 0010 --> 0011 --> 0100 --> 0101 --> 0110 --> 0111 --> 1000 --> 1001 --> 1010 --> 1011 --> 1100 --> 1101 --> 1110 --> 1111 --> 0000

字典扩展的情况下，当前字典哈希表长度为 8，假设在迭代完 cursor 为 010 的 bucket 之后，下一个 cursor 为 011。迭代 011 之前，字典长度扩展成了 16，011 这个 cursor ，在长度为 16 的情况下，就成了 0011，因此开始迭代索引为 0011 的 bucket 中的节点。  
在长度为 8 时，已经迭代过的 cursor 是：000,001,010. 哈希表长度扩展到 16 后，这些索引的 bucket 中的节点，会分布到新的 bucket 中，新 bucket 的索引将会是：0000,1000,0001,1001,0010和1010.现在要开始迭代的 cursor 为 0011，而 1000,1001,1010 这些 bucket 中的节点在后续还是会遍历到，这就产生了重复遍历。  
虽然这种情况不会发生漏掉节点的情况，但是肯定会有重复的情况发生，而且长度变化发生的时机越晚，重复遍历的节点越多，比如长度为 8 时，迭代完 110 后，下一个 cursor 为 111，长度扩展为 16 后，这个 cursor 就成了 0111.  
长度为 8 时，已经迭代过的 cursor 为 000,001,010，011,100，101,110扩展到长度为 16 的哈希表中，这些 bucket 中的节点会分布到索引为：0000,1000,0001,1001,0010,1010,0011,1011,0100,1100,0101,1101,0110，1110. 现在长度为 16，要开始迭代 cursor 为 0111，而 1000,1001,1010,1011和1110 这些节点后续还会遍历到，重复的节点增多了。  

再看下长度缩小的情况下，长度由 16 缩小为 8.在长度为 16 时，迭代完 0100 的 cursor 之后，下一个 cursor 为 0101，此时长度缩小为 8。0101 这个 cursor，在长度为 8 的情况下，就成了 101.  
在长度为 16 时，尚未迭代过的 cursor 是：0101,0110,0111,1000,1001,1010,1011,1100,1101,1110,1111。这些 cursor，在哈希表长度缩小后，分配到新的 bucket 中，索引将会是：000,001,010,011，100,101,110,111.现在要开始迭代的 cursor 为 101，那 101 之前的 000,001,010,011,100 这些 cursor 就不会迭代了，这样，原有的某些节点就被漏掉了。  
另外，还是从 16 缩小为 8 的情况，如果已经迭代完 1100，下一个 cursor 为 1101，在长度为 8 的情况下，就成了 101.  
长度为 16 时，已经迭代过的 cursor 为 0000,0001,0010,0011,0100,0101,0110,0111,1000,1001,1010,1011,1100.这些 cursor，在哈希表长度缩小后，分配到新的 bucket 中，索引分别是：000,001,010,011,100,101,110,111。长度变为 8 后，从 101 开始，很明显，原来已经迭代过的 0101,0110,0111 就会产生重复迭代。  
因此顺序迭代不是一个满足要求的方法。  

扫描执行流程图如下：

<center>
<img src="../../img/redis/dict/dictScan.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">字典扫描执行流程</div>
</center>  

源码如下:

```CPP

/* dictScan() is used to iterate over the elements of a dictionary.
 *
 * dictScan 用于遍历字典中的元素
 *
 * Iterating works the following way:
 * 迭代的工作方式如下
 *
 * 1) Initially you call the function using a cursor (v) value of 0.
 * 1) 最初 你使用游标 0 来调用函数
 * 2) The function performs one step of the iteration, and returns the
 *    new cursor value you must use in the next call.
 * 2) 该函数执行迭代的一个步骤，返回你必须在下一个调用使用的新游标
 * 3) When the returned cursor is 0, the iteration is complete.
 * 3) 当返回的游标为0，迭代就已经完成
 *
 * The function guarantees all elements present in the
 * dictionary get returned between the start and end of the iteration.
 * However it is possible some elements get returned multiple times.
 * 该函数保证在迭代开始和结束之前返回字典中存在的所有元素。
 * 但是有可能一些元素返回多次
 *
 * For every element returned, the callback argument 'fn' is
 * called with 'privdata' as first argument and the dictionary entry
 * 'de' as second argument.
 * 对于每个返回的元素，回调参数 fn 被调用，private 作为第一个参数，字典条目 de 作为第二个参数
 *
 * HOW IT WORKS.
 * 他是如何工作的
 *
 * The iteration algorithm was designed by Pieter Noordhuis.
 * The main idea is to increment a cursor starting from the higher order
 * bits. That is, instead of incrementing the cursor normally, the bits
 * of the cursor are reversed, then the cursor is incremented, and finally
 * the bits are reversed again.
 * 这个迭代算法是由 Pieter Noordhuis 设计的。
 * 主要思想是将游标从二进制高位递增。也就是说，不是正常的增加游标，而是对光标的位进行反转，然后对光标进行递增，最后再次对位进行反转。
 *
 * This strategy is needed because the hash table may be resized between
 * iteration calls.
 * 这个策略是必须的，因为哈希表可能会在迭代调用之间调整大小
 *
 * dict.c hash tables are always power of two in size, and they
 * use chaining, so the position of an element in a given table is given
 * by computing the bitwise AND between Hash(key) and SIZE-1
 * (where SIZE-1 is always the mask that is equivalent to taking the rest
 *  of the division between the Hash of the key and SIZE).
 * 哈希表的大小总是2的幂次，并且他们使用链，因此给定表中元素的位置是通过计算 哈希key和 size-1 之间的按位与来给出的。
 * （其中 size-1 总是掩码，相当于取键的哈希值和size之间的剩余部分）
 *
 * For example if the current hash table size is 16, the mask is
 * (in binary) 1111. The position of a key in the hash table will always be
 * the last four bits of the hash output, and so forth.
 * 例如当前哈希表大小是16，二进制为1111。
 * 键在哈希表中的位置始终是哈希输出的最后四位，以此类推
 *
 * WHAT HAPPENS IF THE TABLE CHANGES IN SIZE?
 * 如果表的大小改变会发生什么？
 *
 * If the hash table grows, elements can go anywhere in one multiple of
 * the old bucket: for example let's say we already iterated with
 * a 4 bit cursor 1100 (the mask is 1111 because hash table size = 16).
 * 如果哈希表增长，元素可以在旧桶的一个倍数内移动到任何地方。
 * 举例来说：假设我们已经迭代了一个 4 位游标 1100（掩码是1111，因为哈希表大小为16）
 *
 * If the hash table will be resized to 64 elements, then the new mask will
 * be 111111. The new buckets you obtain by substituting in ??1100
 * with either 0 or 1 can be targeted only by keys we already visited
 * when scanning the bucket 1100 in the smaller hash table.
 * 如果哈希表大小调整到64个元素，新掩码将为 111111。
 * 通过将 ??1100 替换为0或1而获得的新桶只能通过我们在较小的哈希表中扫描 1100 时已经访问过的键来定位
 *
 * By iterating the higher bits first, because of the inverted counter, the
 * cursor does not need to restart if the table size gets bigger. It will
 * continue iterating using cursors without '1100' at the end, and also
 * without any other combination of the final 4 bits already explored.
 * 通过首先迭代较高的位，由于反向计数器，如果表大小变大，游标不需要重新启动。
 * 它将继续使用游标进行迭代，最后没有 1100 ，也没有最后4位的任何其他组合。
 *
 * Similarly when the table size shrinks over time, for example going from
 * 16 to 8, if a combination of the lower three bits (the mask for size 8
 * is 111) were already completely explored, it would not be visited again
 * because we are sure we tried, for example, both 0111 and 1111 (all the
 * variations of the higher bit) so we don't need to test it again.
 * 同样，当表大小随着时间的推移而缩小时，例如从16到8，如果已经完全探索了较低三位的组合（大小为8的掩码是111），则不会再次访问它，因为我们确信已经尝试过，
 * 例如 0111和1111 （较高位的所有变体），因此我们不会再尝试它
 *
 * WAIT... YOU HAVE *TWO* TABLES DURING REHASHING!
 * 等等，你有两个哈希表在重哈希
 *
 * Yes, this is true, but we always iterate the smaller table first, then
 * we test all the expansions of the current cursor into the larger
 * table. For example if the current cursor is 101 and we also have a
 * larger table of size 16, we also test (0)101 and (1)101 inside the larger
 * table. This reduces the problem back to having only one table, where
 * the larger one, if it exists, is just an expansion of the smaller one.
 * 是的，这是真的，但是我们总是先迭代较小的表，然后我们测试当前游标在较大表中的所有扩展。
 * 例如当前游标是 101 而且我们有大表的大小为16，我们在大表中验证 0101 和 1101.
 * 这将问题减少到只有一个表，其中较大的表（如果存在的话）只是较小表的扩展
 *
 *
 * LIMITATIONS
 * 限制
 *
 * This iterator is completely stateless, and this is a huge advantage,
 * including no additional memory used.
 * 这个迭代器是完全无状态的，这是一个巨大的优势，包括不使用额外的内存
 *
 * The disadvantages resulting from this design are:
 * 这种设计的缺点是
 *
 * 1) It is possible we return elements more than once. However this is usually
 *    easy to deal with in the application level.
 * 1) 我们可能多次返回相同元素。但是它通常在程序级别易于处理
 * 2) The iterator must return multiple elements per call, as it needs to always
 *    return all the keys chained in a given bucket, and all the expansions, so
 *    we are sure we don't miss keys moving during rehashing.
 * 2) 迭代器每次调用必须返回多个元素，因为它总是需要返回给定桶中链接的所有键，以及所有的展开，所以我们确保在重新散列期间不会错过键的移动
 * 3) The reverse cursor is somewhat hard to understand at first, but this
 *    comment is supposed to help.
 * 3) 反向光标一开始有点难以理解，但是这条注释应该会有所帮助
 */
unsigned long dictScan(dict *d,
                       unsigned long v, //对应的游标 从0开始
                       dictScanFunction *fn,
                       dictScanBucketFunction* bucketfn,
                       void *privdata)
{
    //初始化 表0地址和表1地址
    dictht *t0, *t1;
    //初始化当前哈希节点和下一个哈希节点
    const dictEntry *de, *next;
    //初始化 表0和表1对应的掩码
    unsigned long m0, m1;

    //如果哈希表数量为0值返回
    if (dictSize(d) == 0) return 0;

    /* This is needed in case the scan callback tries to do dictFind or alike.
     * 这在扫描回调试图执行 dictFind 或类型操作时是需要的。
     * */
    //暂停重哈希
    dictPauseRehashing(d);

    //判断哈希表是在重哈希过程中
    //若不处于重哈希过程中直接根据 v 找到需要迭代的 Bucket索引，针对该bucket中链表中的所有节点，调用用户提供的fn函数
    if (!dictIsRehashing(d)) {
        //t0 指向表
        t0 = &(d->ht[0]);
        //表0对应的掩码
        m0 = t0->sizemask;

        /* Emit entries at cursor
         * 在游标处发出项
         * */
        //执行对应bucketfn方法
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        //取哈希对应索引位置的链表节点
        de = t0->table[v & m0];
        //遍历链表
        while (de) {
            //取下一个哈希节点
            next = de->next;
            //执行对应fn方法
            fn(privdata, de); //将这个链表里的数据全部入队，准备返回客户端
            //指向下一个哈希节点
            de = next;
        }

        /* Set unmasked bits so incrementing the reversed cursor
         * operates on the masked bits
         * 设置非掩码位，以增加反向游标对掩码位的操作
         * */
        //用于保留 v 的低n位数，其余位全置位1
        v |= ~m0;

        /* Increment the reverse cursor
         * 增加反向光标
         * */
        //主要作用就是游标高位+1
        //游标倒置，将v的二进制位进行翻转，所以，v的低n位数成了高n位数，并且进行翻转
        v = rev(v);
        //游标加1，低位加1
        v++;
        //游标继续倒置，导致高位加1
        v = rev(v);

    } else {
        //表0的位置
        t0 = &d->ht[0];
        //表1的位置
        t1 = &d->ht[1];

        /* Make sure t0 is the smaller and t1 is the bigger table
         * 保证 t0 的数量小于 t1 的数量
         * */
        //保证t0数量最小，为后续遍历做准备，若t0大于t1，会导致部分数据漏掉
        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }

        //表0 对应的掩码
        m0 = t0->sizemask;
        //表1 对应的掩码
        m1 = t1->sizemask;

        /* Emit entries at cursor */
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        //取哈希对应索引位置的链表节点
        de = t0->table[v & m0];
        //遍历链表 同上
        while (de) {
            //取下一个哈希节点
            next = de->next;
            //执行对应fn方法
            fn(privdata, de);
            //指向下一个哈希节点
            de = next;
        }

        /* Iterate over indices in larger table that are the expansion
         * of the index pointed to by the cursor in the smaller table
         * 迭代较大表中的索引，这些索引是游标在较小表中所指向的索引的扩展
         * */
        //同遍历表0
        do {
            /* Emit entries at cursor */
            if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);
            de = t1->table[v & m1];
            while (de) {
                next = de->next;
                fn(privdata, de);
                de = next;
            }

            /* Increment the reverse cursor not covered by the smaller mask.*/
            v |= ~m1;
            v = rev(v);
            v++;
            v = rev(v);

            /* Continue while bits covered by mask difference is non-zero
             * 当掩码差所覆盖的位不为零时继续执行
             * */
        } while (v & (m0 ^ m1)); //表示直到 v 的低m1-m0位 到 低m1位（指 v 的高位都没有1） 之间全为0为止
    }

    //释放重哈希标志
    dictResumeRehashing(d);

    return v;
}


/* Function to reverse bits. Algorithm from:
 * http://graphics.stanford.edu/~seander/bithacks.html#ReverseParallel 
 * 二进制反转  即 0000 0111  ->  1110 0000
 * */
static unsigned long rev(unsigned long v) {
    unsigned long s = CHAR_BIT * sizeof(v); // bit size; must be power of 2
    unsigned long mask = ~0UL;
    while ((s >>= 1) > 0) {
        mask ^= (mask << s);
        v = ((v >> s) & mask) | ((v << s) & ~mask);
    }
    return v;
}
```


> 相关参考：
> https://pdai.tech/md/db/nosql-redis/db-redis-x-redis-ds.html#%E5%AD%97%E5%85%B8-%E5%93%88%E5%B8%8C%E8%A1%A8-dict  
> https://juejin.cn/post/7096097985976598564  
> https://zhuanlan.zhihu.com/p/332480573  
> https://www.cnblogs.com/yinbiao/p/10766357.html  
> https://www.cnblogs.com/bhlsheji/p/5134465.html  
> https://www.cnblogs.com/chenchuxin/p/14191156.html#%E5%93%88%E5%B8%8C%E8%A1%A8  
> https://juejin.cn/post/7189826507110350904  
> https://juejin.cn/post/7065498940883337253  
> https://blog.csdn.net/yangbodong22011/article/details/78467583  
> https://juejin.cn/post/6844903688528461831  
> https://www.cnblogs.com/gqtcgq/p/7247075.html  
> http://chenzhenianqing.com/articles/1101.html  
> https://blog.csdn.net/damanchen/article/details/89468810  
> https://www.lixueduan.com/posts/redis/redis-scan/  
