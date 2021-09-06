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
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/两棵能够表示同一组键的二叉查找树.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="两棵能够表示同一组键的二叉查找树"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/两棵能够表示同一组键的二叉查找树.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">两棵能够表示同一组键的二叉查找树</div>
    </a>
</center>
2. 查找
在二叉查找树中查找一个键的递归算法:如果树是空的,则查找未命中;如果被查找的键和根节点的键相等,则查找命中,否则我们就递归的在适当的子树中继续查找.如果被查找的键较小就选择左子树,较大则选择右子树.
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的查找命中(左)和未命中(右).jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树中的查找命中(左)和未命中(右)"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树中的查找命中(左)和未命中(右).jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">二叉查找树中的查找命中(左)和未命中(右)</div>
    </a>
</center>
3. 插入
如果树是空的,就返回一个含有该键值对的新节点;如果被查找的键小于根节点的键,会继续在左子树中插入该键,否则在右子树中插入该键.  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的插入操作.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树的插入操作"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的插入操作.jpg">
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
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的可能形状.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉查找树的可能形状"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearchTree/二叉查找树的可能形状.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">二叉查找树的可能形状</div>
    </a>
</center>
- - -



### 有序性相关的方法与删除操作
- - -
- - -

### 扩展


### 其他语言
- - -
> C++

``` cpp
xxx
```
> python3

``` python
xxx
```
> go

``` go
xxx
```