---
layout: default
algorithm: true
modal-id: 11
date: 2021-08-13
img: pexels-rachel-claire-4846299.jpg
alt: bubbleSort
project-date: April 2021
client: Start Bootstrap
category: algorithm
description: 冒泡排序
---

### 基本思想
- - -
就是两两比较然后交换位置,将较小值不断向前移动
- - -

### 思路
- - -
1. 首先需要两层循环,外层从 __i=0__ 开始
2. 内层循环的主要作用就根据遍历顺序保证最大值或最小值已经移动到数组的末尾或者开头

内层有两者方式：
- 从前往后:j从0一直遍历到nums.len-1-i 处,同时比较 __nums[j]__ 和 __nums[j+1]__ 的值,只有 __nums[j]>nums[j+1]__ 时才将 __nums[j]__ 向后移动,每次循环结束保证最大值在末尾
- 从后往前:j从nums.len一直遍历到 i 处,同时比较 __nums[j-1]和nums[j]__ 的值,只有 __nums[j-1]>nums[j]__ 时才将 __nums[j]__ 向前移动,每次循环结束保证最小值在开头

- - -

### 内层逆序图解
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/bubbleSort/BubbleSort.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="BubbleSort"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/bubbleSort/BubbleSort.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">bubbleSort</div>
    </a>
</center>

#### 代码
>java

``` java
//内层逆序遍历
public static void bubbleSort(int[] nums){
     int temp;
    for (int i = 0; i< nums.length - 1; i++ ){
        for (int j = nums.length -1; j>i; j--){
            if (nums[j-1]> nums[j]){
                temp = nums[j-1];
                nums[j-1] = nums[j];
                nums[j] = temp;
            }
        }
    }
}

//内层正序遍历
private static void bubbleSort(int[] nums) {
        if(nums==null || nums.length < 2 ){
            return;
        }
        for (int i = 0; i < nums.length - 1; i++) {
            for (int j = 0; j < nums.length - i -1; j++) {   
                if (nums[j] > nums[j + 1]) {
                    int temp = nums[j];
                    nums[j] = nums[j + 1];
                    nums[j + 1] = temp;
                }
            }
        }
    }
```


> 时间复杂度:O(n^2)
>
> 空间复杂度:O(1)

### 算法优化
- - -
没有什么大的优化,主要是若一次遍历过后中间没有移动的动作说明数组已经有序,无需继续往后遍历,直接结束
- - -

``` java
private static void bubbleSort(int[] nums) {
    int temp;
    boolean flag = true;
    for (int i = 0; i< nums.length - 1; i++ ){
        for (int j = nums.length -1; j>i; j--){
            if (nums[j-1]> nums[j]){
                temp = nums[j-1];
                nums[j-1] = nums[j];
                nums[j] = temp;
                flag = false;
            }
        }
        if (flag){
            break;
        }
    }
}
```

### 其他语言
- - -
> C++

``` cpp
class Solution {
public:
	//内部正序
	void bubbleSort(vector<int>& nums)
	{
		int len = nums.size();
		for (int i = 0; i < len - 1; i++)
		{
			for (int j = 0; j < len-i-1; j++)
			{
				if (nums[j + 1] < nums[j])
				{
					int temp = nums[j];
					nums[j] = nums[j + 1];
					nums[j + 1] = temp;
				}
			}
		}
	}

	//内部逆序
	void bubbleSort1(vector<int>& nums)
	{
		int len = nums.size();
		for (int i = 0; i < len; i++)
		{
			for (int  j = len-1; j >i; j--)
			{
				if (nums[j-1]>nums[j])
				{
					int temp = nums[j];
					nums[j] = nums[j - 1];
					nums[j - 1] = temp;
				}
			}
		}
	}
};
```


> python3

``` python
class Solution:
    #内部正序
    def bubbleSort(self, nums:List[int]):
        length = len(nums)
        for i in range(0, length-1):
            for j in range(0, length-i-1):
                if nums[j+1] < nums[j]:
                    temp = nums[j]
                    nums[j] = nums[j+1]
                    nums[j+1] = temp

    #内部正序
    def bubbleSort1(self, nums:List[int]):
        length = len(nums)
        for i in range(0, length):
            for j in (length-1, i, -1):
                if nums[j-1]>nums[j]:
                    temp = nums[j]
                    nums[j] = nums[j-1]
                    nums[j-1] = temp
```
> go

``` go
//内部逆序
func bubbleSort(nums []int) {
	len :=len(nums)
	for i:=0;i<len-1;i++ {
		for j:=0;j<len-i-1;j++ {
			if nums[j+1]<nums[j] {
				temp:=nums[j]
				nums[j]=nums[j+1]
				nums[j+1]=temp
			}
		}
	}
}

func bubbleSort1(nums []int){
	len := len(nums)
	for i:=0; i<len;i++ {
		for j:=len-1;j>i;j-- {
			if nums[j] < nums[j-1] {
				temp:=nums[j]
				nums[j]=nums[j-1]
				nums[j-1]=temp
			}
		}
	}
}
```