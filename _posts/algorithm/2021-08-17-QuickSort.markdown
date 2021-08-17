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

### 思路
- - -
1. 首先以left=0,right=nums.len-1进行排序,最外层需要保证left<right
2. 排序首先以nums[left]为基准值base
3. 然后将右指针从右往左遍历,当left<right且nums[right]>=base时,将右指针右移,否则就将右指针的值替换到左指针处 (此步的意义就是找到最右侧小于基准值base并将右侧值赋值到左指针的值)
4. 接着将左指针从左往右遍历,当left<right且nums[left]<=base时,将左指针左移,否则将左指针的值赋值给右指针 (此步同步骤三相反,即找到最左侧大于基准值base的值,并将此值赋值给右指针的值)
5. 最后当两个指针相交时,将base值赋值给nums[left],并同时将left返回作为下个区间的中间值
6. 接下来就是递归遍历[left,temp-1]和[temp+1,right],遍历结束即数组有序

### 图解
- - -
第一层排序[0, 4]
![quickSort1](){:class="img-responsive img-centered"}
第二层排序[0,-1]和[1,4]
![quickSort2](){:class="img-responsive img-centered"}
第三轮排序[1,1]和[3,4]
![quickSort3](){:class="img-responsive img-centered"}

#### java代码
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
