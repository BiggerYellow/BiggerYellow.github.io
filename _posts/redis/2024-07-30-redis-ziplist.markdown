---
layout: default
redis: true
modal-id: 30000004
date: 2024-07-30
img: pexels-ahmetkurt-12661193.jpg
alt: image-alt
project-date: July 2024
client: Start Bootstrap
category: redis
subtitle: 压缩列表
description: ziplist
---

### 压缩列表 - ziplist    
>  ziplist 是为了提高存储效率而设计的一种特殊编码的双向链表。它可以存储字符串或整数，存储整数时是采用整数的二进制而不是字符串形式存储。它能在 O(1) 的时间复杂度下完成 list 两端的 push 和 pop 操作。但是因为每次操作都需要重新分配 ziplist 的内存，所以实际复杂度和 ziplist 的内存使用量相关。  


**为什么 ZipList 特别省内存**  
- ziplist 节省内存是相对于普通的 list 来说的，如果是普通的数组，那么它每个元素占用的内存是一样的且取决于最大的那个元素（很明显需要预留空间）  
- 所以 ziplist 在设计时就很容易想到尽量让每个元素按照实际的内容大小存储，所以增加 encoding 字段，针对不同的 encoding 来细化存储大小。  
- 这时候还需要解决的一个问题是遍历元素时如何定位下一个元素？在普通数组中每个元素定长，所以不需要考虑这个问题；但是 ziplist 中每个 data 占据的内存不一样，所以为了解决遍历，需要增加记录上一个元素的 length，所以增加了 prelen 字段。  

**ziplist 数据结构**

<center>   
<img src="../../img/redis/ziplist/ziplist结构图.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">ziplist结构图</div>
</center>  
由如下部分组成：  

`zlbytes`：uint32_t 类型，表示整个 ziplist 所占用内存的字节数，在执行内存分配时会被使用。  
`zltail`：uint32_t 类型，即上图到达 zlend 节点的偏移量，通过此偏移量可以快速定位到 zlend 节点。  
`zllen`：uint16_t 类型，ziplist 中节点的数量。当这个值小于 UINT16_MAX(65535) 时，这个值就是 ziplist 中节点的数量；当这个值等于 UINT16_MAX 时，节点的数量需要遍历整个 ziplist 才能计算得出。  
`entry`：ziplist 所保存的节点，各个节点的长度根据内容而定。  
`zlend`：uint8_t 类型，255 的二进制值 1111 1111(UINT8_MAX)，用于标记 ziplist 的末端。  

官方描述如下：  
```C
 * ZIPLIST OVERALL LAYOUT
 * ======================
 *
 * The general layout of the ziplist is as follows:
 * ziplist的总体布局如下：
 *
 *   4字节     4字节     2字节                                1字节
 * <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
 *
 * NOTE: all fields are stored in little endian, if not specified otherwise.
 * 注意：如果没有另外指定，所有字段都以小端序存储（高位高存，顺序存放）
 *
 * <uint32_t zlbytes> is an unsigned integer to hold the number of bytes that
 * the ziplist occupies, including the four bytes of the zlbytes field itself.
 * This value needs to be stored to be able to resize the entire structure
 * without the need to traverse it first.
 * <uint32_t zlbytes> 是一个无符号整数来保存 ziplist 占用的字节数，包含 zlbytes 本身的四个字节。
 * 需要存储此值以便在不需要首先遍历它的情况下调整整个结构的大小。
 *
 * <uint32_t zltail> is the offset to the last entry in the list. This allows
 * a pop operation on the far side of the list without the need for full
 * traversal.
 * <uint32_t zltail> 是列表中最后一个entry的偏移量。它使得一个从列表另一边的 pop 操作无需全部遍历，时间复杂度为O(1)。
 *
 * <uint16_t zllen> is the number of entries. When there are more than
 * 2^16-2 entries, this value is set to 2^16-1 and we need to traverse the
 * entire list to know how many items it holds.
 * <uint16_t zllen> 代表 entry 的数量。 当entry的数量大于等于 2^16-2（即253） 时，它的值被设置为 2^16-1(即254)，并且我们需要遍历整个列表获取数量。为什么不是255，因为zlend默认使用255。
 *
 * <uint8_t zlend> is a special entry representing the end of the ziplist.
 * Is encoded as a single byte equal to 255. No other normal entry starts
 * with a byte set to the value of 255.
 * <uint8_t zlend> 是一个特殊的entry代表ziplist的结尾。 被编码为等于255的单个字节。 没有其他正常entry被设置为255.
```  

**ziplist-entry 数据结构**  

第一种情况，一般结构为：`<prevlen> <encoding> <entry-data>`  
`prevlen` : 前一个 entry 的大小，编码方式见下文；  
`encoding`：不同的情况下值不同，用于表示当前 entry 的类型和长度；  
`entry-data`：真正用于存储 entry 表示的数据;  

第二种情况，在 entry 中存储的是 int 类型时，encoding 和 entry-data 会合并在 encoding 中表示，此时没有 entry-data 字段；  
redis 中，在存储数据时，会先尝试将 string 转换成 int 存储，节省空间；  
此时 entry 结构：`<prevlen> <encoding>`  


- prevlen编码  
当前一个元素长度小于 254 （255用于zlend） 的时候，prevlen长度为 1 个字节，值即为前一个 entry 的长度，如果长度大于等于 254 的时候，prevlen 用 5 个字节表示，第一个字节设置为 254 ，后面 4 个字节存储一个小端的无符号整型，表示前一个 entry 的长度：  
``<prevlen from 0 to 253> <encoding> <entry>`` //长度小于 254 结构  
`0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry>`  //长度大于等于 254 结构  

- encoding编码  
encoding 的长度和值根据保存的是 int 还是 string，还有数据的长度而定；  
前两位用来表示类型，当为 11 时，表示 entry 存储的是 int 类型，其他表示存储的是 string；  
**存储 string 时**:  
`|00pppppp|`：此时 encoding 长度为 1 个字节，该字节的后六位表示 entry 中存储的 string 长度，因为是 6 位，所以entry 中存储的 string 长度不能超过 63；  
`|01pppppp|qqqqqqqq|`：此时 encoding 长度为 2 个字节；此时 encoding 的后 14 位用来存储 string 长度，长度不能超过 16383(2^14 - 1)；  
`|10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt|`：此时 encoding 长度为 5 个字节，后面的 4 个字节用来表示 encoding 中存储的字符串长度，长度不能超过 2^32 - 1；  
**存储 int 时**：  
`|11000000|`：encoding 为 3 个字节，后 2 个字节表示一个 int16；  
`|11010000|`：encoding 为 5 个字节，后 4 个字节表示一个 int32；  
`|11100000|`：encoding 为 9 个字节，后 8 个字节表示一个 int64；  
`|11110000|`：encoding 为 4 个字节，后 3 个字节表示一个有符号整型；  
`|11111110|`：encoding 为 2 个字节，后 1 个字节表示一个有符号整型；  
`|1111xxxx|`：encoding 长度就只有一个字节，xxxx 在 0001 到 1101 之前，表示无符号整数 0-12。 因为 0000和1111 不能被使用，实际编码值为 1-13，所以应该减去 1 获取正确的值；   
`|11111111|`：ziplist 特殊节点结尾；  

官方描述如下：  
```C
 * ZIPLIST ENTRIES
 * ===============
 *
 * Every entry in the ziplist is prefixed by metadata that contains two pieces
 * of information. First, the length of the previous entry is stored to be
 * able to traverse the list from back to front. Second, the entry encoding is
 * provided. It represents the entry type, integer or string, and in the case
 * of strings it also represents the length of the string payload.
 * So a complete entry is stored like this:
 * ziplist中的每个节点都以包含两部分信息的元数据作为前缀。
 * 首先 将前一个节点的长度存储为能够从后到前遍历列表。
 * 第二 提供节点的编码，它代表节点的类型，整数或者字符串，在这种情况下的字符串，它也表示字符串有效载荷的长度。
 * 所以一个完整的节点按如下方式存储：
 *
 * <prevlen> <encoding> <entry-data>
 *
 * Sometimes the encoding represents the entry itself, like for small integers
 * as we'll see later. In such a case the <entry-data> part is missing, and we
 * could have just:
 * 有时编码代表节点自身，像后面看到的短整数。在这种情况下 <entry-data> 部分将消失，类型如下：
 *
 * <prevlen> <encoding>
 *
 * The length of the previous entry, <prevlen>, is encoded in the following way:
 * If this length is smaller than 254 bytes, it will only consume a single
 * byte representing the length as an unsinged 8 bit integer. When the length
 * is greater than or equal to 254, it will consume 5 bytes. The first byte is
 * set to 254 (FE) to indicate a larger value is following. The remaining 4
 * bytes take the length of the previous entry as value.
 * <prevlen> : 上一个节点的长度，按以下方式编码：
 * 如果长度小于254字节，它将消耗 1 字节表示长度为 8bit 的整数。
 * 当长度大于等于254，它将占用 5 字节。第一个字节设置为 254（FE） 表明紧跟着的是较大值。剩下的 4 个字节以前一个节点的长度作为值。
 *
 * So practically an entry is encoded in the following way:
 * 因此 实际上节点是按照以下方式编码的
 *
 * <prevlen from 0 to 253> <encoding> <entry>
 * prevlen = [0, 253]
 *
 * Or alternatively if the previous entry length is greater than 253 bytes
 * the following encoding is used:
 * 或者 如果上一个节点长度大于 253 字节使用下面编码
 *
 * 0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry>
 *
 * The encoding field of the entry depends on the content of the
 * entry. When the entry is a string, the first 2 bits of the encoding first
 * byte will hold the type of encoding used to store the length of the string,
 * followed by the actual length of the string. When the entry is an integer
 * the first 2 bits are both set to 1. The following 2 bits are used to specify
 * what kind of integer will be stored after this header. An overview of the
 * different types and encodings is as follows. The first byte is always enough
 * to determine the kind of entry.
 * 节点的编码字段取决于节点的内容。
 * 当节点是字符串时，编码的第一个字节的前 2 位将保存用于存储字符串的编码类型，然后是字符串的实际长度。
 * 当节点是整数时，前 2 位都设为 1，接下来的 2 位用来指定在这个头之后存储的整数类型。
 * 不同类型和编码的概述如下。第一个字节总是足够确定节点的类型
 *
 * 字符串
 * |00pppppp| - 1 byte
 *      String value with length less than or equal to 63 bytes (6 bits).
 *      "pppppp" represents the unsigned 6 bit length.
 *      长度小于等于 63(2^6-1) 字节的字符串， pppppp 代表无符号 6bit 长度
 * |01pppppp|qqqqqqqq| - 2 bytes
 *      String value with length less than or equal to 16383 bytes (14 bits).
 *      IMPORTANT: The 14 bit number is stored in big endian.
 *      长度小于等于 16383(2^14-1) 字节的字符串
 *      14位数字以大端序存储（高位低存，逆序存放）
 * |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
 *      String value with length greater than or equal to 16384 bytes.
 *      Only the 4 bytes following the first byte represents the length
 *      up to 2^32-1. The 6 lower bits of the first byte are not used and
 *      are set to zero.
 *      IMPORTANT: The 32 bit number is stored in big endian.
 *      长度大于等于 16384（2^14） 字节的字符串。
 *      只有第一个字节之后的4个字节表示长度即 2^32-1。首字节的 6 个低bit位未使用且设置为0.
 *      32 bit 位以大端序存储（高位低存，逆序存放）
 *
 * 整数
 * |11000000| - 3 bytes
 *      Integer encoded as int16_t (2 bytes).
 *      整数编码为 int16_t（2字节）
 * |11010000| - 5 bytes
 *      Integer encoded as int32_t (4 bytes).
 *      整数编码为 int32_t（4字节）
 * |11100000| - 9 bytes
 *      Integer encoded as int64_t (8 bytes).
 *      整数编码为 int64_t（8字节）
 * |11110000| - 4 bytes
 *      Integer encoded as 24 bit signed (3 bytes).
 *      整数编码为24位有符号 （3字节）
 * |11111110| - 2 bytes
 *      Integer encoded as 8 bit signed (1 byte).
 *      整数编码为 8bit有符号 （1字节）
 * |1111xxxx| - (with xxxx between 0001 and 1101) immediate 4 bit integer.
 *      Unsigned integer from 0 to 12. The encoded value is actually from
 *      1 to 13 because 0000 and 1111 can not be used, so 1 should be
 *      subtracted from the encoded 4 bit value to obtain the right value.
 *      XXXX 在 0001到1101之间，即4bit 整数。
 *      无符号整数从0到12。编码值实际从1到13，因为0000和1111不能被使用，所以应该从4bit中减去 1 来获取正确的值
 *
 * |11111111| - End of ziplist special entry.
 *      ziplist特殊节点结尾
 *
 * Like for the ziplist header, all the integers are represented in little
 * endian byte order, even when this code is compiled in big endian systems.
```

**zlentry 数据结构**
```C++
/* We use this function to receive information about a ziplist entry.
 * Note that this is not how the data is actually encoded, is just what we
 * get filled by a function in order to operate more easily.
 * 我们使用这个方法来接收 ziplist 节点的信息。
 * 注意这不是数据的实际编码方式，这只是为了更容易操作而由函数填充的内容
 */
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
                                 //用于编码前一项 len 的字节，存储上一个元素的长度数值所需要的字节数
    unsigned int prevrawlen;     /* Previous entry len. */
                                 //前一个节点的长度
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
                                 //用于编码节点的类型或长度的字节。 例如字符串由 1、2、5字节的头部。 整数通常使用 1 字节
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
                                 //用于表示真正节点的字节。 对于字符串来说就是字符串的长度，对于整型来说依赖数字的范围，可能是 1,2,3，4,8 或 0的4bit位
    unsigned int headersize;     /* prevrawlensize + lensize. */
                                 //头部长度 前一个节点 加 当前节点的长度
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
                                 //依赖节点的类型设置为 ZIP_STR_* or ZIP_INT_*。 然后对于 4bit位整数，可以假设为一个范围的值，必须进行范围检查
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
                                 //指向节点最开始的指针，即指向prev-entry-len(前一个节点长度) 字段
} zlentry;
```
上面说到列表元素的结构只有三个属性，分别是:  
- previous_entry_length: 前一个元素的字节长度
- encoding: 当前元素的编码
- content: 当前元素的内容

previous_entry_length 属性记录了前一个元素的字节长度，却有 1 字节还是 5 字节之分，前一个元素的长度也根据字节数的不同，获取方式也不同。
因此我们可以把这个可变属性拆成 2 个属性:  
- prevrawlensize: 存储前一个元素的长度所需要的字节数  
- prevrawlen: 前一个元素的长度  

encoding 属性表示当前元素的编码，记录着当前节点存储数据的类型及长度。 encoding 字段存储数据的长度也是采用变长的方式，可以是1、2、5字节，整数恒为 1 字节。  
encoding 值的前两位则能表示当前元素存储数据的类型。当 encoding 为 1111xxxx 时，xxxx还表示值的大小，此时 content 字段内容为空，所以连同 content 字段一起，我们可以把 encoding 属性拆分成三个属性:  
- lensize: 存储元素的长度数值所需要的字节数，可以为1、2、5字节，整数恒为 1 字节
- len: 表示元素的长度
- encoding: 标识是字节数组还是整数数组

那么如何获取当前元素的内容呢，Redis是这个做的: 
- *p: 定义一个 char 类型指针，该指针指向当前元素的起始位置
- headersize: headersize 表示当前元素的首部长度，即 prevrawlensize + lensize  

通过指针 p 偏移 headersize 即可得到元素内容。


那么 Redis 是如何对压缩列表进行解码的呢？  
通过 zipEntry 函数进行解码压缩列表元素，并存于 zlentry 结构体。  
```C++
/* Fills a struct with all information about an entry.
 * This function is the "unsafe" alternative to the one blow.
 * Generally, all function that return a pointer to an element in the ziplist
 * will assert that this element is valid, so it can be freely used.
 * Generally functions such ziplistGet assume the input pointer is already
 * validated (since it's the return value of another function).
 * 用节点的所有信息填充结构体。这个函数是不安全的替代方法。
 * 一般来说，所有返回指向 ziplist 中元素指针的函数都回断言该元素是有效的，因此可以自由使用。
 * 一般来说 像ziplistGet的方法，假设输入的指针已经是有效的（因为它是其他方法返回的指针）
 */
static inline void zipEntry(unsigned char *p, zlentry *e) {
    //根据 prevrawlensize字节数解码 prevrawlen长度
    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    //根据 prevrawlensize字节数解码 encoding
    ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
    //根据 prevrawlensize 和 encoding 解析节点长度及存储长度所需要的字节
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    assert(e->lensize != 0); /* check that encoding was valid. */
    //设置节点头部大小 = 上一个节点长度 + 该节点长度
    e->headersize = e->prevrawlensize + e->lensize;
    e->p = p;
}


/* Return the length of the previous element, and the number of bytes that
 * are used in order to encode the previous element length.
 * 'ptr' must point to the prevlen prefix of an entry (that encodes the
 * length of the previous entry in order to navigate the elements backward).
 * The length of the previous entry is stored in 'prevlen', the number of
 * bytes needed to encode the previous entry length are stored in
 * 'prevlensize'.
 * 返回上一个元素的长度 和 用于编码上一个元素长度所需的字节数量。
 * 'ptr'必须指向节点的 prevlen 前缀（它对前一个节点长度进行编码以便向后导航元素）
 * 上一个节点的长度保存在 prevlen 中，编码前一个节点长度所需的字节数保存在 prevlensize 中
 */
#define ZIP_DECODE_PREVLEN(ptr, prevlensize, prevlen) do {                     \
    /*解码 prevlensize 所占的字节数 要么为1字节要么为5字节，获取上一个节点真正的长度*/                                                                           \
    ZIP_DECODE_PREVLENSIZE(ptr, prevlensize);                                  \
    /*若prevlensize为1，则ptr的第一个字节即为上一个节点的长度*/                                                                           \
    if ((prevlensize) == 1) {                                                  \
        (prevlen) = (ptr)[0];                                                  \
    } else { /* prevlensize == 5 若字节长度等于5，则取后面4个字节即32位作为上一个节点的长度 */            \
        /* 读取连续的4个字节（即32位），并将这4个字节的数据组合成一个整数len。
         * 这里的组合方式是按照小端字节序（little-endian）进行的，即最高有效字节（MSB）存放在最高的内存地址，最低有效字节（LSB）存放在最低的内存地址。
         * (ptr)[1]是第一个字节，它不会被移动，成为prevlen的最低8位
         * (pre)[2]是第二个字节，它会被左移8位，成为整数prevlen的次低8位
         * (pre)[3]是第三个字节，它会被左移16位，成为整数的prevlen的次高8位
         * (pre)[4]是第四个字节，它会被左移24位，成为整数的prevlen的最高8为
         * */                                                                       \
        (prevlen) = ((ptr)[4] << 24) |                                         \
                    ((ptr)[3] << 16) |                                         \
                    ((ptr)[2] <<  8) |                                         \
                    ((ptr)[1]);                                                \
    }                                                                          \
} while(0)

/* Return the number of bytes used to encode the length of the previous
 * entry. The length is returned by setting the var 'prevlensize'.
 * 返回用于编码上一个节点长度的字节数。 通过设置变量 prevlensize 来返回长度
 */
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
    /*若指针的第一个元素小于ZIP_BIG_PREVLEN 代表 prevlensize 为 1字节，否则为 5 字节*/  \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
        (prevlensize) = 1;                                                     \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0)


/* Extract the encoding from the byte pointed by 'ptr' and set it into
 * 'encoding' field of the zlentry structure.
 * 从 ptr 所指向的字节中提取编码，并且将它设置到 zlentry 结构中的 encoding 字段
 * */
#define ZIP_ENTRY_ENCODING(ptr, encoding) do {  \
    (encoding) = ((ptr)[0]);                    \
    /* 若 encoding 小于ZIP_STR_MASK（字符串标志位），与ZIP_STR_MASK进行与运算获取 int类型编码 */                                            \
    if ((encoding) < ZIP_STR_MASK) (encoding) &= ZIP_STR_MASK; \
} while(0)


/* Decode the entry encoding type and data length (string length for strings,
 * number of bytes used for the integer for integer entries) encoded in 'ptr'.
 * The 'encoding' variable is input, extracted by the caller, the 'lensize'
 * variable will hold the number of bytes required to encode the entry
 * length, and the 'len' variable will hold the entry length.
 * On invalid encoding error, lensize is set to 0.
 * 解码节点中的编码类型和数据长度（对于字符串的字符串长度，对于整数项的整数所使用的字节数）。
 * encoding变量是输入，由调用者提取。变量 lensize 保留编码节点长度所需的字节。 变量 len 将保留节点的长度。
 * 在编码异常场景下，lensize将设置为0
 * */
#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len) do {                    \
    /*判断编码属于字节数组还是整数类型*/                                                                           \
    if ((encoding) < ZIP_STR_MASK) {                                           \
        /*如果存储长度等于 ZIP_STR_06B（00000000）*/                                                                       \
        if ((encoding) == ZIP_STR_06B) {                                       \
            /*需要 1 字节存储字符串长度*/                                                                   \
            (lensize) = 1;                                                     \
            /* 元素长度为 (ptr)[0] 和 111111 做与运算获取实际长度 */                                                                   \
            (len) = (ptr)[0] & 0x3f;                                           \
        } else if ((encoding) == ZIP_STR_14B) {/*如果存储长度等于 ZIP_STR_14B(0100 0000) */                                \
            (lensize) = 2;                                                     \
            (len) = (((ptr)[0] & 0x3f) << 8) | (ptr)[1];                       \
        } else if ((encoding) == ZIP_STR_32B) {                                \
            (lensize) = 5;                                                     \
             /* 读取连续的4个字节（即32位），并将这4个字节的数据组合成一个整数len。
              * 这里的组合方式是按照大端字节序（big-endian）进行的，即最高有效字节（MSB）存放在最低的内存地址，最低有效字节（LSB）存放在最高的内存地址。
              * (ptr)[1]是第一个字节，它会被左移24位，成为整数的prevlen的最高8为
              * (pre)[2]是第二个字节，它会被左移16位，成为整数的prevlen的次高8位
              * (pre)[3]是第三个字节，它会被左移8位，成为整数prevlen的次低8位
              * (pre)[4]是第四个字节，它不会被移动，成为prevlen的最低8位
              * */                                                                       \
            (len) = ((uint32_t)(ptr)[1] << 24) |                               \
                    ((uint32_t)(ptr)[2] << 16) |                               \
                    ((uint32_t)(ptr)[3] <<  8) |                               \
                    ((uint32_t)(ptr)[4]);                                      \
        } else {                                                               \
            (lensize) = 0; /* bad encoding, should be covered by a previous */ \
            (len) = 0;     /* ZIP_ASSERT_ENCODING / zipEncodingLenSize, or  */ \
                           /* match the lensize after this macro with 0.
                            * 异常场景，赋值0*/ \
        }                                                                      \
    } else {                                                                   \
        /*判断编码属于整数类型*/                                                                           \
        (lensize) = 1;                                                         \
        if ((encoding) == ZIP_INT_8B)  (len) = 1;  /* 若encoding = ZIP_INT_8B 长度为1字节，即可以用 8bit 位数据表示 */                            \
        else if ((encoding) == ZIP_INT_16B) (len) = 2;  /* 若encoding = ZIP_INT_16B 长度为2字节，即可以用 16bit 位数据表示 */                         \
        else if ((encoding) == ZIP_INT_24B) (len) = 3;  /* 若encoding = ZIP_INT_24B 长度为3字节，即可以用 24bit 位数据表示 */                         \
        else if ((encoding) == ZIP_INT_32B) (len) = 4;  /* 若encoding = ZIP_INT_32B 长度为4字节，即可以用 32bit 位数据表示 */                         \
        else if ((encoding) == ZIP_INT_64B) (len) = 8;  /* 若encoding = ZIP_INT_64B 长度为8字节，即可以用 64bit 位数据表示 */                         \
        else if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX)  /* 异常情况 长度设置为0 */ \
            (len) = 0; /* 4 bit immediate */                                   \
        else                                                                   \
            (lensize) = (len) = 0; /* bad encoding */                          \
    }                                                                          \
} while(0)
```
可以看到解码主要分为三个步骤:  
1. 解码 previous_entry_length 字段，此时入参 ptr 指向元素首地址  
2. 解码 encoding 字段，此时入参 ptr 指向元素首地址偏移 previous_entry_length 字段长度的位置  
3. 解码 len 字段，根据 encoding 分别解析计算节点长度


**创建压缩列表**  
源码如下：  
```C++
/* Create a new empty ziplist. */
//创建压缩列表
unsigned char *ziplistNew(void) {
    //初始化默认字节大小=11字节  即 zlbytes(4字节) + zltail(4字节) + zlen(2字节) + zlend(1字节)
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    //分配内存11字节大小
    unsigned char *zl = zmalloc(bytes);
    //初始化 zlbytes 字段
    //intrev32ifbe 意思为 int32 reversal if big endian，即如果当前主机字节序为大端序，那么将它的内存存储进行翻转操作。  见https://blog.csdn.net/WhereIsHeroFrom/article/details/84643017
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    //初始化 zltail 字段
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    //长度设置为0
    ZIPLIST_LENGTH(zl) = 0;
    //初始化结束标志位
    zl[bytes-1] = ZIP_END;
    return zl;
}
```
创建压缩列表的代码很简单，函数无输入参数，只需要分配初始化存储空间 11（4+4+2+1）个，并对 zlytes、zltail、zllen 和 zlend 字段初始化值，最后返回压缩列表的首地址。  


**插入元素**  
源码如下: 
```C++

/* Insert item at "p".
 * 向指针 p 中插入元素
 * zl:压缩列表字节大小
 * p:压缩列表指针
 * s: 待插入的节点
 * slen: 待插入的节点长度
 * */
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    //curlen 表示插入元素前压缩列表的长度
    //reqlen表示新插入元素的长度
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, newlen;
    //prevlensize 表示前一个字节的长度
    //prevlen 表示存储前一个字节需要的字节数
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    //nextdiff 表示插入元素后一个元素长度的变化，取值可能是0（长度不变），4（长度增加4）或-4（长度减少4）
    int nextdiff = 0;
    //encoding 用于存储当前元素编码
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized.
                                    初始化以避免警告。使用该值可以很简单的看出未初始化
                                    */
    zlentry tail;

    /* Find out prevlen for the entry that is inserted.
     * 计算已插入节点的 prevlen
     * */
    //找出待插入节点的前置节点长度，有三种场景
    if (p[0] != ZIP_END) {
        //若非结尾标识，直接计算上一个节点的长度
        //1.如果p[0] 不指向列表末尾，说明列表非空，并且 p 指向其中一个节点，所以新插入节点的前置节点长度可以通过节点 p 指向的节点信息中获取
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen); //通过 ZIP_DECODE_PREVLEN 方法获取 prevlen 长度
    } else {
        //获取尾节点位置，用来判断当前压缩列表是否为空列表
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        //若末尾节点非结束标识
        //2. 如果尾节点指针不指向压缩列表末尾，说明当前压缩列表不为空，那么新插入节点的前置节点长度就是尾节点的长度
        if (ptail[0] != ZIP_END) {
            //获取尾节点的总长度 头部大小+总长度
            prevlen = zipRawEntryLengthSafe(zl, curlen, ptail);
        }
        //3. 第三种情况就是 p 指向压缩列表末尾，但是压缩列表中节点为空，所以 p 的前置节点长度为 0 ，故不做处理
    }

    /* See if the entry can be encoded
     * 尝试将节点从String转为int，计算当前节点的长度
     * */
    //s指针指向新增节点数据  slen为数据长度
    //确定数据编码，为整数时返回对应固定长度，为字符串时使用slen
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding
         * encoding 已经设置为最合适的整数编码，并获取编码长度
         * */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
         * string length to figure out how to encode it.
         * encoding 未受影响，但是 zipStoreEntryEncoding 将使用字符串长度来决定如何编码
         * */
        reqlen = slen;
    }

    /* We need space for both the length of the previous entry and
     * the length of the payload.
     * 我们需要为前一个节点的长度和有效负载的长度分配空间 即更新当前节点场景
     * */
    //编码前置节点长度所需字节数
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    //编码当前字符串长度所需字节数
    //此时 reqlen 为新加入节点的整体长度
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field.
     * 当插入的位置不在尾部时，我们需要保证下一个节点可以在 prevlen 字段中保存当前节点的长度
     * */
    int forcelarge = 0; //在nextdiff==-4 && reqlen<4 时使用，该条件说明，插入元素导致压缩列表变小了，即函数 ziplistResize内部调用 realloc 重新分配空间小于 zl 指向的空间，此时 realloc 会将多余空间回收，导致数据丢失（丢失了尾部），所以为了避免这种情况，我们使用forcelarge来标记这种情况，并将 nextdiff 置为0，详情见https://segmentfault.com/a/1190000018878466?utm_source=tag-newest
    //计算原来 p 位置上的节点将要保存的 prevlen（当前待插入节点的大小） 是否变化
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    //连锁更新的时候会出现问题
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0; //将nextdiff 设置为0，此时内存重分配不会出现回收空间的情况，造成数据丢失
        forcelarge = 1;
    }

    /* Store offset because a realloc may change the address of zl.
     * 重新分配内存肯呢个会改变 zl 的地址，所以存储偏移
     * */
    offset = p-zl; //偏移量，用来表示 p 相对于压缩列表首地址的偏移量，由于重新分配了空间，新元素插入的位置指针 p 会时效，可以预先计算好指针 p 对于压缩列表首地址的偏移量，待空间分配之后再便宜
    //最新空间大小   当前压缩列表大小 + 插入元素大小 + 差值长度
    newlen = curlen+reqlen+nextdiff;
    //重新分配压缩列表大小
    zl = ziplistResize(zl,newlen);
    //根据偏移找到新插入元素 P 的位置
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset.
     * 必要时应用内存移动并更新尾偏移量
     * */
    //非空列表插入
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes
         * 因为是 ZIP_END 字节，所以减去1
         * */
        //将 p 节点后移（没有移动 p 节点前一节点长度信息），留出当前节点位置
        //p+reqlen: 偏移量是这个，就是将原来的数据移动到新插入节点之后
        //curlen-offset-1+nextdiff: 移动的长度，就是位置 P 之后的所有元素的长度 -1（结束符大小恒为 0XFF，不需要移动），再加上 nextdiff（下一个元素长度的变化）
        //p-nextdiff: 表示从哪个位置需要复制移动，因为下一个元素长度会发生变化，所以需要提前预留出这部分空间，就多复制一块空间，到时候覆盖即可
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry.
         * 在下一个节点中编码节点原始长度 并且判断是否需要加大上一个节点长度
         * */
        //写入 p 节点以前一个节点长度信息（要插入节点的长度）
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        /* Update offset for tail
         * 更新末尾节点的偏移
         * */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset.
         * 当尾部包含多个节点时，还需要考虑 nextdiff，否则，prevlen大小的变化不会对尾部偏移量产生影响
         * */
        assert(zipEntrySafe(zl, newlen, p+reqlen, &tail, 1));
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail.
         * 如果在末尾 则该节点将变成新的末尾节点
         * */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist
     * 当 nextdiff 不为0，下一个节点的原始长度已经改变，所以需要级联更新
     * */
    if (nextdiff != 0) {
        //计算偏移
        offset = p-zl;
        //级联更新压缩链表
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        //更新后添加偏移
        p = zl+offset;
    }

    /* Write the entry
     * 写节点信息
     * */
    //写入前一节点长度信息
    p += zipStorePrevEntryLength(p,prevlen);
    //写入节点编码与长度信息
    p += zipStoreEntryEncoding(p,encoding,slen);
    //如果是字符串则分配内存
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        //是整数则添加整数值
        zipSaveInteger(p,value,encoding);
    }
    //压缩列表长度加1
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}


/* Return the length of the previous element, and the number of bytes that
 * are used in order to encode the previous element length.
 * 'ptr' must point to the prevlen prefix of an entry (that encodes the
 * length of the previous entry in order to navigate the elements backward).
 * The length of the previous entry is stored in 'prevlen', the number of
 * bytes needed to encode the previous entry length are stored in
 * 'prevlensize'.
 * 返回上一个元素的长度 和 用于编码上一个元素长度所需的字节数量。
 * 'ptr'必须指向节点的 prevlen 前缀（它对前一个节点长度进行编码以便向后导航元素）
 * 上一个节点的长度保存在 prevlen 中，编码前一个节点长度所需的字节数保存在 prevlensize 中
 */
#define ZIP_DECODE_PREVLEN(ptr, prevlensize, prevlen) do {                     \
    /*解码 prevlensize 所占的字节数 要么为1字节要么为5字节，获取上一个节点真正的长度*/                                                                           \
    ZIP_DECODE_PREVLENSIZE(ptr, prevlensize);                                  \
    /*若prevlensize为1，则ptr的第一个字节即为上一个节点的长度*/                                                                           \
    if ((prevlensize) == 1) {                                                  \
        (prevlen) = (ptr)[0];                                                  \
    } else { /* prevlensize == 5 若字节长度等于5，则取后面4个字节即32位作为上一个节点的长度 */            \
        /* 读取连续的4个字节（即32位），并将这4个字节的数据组合成一个整数len。
         * 这里的组合方式是按照小端字节序（little-endian）进行的，即最高有效字节（MSB）存放在最高的内存地址，最低有效字节（LSB）存放在最低的内存地址。
         * (ptr)[1]是第一个字节，它不会被移动，成为prevlen的最低8位
         * (pre)[2]是第二个字节，它会被左移8位，成为整数prevlen的次低8位
         * (pre)[3]是第三个字节，它会被左移16位，成为整数的prevlen的次高8位
         * (pre)[4]是第四个字节，它会被左移24位，成为整数的prevlen的最高8为
         * */                                                                       \
        (prevlen) = ((ptr)[4] << 24) |                                         \
                    ((ptr)[3] << 16) |                                         \
                    ((ptr)[2] <<  8) |                                         \
                    ((ptr)[1]);                                                \
    }                                                                          \
} while(0)


/* Return the total number of bytes used by the entry pointed to by 'p'.
 * 返回 p 所指向的节点所使用的总字节数
 * */
static inline unsigned int zipRawEntryLengthSafe(unsigned char* zl, size_t zlbytes, unsigned char *p) {
    zlentry e;
    assert(zipEntrySafe(zl, zlbytes, p, &e, 0));
    return e.headersize + e.len;
}


/* Check if string pointed to by 'entry' can be encoded as an integer.
 * Stores the integer value in 'v' and its encoding in 'encoding'.
 * 检查 entry 所指向的字符串是否可以被编码为整数。
 * 将整数值存储在 v 中，它的编码存储在 encoding 字段中
 * */
int zipTryEncoding(unsigned char *entry, unsigned int entrylen, long long *v, unsigned char *encoding) {
    long long value;

    if (entrylen >= 32 || entrylen == 0) return 0;
    if (string2ll((char*)entry,entrylen,&value)) {
        /* Great, the string can be encoded. Check what's the smallest
         * of our encoding types that can hold this value. */
        if (value >= 0 && value <= 12) {
            *encoding = ZIP_INT_IMM_MIN+value;
        } else if (value >= INT8_MIN && value <= INT8_MAX) {
            *encoding = ZIP_INT_8B;
        } else if (value >= INT16_MIN && value <= INT16_MAX) {
            *encoding = ZIP_INT_16B;
        } else if (value >= INT24_MIN && value <= INT24_MAX) {
            *encoding = ZIP_INT_24B;
        } else if (value >= INT32_MIN && value <= INT32_MAX) {
            *encoding = ZIP_INT_32B;
        } else {
            *encoding = ZIP_INT_64B;
        }
        *v = value;
        return 1;
    }
    return 0;
}


/* Return bytes needed to store integer encoded by 'encoding'
 * 返回存储由 encoding 编码的整数所需的字节数
 * */
static inline unsigned int zipIntSize(unsigned char encoding) {
    switch(encoding) {
    case ZIP_INT_8B:  return 1;
    case ZIP_INT_16B: return 2;
    case ZIP_INT_24B: return 3;
    case ZIP_INT_32B: return 4;
    case ZIP_INT_64B: return 8;
    }
    if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX)
        return 0; /* 4 bit immediate */
    /* bad encoding, covered by a previous call to ZIP_ASSERT_ENCODING */
    redis_unreachable();
    return 0;
}


/* Encode the length of the previous entry and write it to "p". Return the
 * number of bytes needed to encode this length if "p" is NULL.
 * 编码上一个节点的长度且将它写入到 p 节点中。
 * 如果节点 p 为null，返回编码该长度需要的字节数
 * */
unsigned int zipStorePrevEntryLength(unsigned char *p, unsigned int len) {
    //如果节点 p 为空，返回当前 len 对应的编码即可
    if (p == NULL) {
        //如果 len 小于254返回1字节，否则返回 4字节 + 1字节
        return (len < ZIP_BIG_PREVLEN) ? 1 : sizeof(uint32_t) + 1;
    } else {
        //若节点 p 不为空
        //当前长度小于 254 更新节点 p 长度并返回 1字节
        if (len < ZIP_BIG_PREVLEN) {
            p[0] = len;
            return 1;
        } else {
            //返回上一个节点的更大字节 用于更新场景
            return zipStorePrevEntryLengthLarge(p,len);
        }
    }
}


/* Encode the length of the previous entry and write it to "p". This only
 * uses the larger encoding (required in __ziplistCascadeUpdate).
 * 对前一项的长度进行编码，并将其写入 p 。这只用于更大的编码 用于 __ziplistCascadeUpdate 方法
 * */
int zipStorePrevEntryLengthLarge(unsigned char *p, unsigned int len) {
    uint32_t u32;
    //若p 不为null，更新长度编码即长度
    if (p != NULL) {
        p[0] = ZIP_BIG_PREVLEN;
        u32 = len;
        memcpy(p+1,&u32,sizeof(u32));
        memrev32ifbe(p+1);
    }
    //返回 1 + 4 字节
    return 1 + sizeof(uint32_t);
}


/* Write the encoding header of the entry in 'p'. If p is NULL it just returns
 * the amount of bytes required to encode such a length. Arguments:
 * 将节点的编码头部写入到 p 中，如果 p 为null则返回编码头部需要的长度
 *
 * 'encoding' is the encoding we are using for the entry. It could be
 * ZIP_INT_* or ZIP_STR_* or between ZIP_INT_IMM_MIN and ZIP_INT_IMM_MAX
 * for single-byte small immediate integers.
 * encoding 是我们为节点使用的编码。它可以是 ZIP_INT_* or ZIP_STR_* 或 大小在 ZIP_INT_IMM_MIN and ZIP_INT_IMM_MAX 表示单字节小的直接整数。
 *
 * 'rawlen' is only used for ZIP_STR_* encodings and is the length of the
 * string that this entry represents.
 * rawlen 只用于 ZIP_STR_* 编码且该节点所代表的字符串长度
 *
 * The function returns the number of bytes used by the encoding/length
 * header stored in 'p'.
 * 该方法返回  存储在 p 中用于 encoding/length 头部的字节大小
 * */
unsigned int zipStoreEntryEncoding(unsigned char *p, unsigned char encoding, unsigned int rawlen) {
    unsigned char len = 1, buf[5];

    //判断是否字符串 并更新 编码字段
    if (ZIP_IS_STR(encoding)) {
        /* Although encoding is given it may not be set for strings,
         * so we determine it here using the raw length. */
        if (rawlen <= 0x3f) {
            if (!p) return len;
            buf[0] = ZIP_STR_06B | rawlen;
        } else if (rawlen <= 0x3fff) {
            len += 1;
            if (!p) return len;
            buf[0] = ZIP_STR_14B | ((rawlen >> 8) & 0x3f);
            buf[1] = rawlen & 0xff;
        } else {
            len += 4;
            if (!p) return len;
            buf[0] = ZIP_STR_32B;
            buf[1] = (rawlen >> 24) & 0xff;
            buf[2] = (rawlen >> 16) & 0xff;
            buf[3] = (rawlen >> 8) & 0xff;
            buf[4] = rawlen & 0xff;
        }
    } else {
        /* Implies integer encoding, so length is always 1.
         * 暗示整数编码，长度总为1
         * */
        if (!p) return len;
        buf[0] = encoding;
    }

    /* Store this length at p.
     * 将长度保存在 p 中
     * */
    memcpy(p,buf,len);
    return len;
}


/* Given a pointer 'p' to the prevlen info that prefixes an entry, this
 * function returns the difference in number of bytes needed to encode
 * the prevlen if the previous entry changes of size.
 * 给定一个指针 p ，指向一个节点的 prevlen 信息，如果前一个节点的大小改变了，该方法则返回 编码 prevlen长度的差值
 *
 * So if A is the number of bytes used right now to encode the 'prevlen'
 * field.
 * 如果 A 是当前 prevlen字段编码的字节大小
 *
 * And B is the number of bytes that are needed in order to encode the
 * 'prevlen' if the previous element will be updated to one of size 'len'.
 * 如果前一个节点将被更新为大小为 len 的元素， 则B 是编码 prevlen 所需的字节数
 *
 * Then the function returns B - A
 * 该方法返回 b-a
 *
 * So the function returns a positive number if more space is needed,
 * a negative number if less space is needed, or zero if the same space
 * is needed.
 * 如果该方法返回正数代表需要更多的空间，返回负数代表需要更少的空间，或 0 代表无需改动
 * */
int zipPrevLenByteDiff(unsigned char *p, unsigned int len) {
    unsigned int prevlensize;
    //当前节点的 prevlen 字节大小 A
    ZIP_DECODE_PREVLENSIZE(p, prevlensize);
    //最新节点长度的 prevlen 字节大小 B
    return zipStorePrevEntryLength(NULL, len) - prevlensize;
}


/* Return the number of bytes used to encode the length of the previous
 * entry. The length is returned by setting the var 'prevlensize'.
 * 返回用于编码上一个节点长度的字节数。 通过设置变量 prevlensize 来返回长度
 */
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
    /*若指针的第一个元素小于ZIP_BIG_PREVLEN 代表 prevlensize 为 1字节，否则为 5 字节*/  \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
        (prevlensize) = 1;                                                     \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0)


/* Resize the ziplist.
 * 重新调整压缩列表的大小
 * */
unsigned char *ziplistResize(unsigned char *zl, size_t len) {
    assert(len < UINT32_MAX);
    zl = zrealloc(zl,len);
    ZIPLIST_BYTES(zl) = intrev32ifbe(len);
    zl[len-1] = ZIP_END;
    return zl;
}


/* Encode the length of the previous entry and write it to "p". Return the
 * number of bytes needed to encode this length if "p" is NULL.
 * 编码上一个节点的长度且将它写入到 p 节点中。
 * 如果节点 p 为null，返回编码该长度需要的字节数
 * */
unsigned int zipStorePrevEntryLength(unsigned char *p, unsigned int len) {
    //如果节点 p 为空，返回当前 len 对应的编码即可
    if (p == NULL) {
        //如果 len 小于254返回1字节，否则返回 4字节 + 1字节
        return (len < ZIP_BIG_PREVLEN) ? 1 : sizeof(uint32_t) + 1;
    } else {
        //若节点 p 不为空
        //当前长度小于 254 更新节点 p 长度并返回 1字节
        if (len < ZIP_BIG_PREVLEN) {
            p[0] = len;
            return 1;
        } else {
            //返回上一个节点的更大字节 用于更新场景
            return zipStorePrevEntryLengthLarge(p,len);
        }
    }
}
```
插入元素可以简要分为 3 个步骤:  
1. 将元素内容编码
2. 重新分配内存空间
3. 复制数据  

下面详细介绍每个步骤的实现:  
**编码**  
编码即计算 previous_entry_length 字段， encoding 字段和 content 字段的内容。那么如何获取前一个元素的长度呢？此时就需要根据元素的插入位置分情况讨论了。插入元素的位置如下图所示:  

<center>   
<img src="../../img/redis/ziplist/插入元素位置示意.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">插入元素位置示意</div>
</center>

1. 当插入列表为空、插入位置为 P0 时，不存在前一个元素，即前一个元素的长度为0； 
2. 当插入位置为 P1 时，需要获取 entryX 元素的长度，而 entryX+1 元素的 previous_len 字段存储的就是 entryX 元素的长度，比较容易获取
3. 当插入位置为 P2 时，需要获取 entryN 元素的长度，entryN 是压缩列表的尾元素，计算元素长度时需要将其 3 个字段长度相加  
找出元素插入位置代码如下：  

```C++
/* Find out prevlen for the entry that is inserted.
 * 计算已插入节点的 prevlen
 * */
//找出待插入节点的前置节点长度，有三种场景
if (p[0] != ZIP_END) {
    //若非结尾标识，直接计算上一个节点的长度
    //1.如果p[0] 不指向列表末尾，说明列表非空，并且 p 指向其中一个节点，所以新插入节点的前置节点长度可以通过节点 p 指向的节点信息中获取
    ZIP_DECODE_PREVLEN(p, prevlensize, prevlen); //通过 ZIP_DECODE_PREVLEN 方法获取 prevlen 长度
} else {
    //获取尾节点位置，用来判断当前压缩列表是否为空列表
    unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
    //若末尾节点非结束标识
    //2. 如果尾节点指针不指向压缩列表末尾，说明当前压缩列表不为空，那么新插入节点的前置节点长度就是尾节点的长度
    if (ptail[0] != ZIP_END) {
        //获取尾节点的总长度 头部大小+总长度
        prevlen = zipRawEntryLengthSafe(zl, curlen, ptail);
    }
    //3. 第三种情况就是 p 指向压缩列表末尾，但是压缩列表中节点为空，所以 p 的前置节点长度为 0 ，故不做处理
}
```

当插入位置为压缩列表末尾且压缩列表不为空时，计算前一个元素长度代码如下所示：

```C++
/* Return the total number of bytes used by the entry pointed to by 'p'.
 * 返回 p 所指向的节点所使用的总字节数
 * */
static inline unsigned int zipRawEntryLengthSafe(unsigned char* zl, size_t zlbytes, unsigned char *p) {
    zlentry e;
    assert(zipEntrySafe(zl, zlbytes, p, &e, 0));
    return e.headersize + e.len;
}
```

encoding 字段标识的是当前元素存储的数据类型和数据长度。  
- 编码时首先尝试将数据内容解析为整数，如果解析成功，则按照压缩列表整数类型编码存储
- 如果解析失败，则按照压缩列表字节数类型编码存储

```C++
/* See if the entry can be encoded
 * 尝试将节点从String转为int，计算当前节点的长度
 * */
//s指针指向新增节点数据  slen为数据长度
//确定数据编码，为整数时返回对应固定长度，为字符串时使用slen
if (zipTryEncoding(s,slen,&value,&encoding)) {
    /* 'encoding' is set to the appropriate integer encoding
     * encoding 已经设置为最合适的整数编码，并获取编码长度
     * */
    reqlen = zipIntSize(encoding);
} else {
    /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
     * string length to figure out how to encode it.
     * encoding 未受影响，但是 zipStoreEntryEncoding 将使用字符串长度来决定如何编码
     * */
    reqlen = slen;
}

/* We need space for both the length of the previous entry and
 * the length of the payload.
 * 我们需要为前一个节点的长度和有效负载的长度分配空间 即更新当前节点场景
 * */
//编码前置节点长度所需字节数
reqlen += zipStorePrevEntryLength(NULL,prevlen);
//编码当前字符串长度所需字节数
//此时 reqlen 为新加入节点的整体长度
reqlen += zipStoreEntryEncoding(NULL,encoding,slen);
```
上述代码尝试按照整数解析新添加元素的数据内容，数值存储在变量的 value 中，编码存储在变量 encoding 中。如果解析成功，还需要计算整数整数所占字节数。
变量 reqlen 最终存储的是当前元素所需空间大小，初始赋值为元素 content 字段所需空间的大小，再累加 previous_entry_length 和 encoding 字段所需空间大小  


**重新分配空间**  
由于新插入了元素，压缩列表所需空间增大，因此需要重新分配存储空间。那么空间大小是不是添加元素前的压缩列表长度与新添加元素长度之和呢？并不完全是，因为 previous_entry_length 字段长度是根据前一个字段长度变化的。  

我们假设插入元素前， entryX 元素的长度为 128 字节， entryX+1 元素的 previous_entry_length 字段占 1 字节；
添加元素 entryNew，元素长度为 1024 字节，此时 entryX+1 元素的 previous_entry_length 字段需要占 5 个字节，即压缩列表的长度不仅增加了 1024 字节，还要加上 entryX+1 元素扩展的 4 个字节。而 entryX+1 元素的长度可能增加 4 个字节、减少 4 个字节或者不变。  
压缩列表长度变化如下图所示：  

<center>   
<img src="../../img/redis/ziplist/压缩列表长度变化示意.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">压缩列表长度变化示意</div>
</center>

由于重新分配了空间，新元素插入的位置指针 P 会失效，可以预先计算好指针 P 相对于压缩列表首地址的偏移量，待分配空间之后再偏移即可。  
代码实现如下：  
```C++
/* We need space for both the length of the previous entry and
 * the length of the payload.
 * 我们需要为前一个节点的长度和有效负载的长度分配空间 即更新当前节点场景
 * */
//编码前置节点长度所需字节数
reqlen += zipStorePrevEntryLength(NULL,prevlen);
//编码当前字符串长度所需字节数
//此时 reqlen 为新加入节点的整体长度
reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

/* When the insert position is not equal to the tail, we need to
 * make sure that the next entry can hold this entry's length in
 * its prevlen field.
 * 当插入的位置不在尾部时，我们需要保证下一个节点可以在 prevlen 字段中保存当前节点的长度
 * */
int forcelarge = 0; //在nextdiff==-4 && reqlen<4 时使用，该条件说明，插入元素导致压缩列表变小了，即函数 ziplistResize内部调用 realloc 重新分配空间小于 zl 指向的空间，此时 realloc 会将多余空间回收，导致数据丢失（丢失了尾部），所以为了避免这种情况，我们使用forcelarge来标记这种情况，并将 nextdiff 置为0，详情见https://segmentfault.com/a/1190000018878466?utm_source=tag-newest
//计算原来 p 位置上的节点将要保存的 prevlen（当前待插入节点的大小） 是否变化
nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
//连锁更新的时候会出现问题
if (nextdiff == -4 && reqlen < 4) {
    nextdiff = 0; //将nextdiff 设置为0，此时内存重分配不会出现回收空间的情况，造成数据丢失
    forcelarge = 1;
}

/* Store offset because a realloc may change the address of zl.
 * 重新分配内存肯呢个会改变 zl 的地址，所以存储偏移
 * */
offset = p-zl; //偏移量，用来表示 p 相对于压缩列表首地址的偏移量，由于重新分配了空间，新元素插入的位置指针 p 会时效，可以预先计算好指针 p 对于压缩列表首地址的偏移量，待空间分配之后再便宜
//最新空间大小   当前压缩列表大小 + 插入元素大小 + 差值长度
newlen = curlen+reqlen+nextdiff;
//重新分配压缩列表大小
zl = ziplistResize(zl,newlen);
//根据偏移找到新插入元素 P 的位置
p = zl+offset;
```
nextdiff 用来表示 entryX+1 元素长度变化，取值可能为 0 （长度不变）、4（长度增加4）、-4（长度减少4）  
forcelarge 用来表示一种特殊的情况，即 nextdiff==-4&&reqlen<4 ，该情况有可能会导致内存重分配时回收内存空间，进而数据丢失。所以表示这种特殊情况并做处理。  
offset 表示指针 P 相对于压缩列表首地址的偏移位置，内存重分配后计算指针 P 的新的位置时使用  

> 下面详细说下 nextdiff == -4 && reqlen < 4 这个情况  
> nextdiff == -4 && reqlen < 4 时会发生什么呢？插入元素导致压缩列表所需空间变小了。即函数 ziplistResize 内部调研 realloc 重新分配的空间小于指针 zl指向的空间。我们知道 realloc 重新分配空间时，返回的地址可能不变（当前位置有足够的内存空间可供分配），当重新分配的空间减少时，realloc 可能会将多余的空间回收，导致数据丢失（压缩列表后面一部分数据丢失）。因此需要避免这种情况的发生，Redis 采用的方法是重新赋值 nextdiff=0，同时使用 forcelarge 标记这种情况。  
> 那么，nextdiff=-4 时，reqlen 会小于 4 吗？nextdiff=-4 说明插入元素之前 entryX+1 的元素的总长度大于等于 254 字节，所以 entryNew 元素的 previous_entry_length 字段同样需要 5 个字节，即 entryNew 元素的总长度肯定是大于 5 个字节的，reqlen 又怎么会小于 4 呢？正常情况下是不会出现这种情况的，但是由于连锁更新，可能会出现 nextdiff=-4 但 entryX 元素的总长度小于 254 字节的情况，此时 reqlen 可能会小于 4 。


**数据复制**  
重新分配空间之后，需要将位置 P 后的元素移动到指定位置，将新元素插入到位置 P。我们假设 entryX+1 元素的长度增加 4（即nextdiff=4）。代码如下：  
```C++
  /* Apply memory move when necessary and update tail offset.
     * 必要时应用内存移动并更新尾偏移量
     * */
    //非空列表插入
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes
         * 因为是 ZIP_END 字节，所以减去1
         * */
        //将 p 节点后移（没有移动 p 节点前一节点长度信息），留出当前节点位置
        //p+reqlen: 偏移量是这个，就是将原来的数据移动到新插入节点之后
        //curlen-offset-1+nextdiff: 移动的长度，就是位置 P 之后的所有元素的长度 -1（结束符大小恒为 0XFF，不需要移动），再加上 nextdiff（下一个元素长度的变化）
        //p-nextdiff: 表示从哪个位置需要复制移动，因为下一个元素长度会发生变化，所以需要提前预留出这部分空间，就多复制一块空间，到时候覆盖即可
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry.
         * 在下一个节点中编码节点原始长度 并且判断是否需要加大上一个节点长度
         * */
        //写入 p 节点以前一个节点长度信息（要插入节点的长度）
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        /* Update offset for tail
         * 更新末尾节点的偏移
         * */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset.
         * 当尾部包含多个节点时，还需要考虑 nextdiff，否则，prevlen大小的变化不会对尾部偏移量产生影响
         * */
        assert(zipEntrySafe(zl, newlen, p+reqlen, &tail, 1));
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail.
         * 如果在末尾 则该节点将变成新的末尾节点
         * */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }
```
数据复制分为两种情况：  
1. 一种是当前插入的位置不是最后，因此需要进行数据赋值  
2. 另一种是当前插入的位置为尾节点后面，就不需要进行数据复制了  

主要看第一种情况，其中下面这段代码需要仔细理解下： 
> memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);  
> C库函数 void *memmove(void *str1, const void *str2, size_t n) 从 str2 复制 n 个字符到 str1。  

- p+reqlen 很好理解，就是将 P 后面的数据复制到新节点之后
- curlen-offset-1+nextdiff 表示复制的字符数量。即 P 后面元素的长度 + entryX+1 元素长度的变化 nextdiff，再减去 1 ，因为结束符恒为 0XFF,不需要移动
- p-nextdiff 将 P 后面的内容复制过去，然后多复制一块，因为下一个元素的长度空间会发生变化，供下一个元素的 previous_entry_length 使用

为什么第一个更新尾元素偏移量之后，指向的元素可能不是尾元素呢？因为当 entryX+1 元素就是尾元素时，只需要更新一次尾元素的偏移量；但是当 entryX+1 元素不是尾元素时且 entryX+1 元素的长度发生了改变时，尾元素偏移量还需要加上 nextdiff 的值。  


**连锁更新**  
每个节点的 previous_entry_length 属性都记录了前一个节点的长度：  
- 如果前一个节点的长度小于 254 字节，那么 previous_entry_length 属性需要用 1 字节长度的空间来保存这个长度值  
- 如果前一个节点的长度大于等于 254 字节，那么 previous_entry_length 属性需要用 5 个字节的空间来保存这个长度值  

现在，考虑这样一种情况：在一个压缩列表中，有多个连续的、长度介于 250 字节到 253 字节之间的节点 e1 到 eN ，如下图所示：  

<center>   
<img src="../../img/redis/ziplist/包含节点 e1 至 eN 的压缩列表.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">包含节点 e1 至 eN 的压缩列表</div>
</center>  

因为 e1 到 eN 的所有节点的长度都小于 254 字节，所以记录这些节点的长度只需要 1 字节长的 previous_entry_length 属性。
这时，如果我们将一个长度大于等于 254 字节的新节点 new 设置为压缩列表的表头节点，那么 new 将成为 e1 的前置节点，如下图所示：  

<center>   
<img src="../../img/redis/ziplist/插入新节点到压缩列表.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">插入新节点到压缩列表</div>
</center>  

因为 e1 的 previous_entry_length 属性仅长 1 字节，它没办法保存新节点 new 的长度，所以程序将对压缩列表执行空间重分配操作，并将 e1 节点的 previous_entry_length 属性从原来的 1 字节扩展为 5 字节长。  
现在，麻烦的事情来了，e1 原本的长度介于 250 字节至 253 字节之间，在为 previous_entry_length 属性新增 4 个字节空间之后， e1 的长度就变成了介于 254 字节至 257 字节之间，而这种长度使用 1 字节长的 previous_entry_length 属性是没办法保存的。  

因此，为了让 e2 的 previous_entry_length 属性可以记录下 e1 的长度，程序需要再次对压缩列表执行空间重分配操作，并将 e2 节点的 previous_entry_length 属性从原来的 1 字节扩展为 5 字节。  

正如扩展 e1 引发了对 e2 的扩展一样，扩展 e2 也会引发对 e3 的扩展，而扩展 e3 又会引发对 e4 的扩展。为了让每个节点的 previous_entry_length 属性都符合压缩列表对节点的要求，程序需要不断地对压缩列表执行空间分配操作，指导 eN 位置。  

Redis 将这种在特殊情况下产生的连续多次空间扩展操作称之为 连续更新，下图展示了这一过程：  

<center>   
<img src="../../img/redis/ziplist/连锁更新过程.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">连锁更新过程</div>
</center>  

除了添加新节点可能会引发连锁更新之外，删除节点也可能会引发连锁更新。  

考虑到上图所示的压缩列表，如果 e1 至 eN 都是大小介于 250 字节至 253 字节的节点，big 节点的长度大于等于 254 字节（需要 5 字节的 previous_entry_length 来保存），而 small 节点的长度小于 254 字节（只需要 1 字节的 previous_entry_length 来保存），那么当我们将 small 节点从压缩列表中删除之后，为了让 e1 的 pervious_entry_length 属性可以记录 big 节点的长度，程序将扩展 e1 的空间，并由此引发之后的连锁更新。  

<center>   
<img src="../../img/redis/ziplist/删除节点引发连锁更新.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">删除节点引发连锁更新</div>
</center>  

因为连锁更新在最坏情况下需要对压缩列表执行 N 次空间重分配操作，而每次空间重分配的最坏复杂度为 O(N) ，所以连锁更新的最坏复杂度为 O(N^2)。  

需要注意的是，尽管连锁更新的复杂度较高，但它真正造成性能问题的几率是很低的：  
- 首先，压缩列表里要恰好有多个连续、长度介于 250 字节至 253 字节之间的节点，连锁更新才有可能被引发，在实际中，这种情况并不多见；  
- 其次，即使出现连锁更新，但只要被更新的节点数量不多，就不会对性能造成任何影响；比如说，对三五个节点进行连锁更新是绝对不会影响性能的；  

因为以上原因，ziplistPush 等命令的平均复杂度仅为 O(N) ，在实际中，可以放心使用这些函数，而不必担心连锁更新会影响压缩列表的性能。  


源码代码如下：  
```C++  
/* When an entry is inserted, we need to set the prevlen field of the next
 * entry to equal the length of the inserted entry. It can occur that this
 * length cannot be encoded in 1 byte and the next entry needs to be grow
 * a bit larger to hold the 5-byte encoded prevlen. This can be done for free,
 * because this only happens when an entry is already being inserted (which
 * causes a realloc and memmove). However, encoding the prevlen may require
 * that this entry is grown as well. This effect may cascade throughout
 * the ziplist when there are consecutive entries with a size close to
 * ZIP_BIG_PREVLEN, so we need to check that the prevlen can be encoded in
 * every consecutive entry.
 * 当插入一个节点时，我们需要将下一个节点的 prevlen 字段设置为等于插入节点的长度。
 * 可能会出现这种情况，这个长度不能被 1 字节编码，下一个节点需要变大以存储 5字节的 prevlen 编码。
 * 这可以免费完成，因为这只会发生在节点已经被插入（这会导致realloc和memmove）。
 * 但是，编码 prevlen 同样需要节点增长。
 * 当存在大小接近 ZIP_BIG_PREVLEN 的连续节点时，这种效果可能会在整个 ziplist 中级联，因为我们需要检查 prevlen 是否可以在每个连续条目中编码
 *
 *
 * Note that this effect can also happen in reverse, where the bytes required
 * to encode the prevlen field can shrink. This effect is deliberately ignored,
 * because it can cause a "flapping" effect where a chain prevlen fields is
 * first grown and then shrunk again after consecutive inserts. Rather, the
 * field is allowed to stay larger than necessary, because a large prevlen
 * field implies the ziplist is holding large entries anyway.
 * 请注意，这种效果也可以反过来发生，在这种情况下，编码 prevlen 字段所需的字节会缩小。
 * 这种效果被故意忽略了，因为它会导致扑动效果，即在连续插入之后，链上的字段首先增长，然后再次收缩。
 * 相反，允许字段保持比需要的更发，因为较大的 prevlen 字段意味着 ziplist 无论如何多包涵较大的节点。
 *
 *
 * The pointer "p" points to the first entry that does NOT need to be
 * updated, i.e. consecutive fields MAY need an update.
 * 指针 p 指向第一个不需要被更新节点，即连续的字段可能需要更新
 * */
unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p) {
    zlentry cur;
    size_t prevlen, prevlensize, prevoffset; /* Informat of the last changed entry. 最后更改的节点信息 */
    size_t firstentrylen; /* Used to handle insert at head. 用于处理头部插入 */
    size_t rawlen, curlen = intrev32ifbe(ZIPLIST_BYTES(zl));
    size_t extra = 0, cnt = 0, offset;
    size_t delta = 4; /* Extra bytes needed to update a entry's prevlen (5-1).  更新节点的 prevlen 所需的额外字节（5-1） */
    unsigned char *tail = zl + intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl));

    /* Empty ziplist */
    if (p[0] == ZIP_END) return zl;

    //将 p 所指向的节点的信息保存到 cur 结构中
    zipEntry(p, &cur); /* no need for "safe" variant since the input pointer was validated by the function that returned it. */
    //当前节点的长度
    firstentrylen = prevlen = cur.headersize + cur.len;
    //计算编码当前节点的长度所需的字节数
    prevlensize = zipStorePrevEntryLength(NULL, prevlen);
    //记录 p 的偏移量
    prevoffset = p - zl;
    //记录下一节点的偏移量
    p += prevlen;

    /* Iterate ziplist to find out how many extra bytes do we need to update it.
     * 迭代压缩列表找出需要更新的额外字节数
     * */
    while (p[0] != ZIP_END) {
        assert(zipEntrySafe(zl, curlen, p, &cur, 0));

        /* Abort when "prevlen" has not changed. 当 prevlen 没有改变时终止 */
        if (cur.prevrawlen == prevlen) break;

        /* Abort when entry's "prevlensize" is big enough. 当节点的 prevlensize 足够大时宗旨 */
        if (cur.prevrawlensize >= prevlensize) {
            if (cur.prevrawlensize == prevlensize) {
                zipStorePrevEntryLength(p, prevlen);
            } else {
                /* This would result in shrinking, which we want to avoid.
                 * So, set "prevlen" in the available bytes. */
                zipStorePrevEntryLengthLarge(p, prevlen);
            }
            break;
        }

        /* cur.prevrawlen means cur is the former head entry. */
        assert(cur.prevrawlen == 0 || cur.prevrawlen + delta == prevlen);

        /* Update prev entry's info and advance the cursor. 更新前一个节点的信息并移动光标 */
        rawlen = cur.headersize + cur.len;
        prevlen = rawlen + delta; 
        prevlensize = zipStorePrevEntryLength(NULL, prevlen);
        prevoffset = p - zl;
        p += rawlen;
        extra += delta;
        cnt++;
    }

    /* Extra bytes is zero all update has been done(or no need to update).
     * 额外字节为零，表示更新已完成（或不需要更新）
     * */
    if (extra == 0) return zl;

    /* Update tail offset after loop.
     * 循环后更新尾部偏移量
     * */
    if (tail == zl + prevoffset) {
        /* When the the last entry we need to update is also the tail, update tail offset
         * unless this is the only entry that was updated (so the tail offset didn't change). */
        if (extra - delta != 0) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra-delta);
        }
    } else {
        /* Update the tail offset in cases where the last entry we updated is not the tail. */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
    }

    /* Now "p" points at the first unchanged byte in original ziplist,
     * move data after that to new ziplist.
     * 现在 p 指向原始 ziplist 中第一个未更改的字节，将其之后的数据移动到ziplist中
     * */
    offset = p - zl;
    zl = ziplistResize(zl, curlen + extra);
    p = zl + offset;
    memmove(p + extra, p, curlen - offset - 1);
    p += extra;

    /* Iterate all entries that need to be updated tail to head.
     * 迭代所有需要从头到尾更新的节点
     * */
    while (cnt) {
        zipEntry(zl + prevoffset, &cur); /* no need for "safe" variant since we already iterated on all these entries above. */
        rawlen = cur.headersize + cur.len;
        /* Move entry to tail and reset prevlen. 将入口移动至尾部并重置预设值 */
        memmove(p - (rawlen - cur.prevrawlensize), 
                zl + prevoffset + cur.prevrawlensize, 
                rawlen - cur.prevrawlensize);
        p -= (rawlen + delta);
        if (cur.prevrawlen == 0) {
            /* "cur" is the previous head entry, update its prevlen with firstentrylen. */
            zipStorePrevEntryLength(p, firstentrylen);
        } else {
            /* An entry's prevlen can only increment 4 bytes. */
            zipStorePrevEntryLength(p, cur.prevrawlen+delta);
        }
        /* Foward to previous entry. 跳转到上一个节点 */
        prevoffset -= cur.prevrawlen;
        cnt--;
    }
    return zl;
}
```


**删除元素**  
ziplistDelete 函数可以同时删除多个元素，输入参数 p 指向的是首个待删除元素的地址，num 表示待删除元素数据。  
删除元素同样可以简要分为三个步骤：  
1. 计算待删除元素的总长度
2. 数据复制
3. 重新分配空间

计算待删除元素的总长度  
相关代码如下：  
```C++
//解码第一个待删除元素
zipEntry(p, &first); /* no need for "safe" variant since the input pointer was validated by the function that returned it. */
//遍历所有待删除元素，通知指针后移
for (i = 0; p[0] != ZIP_END && i < num; i++) {
    p += zipRawEntryLengthSafe(zl, zlbytes, p);
    deleted++;
}

assert(p >= first.p);
//待删除元素的总长度
totlen = p-first.p; /* Bytes taken by the element(s) to delete. 要删除的元素所占的字节数*/
```
即遍历压缩列表，计算待删除元素的长度之和。  

数据复制  
第一步完成之后，指针 first 与指针 p 之间的元素都是待删除的。当指针 p 恰好指向 zlend 字段时，不需要复制数据，只需要更新尾节点的偏移量即可。  

接下来，我们看另外一种情况，即指针 p 指向的是某一个元素，而不是 zlend 字段。 删除元素时，压缩列表所需空间减少，那么减少的量是否仅为待删除元素的总长度呢？那肯定不是了，因为需要更新下一个节点的 previous_entry_length 的值。见下图：
<center>   
<img src="../../img/redis/ziplist/压缩列表删除元素示意.png" class="img-responsive img-centered" alt="image-alt">
<div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">压缩列表删除元素示意</div>
</center>  

删除元素 entryX+1 到元素 entryN-1 之间的 N-X-1 个元素，元素 entryN-1 的长度为 12 字节，因此元素 entryN 的 previous_entry_length 字段占 1 字节；  
删除这些元素之后，entryX 成为了 entryN 的前一个元素，元素 entryX 的长度为 512 字节，因此元素 entryN 的 previous_entry_length 字段还需要占 5 个字节，即删除元素之后的压缩列表的总长度还与元素 entryN 长度的变化量有关。  
源码如下：  
```C++
/* Storing `prevrawlen` in this entry may increase or decrease the
 * number of bytes required compare to the current `prevrawlen`.
 * There always is room to store this, because it was previously
 * stored by an entry that is now being deleted.
 * 在此表项中存储 prevrawlen 可能会比当前 prevrawlen 增加或减少所需的字节数。
 * 总是有空间来存储它，因为它以前是一个现在被删除的节点存储的。
 * 计算删除的 最后一个元素 entryN-1 之后的元素 entryN 的长度变化量
 * */
nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);

/* Note that there is always space when p jumps backward: if
 * the new previous entry is large, one of the deleted elements
 * had a 5 bytes prevlen header, so there is for sure at least
 * 5 bytes free and we need just 4.
 * 请注意，当 p 向后跳转时总是有空间：如果新的前一个节点很大，则删除的元素之一具有 5 字节的 prevlen 头部，因此肯定至少有 5 字节的空闲空间，而我们只需要 4 字节。
 * */
//更新 entryN 的 previous_entry_length 字段
p -= nextdiff;
assert(p >= first.p && p<zl+zlbytes-1);
zipStorePrevEntryLength(p,first.prevrawlen);

/* Update offset for tail
 * 尾部更新偏移
 * */
set_tail = intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen;

/* When the tail contains more than one entry, we need to take
 * "nextdiff" in account as well. Otherwise, a change in the
 * size of prevlen doesn't have an effect on the *tail* offset.
 * 当尾部包含多个节点时，我们还需要考虑 nextdiff 。
 * 否则， prevlen  大小的变化不会对 尾 偏移量产生影响。
 * */
assert(zipEntrySafe(zl, zlbytes, p, &tail, 1));
if (p[tail.headersize+tail.len] != ZIP_END) {
    set_tail = set_tail + nextdiff;
}

/* Move tail to the front of the ziplist
 * 将尾部移到压缩列表的签名
 * */
/* since we asserted that p >= first.p. we know totlen >= 0,
 * so we know that p > first.p and this is guaranteed not to reach
 * beyond the allocation, even if the entries lens are corrupted.
 * 因为我们断言 p >= first.p。 我们知道 totlen>=0，所以我们知道 p>first.p ，这保证不会超出分哦，即使节点长度被损坏
 * */
size_t bytes_to_move = zlbytes-(p-zl)-1;
memmove(first.p,p,bytes_to_move);
```


重新分配空间  
重新分配空间与插入元素的类似，代码如下：  
```C++
/* Resize the ziplist
 * 重新调整压缩列表大小
 * */
offset = first.p-zl;
zlbytes -= totlen - nextdiff;
zl = ziplistResize(zl, zlbytes);
p = zl+offset;
```

另外，在插入元素时，调用 ziplistResize 函数重新分配空间时，如果重新分配的空间小于指针 zl 指向的空间时，可能会出现问题。而删除元素时，压缩列表的长度肯定时减小的。  
因为删除元素时，先复制数据，再重新分配空间，即调用 ziplistResize 函数时，多余的那部分空间存储的数据已经被复制，此时回收这部分空间并不会造成数据的损失。

> https://pdai.tech/md/db/nosql-redis/db-redis-x-redis-ds.html#ziplist%E7%BB%93%E6%9E%84  
> http://zhangtielei.com/posts/blog-redis-ziplist.html  
> https://juejin.cn/post/6914456200650162189#heading-17
> https://juejin.cn/post/7097856013776191518
> https://segmentfault.com/a/1190000018878466?utm_source=tag-newest
> https://wenfh2020.com/2020/01/30/redis-ziplist/