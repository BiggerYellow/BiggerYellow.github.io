---
layout: default
algorithm: true
modal-id: 100000017
date: 2021-11-26
img: pexels-nuta-sorokina-10005736.jpg
alt: hashTable
project-date: November 2021
client: HashTable
category: algorithm
subtitle: HashTable
description: 散列表
---
### 定义
- - -

&emsp;&emsp;根据关键码值(Key Value)而直接进行访问的数据结构.也就是说,它通过把关键码值映射到表中一个位置来访问记录,以加快查找的速度.这个映射函数叫做散列函数,存放记录的数组叫做散列表.  
&emsp;&emsp;使用散列的查找算法分为两步,第一步是用散列函数将被查找的键转化为数组的一个索引;第二步就是一个处理碰撞冲突的过程,接下来将介绍两种解决碰撞的方法:拉链法和线性探测法.
- - -

### 散列函数
- - -
&emsp;&emsp;如果我们有一个能够保存M个键值对的数组,那么我们就需要一个能够将任意键转化为该数组范围内的索引([0,M-1]范围内的整数)的散列函数.
我们要找的散列函数应该易于计算并且能够均匀分布所有的键,即对于任意键,0到M-1之间的每个整数都有相等的可能性与之对应.  
&emsp;&emsp;散列函数和键的类型有关,严格地说,对于每种类型的键我们都需要一个与之对应的散列函数.  
&emsp;&emsp;主要介绍一下几种类型:
1. 正整数  
&emsp;&emsp;将整数散列最常用的方法是除留余数法.我们选择大小为素数M的数组,对于任意正整数k,计算k除以M的余数.这个函数的计算非常容易(在Java中为k%M)并能够有效地将键散布在0到M-1的范围内.
如果M不是素数,我们可能无法利用键中包含的所有信息,这可能导致我们无法均匀的散列散列值.
2. 浮点数  
&emsp;&emsp;如果键是0到1之间的实数,我们可以将它乘以M并四舍五入得到一个0至M-1之间的索引值.尽管这个方法很容易理解,但它是有缺陷的,因为这种情况下键的高位起的作用更大,最低位对散列的结果没有影响.修正这个问题的办法是将键表示为二进制数然后再使用除留余数法.
3. 字符串  
&emsp;&emsp;除留余数法也可以处理较长的键,
4. 组合键  
&emsp;&emsp;如果键的类型含有多个整型变量,我们可以和String类型一样将它们混合起来.
&emps;&emsp;例如:假设被查找的键的类型是Date,其中含有几个整型的域:day(两个数字表示的日)、month(两个数字表示的月)和year(4个数字表示的年).我们可以这样计算它的散列值:  
``` java
int hash = (((day*R + month)%M)*R + year)%M
```

- Java的约定  
&emsp;&emsp;每种数据类型都需要相应的散列函数,于是Java令所有数据类型都继承了一个能够返回一个32比特整数的hashCode().每一种数据结构的hashCode()方法都必须和equals()方法一致.
也就是说,如果a.equals(b)返回true,那么a.hashCode()的返回值必然和b.hashCode()的返回值相同.相反,如果两个对象的hashCode()方法的返回值不同,那么我们就知道这两个对象是不同的.
但是如果两个对象的hashCode()方法的返回值相同,这两个对象也有可能不同,我们还需要使用equals()方法进行判断.请注意,这说明你要为自定义的数据类型定义散列函数,你需要同时重写hashCode()和equals()两个方法.
- 将hashCode()的返回值转化为一个数组索引  
&emsp;&emsp;因为我们需要的是数组的索引而不是一个32位的整数,我们在实现中会将默认的hashCode()方法和除留余数法结合起来产生一个0到M-1的整数,方法如下:
``` java
private int hash(Key x){
    return (x.hashCode() & 0x7fffffff) % M;
}
```
&emsp;&emsp;这段代码会将符号位屏蔽(将一个32位整数变为一个31位非负整数),然后用除留余数法计算它除以M的余数.在使用这样的代码时我们一般会将数组的大小M取为素数以充分利用原散列值的所有位.  
- 自定义的hashCode()方法  
&emsp;&emsp;散列表的用例希望hashCode()方法能够将键平均地散布为所有可能的32位整数.也就是说,对于任意对象x,你可以调用x.hashCode()并认为有均等的机会得到2^32个不同整数中的任意一个32位整数值.
对于自定义的数据类型,必须试着自己实现这一点,示例如下:
``` java
public class Transaction{
    ...
    private final String who;
    private final Date when;
    private final double amount;
    
    public int hashCode(){
        int hash = 17;
        hash = 31*hash + who.hashCode();
        hash = 31*hash + when.hashCode();
        hash = 31*hash + ((Double) amount).hashCode();
        return hash;
    }
    ...
}
```
&emsp;&emsp;在Java中,所有的数据类型都继承了hashCode()方法,因此还有一个更简单的做法:将对象中的每个变量的hashCode()返回值转化为32位整数并计算得到散列值,如上示例所示.
对于原始类型的对象,可以将其转化为对于的数据类型然后再调用hashCode()方法.
- 软缓存  
&emsp;&emsp;如果散列值的计算很耗时,那么我们或许可以将每个键的散列值缓存起来,即在每个键中使用一个hash变量来保存它的hashCode()的返回值.
第一次调用hashCode()方法时,我们需要计算对象的散列值,但之后对hashCode()方法的调用会直接返回hash变量的值.
&emsp;&emsp;总的来说,要为一个数据类型实现一个优秀的散列方法需要使用满足三个条件:
  - 一致性:等价的键必然产生相等的散列值
  - 高效性:计算简便
  - 均匀性:均匀地散列所有的键

- - -

### 基于拉链法的散列表
- - -
&emsp;&emsp;一个散列函数能够将键转化为数组索引.散列算法的第二步是碰撞处理,也就是处理两个或多个键的散列值相同的情况.一种直接的办法是将大小为M的数组中的每个元素指向一条链表,
链表中的每个节点都存储了散列值为该元素的索引的键值对.这种方法被称为拉链法,因为发生冲突的元素都被存储在链表中.这个方法的基本思想就是选择足够大的M,使得所有链表都尽可能短以保证高效的查找.
查找分两步:首先根据散列值找到对应的链表,然后沿着链表顺序查找相应的键.  
&emsp;&emsp;拉链法的一种实现方法是使用原始的链表数据类型类扩展SequentialSearchST.另一种更简单的方法是采用一般性的策略,为M个元素分别构建符号表来保存散列到这里的键,这样也可以重用之前的代码.  
&emsp;&emsp;因为我们要用M条链表保存N个键,无论键在各个链表中的分布如何,链表的平均长度肯定是N/M.在标准索引用例中使用基于拉链法的散列表如下图所示:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/hashTable/标准索引用例使用基于拉链法的散列表.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="标准索引用例使用基于拉链法的散列表"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/hashTable/标准索引用例使用基于拉链法的散列表.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">标准索引用例使用基于拉链法的散列表</div>
    </a>
</center>

``` java
public class SeparateChainingHashST<Key, Value>
{
  private int N; // 键值对总数
  private int M; // 散列表的大小
  private SequentialSearchST<Key, Value>[] st; // 存放链表对象的数组
  public SeparateChainingHashST()
  { this(997); }
  public SeparateChainingHashST(int M)
  { // 创建M条链表
    this.M = M;
    st = (SequentialSearchST<Key, Value>[]) new SequentialSearchST[M];
    for (int i = 0; i < M; i++)
    st[i] = new SequentialSearchST();
  }
  private int hash(Key key)
  { return (key.hashCode() & 0x7fffffff) % M; }
  public Value get(Key key)
  { return (Value) st[hash(key)].get(key); }
  public void put(Key key, Value val)
  { st[hash(key)].put(key, val); }
  public Iterable<Key> keys()
  ...
}
```
&emsp;&emsp;这段简单的符号表实现维护这一条链表的数组,用散列函数来为每个键选择一条链表.

- 散列表的大小  
&emsp;&emsp;在实现基于拉链法的散列表时,目标是选择适当的数组大小M,既不会因为空链表而浪费大量内存,也不会因为链表太长而在查找上浪费太多时间.
- 删除操作  
&emsp;&emsp;要删除一个键值对,先用散列值找到含有该键的SequencetialSearchST对象,然后再调用该对象的delete()方法.
- 有序性相关操作  
&emsp;&emsp;散列最主要的目的在于均匀地将键散布开来,因此在计算散列后键的顺序信息就会丢失了.所以散列不适合有序操作.

- - -

### 基于线性探测的散列表
- - -
&emsp;&emsp;实现散列表的另一种方式就是用大小为M的数组保存N个键值对,其中M>N.我们需要依靠数组中的空位解决碰撞冲突.基于这种策略的所有方法被统称为开放地址散列表.  
&emsp;&emsp;开放地址散列表中最简单的方法叫做线性探测法:当碰撞发生时(当一个键的散列值已经被另一个不同的键占用),我们直接检查散列表中的下一个位置(将索引值加1).这样的线性探测可能会产生三种结果:
  - 命中,该位置的键和被查找的键相同
  - 未命中,键为空(该位置没有键)
  - 继续查找,该位置的键和被查找的键不同

&emsp;&emsp;我们用散列函数找到键在数组中的索引,检查其中的键和被查找的键是否相同.如果不同则继续查找(将索引增大,到达数组结尾时折回数组的开头),直到找到该键或者遇到一个空元素,我们习惯将检查一个数组位置是否含有被查找的键的操作称作探测.  
&emsp;&emsp;开放地址类的散列表的核心思想是与其将内存用作链表,不如将它们作为在散列表的空元素.这些空元素可以作为查找结束的标志.
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/hashTable/标准索引用例使用的基于线性探测的符号表的轨迹.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="标准索引用例使用的基于线性探测的符号表的轨迹"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/hashTable/标准索引用例使用的基于线性探测的符号表的轨迹.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">标准索引用例使用的基于线性探测的符号表的轨迹</div>
    </a>
</center>

``` java
public class LinearProbingHashST<Key, Value>
{
  private int N; // 符号表中键值对的总数
  private int M = 16; // 线性探测表的大小
  private Key[] keys; // 键
  private Value[] vals; // 值
  public LinearProbingHashST()
  {
    keys = (Key[]) new Object[M];
    vals = (Value[]) new Object[M];
  }
  private int hash(Key key)
  { return (key.hashCode() & 0x7fffffff) % M; }
  private void resize() // 请见3.4.4节
  public void put(Key key, Value val)
  {
    if (N >= M/2) resize(2*M); // 将M加倍（请见正文）
    int i;
    for (i = hash(key); keys[i] != null; i = (i + 1) % M)
    if (keys[i].equals(key)) { vals[i] = val; return; }
    keys[i] = key;
    vals[i] = val;
    N++;
  }
  public Value get(Key key)
  {
    for (int i = hash(key); keys[i] != null; i = (i + 1) % M)
    if (keys[i].equals(key))
    return vals[i];
    return null;
  }
}
```
&emsp;&emsp;这段符号表的实现将键和值分别保存在两个数组中,使用空来表示一簇键的结束.如果一个新键的散列值是一个空元素,那么就将它保存在那里;如果不是,
我们就顺序查找一个空元素来保存它.要查找一个键,我们从它的散列值开始顺序查找,如果找到则是命中,如果遇到空元素则未命中.

- 删除操作  

&emsp;&emsp;如何从基于线性探测的散列表中删除一个键?仔细想想,直接将该键所在的位置设为null是不行的,因为这会使得此位置之后的元素无法被查找.因此我们需要将簇中被删除键的右侧的所有键重新插入散列表.这个过程比想象的复杂.删除代码如下所示:
``` java
public void delete(Key key)
{
  if (!contains(key)) return;
  int i = hash(key);
  while (!key.equals(keys[i]))
    i = (i + 1) % M;
  keys[i] = null;
  vals[i] = null;
  i = (i + 1) % M;
  while (keys[i] != null)
  {
    Key keyToRedo = keys[i];
    Value valToRedo = vals[i];
    keys[i] = null;
    vals[i] = null;
    N--;
    put(keyToRedo, valToRedo);
    i = (i + 1) % M;
  }
  N--;
  if (N > 0 && N == M/8) resize(M/2);
}
```
- 键簇  

&emsp;&emsp;线性探测的平均成本取决于元素在插入数组后聚集成的一组连续的条目,也叫做键簇.如下图所示:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/hashTable/线性探测法中的键簇.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="线性探测法中的键簇"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/hashTable/线性探测法中的键簇.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">线性探测法中的键簇</div>
    </a>
</center>

- - -

### 调整数组大小
- - -
&emsp;&emsp;首先,我们的LinearProbingHashST需要一个新的构造函数,它接受一个固定的容量作为参数.然后,我们需要右边给出的resize()方法.它会创建出一个新的给定大小的LinearProbingHashST,保存原表中的keys和values变量,
然后将原表中所有的键重新散列并插入到新表中.这使我们可以将数组的长度加倍.put()方法中的第一条语句会调用resize()来保证散列表最多为半满状态.这段代码构造的散列表比原来大一倍.因此α的值就会减半.
和其他需要调整数组大小的应用场景一样,我们也需要在delete()方法的最后加上 __if (N > 0 && N <= M/8) resize(M/2);__ 以保证所使用的内存量和表中的键值对数量的比例总在一定范围之内.
``` java
private void resize(int cap)
{
  LinearProbingHashST<Key, Value> t;
  t = new LinearProbingHashST<Key, Value>(cap);
  for (int i = 0; i < M; i++)
  if (keys[i] != null)
  t.put(keys[i], vals[i]);
  keys = t.keys;
  vals = t.vals;
  M = t.M;
}
```

### 拉链法
&emsp;&emsp;我们可以用相同的方法在拉链法中保持较短的链表(平均长度在2到8之间):当N>=8*M时调用resize(2*M),并在delete()中(在N>0&&N<=2*M时)调用resize(M/2).

- - -