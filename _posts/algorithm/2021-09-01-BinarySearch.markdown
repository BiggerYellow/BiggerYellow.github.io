---
layout: default  
algorithm: true  
modal-id: 10000010
date: 2021-09-01
img: pexels-victoria-borodinova-9201864.jpg
alt: BinarySearch
project-date: April 2021
client: Search
category: algorithm
subtitle: BinarySearch
description: 二分查找
---
### 定义
- - -
二分查找也称为折半查找,它是一种效率较高的查找方法.但是它要求是线性表必须采用顺序存储结构,而且表中的元素是有序的.  
- - -

### 思路
- - -
例如:查找 __nums__ 有序数组中, __target__ 在nums中的位置
1. 首先定义左右指针 __left=0__ 和 __right=nums.length-1__
2. 遍历数组,退出条件为 __left<=right__ 
3. 找到中间位置 __mid=left+(right-left)/2__ ,此时 __nums[0...mid-1]__ 的值都小于 __nums[mid]__ , __nums[mid+1...len]__ 的值都大于 __nums[mid]__
4. 比较中间位置的值 __nums[mid]__ 和 __target__ 的值,有三种情况:
    - 若 __nums[mid]=target__ ,直接返回 __mid__
    - 若 __nums[mid]<target__ ,中间值小于目标值,那么接下来应该在区间 __[mid+1,len]__ 中查找,即 __left=mid+1__ 
    - 若 __nums[mid]>target__ ,中间值大于目标值,那么接下来应该在区间 __[0,mid-1]__ 中查找,即 __right=mid-1__
5. 若按步骤4没有找到,说明数组中不存在等于 __target__ 的索引
- - -

### 遍历图解
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearch/二分查找.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="二分查找"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/binarySearch/二分查找.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">二分查找</div>
    </a>
</center>


### 代码
>java

``` java
public static int binarySearch(int[] array, int target){
    int left = 0;
    int right = array.length-1;
    while (left <= right){
        int mid = left + (right-left)/2;
        if (array[mid] == target){
            return mid;
        }else if (array[mid] < target){
            left = mid + 1;
        }else if (array[mid] > target){
            right = mid - 1;
        }
    }
    return -1;
}
```

> 时间复杂度:O(logn)  
> 空间复杂度为:O(1)  


### 扩展
- - - 
正常遇到的二分查找一般不会这么简单.我们还需要了解一些二分查找的扩展.  
例如给定的数组中有多个重复元素,那么我们如何找出给定 __target__ 的最左或最右索引位置.  
整体逻辑同二分查找类似, 不同点在于:  
1. 退出循环条件, __left<right__ 
2. 当 __nums[mid]>target__ ,直接将 __mid__ 赋值给 __right__  
3. 当 __nums[mid]==target__ 时,有两种情况:
    - 若取最左位置,则继续向左搜索,即限制右指针, __right=mid__ 
    - 若取最右位置,则继续向右搜索,即限制左指针, __left=mid+1__  

> java代码

``` java
public static int binarySearchLeft(int[] array, int target){
    int left = 0;
    int right = array.length;
    while (left < right){
        int mid = left + (right-left)/2;
        if (array[mid] == target){
            right = mid;
        }else if (array[mid] < target){
            left = mid +1;
        }else if (array[mid] > target){
            right = mid;
        }
    }
    if (left == array.length) return -1;
    return array[left] == target? left : -1;
}

public static int binarySearchRight(int[] array, int target){
    int left =0;
    int right = array.length;
    while (left<right){
        int mid = left + (right-left)/2;
        if (array[mid] == target){
            left = mid+1;
        }else if (array[mid] < target){
            left = mid+1;
        }else if (array[mid] > target){
            right = mid;
        }
    }
    if (left == 0) return -1;
    return array[left-1] == target? left-1:-1;
}
```

### 其他语言
- - -
> C++

``` cpp
public:
	int binarySearch(vector<int>& nums, int target)
	{
		int left = 0;
		int right = nums.size() - 1;
		while (left<=right)
		{
			int mid = left + (right - left) / 2;
			if (nums[mid]==target)
			{
				return mid;
			}
			else if (nums[mid]>target)
			{
				right = mid - 1;
			}
			else
			{
				left = mid + 1;
			}
		}
		return -1;
	}
};
```
> python3

``` python
def binarySearch(self, nums:List[int], target:int) ->int:
    left=0
    right=len(nums)-1
    while left<=right:
        mid = left+(right-left)//2
        if nums[mid]==target:
            return mid
        elif nums[mid]>target:
            right=mid-1
        else:
            left=mid+1
    return -1
```
> go

``` go
func binarySearch(nums []int, target int) int {
	left:=0
	right:=len(nums)-1
	for left<=right {
		mid :=left+(right-left)/2
		if nums[mid]==target {
			return mid
		}else if nums[mid]>target {
			right = mid-1
		}else {
			left = mid+1
		}
	}
	return -1
}
```