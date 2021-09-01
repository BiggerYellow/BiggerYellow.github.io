---
layout: default
algorithm: true
modal-id: 10000009
date: 2021-08-25
img: pexels-karina-zhukovskaya-7260333.jpg
alt: binaryTree
project-date: April 2021
client: Sort
category: algorithm
subtitle: BinaryTree
description: 二叉树基本概念
---
### 定义
- - -
二叉树是指树中节点的度不大于2的有序树,他是一种最简单且最重要的树.二叉树的递归定义为:二叉树是一棵空树,或者是一棵由一个根节点和两棵互不相交的,分别称作根的左子树和右子树组成的非空树;左子树和右子树又同样都是二叉树.
- - -

### 相关术语
- - -
1. 节点:包含一个数据元素及若干指向子树分支的信息
2. 节点的度:一个节点拥有子树的数目称为节点的度
3. 叶子节点:也成为终端节点,没有子树的节点或者度为零的节点
4. 分支节点:也称为非终端节点,度不为零的节点称为非终端节点
5. 树的度:树中所有节点的度的最大值
6. 节点的层次:从根节点开始,假设根节点为第1层,根节点的子节点为第2层,以此类推,如果第一个节点位于第L层,则其子节点位于第L+1层
7. 树的深度:也称为树的高度,树中所有节点的层次最大值称为树的深度
8. 有序树:如果树中各棵树的次序是有先后次序的,则称该树为有序树
9. 无序树:如果树中各棵子树的次序没有先后次序,则称该树为无序树
10. 森林:由m(m>=0)棵互不相交的树构成一片森林.如果把一棵非空的树的根节点删除,则该树就变成了一片森林,森林中的树由原来根节点的各棵子树构成

- - -
### 常见的树类型
- - -
1. 满二叉树:如果一棵二叉树只有度为0的节点和度为2的节点,并且度为0的节点在同一层上,则这棵二叉树为满二叉树
2. 完全二叉树:深度为k,有n个节点的二叉树当且仅当其每一个节点都与深度为k的满二叉树中编号从1到n的节点一一对应时,称为完全二叉树

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binaryTree/相关二叉树图解.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="相关二叉树图解"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binaryTree/相关二叉树图解.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">相关二叉树图解</div>
    </a>
</center>


### 二叉树性质
- - -
1. 二叉树的第 __i__ 层上最多有 __2^i-1__ (i>=1)个节点
2. 深度为 __h__ 的二叉树中至多含有 __2^h -1__ 个节点
3. 若在任意一棵二叉树中,有 __n0__ 个叶子节点,有 __n2__ 个度为2的节点,则必有 __n0=n2+1__
4. 具有 __n__ 个节点的完全二叉树深为 __log2x+1__
5. 若对一棵有 __n__ 个节点的完全二叉树进行顺序编号(1<=i<=n),那么对于编号为 __i__ (i>=1)的节点:
- 当 __i=1__ 时,该节点为根,它无双亲节点
- 当 __i>1__ 时,该节点的双亲节点的编号为 __i/2__
- 若 __2i<=n__ ,则有编号为 __2i__ 的左节点,否则没有左节点
- 若 __2i+1<=n__ ,则有编号为 __2i+1__ 的右节点,否则没有右节点

- - -

### 二叉树的存储方式
- - -
一般我们使用TreeNode对象来表示一个节点, __val__ 代表该节点额值, __left__ 代表该节点的左子节点, __right__ 代表该节点的右子节点
``` java
public class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
}
```

### 二叉树的遍历方式
- - -
二叉树可以使用深度优先遍历(DFS)或广度优先遍历(BFS).  
深度优先遍历又可以分为先序遍历、中序遍历、后序遍历三种方式.每种方式只是访问节点的顺序不同.  
- 先序遍历:根节点->左子节点->右子节点
- 中序遍历:左子节点->根节点->右子节点 
- 后序遍历:左子节点->右子节点->根节点

广度优先遍历又被称为层序遍历,即按照二叉树的每一层进行遍历

__特殊性质__ :二叉搜索树采用中序遍历,其实就是一个有序数组.  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binaryTree/二叉树的遍历.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉树的遍历"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binaryTree/二叉树的遍历.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">二叉树的遍历</div>
    </a>
</center>

### 遍历图解
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binaryTree/二叉树遍历图解.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二叉树遍历图解"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binaryTree/二叉树遍历图解.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">二叉树遍历图解</div>
    </a>
</center>


### 代码
>java

``` java
public static List<Integer> res = new ArrayList<>();

public static void dfs(TreeNode node){
    if (node != null){
        //1.先序遍历
        res.add(node.val);
        dfs(node.left);
        //2.中序遍历
        res.add(node.val);
        dfs(node.right);
        //3.后序遍历
        res.add(node.val);
    }
}

public static void bfs(TreeNode node){
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(node);
    while (!queue.isEmpty()){
        int size = queue.size();
        for (int i=0;i<size;i++){
            TreeNode poll = queue.poll();
            res.add(poll.val);
            if (poll.left != null){
                queue.offer(poll.left);
            }
            if (poll.right != null){
                queue.offer(poll.right);
            }
        }
    }
}
```

> DFS时间复杂度:O(logn)  BFS时间复杂度:O(n)  
> 空间复杂度都为:O(1)  

### 其他语言
- - -
> C++

``` cpp
class Solution {
public:

	vector<int> res;

	vector<int> sort(struct TreeNode* node)
	{
		dfs(node);
		return res;
	}

	void dfs(TreeNode* node)
	{
		if (node)
		{
			//1.先序遍历
			res.push_back(node->val);
			dfs(node->left);
			//2.中序遍历
			res.push_back(node->val);
			dfs(node->right);
			//3.后序遍历
			res.push_back(node->val);
		}
	}

	vector<int> sortBFS(struct TreeNode* node)
	{
		queue<TreeNode*> queue;
		queue.push(node);
		while (!queue.empty())
		{
			int size = queue.size();
			for (int i = 0; i < size; i++)
			{
				TreeNode* pop = queue.front();
				queue.pop();
				res.push_back(pop->val);
				if (pop->left)
				{
					queue.push(pop->left);
				}
				if (pop->right)
				{
					queue.push(pop->right);
				}
			}
		}
		return res;
	}
};
```
> python3

``` python
class Solution:
    def traverse(self, node:TreeNode) -> List[int]:
        res = list()
        def dfs(node:TreeNode):
            if node:
                #1.先序遍历
                #res.append(node.val)
                dfs(node.left)
                #2.中序遍历
                res.append(node.val)
                dfs(node.right)
                #3.后序遍历
                #res.append(node.val)
        dfs(node)
        return res

    def traverseBSF(self,node:TreeNode) -> List[int]:
        res = list()
        queue = list()
        queue.append(node)
        while len(queue) != 0:
            size = len(queue)
            for i in range(size):
                temp = queue.pop(0)
                res.append(temp.val)
                if temp.left:
                    queue.append(temp.left)
                if temp.right:
                    queue.append(temp.right)
        return res
```
> go

``` go
func traverse(node *TreeNode) []int {
	res := []int{}
	var dfs func(node *TreeNode)
	dfs = func(node *TreeNode) {
		if node != nil {
			//1.先序遍历
			res = append(res, node.Val)
			dfs(node.Left)
			//2.中序遍历
			res = append(res, node.Val)
			dfs(node.Right)
			//3.后序遍历
			res = append(res, node.Val)
		}
	}
	dfs(node)
	return res
}

func BFS(node *TreeNode) []int {
	res := make([]int, 0)
	queue := make([]*TreeNode , 0)
	queue = append(queue, node)
	for len(queue)!=0 {
		temp := queue[0]
		res = append(res, temp.Val)
		if temp.Left!=nil {
			queue = append(queue, temp.Left)
		}
		if temp.Right!=nil {
			queue = append(queue, temp.Right)
		}
		queue = queue[1:]
	}
	return res
}
```