---
layout: default
algorithm: true
modal-id: 12
date: 2021-08-17
img: pexels-jo-jesus-4198220.jpg
alt: quickSort
project-date: April 2021
client: Start Bootstrap
category: algorithm
description: 快速排序
---
### 基本思想
- - -
是一种分治的排序算法,他将一个数组分成两个子数组,将两部分独立的排序.
- - -

### 思路
- - -
1. 首先以 __left=0,right=nums.len-1__ 进行排序,最外层需要保证 __left<right__
2. 排序首先以 __nums[left]__ 为基准值base
3. 然后将右指针从右往左遍历,当 __left<right且nums[right]>=base__ 时,将右指针右移,否则就将右指针的值替换到左指针处 (此步的意义就是找到最右侧小于基准值base并将右侧值赋值到左指针的值)
4. 接着将左指针从左往右遍历,当 __left<right且nums[left]<=base__ 时,将左指针左移,否则将左指针的值赋值给右指针 (此步同步骤三相反,即找到最左侧大于基准值base的值,并将此值赋值给右指针的值)
5. 最后当两个指针相交时,将base值赋值给nums[left],并同时将left返回作为下个区间的中间值
6. 接下来就是递归遍历 __[left,temp-1]和[temp+1,right]__ ,遍历结束即数组有序

- - -

### 图解
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/quickSort/quickSort1.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="quickSort1"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/quickSort/quickSort1.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">第一层排序[0, 4]</div>
    </a>
</center>
<br>
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/quickSort/quickSort2.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="quickSort1"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/quickSort/quickSort2.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">第二层排序[0,-1]和[1,4]</div>
    </a>
</center>
<br>
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/quickSort/quickSort3.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="quickSort1"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/quickSort/quickSort3.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">第三轮排序[1,1]和[3,4]</div>
    </a>
</center>

#### 代码
>java

```
public static void quickSort(int[] nums, int left, int right){
    if (left< right){
        int base = division(nums, left, right);
        quickSort(nums, left, base-1);
        quickSort(nums, base+1, right);
    }
}

public static int division(int[] nums, int left, int right){
    int base = nums[left];
    while (left<right){
        while (left<right && nums[right]>=base){
            right--;
        }
        nums[left] = nums[right];
        while (left<right && nums[left]<=base){
            left++;
        }
        nums[right] = nums[left];
    }
    nums[left]=base;
    return left;
}
```
>时间复杂度:O(nlogn)
>
>空间复杂度:O(1)

### 其他语言
- - -
> c++

```
class Solution {
public:
	void quickSort(vector<int>& nums, int left, int right) 
	{
		if (left<right)
		{
			int temp = division(nums, left, right);
			quickSort(nums, left, temp - 1);
			quickSort(nums, temp + 1, right);
		}
	}

	int division(vector<int>& nums, int left, int right)
	{
		int base = nums[left];
		while (left<right)
		{
			while (left < right && nums[right] >= base) {
				right--;
			}
			nums[left] = nums[right];
			while (left < right && nums[left] <= base) {
				left++;
			}
			nums[right] = nums[left];
		}
		nums[left] = base;
		return left;
	}
};
```
> python3

```
class Solution:
    def quickSort(self, nums:List[int], left:int, right:int):
        def division(nums:List[int], left:int, right:int)->int:
            base = nums[left]
            while left<right:
                while left<right and nums[right]>=base:
                    right-=1
                nums[left]=nums[right]
                while left<right and nums[left]<=base:
                    left+=1
                nums[right]=nums[left]
            nums[left]=base
            return left
        if left<right:
            temp = division(nums, left ,right)
            self.quickSort(nums, left, temp-1)
            self.quickSort(nums, temp+1, right)
```
> go

```
func quickSort(nums []int, left int, right int)  {
		var division func(nums []int, left int, right int) int
		division = func(nums []int, left int, right int) int {
			base := nums[left]
			for left<right {
				for left<right && nums[right]>=base {
					right--
				}
				nums[left] = nums[right]
				for left<right && nums[left]<=base {
					left++
				}
				nums[right] = nums[left]
			}
			nums[left]=base
			return left
		}

		if left<right {
			temp:=division(nums, left, right)
			quickSort(nums, left, temp-1)
			quickSort(nums, temp+1, right)
		}
}
```