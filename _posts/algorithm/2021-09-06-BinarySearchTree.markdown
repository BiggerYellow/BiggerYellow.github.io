---
layout: default
algorithm: true
modal-id: 10000011
date: 2021-08-25
img: pexels-julia-volk-5273316.jpg
alt: BinarySearchTree
project-date: September 2021
client: Tree
category: algorithm
subtitle: BinarySearchTree
description: 二查叉找树
---
### 定义
- - -
二叉查找树(BinarySearchTree,简称BST)又称为二叉排序树(BinarySortTree).  
一棵二叉查找树是一棵二叉树,其中每个节点都含有一个 __Comparable__ 的键(以及相关联的值)且每个节点的键都大于等于其左子树中的任意节点而小于右子树的任意节点.  
- - -

### 基本实现
- - -  
1. 数据表示     
    
    和链表一样,我们嵌套定义了一个所有类来表示二叉查找树上的一个节点.每个节点都含有一个键、一个值、一条左链接、一条右链接和一个节点计数器.左链接指向一棵由小于该节点的所有键组成的二叉树,右链接指向一棵由大于该节点的所有键组成的二叉查找树.变量 __N__ 给出了以该节点为根的子树的节点总数.  
    一棵二叉查找树代表了一组键(及其相应的值)的集合,而同一个集合可以用多棵不同的二叉查找树表示.如果我们将一棵二叉查找树的所有键投影到一条直线上,保证一个节点的左子树中的键出现在它的左边,右子树中的键出现在它的右边,那么我们一定可以得到一条有序的键列.  
    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/两棵能够表示同一组键的二叉查找树.png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="两棵能够表示同一组键的二叉查找树"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/两棵能够表示同一组键的二叉查找树.png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">两棵能够表示同一组键的二叉查找树</div>
        </a>
    </center>  
    
2. 查找   
    在二叉查找树中查找一个键的递归算法:如果树是空的,则查找未命中;如果被查找的键和根节点的键相等,则查找命中,否则我们就递归的在适当的子树中继续查找.如果被查找的键较小就选择左子树,较大则选择右子树.  
    
    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的查找命中(左)和未命中(右).png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树中的查找命中(左)和未命中(右)"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的查找命中(左)和未命中(右).png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">二叉查找树中的查找命中(左)和未命中(右)</div>
        </a>
    </center>  
    
3. 插入   
    如果树是空的,就返回一个含有该键值对的新节点;如果被查找的键小于根节点的键,会继续在左子树中插入该键,否则在右子树中插入该键.  
    
    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的插入操作.png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树的插入操作"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的插入操作.png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">二叉查找树的插入操作</div>
        </a>
    </center>

``` java
public class BST<Key extends Comparable<Key>, Value> {
    private Node root;

    private class Node{
        private Key key;
        private Value value;
        private Node left,right;
        private int N;

        public Node(Key key, Value value, int N){
            this.key = key;
            this.value=value;
            this.N = N;
        }
    }

    public int size(){
        return size(root);
    }

    private int size(Node x){
        if (x==null) return 0;
        else return x.N;
    }

    public Value get(Key key){
        return get(root, key);
    }

    private Value get(Node x,Key key){
        if (x==null) return null;
        int cmp = key.compareTo(x.key);
        if (cmp<0) return get(x.left, key);
        else if (cmp>0) return get(x.right, key);
        else return x.value;
    }

    public void put(Key key, Value value){
        root = put(root, key, value);
    }

    private Node put(Node x, Key key, Value value){
        if (x==null) return new Node(key, value, 1);
        int cmp = key.compareTo(x.key);
        if (cmp<0) x.left = put(x.left, key, value);
        else if (cmp>0) x.right = put(x.right, key, value);
        else x.value = value;
        x.N = size(x.left) + size(x.right) +1;
        return x;
    }
}
```

- - -
### 分析
- - -
使用二叉查找树的算法运行时间取决于树的形状,而树的形状取决于键被插入的先后顺序.在最好的情况下,一棵含有 __N__ 个节点的树是完全平衡的,每条空链接和根节点的距离都为 __lgN__ .在最坏的情况下,搜索路径上可能有 __N__ 个节点.

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的可能形状.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树的可能形状"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的可能形状.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">二叉查找树的可能形状</div>
    </a>
</center>
- - -



### 有序性相关的方法与删除操作
- - -
二叉查找树得以广泛应用的一个重要原因就是它能够保持键的有序性,因此它可以作为实现有序符号表API中众多方法的基础.  
    
1. 最大键和最小键  
    最小键:如果根节点的左链接为空,那么一棵二叉查找树中最小的键就是根节点;如果左链接非空,那么树中最小键就是左子树中的最小键.见下方min()方法的递归实现  
    最大键:如果根节点的右链接为空,那么一棵二叉查找树中最大的键就是根节点;如果右链接非空,那么树中最大键就是右子树中的最大键.见下方max()方法的递归实现  

    ``` java
    public Key min()
    {
        return min(root).key;
    }
    
    private Node min(Node x)
    {
        if(x.left==null) return x;
        return min(x.left);
    }
    
    public Key max()
    {
        return max(root).key;
    }
    
    private Node max(Node x)
    {
        if(x.right==null) return x;
        return max(x.right);
    }
    ```
    
2. 向上取整和向下取整  
    向下取整: 如果给定的键 __key__ 小于二叉查找树的根节点的键,那么小于等于 __key__ 的最大键 __floor(key)__ 一定在根节点的左子树中;如果给定的键 __key__ 大于二叉查找树的根节点,那么只有当根节点右子树中存在小于等于 __key__ 的节点时,小于等于 __key__ 的最大键才会出现在右子树中,否则根节点就是小于等于 __key__ 的最大键.  
    向上取整: 将向下取整中的 左变为右(同时将小于变为大于)就能得到 __ceiling()__ 的算法.  

    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/向下取整图解.png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="向下取整图解"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/向下取整图解.png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">向下取整图解</div>
        </a>
    </center>
    
    ``` java
    public Key floor(Key key)
    {
        Node x = floor(root, key);
        if (x==null) return null;
        return x.key;
    }
    
    public Node floor(Node x, Key key)
    {
        if(x==null) return null;
        int cmp = key.compareTo(x.key);
        if(cmp==0) return x;
        if(cmp<0) return floot(x.left, key);
        Node t = floot(x.right, key);
        if(t!=null){
            return t;
        }else{
            return x;
        } 
    }
    ```

3. 选择操作  
    在二叉查找树的每个节点中维护的子树节点计数器变量N就是用来支持此操作的.  
    假设我们想找到排名为k的键(即树中正好有k个小于它的键).有如下三种情况:  
    - 如果左子树中的节点数t大于k,那么我们就继续递归在左子树中查找排名为k的键;  
    - 如果t等于k,就返回根节点中的键;
    - 如果t小于k,就递归的在右子树中查找排名为 __k-t-1__ 的键.  

    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的select()操作.png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树中的select()操作"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的select()操作.png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">二叉查找树中的select()操作</div>
        </a>
    </center>

    ``` java
    public Key select(int k){
        return select(root, k).key;
    }
    
    public Node select(Node x, int k){
        if(x==null) return null;
        int t = size(x.left);
        if(t>k) return select(x.left, k);
        else if (t<k) return select(x.right, k-t-1);
        else return x;
    }
    ```

4. 排名  
    rank()是select()的逆方法,它会返回给定键的排名.它的实现和select()类似.  
    如果给定的键和根节点的键相等,我们返回左子树中的节点总数t;  
    如果给定的键小于根节点,我们会返回该键在左子树中的排名;  
    如果给定的键大于根节点,我们会返回t+1(根节点)加上它在右子树中的排名.  

    ``` java
    public int rank(int k){
        return rank(root, k);
    }
    
    public int rank(Node x, int k){
        if(x==null) return null;
        int cmp = k.compareTo(x.key);
        if(cmp<0) return rank(x.left, k);
        else if (cmp >0) return 1+size(x.left, k)+rank(x.right, k);
        else return size(x.left);
    }
    ```

5. 删除最大键和删除最小键  
    我们要不断深入根节点的左子树直到遇到一个空链接,然后将指向该节点的连接指向该节点的右子树(只需要在递归调用中返回它的右链接即可).  
    此时已经没有任何链接指向要被删除的节点,因此它会被垃圾收集器清理掉.然后在更新它到根节点的路径上所有计数器的值.  
    deleteMax()方法的实现和deleteMin()完全类似.  

    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/删除二叉查找树中的最小结点.png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="删除二叉查找树中的最小结点"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/删除二叉查找树中的最小结点.png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">删除二叉查找树中的最小结点</div>
        </a>
    </center>

    ``` java
    public void deleteMin(){
        root = deleteMin(root);
    }

    private Node deleteMin(Node x){
        if(x.left==null) return x.right;
        x.left = deleteMin(x.left);
        x.N = size(x.left)+size(x.right)+1;
        return x;
    }
    ```

6. 删除操作  
    我们可以用类似的方式删除任意只有一个子节点或者没有子节点的节点,但应该如何删除一个拥有两个子节点的节点呢?  
    删除之后我们要处理两棵子树,但被删除节点的父节点只有一条空出来的链接.  
    在删除节点x后用它的后继节点填补它的位置.因为x有一个右节点,因此它的后继节点就是其右子树的最小节点.这样的替换仍然能够保证树的有序性,因为x.key和它的后继节点的键之间不存在其他的键.  
    我们能够用4个简单的步骤完成将x替换为它的后继节点的任务.  
    - 将指向即将被删除的节点的链接保存为t
    - 将x指向它的后继节点 __min(t.right)__
    - 将x的右链接(原本指向一棵所有节点都大于x.key的二叉查找树)指向 __deleteMin(t.right)__ ,也就是在删除后所有节点仍然大于x.key的子二叉查找树
    - 将x的左链接(本为空)设为t.left(其下所有的键都小于被删除节点和它的后继节点)

    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的删除操作.png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树中的删除操作"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的删除操作.png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">二叉查找树中的删除操作</div>
        </a>
    </center>
    
    ``` java
    public void delete(Key key){
        root = delete(root, key);
    }
    
    private Node delete(Node x, Key key){
        if(x==null) return null;
        int cmp = key.compareTo(x.key);
        if(cmp<0) x.left = delete(x.left, key);
        else if (cmp>0) x.right = delete(x.right, key);
        else{
            if(x.right==null) return x.left;
            if(x.left==null) return x.right;
            Node t = x;
            x = min(t.right);
            x.right = deleteMin(x.right);
            x.left = t.left;
        }
        x.N = size(x.left)+size(x.right)+1;
        return x;
    }
    ```

7. 范围查找  

    首先需要一个遍历二叉查找树的基本方法,叫做中序遍历.即左->根->右。
    使用队列Queue,将所有落在给定范围内的键加入一个队列Queue并跳过那些不可能含有所查找键的子树.  
    为了确保以给定节点为根的子树中所有在指定范围内的键加入队列,我们会递归地查找根节点的左子树,然后查找根节点,然后递归地查找根节点的右子树.  

    <center>
        <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的范围查找.png">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树的范围查找"
        src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的范围查找.png">
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">二叉查找树的范围查找</div>
        </a>
    </center>
    
    ``` java
    public Iterable<Key> keys(){
        return keys(min(), max());
    }
    
    public Iterable<Key> keys(Key lo, Key hi){
        Queue<Key> queue = new Queue<Key>();
        keys(root, queue, lo, hi);
        return queue;
    }
    
    private void keys(Node x, Queue<Key> queue, Key lo, Key hi){
        if(x==null) return;
        int cmplo = lo.compareTo(x.key);
        int cmphi = hi.compareTo(x.key);
        if(cmp<0) keys(x.left, queue, lo, hi);
        if(cmplo<=0 &&cmphi>=0) queue.enqueue(x.key);
        if(cmphi>0) keys(x.right, queue, lo, hi);
    }
    ```
- - -

### 总结
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/简单的符号表实现的成本总结.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="简单的符号表实现的成本总结"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/简单的符号表实现的成本总结.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">简单的符号表实现的成本总结</div>
    </a>
</center>
- - -