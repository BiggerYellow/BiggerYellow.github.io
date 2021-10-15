---
layout: default
algorithm: true
modal-id: 10000012
date: 2021-09-18
img: pexels-jue-5806480.jpg
alt: BalancedBinaryTree
project-date: Septemper 2021
client: Tree
category: algorithm
subtitle: BalancedBinaryTree
description: 平衡二叉查找树
---
### 定义
- - -
&emsp;&emsp;因为二叉查找树在极端情况会退化成链表,导致查询的时间复杂度为O(n).因为平衡二叉查找树应运而生.  
&emsp;&emsp;它是一种结构平衡的二查叉找树,即叶节点高度差的绝对值不超过1,并且左右两个子树都是一棵平衡二叉查找树.  
&emsp;&emsp;它能在 __O(logn) __ 内完成插入、查找和删除操作.  
- - -

### 2-3查找树
- - -
&emsp;&emsp;一棵2-3查找树或为一棵空树,或由以下节点组成:
- 2-节点:含有一个键(及其对应的值)和两条连接,左连接指向的2-3树中的键都小于该节点,右连接指向的2-3树中的键都大于该节点
- 3-节点:含有两个键(及其对应的值)和三条连接,左连接指向的2-3树中的键都小于该节点,中连接指向的2-3树中的键都位于该节点的两个键之间,右连接指向的2-3树中的键都大于该节点 
&emsp;&emsp;将指向一棵空树的连接称为空连接,一棵完美平衡的2-3查找树中的所有空连接到根节点的距离都应该相同,2-3查找树如下图:    
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/2-3 查找树示意图.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="2-3 查找树示意图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/2-3 查找树示意图.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">2-3 查找树示意图</div>
    </a>
</center>
  
1.查找  
&emsp;&emsp;判断一个键是否在树中,我们先将它和根节点中的键比较.如果它和其中任意个相等,查找命中;否则我们就根据比较的结果找到指向相应区间的连接,并在其指向的子树中递归地继续查找,如果这是个空连接,查找未命中.具体查询过程如图:    

<center>  
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/2-3 树中的查找命中(左)和未命中(右).png">  
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="2-3 树中的查找命中(左)和未命中(右)"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/2-3 树中的查找命中(左)和未命中(右).png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">2-3 树中的查找命中(左)和未命中(右)</div>  
    </a>  
</center>  
    
2.向2-节点中插入新键  
&emsp;&emsp;要在2-3树中插入一个新节点,可以和二叉查找树一样先进行一次未命中的查找,然后把新节点挂在树的底部,但是这样的话无法保持完美平衡性.  
&emsp;&emsp;我们使用2-3树的主要原因就在于它能够在插入后继续保持平衡.如果未命中的查找结束于一个2-节点:我们只要把这个2-节点替换为一个3-节点,将要插入的键保存到其中即可(如下图);如果未命中的查找结束于一个3-节点,事情就麻烦一点.  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向2- 结点中插入新的键.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向2- 结点中插入新的键"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向2- 结点中插入新的键.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">向2- 结点中插入新的键</div>
    </a>
</center>  
    
3.向一棵只含有一个3-节点的树中插入新键  
&emsp;&emsp;假设我们需要向一棵只含有一个3-节点的树中插入一个新键.这棵树中有两个键,所以在它唯一的节点中已经没有可插入新键的空间了.  
&emsp;&emsp;为了将新键插入,我们先临时将新键存入该节点中,使之成为一个4-节点.创建一个4-节点很方便,因为很容易将它转换为一棵由3个2-节点组成的2-3树,其中一个节点(根)含有中键,一个节点含有3个键中的最小者(和根节点的左链接相连),一个节点含有3个键中的最大者(和根节点的右链接相连).  
&emsp;&emsp;这棵树既是一棵含有3个节点的二叉查找树,同时也是一棵完美平衡的2-3树,因为其中所有的空连接到根节点的距离都相等.插入前树的高度为0,插入后树的高度为1.它说明了树是如何生长的.如下图所示:  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一棵只含有一个3-结点的树中插入新键.png">  
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向一棵只含有一个3-结点的树中插入新键"  
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一棵只含有一个3-结点的树中插入新键.png">  
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">向一棵只含有一个3-结点的树中插入新键</div>  
    </a>  
</center>  
    
4.向一个父节点为2-节点的3-节点插入新键  
&emsp;&emsp;假设未命中的查找结束于一个3-节点,而它的父节点是一个2-节点.在这种情况下我们需要在维持树的完美平衡的前提下为新键腾出空间.  
&emsp;&emsp;我们先像刚才一样构造一个临时的4-节点并将其分解,但此时我们不会为中键创建一个新节点,而是将其移动至原来的父节点中.  
&emsp;&emsp;你可以将这次转换看成将指向原3-节点的一条连接替换为新父节点中的原中键左右两边的两条连接,并分别指向两个新的2-节点.  
&emsp;&emsp;根据我们的假设,父节点是有空间的:父节点是一个2-节点(一个键两条连接),插入之后变为了一个3-节点(两个键三条连接).  
&emsp;&emsp;这次转换也并不影响2-3树的主要性质.树仍然是有序的,因为中键被移动到父节点中去了;树仍然是完美平衡的,插入后所有的空连接到根节点的距离仍然相同,过程如下图所示:

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一个父结点为2-结点的3-结点中插入新键.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向一个父结点为2-结点的3-结点中插入新键"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一个父结点为2-结点的3-结点中插入新键.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">向一个父结点为2-结点的3-结点中插入新键</div>
    </a>
</center>
    
5.向一个父节点为3-节点的3-节点中插入新键  
&emsp;&emsp;现在假设未命中的查找结束于一个父节点为3-节点的节点.我们再次和刚才一样构造一个临时的4-节点并分解它,然后将它的中键插入它的父节点中.但父节点也是一个3-节点,因此我们再用和这个中键构造一个新的临时4-节点,然后在这节点上进行相同的变换,即分解这个父节点并将它的中键插入到它的父节点中.  
&emsp;&emsp;我们就这样一直向上不断分解临时的4-节点并将中间插入到它的父节点中,直到遇到一个2-节点并将它替换为一个不需要继续分解的3-节点,或者是到达3-节点的根.过程如下图:  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一个父结点为3-结点的3-结点中插入新键.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向一个父结点为3-结点的3-结点中插入新键"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一个父结点为3-结点的3-结点中插入新键.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">向一个父结点为3-结点的3-结点中插入新键</div>
    </a>
</center>
    
6.分解根节点  
&emsp;&emsp;如果从插入节点到根节点的路径上全都是3-节点,我们的根节点最终变成一个临时的4-节点.此时我们可以按照向一棵只有一个3-节点的树中插入新键的方法处理这个问题.我们将临时的4-节点分解为3个2-节点,使得树高加1,如下图所示:  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/分解根结点.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="分解根结点"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/分解根结点.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">分解根结点</div>
    </a>
</center>
    
7.局部变换  
&emsp;&emsp;将一个4-节点分解为一棵2-3树可能有6种情况,总结在下图.  
&emsp;&emsp;这个4-节点可能是根节点,可能是一个2-节点的左子节点或右子节点,也可能是一个3-节点的左子节点、中子节点或者右子节点.  
&emsp;&emsp;2-3树插入算法的根本在于这些变换都是局部的:除了相关的节点和连接之外不必修改或者检查树的其他部分.每次变换中,变更的连接数量不会超过一个很小的常数.需要特别指出的是,不光是在树的底部,树中的任何地方只要符合相应的模式,变换都可以进行.每个变换都会将4-节点中的一个键送入它的父节点中,并重构相应的连接而不必涉及树的其他部分.  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/在一棵2-3树中分解一个4-结点的情况汇总.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="在一棵2-3树中分解一个4-结点的情况汇总"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/在一棵2-3树中分解一个4-结点的情况汇总.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">在一棵2-3树中分解一个4-结点的情况汇总</div>
    </a>
</center>
    
8.全局性质  
&emsp;&emsp;这些局部变换不会影响树的全局有序性和平衡性:任意空连接到根节点的路径长度都是相等的.  
&emsp;&emsp;下图所示的是当一个4-节点是一个3-节点的中子节点时的完整变换情况.如果在变换之前根节点到所有空连接的路径长度为h,那么变换之后该长度仍然为h.所有的变换都是这个性质,即使是将一个4-节点分解为两个2-节点并将其父节点由2-节点变为3-节点,或是由3-节点变为一个临时的4-节点也是如此.  
&emsp;&emsp;当根节点被分解为3个2-节点时,所有空连接到根节点的路径长度才会加1.

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/4-结点的分解是一次局部变换,不会影响树的有序性和平衡性.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="4-结点的分解是一次局部变换,不会影响树的有序性和平衡性"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/4-结点的分解是一次局部变换,不会影响树的有序性和平衡性.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">4-结点的分解是一次局部变换,不会影响树的有序性和平衡性</div>
    </a>
</center>


&emsp;&emsp;和标准的二叉查找树由上向下生长不同,2-3树的生长是由下向上的.如下图所示:

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/2-3树的构造轨迹.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="2-3树的构造轨迹"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/2-3树的构造轨迹.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">2-3树的构造轨迹</div>
    </a>
</center>

>结论
>在一棵大小为N的2-3树中,查找和插入操作访问的节点必然不超过lgN个

- - -

### 红黑二叉查找树
- - -
1.替换3-节点  
&emsp;&emsp;红黑二叉查找树背后的基本思想是用标准的二叉查找树(完全由2-节点构成)和一些额外的信息(替换3-节点)来表示2-3树.
我们将树中的链接分为两种类型:红链接将两个2-节点连接起来构成一个3-节点,黑链接则是2-3树中的普通链接.确切地说,我们将3-节点表示为由一条左斜的红色链接(两个2-节点其中之一是另一个左子节点)相连的两个2-节点,如下图所示.
这种表示法的一个优点是,无需修改就可以直接只用标准二叉查找树的get()方法.对于任意的2-3树,只要对节点进行转换,我们都可以立即派生出一颗对应的二叉查找树.将用这种方式表示2-3树的二叉查找树称为红黑二叉查找树.
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/由一条红色左链接相连的两个2-结点表示一个3-结点.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="由一条红色左链接相连的两个2-结点表示一个3-结点"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/由一条红色左链接相连的两个2-结点表示一个3-结点.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">由一条红色左链接相连的两个2-结点表示一个3-结点</div>
    </a>
</center>

2.等价的定义  
&emsp;&emsp;红黑树的另一种定义是含有红黑链接并满足下列条件的二叉查找树:
- 红链接均为左链接
- 没有任何一个节点同时和两条红链接相连
- 该树是完美黑色平衡的,即任意空链接到根节点的路径上的黑链接数量相同

3.一一对应  
&emsp;&emsp;如果我们将一棵红黑树的红链接画平,那么所有的空链接到根节点的距离都将使相同的.如下图所示.
如果将由红链接相连的节点合并,得到的就是一棵2-3树.相反,如果将一棵2-3树中的3-节点画做由红色左链接相连的两个2-节点,那么不会存在能够和两条红链接相连的节点,且树必然是完美黑色平衡的,因为黑链接即2-3树中的普通连接,根据定义这些连接必然是完美平衡的.
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/将红链接画平时,一棵红黑树就是一棵2-3树.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="将红链接画平时,一棵红黑树就是一棵2-3树"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/将红链接画平时,一棵红黑树就是一棵2-3树.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">将红链接画平时,一棵红黑树就是一棵2-3树</div>
    </a>
</center>
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/红黑树和2-3树的一一对应关系.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="红黑树和2-3树的一一对应关系"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/红黑树和2-3树的一一对应关系.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">红黑树和2-3树的一一对应关系</div>
    </a>
</center>
4.颜色表示  
&emsp;&emsp;因为每个节点都只会有一条指向自己的链接(从它的父节点指向它),我们将链接的颜色保存在表示节点的Node数据类型的布尔变量color中.如果指向它的链接的颜色是红色的,那么该变量为true,黑色为false.约定空链接为黑色.
使用私有方法isRed()来测试一个节点和它的父节点之间的链接的颜色.当我们提到一个节点的颜色时,我们指的是指向该节点的链接的颜色,反之亦然.代码及图示如下:

``` java
private static final boolean RED = true;
private static final boolean BLACK = false;
private class Node{
    Key key;        //键
    Value value;    //相关联的值
    Node left,right;//左右子树
    int N;          //这棵子树中的节点总数
    boolean color;  //由其父节点指向它的链接的颜色

    Node(Key key, Value val, int N, boolean color){
        this.key=key;
        this.val=val;
        this.N=N;
        this.color=color;
    }
}
private boolean isRed(Node x){
    if(x==null) return false;
    return x.color == RED;
}
```

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/红黑树的结点表示.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="红黑树的结点表示"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/红黑树的结点表示.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">红黑树的结点表示</div>
    </a>
</center>

5.旋转  
&emsp;&emsp;在我们实现的某些操作中可能会出现红色右链接或者两条连续的红链接,但在操作完成前这些情况都会被小心地旋转并修复.旋转操作会改变红链接的指向.
首先,假设我们有一条红色的右链接需要被转化为左链接,如下图.这个操作叫做左旋转,它对应的方法接受一条并指向红黑树中的某个节点的链接作为参数.
假设被指向的节点的右链接是红色的,这个方法会对树进行必要的调整并返回一个指向包含同一组键的子树且左链接为红色的根节点的链接.
这个操作很容易理解:我们只是将用两个键中的较小者作为根节点变为较大者作为根节点.实现将一个红色左链接转换为一个红色右链接的一个右旋转的代码完全相同,只需要将left和right互换即可.
- 左旋转
``` java
Node rotateLeft(Node h){
    Node x = h.right;
    h.right = x.left;
    x.left = h;
    x.color = h.color;
    h.color = RED;
    x.N = h.N;
    h.N = 1+size(h.left)+size(h.right);
    return x;
}
```
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/左旋转h的右链接.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="左旋转h的右链接"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/左旋转h的右链接.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">左旋转h的右链接</div>
    </a>
</center>

- 右旋转
``` java
Node rotateRight(Node h){
    Node x = h.left;
    h.left = x.right;
    x.right = h;
    x.color = h.color;
    h.color = RED;
    x.N = h.N;
    h.N = 1+size(h.left)+size(h.right);
    return x;
}
```
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/右旋转h的左链接.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="右旋转h的左链接"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/右旋转h的左链接.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">右旋转h的左链接</div>
    </a>
</center>

6.在旋转后重置父节点的链接  
&emsp;&emsp;无论坐旋转还是右旋转,旋转操作都会返回一条链接.我们总是会用rotateRight()或rotateLeft()的返回值重置父节点(或是根节点)中相应的链接.
返回的链接可能是左链接也可能是右链接,但是我们总会将它赋予父节点中的链接.这个链接可能是红色也可能黑色--rotateLeft()和rotateRight()都通过将x.color设为h.color保留它原来的颜色.
这可能会产生两条连续的红链接,但我们的算法会继续用旋转操作修正这种情况.  
&emsp;&emsp;在插入新的键时,我们可以使用旋转操作帮助我们保证2-3树和红黑树之间的一一对应关系,因为旋转操作可以保证红黑树的两个重要性质:有序性和完美平衡性.
也就是说,我们在红黑树中进行旋转时无需为树的有序性或完美平衡性担心.下面我们继续看看如何使用旋转操作来保持红黑树的另外两个重要性质.

7.向单个2-节点中插入新键  
&emsp;&emsp;一棵树只含有一个键的红黑树只含有一个2-节点.插入另一个键后,马上就需要将它们旋转.如果新键小于老键,只需要新增一个红色的节点即可,新的红黑树和单个3-节点完全等价.
如果新键大于老键,那么新增的红色节点将会产生一条红色的右链接.我们需要使用 __root=rotateLeft(root)__ 将其旋转为红色左链接并修正根节点,插入操作才算完成.
两种情况的结果为一棵和单个3-节点等价的红黑树,其中含有两个键,一条红链接,树的黑链接高度为1,如下图所示:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向单个2-结点中插入一个新键.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向单个2-结点中插入一个新键"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向单个2-结点中插入一个新键.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">向单个2-结点中插入一个新键</div>
    </a>
</center>

8.向树底部的2-节点插入新键  
&emsp;&emsp;用和二叉树查找树相同的方式向一棵红黑树中插入一个新键会在树的底部新增一个节点(为了保证有序性),但总是用红链接将新节点和它的父节点相连.
如果它的父节点是一个2-节点,那么刚才讨论的两种处理方法仍然适用.如果指向新节点的是父节点的左链接,那么父节点就直接成为了一个3-节点;
如果指向新节点的是父节点的右链接,这就是一个错误的3-节点,但一次左旋转就能修正它,如下图所示:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向树底部的2-结点插入一个新键.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向树底部的2-结点插入一个新键"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向树底部的2-结点插入一个新键.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">向树底部的2-结点插入一个新键</div>
    </a>
</center>

9.向一棵双键树(即一个3-节点)中插入新键  
&emsp;&emsp;这种情况又可分为三种情况:新键小于树中的两个键,在两者之间,或是大于树中的两个键.每种情况都会产生一个同时连接到两条红链接的节点,而我们的目标就是修正这一点.  
- 三者中最简单的情况就是新键大于原树中的两个键,因此它被连接到3-节点的右链接.此时树是平衡的,根节点为中间大小的键,它有两条红链接分别和较小和较大的节点相连.
如果我们将两条链接的颜色都由红变黑,那么我们就得到了一棵有三个节点组成,高度为2的平衡树.它正好能对应一棵2-3树,如下图左侧.其他两种情况最终也会转换为这种情况.
- 如果新键小于原树中的两个键,它就会被连接到最左边的空链接,这种就产生了两条连续的红链接,如下图中间侧.此时我们只需要将上层的红链接右旋转即可得到第一种情况(中值键为根节点并和其他两个节点用红链接相连)
- 如果新键介于原树中的两个键之间,这又会产生两条连续的红链接,一条红色左链接接一条红色右链接,如下图右侧.此时我们只需要将下层的红链接左旋转即可得到第二种情况(两条连续的红色链接)

&emsp;&emsp;总的来说,我们通过0次,1次和2次旋转以及颜色的变化得到了期望的结果.
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一棵双键树(即一个3-结点)中插入一个新键的三种情况.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向一棵双键树(即一个3-结点)中插入一个新键的三种情况"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/向一棵双键树(即一个3-结点)中插入一个新键的三种情况.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">向一棵双键树(即一个3-结点)中插入一个新键的三种情况</div>
    </a>
</center>

10.颜色转换  
&emsp;&emsp;如下图,我们专门用一个方法 __flipColors()__ 来转换一个节点的两个红色子节点的颜色.除了将子节点的颜色由红变黑之外,我们同时还要将父节点的颜色由黑变红.
这项操作最重重要的性质在于它和旋转操作一样是局部变换,不会影响整棵树的黑色平衡.
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/通过转换链接的颜色来分解4-结点.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="通过转换链接的颜色来分解4-结点"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/balancedBinaryTree/通过转换链接的颜色来分解4-结点.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">通过转换链接的颜色来分解4-结点</div>
    </a>
</center>

11.根节点总是黑色  
&emsp;&emsp;在第9种情况中,颜色转换会使根节点变为红色.严格的说,红色的根节点说明根节点是一个3-节点的一部分,但是实际情况并不是这样.因此我们在每次插入后都会将根节点设为黑色.注意,每当根节点由红变黑时树的黑链接高度就会加1.


- - -

