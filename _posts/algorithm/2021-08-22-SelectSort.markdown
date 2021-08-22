---
layout: default
algorithm: true
modal-id: 14
date: 2021-08-22
img: pexels-8906527.jpg
alt: selectSort
project-date: April 2021
client: Start Bootstrap
category: algorithm
description: 选择排序
---
### 基本思想
- - -
主要就是遍历数组,找到最小值和最前面的元素交换即可.
- - -

### 思路
- - -
1. 首先找到数组中最小的那个元素
2. 其次将它和数组的第一个元素交换位置(如果第一个元素就是最小元素那么它就和自己交换)
3. 再次在剩下的元素中找到最小的元素,将它与数组的第二个元素交换位置
4. 如此往复,直到整个数组有序
- - -

### 堆排序图解
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/selectSort/selectSort.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="选择排序"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/selectSort/selectSort.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">HeapSort</div>
    </a>
</center>

#### 代码
>java

``` java
public static void selectSort(int[] nums){
    int len = nums.length;
    for (int i=0;i<len;i++){
        int min = i;
        for (int j=i+1;j<len;j++){
            if (nums[min] > nums[j]){
                min=j;
            }
        }
        swap(nums, i, min);
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
	void selectSort(vector<int>& nums)
	{
		int length = nums.size();
		for (int i = 0; i < length; i++)
		{
			int min = i;
			for (int j = i+1; j < length; j++)
			{
				if (nums[min] > nums[j])
				{
					min = j;
				}
			}
			swap(nums, min, i);
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
    def selectSort(self, nums:List[int]):
        length = len(nums)
        for i in range(0, length):
            min = i
            for j in range(i+1, length):
                if nums[min]>nums[j]:
                    min = j
            self.swap(nums, min, i)

    def swap(self, nums:List[int], i:int, j:int):
        temp = nums[i]
        nums[i] = nums[j]
        nums[j] = temp
```
> go

``` go
func selectSort(nums []int)  {
	len := len(nums)
	for i:=0;i<len;i++ {
		min:=i
		for j:=i+1;j<len;j++ {
			if nums[min]>nums[j] {
				min = j
			}
		}
		swap1(nums, min, i)
	}
}

func swap1(nums []int, i int, j int){
	temp := nums[i]
	nums[i] = nums[j]
	nums[j] = temp
}
```