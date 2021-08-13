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
# 思路
基本思想就是两两比较然后交换位置，将较小值不断向前移动
1. 首先需要两层循环，外层从i=0开始
2. 内层循环的主要作用就根据遍历顺序保证最大值或最小值已经移动到数组的末尾或者开头

内层有两者方式：
- 从前往后：j从0一直遍历到nums.len-1-i 处，同时比较nums[j]和nums[j+1]的值，只有nums[j]>nums[j+1]时才将nums[j]向后移动，每次循环结束保证最大值在末尾
- 从后往前：j从nums.len一直遍历到 i 处，同时比较nums[j-1]和nums[j]的值，只有nums[j-1]>nums[j]时才将nums[j]向前移动，每次循环结束保证最小值在开头

# 内层逆序图解
![bubbleSort](https://raw.githubusercontent.com/BiggerYellow/BiggerYellow.github.io/master/img/algorithm/bubbleSort/BubbleSort.jpg){:class="img-responsive img-centered"}

```
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
时间复杂度:O(n^2)
空间复杂度:O(1)

# 算法优化
没有什么大的优化，主要是若一次遍历过后中间没有移动的动作说明数组已经有序，无需继续往后遍历，直接结束

```java
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
