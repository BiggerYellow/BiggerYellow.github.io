---
layout: default
algorithm: true
modal-id: 15
date: 2021-08-22
img: pexels-nick-wehrli-6524730.jpg
alt: insertSort
project-date: April 2021
client: Start Bootstrap
category: algorithm
description: 插入排序
---
### 基本思想
- - -
类似于我们打牌时,将每一张牌插入到其他已经有序的牌中的适当位置.  
为了要给插入的元素腾出空间,需要将其余所有元素在插入之前都向右移动一位.
- - -

### 思路
- - -
1. 首先从外层i=1开始遍历数组,保证前i个元素有序
2. 内层j从i向前遍历,同时比较相邻的两个数,将nums[i]插入到合适位置
3. 每次内循环保证前i个元素有序
4. 重复以上步骤直到外层遍历结束即数组有序
- - -

### 图解
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/insertSort/insertSort.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="插入排序"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/insertSort/insertSort.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">InsertSort</div>
    </a>
</center>

#### 代码
>java

``` java
public static void insertSort(int[] nums){
    int len = nums.length;
    for (int i=1;i<len;i++){
        for (int j=i;j>0 && nums[j]<nums[j-1];j--){
            swap(nums, j, j-1);
        }
    }
}

public static void swap(int[] nums, int i, int j){
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

> 时间复杂度:O(n^2)
>
> 空间复杂度:O(1)



### 其他语言
- - -
> C++

``` cpp
class Solution {
public:
	void insertSort(vector<int>& nums)
	{
		int length = nums.size();
		for (int i = 1; i < length; i++)
		{
			for (int j= i; j>0; j--)
			{
				if (nums[j] < nums[j - 1]) {
					swap(nums, j, j - 1);
				}
				
			}
		}
	}

	void swap(vector<int>& nums, int i, int j)
	{
		int temp = nums[i];
		nums[i] = nums[j];
		nums[j] = temp;
	}
};
```
> python3

``` python
class Solution:
    def insertSort(self, nums:List[int]):
        length = len(nums)
        for i in range(1, length):
            for j in range(i, 0,-1):
                if(nums[j]<nums[j-1]):
                    self.swap(nums, j, j-1)

    def swap(self, nums:List[int], i,j:int):
        temp = nums[i]
        nums[i] = nums[j]
        nums[j] = temp
```
> go

``` go
func insertSort(nums []int)  {
	len := len(nums)
	for i:=1;i<len;i++ {
		for j:=i;j>0 && nums[j]< nums[j-1];j-- {
			swap2(nums, j, j-1)
		}
	}
}

func swap2(nums []int, i, j int)  {
	temp := nums[i]
	nums[i] = nums[j]
	nums[j] = temp
}
```