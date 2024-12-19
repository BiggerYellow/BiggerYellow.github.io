---
layout: default
redis: true
modal-id: 30000007
date: 2024-12-18
img: pexels-tomas-malik-793526-27244374.jpg
alt: image-alt
project-date: December 2024
client: Start Bootstrap
category: redis
subtitle: 整数集
description: intSet
---


### 整数集 - intSet 
***
整数集合（intSet）是集合类型的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis 就会使用整数集合作为集合键的底层实现。
有以下几个特点：  
- 升序排序，无重复元素  
- 可通过属性自定义编码方式 (int16_t/int32_t/int64_t)  
- 插入新元素时，如果新元素的类型比 intSet 的原编码类型长，那么要先进行升级操作（不可逆操作）  
- 节省内存  


### intSet 数据结构  
***   
```CPP
typedef struct intset {
    // 整数编码方式  取值有三个 INTSET_ENC_INT16、INTSET_ENC_INT32、INTSET_ENC_INT64  占4字节
    uint32_t encoding;
    // 代表存储的整数个数    占4字节
    uint32_t length;
    // 指向实际存储数值的连续内存区域，是一个数组；整数集合的每个元素都是contents数组的一个数组项（item），各个项在数组中按值大小从小到大有序排序，且数组中不包含任何重复项     占1字节
    int8_t contents[];
} intset;
```

字段含义：  
- encoding：数据编码，表示 intSet 中的每个数据元素用几个字节来存储。它有三种可能得取值：INTSET_ENC_INT16 表示每个元素用 2 个字节存储、INTSET_ENC_INT32 表示每个元素用 4 个字节存储、INTSET_ENC_INT64 表示每个元素用 8 个字节存储。因此，intSet 中存储的整数最多只能占用 64 bit  
- length：表示 intSet 中的元素个数。encoding 和 length 两个字段构成了 intSet 的头部（header）  
- contents：是一个柔性数组，表示 intSet 的 header 后面紧跟着数据元素。这个数组的总长度（即总字节数）等于 encoding*length。柔性数组在 Redis 的很多数据结构的定义中都出现过，用于表达一个偏移量。contents 需要单独为其分配空间，这部分内存不包含在 intSet 结构当中。  


整数集的结构如下所示：  

+--------+--------+--------+--------+--------+--------+  
| header |  data  |  data  |  data  |  data  |  data  |  
+--------+--------+--------+--------+--------+--------+   

注意，intSet 的存储直接涉及到内存编码，所以需要考虑主机的字节序问题。  
intrev32ifbe 的意思是 int32 reversal if big endian。即 如果当前主机字节序为大端序，那么将它的内存存储进行翻转操作。简言之，intSet 的所有成员存储方式都采用小端序。所以创建一个空的整数集合，内存分布如下：   

<center>   
<img src="../../img/redis/intset/整数集合内存分布.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">整数集合内存分布</div>
</center>

需要注意的是，intSet 可能会随着数据的添加而改变它的数据编码：  
最开始，新创建的 intSet 使用占用内存最小的 INTSET_ENC_INT16（2字节）作为数据编码，每添加一个新元素，则根据元素大小决定是否对数据编码进行升级。  

下面是添加数据的具体例子：

<center>   
<img src="../../img/redis/intset/redis_intset_add_example.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">整数集合添加元素示例</div>
</center> 

上图中：  
1. 新建的 intSet 只有一个 header，总共 8 个字节。其中 encoding=2，length=0  
2. 添加 13、5 两个元素之后，因为它们是比较小的整数，都能使用 2 个字节表示，所以 encoding 不变，值还是 2  
3. 当添加 32768 的时候，它不能再用 2 个字节来表示了（2个字节能表示的数据范围是 -2^15 ~ 2^15 - 1，而 32768 = 2^15 ，超过范围了），因此 encoding 必须升级到 INTSET_ENC_INT32（值为4），即用 4 个字节表示一个元素  
4. 在添加每个元素的过程中，intSet 始终保持从小到大有序  
5. 与 ziplist 类似，intSet 也是按小端序模式存储的。比如在上图中 intSet 添加完所有数据之后，表示 encoding 字段的 4 个字节应该解释成 0x00000004，而第 5 个数据应该解释成 0x000186A0 = 100000。  


intSet 与 zipList 相比：  
- zipList 可以存储任意二进制串，而 intSet 只能存储整数  
- zipList 是无需的，而 intSet 是从小到大有序的。因此，在 zipList 上查找只能遍历，而在 intSet 上可以进行二分查找，性能更高  
- zipList 可以对每个数据项进行不同的变长编码（每个数据项前面都有数据长度字段 len），而 intSet 只能整体使用一个统一的编码（encoding）  

### intSet 核心API 
***  

intset *intsetNew(void);  
intset *intsetAdd(intset *is, int64_t value, uint8_t *success);  
intset *intsetRemove(intset *is, int64_t value, int *success);  
uint8_t intsetFind(intset *is, int64_t value);  
int64_t intsetRandom(intset *is);  
uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value);  
uint32_t intsetLen(const intset *is);  
size_t intsetBlobLen(intset *is);  
int intsetValidateIntegrity(const unsigned char *is, size_t size, int deep);  


**源码解析**  

- intsetNew  

```CPP
/* Create an empty intset.
 * 创建空的整数集合
 * */
intset *intsetNew(void) {
    //分配内存
    intset *is = zmalloc(sizeof(intset));
    //初始化整数类型为 int16    intset的所有成员存储方式都采用小端序
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    //初始化集合长度为 0
    is->length = 0;
    return is;
}
```
创建空的整数集，分配内存及初始化属性字段。   

- intsetAdd  

```CPP
/* Insert an integer in the intset
 * 向整数集合中插入一个整数
 * */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    //获取该值对应的编码
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values.
     * 如果需要的话升级编码。如果我们需要升级，我们要知道 这个值应该被附加（如果>0）或加在前面（如果<0），因为它位于现有值的范围之外
     * */
    //如果待插入值的编码大于当前整数集合编码
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success.
         * 他总是成功，所以我们不需要追求 *success
         * */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found.
         * 如果该值已经存在于集合中则终止。
         * 这个调用将用正确的位置填充 pos 以便在找不到值时插入值
         * */
        if (intsetSearch(is,value,&pos)) {
            //找到则说明不用插入直接返回 说明插入未成功
            if (success) *success = 0;
            return is;
        }

        //该值不存在，首先扩容集合数量加1
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        //如果待插入位置在中间 将待插入位置的值全部移动到末尾
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    //将值插入到指定位置
    _intsetSet(is,pos,value);
    //更新长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}


/* Return the required encoding for the provided value.
 * 返回所提供值需要的编码
 * */
static uint8_t _intsetValueEncoding(int64_t v) {
    //若该值小于 32位的最小值 或 大于32位的最大值 则为 64位
    if (v < INT32_MIN || v > INT32_MAX)
        return INTSET_ENC_INT64;
    else if (v < INT16_MIN || v > INT16_MAX) //若该值小于16位的最小值 或 大于16位的最大值 则为 32位
        return INTSET_ENC_INT32;
    else
        return INTSET_ENC_INT16; //否则就是 16位
}

/* Upgrades the intset to a larger encoding and inserts the given integer.
 * 升级整数集合到更大的编码 插入给定的整数
 * */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    //当前整数集合编码
    uint8_t curenc = intrev32ifbe(is->encoding);
    //待插入值的编码
    uint8_t newenc = _intsetValueEncoding(value);
    //当前整数集合数量
    int length = intrev32ifbe(is->length);
    //拼接位置  如果小于0放在前面 如果大于0放在后面
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize
     * 首先设置新编码并扩容
     * */
    //设置新编码
    is->encoding = intrev32ifbe(newenc);
    //数量扩容加1
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset.
     * 从后到前升级，这样我们就不会覆盖
     * 请注意 prepend 比那里用来确保我们有空余空间在整数集的开头或结尾
     * */
    //递归处理 将原有整数插入到扩容后的集合中，需要注意将待插入索引空出来
    while(length--)
        //将整数插入到指定位置
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end.
     * 将值插入到开头或者末尾
     * */
    //如果 prepend为1插入到开头，为0插入到末尾
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    //更新整数集合大小 加1
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

/* Resize the intset
 * 整数集合扩容
 * */
static intset *intsetResize(intset *is, uint32_t len) {
    //计算最终大小 扩容后的数量*最新编码
    uint64_t size = (uint64_t)len*intrev32ifbe(is->encoding);
    assert(size <= SIZE_MAX - sizeof(intset));
    //重新分配内存 intset原始大小 + 最新数组大小
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}

/* Set the value at pos, using the configured encoding.
 * 将值插入到指定位置，并使用配置的编码
 * */
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}

/* Search for the position of "value". Return 1 when the value was found and
 * sets "pos" to the position of the value within the intset. Return 0 when
 * the value is not present in the intset and sets "pos" to the position
 * where "value" can be inserted.
 * 搜索value的位置。如果值被找到则返回1并将值在集合中的位置设置到 pos 中。
 * 如果值不存在于集合中返回0，并值可以被插入的位置设置到 pos 中
 * */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    //初始化变量
    //min = 0  最小
    //max = length-1 最大
    //mid = -1 中间位置
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    //cur = -1 当前位置
    int64_t cur = -1;

    /* The value can never be found when the set is empty
     * 如果集合为空则值永远不会被发现
     * */
    //集合长度为0 代表空集合
    if (intrev32ifbe(is->length) == 0) {
        //pos设置为0 表明可以插入到第一个位置
        if (pos) *pos = 0;
        //插入失败返回0
        return 0;
    } else {

        /* Check for the case where we know we cannot find the value,
         * but do know the insert position.
         * 检查我们找不到该值但是知道要插入哪里的情况
         * */
        //如果该值大于 最大索引对应的值
        if (value > _intsetGet(is,max)) {
            //说明待插入位置为最后
            if (pos) *pos = intrev32ifbe(is->length);
            //返回0
            return 0;
        } else if (value < _intsetGet(is,0)) { //如果该值小于最小索引对应的值
            //说明待插入位置为最开始
            if (pos) *pos = 0;
            //返回0
            return 0;
        }
    }

    //二分查找搜索找到对应的位置
    while(max >= min) {
        //取中间位置
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        //获取中间位置对应的值
        cur = _intsetGet(is,mid);
        //如果待插入值大于当前值  缩小左边
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {//如果待插入值小于当前值 缩小右边
            max = mid-1;
        } else {
            break;
        }
    }

    //如果当前值 等于 待插入值
    if (value == cur) {
        //设置位置为mid
        if (pos) *pos = mid;
        //返回1
        return 1;
    } else {
        //不存在则说明要插入到 二分查找排序的最小位置
        if (pos) *pos = min;
        //返回0
        return 0;
    }
}

static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    //*src 起始值
    //*dst 目标值
    void *src, *dst;
    //from 到 length 的字节数
    uint32_t bytes = intrev32ifbe(is->length)-from;
    //当前编码
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        //真正的起始值
        src = (int64_t*)is->contents+from;
        //真正的模板值
        dst = (int64_t*)is->contents+to;
        //增加字节大小
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }
    //结构移动到末尾
    memmove(dst,src,bytes);
}
```
集合插入元素流程图如下：  
<center>   
<img src="../../img/redis/intset/intsetAdd.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">插入流程图</div>
</center> 

主要思想就是找到插入位置，扩容集合大小并插入，最后更新结构。  
- 首先判断新插入的值对应的编码是否大于现有编码，大于则需要升级整数集合并将元素插入到最后  
- 若不用升级编码则判断该值是否存在于集合中同时找到待插入位置，若存在则直接返回（注意因为集合是有序的，使用二分查找提高效率）  
- 若不存在于集合中，那必定找到要插入的位置了，先将集合扩容，并将待插入位置的现有值全部后移一位腾出空间  
- 将值插入到指定位置，并更新集合长度    


- intsetRemove  

```CPP
/* Delete integer from intset
 * 从整数列表中删除整数值
 * */
intset *intsetRemove(intset *is, int64_t value, int *success) {
    //确定待删除值的编码
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    //成功标识 默认为0 失败
    if (success) *success = 0;

    //待删除值编码 小于等于 整数集合编码  且  该值存在于整数集合中
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        //取出集合数量
        uint32_t len = intrev32ifbe(is->length);

        /* We know we can delete */
        //此时表明可以删除  设置删除标识为1
        if (success) *success = 1;

        /* Overwrite value with tail and update length
         * 用尾部覆盖值 且 更新长度
         * */
        //如果待删除值的位置 在集合中  直接用 pos+1 位置的值覆盖 pos位置
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        //整数集合大小减1
        is = intsetResize(is,len-1);
        //更新结构长度字段
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```  
删除就简单很多了，直接查找元素是否存在于集合中，存在则删除元素并将集合大小减 1   

- intsetFind  

```CPP
/* Determine whether a value belongs to this set
 * 确定该值是否属于该集合
 * */
uint8_t intsetFind(intset *is, int64_t value) {
    //确定该值对应的编码
    uint8_t valenc = _intsetValueEncoding(value);
    //保证该类型要小于集合中的类型  且 集合中可以找到该值
    return valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,NULL);
}
```
查看该元素是否存在于集合中即可。  

- intsetGet  

```CPP
/* Get the value at the given position. When this position is
 * out of range the function returns 0, when in range it returns 1.
 * 返回指定位置的值。
 * 如果位置超过集合长度返回 0 ，在范围内返回 1
 * */
uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value) {
    //是否越界
    if (pos < intrev32ifbe(is->length)) {
        //查询索引对应的值
        *value = _intsetGet(is,pos);
        return 1;
    }
    return 0;
}
```
返回指定位置对应的值，成功则返回 1，失败返回0，同时值存储在 value 中。  


> 参考博客  
> http://zhangtielei.com/posts/blog-redis-intset.html  
> https://pdai.tech/md/db/nosql-redis/db-redis-x-redis-ds.html#%E6%95%B4%E6%95%B0%E9%9B%86-intset  
> https://juejin.cn/post/6952824550140674061
> https://developer.aliyun.com/article/1242611  
> https://redisbook.readthedocs.io/en/latest/compress-datastruct/intset.html  
> https://www.cnblogs.com/yinbiao/p/11249161.html  