---
layout: default
redis: true
modal-id: 30000002
date: 2024-06-24
img: pexels-keval-padhiyar-245700856-19915666.jpg
alt: image-alt
project-date: June 2024
client: Start Bootstrap
category: redis
subtitle: redisObject
description: redisObject
---

<center>   
<img src="../../img/redis/redisObject/redisObject.jpg" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">redisObject概览</div>
</center> 

&nbsp;&nbsp;&nbsp;&nbsp;上图展示了redisObject，Redis所有数据类型、Redis所有编码方式以及底层数据结构之间的关系。  
&nbsp;&nbsp;&nbsp;&nbsp;它反映了redis的每种对象其实都由 **对象结构（redisObject）** 与 **对应编码的数据结构** 组合而成，而每种对象类型对应若干编码方式，不同的编码方式所对应的底层数据结构是不同的。  
所以，我们需要从几个角度来研究：
- 对象设计机制：对象结构（redisObject）
- 编码类型和底层数据结构：对应编码的数据结构  


### 为什么设计redisObject对象    
   
&nbsp;&nbsp;&nbsp;&nbsp;在redis的命令中，用于对键进行处理的命令占了很大一部分，而对于键所保存的值的类型（键的类型），键能执行的命令又各不相同。
如：LPUSH和LLEN只能用于列表键，而SADD和SRANDMEMBER只能用于集合键等；另外一些命令，比如DEL、TTL和TYPE，可以用于任何类型的键；但是要正确实现这些命令，
必须为不同类型的键设置不同的处理方式，比如说，删除一个列表键和删除一个字符串键的操作过程就不太一样。  
&nbsp;&nbsp;&nbsp;&nbsp;故：**Redis必须让每个键都带有类型信息，使得程序可以检查键的类型，并为它选择合适的处理方式。**  
&nbsp;&nbsp;&nbsp;&nbsp;比如说，集合类型就可以由字典和整数集合两种不同的数据结构实现，但是当用户执行ZADD命令时，它不关心集合使用的是什么编码，只要Redis能按照ZADD命令的指示，将新元素添加到集合就可以了。  
&nbsp;&nbsp;&nbsp;&nbsp;这说明，**操作数据类型的命令除了要对键的类型进行检查之外，还需要根据数据类型的不同编码进行多态处理。**  
&nbsp;&nbsp;&nbsp;&nbsp;为了解决以上问题，Redis构建了自己的类型系统，这个系统的主要功能包括：
- redisObject对象
- 基于redisObject对象的类型检查
- 基于redisObject对象的显示多态函数
- 对redisObject进行分配、共享和销毁的机制

### redisObject数据结构  
&nbsp;&nbsp;&nbsp;&nbsp;redisObject是Redis类型系统的核心，数据库中的每个键、值以及Redis本身处理的参数，都表示为这种数据类型。  
``` C
typedef struct redisObject {
    unsigned type:4;            //redisObject的类型    4bit
    unsigned encoding:4;        //同一种类型的不同编码方式  4bit
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or  
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
                            //记录RedisObject的访问时间信息      24bit
                            //LRU时间（相对于全局 lru_clock）
                            //LFU数据(最低有效的8位频率和最高有效的16位访问时间)
    int refcount;           //引用计数      32bit
    void *ptr;              //指向底层实现数据结构的指针     64bit
} robj;
```  

redisObject占用 16 个字节( 4 + 4 + 24 + 32 + 64 = 128 位)。  
其中type、encoding和ptr是最重要的三个属性。  
- type记录了对象所保存的值的类型，它的值可能是以下常量中的一个：  

```C
/* A redis object, that is a type able to hold a string / list / set */

/* The actual Redis Object */
//真正的redis对象
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
```
- encoding记录了对象所保存的值的编码，它的值可能是以下常量中的一个：  

```C
/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object.
    对象编码。像字符串和哈希这种对象内部可以有多种方式标识。 encoding字段被设置为该对象的其中一个字段
 */
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```
- ptr是一个指针，指向实际保存值的数据结构，这个数据结构由type和encoding属性决定。  
举个例子，如果一个redisObject的type属性为 OJB_LIST ，encoding属性为 OBJ_ENCODING_QUICKLIST，那么这个对象就是一个Redis列表（List），它的值保存在一个QuickList的数据结构内，而ptr指针就指向quickList的对象；  

- lru属性：记录了对象最后一次被命令程序访问的时间  
&nbsp;&nbsp;&nbsp;&nbsp;空转时长：当前时间减去键的值对象lru时间，就是该键的空转时长。Redis idletime命令可以打印出给定键的空转时长。  
&nbsp;&nbsp;&nbsp;&nbsp;如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存。  


### 命令的类型检查和多态   

当执行一个处理数据类型命令的时候，redis执行以下步骤：
- 根据给定的key，在数据库字典中查找和他相对应的redisObject，如果没找到，就返回NULL；  
- 检查redisObject的type属性和执行命令所需的类型是否相符，如果不相符，返回类型错误；  
- 根据redisObject的encoding属性所指定的编码，选择合适的操作函数来处理底层的数据结构；  
- 返回数据结构的操作结果作为命令的返回值；  


### 对象共享  

&nbsp;&nbsp;&nbsp;&nbsp;redis一般会把一些常见的值放在一个共享对象中，这样可使程序避免了重复分配的麻烦，也节约了一些CPU时间。  
redis预分配的值对象如下：  
- 各种命令的返回值，比如成功时返回的OK，错误时返回的ERROR，命令入队事务时返回的QUEUE等等
- 包括0在内，小于REDIS_SHARED_INTEGERS的所有整数  

> 注意：共享对象只能被字典和双向链表这类能带有指针的数据结构使用。像整数集合和压缩列表这些只能保存字符串、整数等的内存数据结构。  

&nbsp;&nbsp;&nbsp;&nbsp;为什么redis不共享列表对象、哈希对象、集合对象、有序集合对象，只共享字符串对象？  
- 列表对象、哈希对象、集合对象、有序集合对象，本身可以包含字符串对象，复杂度较高
- 如果共享对象是保存字符串对象，那么验证操作的复杂度为O(1)
- 如果共享对象是保存字符串值的字符串对象，那么验证操作的复杂度是O(N)
- 如果共享对象是包含多个值的对象，其中值本身又是字符串对象，即其他对象中嵌套了字符串对象，比如列表对象、哈希对象，那么验证操作的复杂度将会是O(N^2)


### 引用计数以及对象的销毁      

&nbsp;&nbsp;&nbsp;&nbsp;redisObject中有refcount属性，是对象的引用计数，显然计数0那么就是可以回收。  
- 每个redisObject结构都带有一个refcount属性，指示这个对象被引用了多少次；
- 当新创建一个对象时，它的refcount属性被设置为1；
- 当对一个对象进行共享时，redis将这个对象的rfcount加一；
- 当使用完一个对象后，或者消除对一个对象的引用之后，程序将对象的rfcount减一；
- 当对象的rfcount降至0时，这个RedisObject结构，以及它引用的数据结构的内存都会被释放；  


### Redis底层数据结构
- 简单动态字符串 - sds
- 压缩列表 - ZipList
- 快表 - QuickList
- 字典/哈希表 - Dict
- 整数集 - IntSet
- 跳表 - ZSkipList  
下面就逐一介绍下以上数据结构。  

> https://pdai.tech/md/db/nosql-redis/db-redis-x-redis-object.html
> https://axlgrep.github.io/tech/redis-object.html