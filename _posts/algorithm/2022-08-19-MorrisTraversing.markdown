---
layout: default
algorithm: true
modal-id: 10000020
date: 2022-08-19
img: pexels-thiago-matos-4037037.jpg 
alt: Morris Traversing
project-date: August 2022
client: algorithm
category: algorithm
subtitle: Morris Traversing
description: 莫里斯遍历
---
### 定义
- - -
&emsp;&emsp;对于二叉树的遍历我们常用的方法就是迭代和递归遍历，两种遍历方式使用压栈的方式，递归调用的是虚拟机的栈，迭代使用的是自定义的栈，两者的空间复杂度都为 __O(n)__ 。  
&emsp;&emsp;为了追求极致的效率,莫里斯遍历应运而生,它使用一种巧妙的方式将空间复杂度降为 __O(1)__ .下面我们就一起来看看如何巧妙:  
- - -

### 基本思路
- - -
&emsp;&emsp;遍历过程中用到的两个概念先说明一下,当前遍历的根节点用 __node__ 来表示,当前根节点左子树中的最右节点 __pre__ .  
从当前根节点 __node__ 开始遍历:
1. 若当前根节点 __node__ 的左节点为空,将当前根节点的右节点指向当前根节点 __node = node.right__ ,说明不存在左子树,即左子树遍历结束,开始遍历右子树(这里之所以遍历右子树,因为在第二步中已经将右子树和最近的一个根节点建立连接形成环)  
2. 若当前根节点 __node__ 的左节点不为空,将当前根节点的左节点指向最右节点 __pre = node.left__ ,从当前根节点的左节点继续向右遍历一直到底,因为会通过步骤2.1形成环,所有这里的结束条件为右节点为空或右节点等于当前根节点,判断最右节点的右节点 __pre.right__ 是否为空:
- 2.1 若最右节点的右节点 __pre.right__ 为空,将最右节点的右节点指向根节点 __pre.right = root__ ,继续遍历当前根节点的左子树 __node = node.left__ ,说明此最右节点与当前根节点形成环,成环并继续从当前根节点向左遍历  
- 2.2 若最右节点的右节点 __pre.right__ 不为空,将最右节点的右节点指向空 __pre.right = null__ ,继续遍历当前根节点的右子树 __node = node.right__ ,说明此时遇到步骤2.1生成的环了,将环断开并继续从当前根节点向右遍历  

&emsp;&emsp;可见的莫里斯遍历的核心思想就是将根节点与根节点左子树中的最右节点相连接形成环,这样在遍历的时候就可以重新回到根节点,无须借助其他数据结构降低空间复杂度.  
- - -

### 遍历过程
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mirrorsTraversing/莫里斯遍历.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="莫里斯遍历"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mirrorsTraversing/莫里斯遍历.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">莫里斯遍历</div>
    </a>
</center>
- - -




### 例题
- - -
1.[恢复二叉搜索树](https://leetcode.cn/problems/recover-binary-search-tree/)
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mirrorsTraversing/恢复二叉搜索树.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="恢复二叉搜索树"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mirrorsTraversing/恢复二叉搜索树.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">恢复二叉搜索树</div>
    </a>
</center>

>java

``` java
public static List<Integer> res = new ArrayList<>();

public static void morris(TreeNode root){
    TreeNode pre = null;
    while (root != null){
        //判断当前节点是否有左子树
        // 1.有则说明左子树未遍历完继续遍历左子树,再根据最右节点进行判断
        // 2.无则说明左子树遍历完,取当前节点值,并继续遍历右子树
        if(root.left != null){
            //pre意在找到当前节点左子树中的最右节点
            pre = root.left;
            //若 pre存在右节点 && pre的右节点不等于当前遍历的根节点 (因为在下一步若pre不存在右节点会将pre的右节点指向当前节点x形成一个环)
            while(pre.right != null && pre.right != root){
                pre = pre.right;
            }
            //1.若pre最右节点不存在右节点(也可以理解为未指向其根节点形成环),则将pre的右节点指向根节点形成环,并继续遍历根节点的左子树
            //2.若pre最右节点存在右节点(可以理解为已经指向根节点形成环了),取当前root值并将pre最右节点置为null(即将环断开),继续遍历右子树 (若经过步骤1成环了,即变成从子节点回到根节点)
            if (pre.right == null){
                pre.right = root;
//                    //先序遍历
//                    res.add(root.val);
                root = root.left;
            }else{
                //中序遍历
                res.add(root.val);
                pre.right = null;
                root = root.right;
            }

        }else {
            res.add(root.val);
            root = root.right;

        }
    }
}
```

> 时间复杂度:O(n)  
> 空间复杂度为:O(1)  

### 延伸-后序遍历
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mirrorsTraversing/莫里斯后序遍历.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="莫里斯后序遍历"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mirrorsTraversing/莫里斯后序遍历.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">莫里斯后序遍历</div>
    </a>
</center>

``` java
public static List<Integer> res = new ArrayList<>();


public static void morrisPostOrder(TreeNode root){
    if(root == null){
        return;
    }
    //自定义头结点形成环
    TreeNode node = new TreeNode(-1);
    node.left = root;
    TreeNode pre = node;
    while(pre != null){
        if (pre.left != null){
            TreeNode next = pre.left;
            while(next.right != null && next.right != pre){
                next = next.right;
            }
            if (next.right == null){
                next.right = pre;
                pre = pre.left;
            }else{
                next.right = null;
                //使用头插法在断开环时将右节点和根节点加入结果汇总
                List<Integer> temp = new ArrayList<>();
                TreeNode cur = pre.left;
                while (cur != null){
                    temp.add(0, cur.val);
                    cur = cur.right;
                }
                res.addAll(temp);
                pre = pre.right;
            }
        }else{
            pre = pre.right;
        }
    }
}
```
- - -

### 其他语言
- - -
> C++

``` cpp

```
> python3

``` python

```
> go

``` go

```