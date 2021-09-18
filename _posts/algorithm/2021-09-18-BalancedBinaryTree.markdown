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
因为二叉查找树在极端情况会退化成链表,导致查询的时间复杂度为O(n).因为平衡二叉查找树应运而生.  
它是一种结构平衡的二查叉找树,即叶节点高度差的绝对值不超过1,并且左右两个子树都是一棵平衡二叉查找树.  
它能在 __O(logn) __ 内完成插入、查找和删除操作.  
- - -

### 2-3查找树
- - -
一棵2-3查找树或为一棵空树,或由以下节点组成:
    - 2-节点:含有一个键(及其对应的值)和两条连接,左连接指向的2-3树中的键都小于该节点,右连接指向的2-3树中的键都大于该节点
    - 3-节点:含有两个键(及其对应的值)和三条连接,左连接指向的2-3树中的键都小于该节点,中连接指向的2-3树中的键都位于该节点的两个键之间,右连接指向的2-3树中的键都大于该节点 
将指向一棵空树的连接称为空连接,一棵完美平衡的2-3查找树中的所有空连接到根节点的距离都应该相同,2-3查找树如下图:    
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
判断一个键是否在树中,我们先将它和根节点中的键比较.如果它和其中任意个相等,查找命中;否则我们就根据比较的结果找到指向相应区间的连接,并在其指向的子树中递归地继续查找,如果这是个空连接,查找未命中.具体查询过程如图:    

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
要在2-3树中插入一个新节点,可以和二叉查找树一样先进行一次未命中的查找,然后把新节点挂在树的底部,但是这样的话无法保持完美平衡性.  
我们使用2-3树的主要原因就在于它能够在插入后继续保持平衡.如果未命中的查找结束于一个2-节点:我们只要把这个2-节点替换为一个3-节点,将要插入的键保存到其中即可(如下图);如果未命中的查找结束于一个3-节点,事情就麻烦一点.  

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
假设我们需要向一棵只含有一个3-节点的树中插入一个新键.这棵树中有两个键,所以在它唯一的节点中已经没有可插入新键的空间了.  
为了将新键插入,我们先临时将新键存入该节点中,使之成为一个4-节点.创建一个4-节点很方便,因为很容易将它转换为一棵由3个2-节点组成的2-3树,其中一个节点(根)含有中键,一个节点含有3个键中的最小者(和根节点的左链接相连),一个节点含有3个键中的最大者(和根节点的右链接相连).  
这棵树既是一棵含有3个节点的二叉查找树,同时也是一棵完美平衡的2-3树,因为其中所有的空连接到根节点的距离都相等.插入前树的高度为0,插入后树的高度为1.它说明了树是如何生长的.如下图所示:  

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
假设未命中的查找结束于一个3-节点,而它的父节点是一个2-节点.在这种情况下我们需要在维持树的完美平衡的前提下为新键腾出空间.  
我们先像刚才一样构造一个临时的4-节点并将其分解,但此时我们不会为中键创建一个新节点,而是将其移动至原来的父节点中.  
你可以将这次转换看成将指向原3-节点的一条连接替换为新父节点中的原中键左右两边的两条连接,并分别指向两个新的2-节点.  
根据我们的假设,父节点是有空间的:父节点是一个2-节点(一个键两条连接),插入之后变为了一个3-节点(两个键三条连接).  
这次转换也并不影响2-3树的主要性质.树仍然是有序的,因为中键被移动到父节点中去了;树仍然是完美平衡的,插入后所有的空连接到根节点的距离仍然相同,过程如下图所示:

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
现在假设未命中的查找结束于一个父节点为3-节点的节点.我们再次和刚才一样构造一个临时的4-节点并分解它,然后将它的中键插入它的父节点中.但父节点也是一个3-节点,因此我们再用和这个中键构造一个新的临时4-节点,然后在这节点上进行相同的变换,即分解这个父节点并将它的中键插入到它的父节点中.  
我们就这样一直向上不断分解临时的4-节点并将中间插入到它的父节点中,直到遇到一个2-节点并将它替换为一个不需要继续分解的3-节点,或者是到达3-节点的根.过程如下图:  

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
如果从插入节点到根节点的路径上全都是3-节点,我们的根节点最终变成一个临时的4-节点.此时我们可以按照向一棵只有一个3-节点的树中插入新键的方法处理这个问题.我们将临时的4-节点分解为3个2-节点,使得树高加1,如下图所示:  

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
将一个4-节点分解为一棵2-3树可能有6种情况,总结在下图.  
这个4-节点可能是根节点,可能是一个2-节点的左子节点或右子节点,也可能是一个3-节点的左子节点、中子节点或者右子节点.  
2-3树插入算法的根本在于这些变换都是局部的:除了相关的节点和连接之外不必修改或者检查树的其他部分.每次变换中,变更的连接数量不会超过一个很小的常数.需要特别指出的是,不光是在树的底部,树中的任何地方只要符合相应的模式,变换都可以进行.每个变换都会将4-节点中的一个键送入它的父节点中,并重构相应的连接而不必涉及树的其他部分.  

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
这些局部变换不会影响树的全局有序性和平衡性:任意空连接到根节点的路径长度都是相等的.  
下图所示的是当一个4-节点是一个3-节点的中子节点时的完整变换情况.如果在变换之前根节点到所有空连接的路径长度为h,那么变换之后该长度仍然为h.所有的变换都是这个性质,即使是将一个4-节点分解为两个2-节点并将其父节点由2-节点变为3-节点,或是由3-节点变为一个临时的4-节点也是如此.  
当根节点被分解为3个2-节点时,所有空连接到根节点的路径长度才会加1.

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


和标准的二叉查找树由上向下生长不同,2-3树的生长是由下向上的.如下图所示:

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
- - -

