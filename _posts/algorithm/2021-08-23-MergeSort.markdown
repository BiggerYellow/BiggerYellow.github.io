---
layout: default
algorithm: true
modal-id: 17
date: 2021-08-24
img: pexels-anete-lusina-5721167.jpg
alt: mergeSort
project-date: April 2021
client: Start Bootstrap
category: algorithm
description: 归并排序
---
### 基本思想
- - -
要将一个数组排序,可以先递归地将它分成两半分别排序,然后将结果归并起来.  
归并排序最吸引人的性质是它能够保证将任意长度为N的数组排序所需时间和 __NlogN__ 成正比;它的主要缺点则是它所需的额外空间和 __N__ 成正比.  
- - -

### 原地归并的抽象方法
- - -
实现的方法很简单,创建一个适当大小的数组然后将两个输入数组中的元素一个个从大到小放入这个数组中.  
之所以需要原地归并,是因为当用归并一个大数组排序时,我们需要进行很多次归并,因此在每次归并时都创建一个新数组来存储排序结果会大大增加空间复杂度.  
如果使用原地归并的方法,这样就可以先将前半部分排序,再将后半部分排序, 然后在数组中移动元素而不需要使用额外的空间.  
原地归并的方法签名为: __merge(a, lo, mid, hi)__ ,它会将子数组 __a[lo..mid]__ 和 __a[mid+1..hi]__ 归并成一个有序的数组并将结果存放在 __a[lo..hi]__ 中.
- - -

### 代码
>java

``` java
public static void merge(int[] nums, int lo, int mid, int hi){
        int i=lo, j=mid+1;
        for (int k=lo;k<=hi;k++){
            temp[k] = nums[k];
        }
        for (int k=lo;k<=hi;k++){
            if (i>mid){
                nums[k] = temp[j++];
            }
            else if (j>hi) {
                nums[k] = temp[i++];
            }else if (temp[j]<temp[i]){
                nums[k] = temp[j++];
            }else {
                nums[k] = temp[i++];
            }
        }
    }
```

### 代码分析
- - -
该方法先将所有数组复制到temp[]中,然后在归并回a[]中.  
方法在归并时(第二个for循环)进行了4个判断:  
1. i>mid:左半边用完(取右半边的元素)
2. j>hi:右半边用尽(取左半边的元素)
3. temp[j]<temp[i]:右半边的当前元素小于左半边的当前元素(取右半边的元素)
4. temp[j]>temp[i]:右半边的放弃元素大于左半边的当前元素(取左半边的元素)

仔细观察如上规则,其实就是给你两个有序集合,要把这两个有序集合合并成一个有序集合,使用双指针的方式合并.
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mergeSort/归并排序区间分布图.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="归并排序区间分布图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mergeSort/归并排序区间分布图.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">归并排序区间分布图</div>
    </a>
</center>

### 自顶向下的归并排序
- - -
这是应用高效算法设计中分治思想的最典型的一个例子.
1. 首先归并的排序每一个区间,保证区间大小为1的区间全都有序
2. 其次就是归并合并两个区间保证整个区间有序
3. 重复以上步骤即整个数组有序  

- - -

### 图解
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mergeSort/归并排序.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="归并排序"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/mergeSort/归并排序.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">MergeSort</div>
    </a>
</center>

#### 代码
>java

``` java
public static int[] temp;

public static void mergeSort(int[] nums){
    temp = new int[nums.length];
    sort(nums, 0, nums.length-1);
}

public static void sort(int[] nums, int lo, int hi){
    if (hi<=lo){
        return;
    }
    int mid = lo + (hi-lo)/2;
    sort(nums, lo, mid);
    sort(nums, mid+1, hi);
    merge(nums, lo, mid, hi);
}

public static void merge(int[] nums, int lo, int mid, int hi){
    int i=lo, j=mid+1;
    for (int k=lo;k<=hi;k++){
        temp[k] = nums[k];
    }
    for (int k=lo;k<=hi;k++){
        if (i>mid){
            nums[k] = temp[j++];
        }
        else if (j>hi) {
            nums[k] = temp[i++];
        }else if (temp[j]<temp[i]){
            nums[k] = temp[j++];
        }else {
            nums[k] = temp[i++];
        }
    }
}
```

> 时间复杂度:O(NlogN)
>
> 空间复杂度:O(1)

### 归并排序的缺点及改进方案
- - -
- 缺点: 辅助数组所使用的额外空间和N的大小成正比.

- 改进点
1. 对小规模子数组使用插入排序:用不同的方法处理小规模问题能改进大多数递归算法的性能,因为递归会使小规模问题中方法的调用过于频繁,所以改进对它们的处理方法就是改进整个算法.
2. 测试数组是否已经有序:可以添加一个判断条件,如果 __a[mid]__ 小于等于 __a[mid+1]__ ,我们就冉伟数组已经有序的并跳过merge()方法.  
3. 不将元素复制到辅助数组:可以节省将数组元素复制到用于归并的辅助数组所用的时间(但空间不行).要做到这一点我们要调用两种排序方法,一种将数据从输入数组排序到辅助数组,一种将数据从辅助数组排序到输入数组.
- - -

### 自底向上的归并排序


### 其他语言
- - -
> C++

``` cpp
class Solution {
public:
	void mergeSort(vector<int>& nums)
	{
		int length = nums.size();
		vector<int> temp(length);
		sort(nums, 0, length - 1, temp);
	}

	void sort(vector<int>& nums, int lo, int hi, vector<int>& temp)
	{
		if (hi<=lo)
		{
			return;
		}
		int mid = (hi + lo) / 2;
		sort(nums, lo, mid, temp);
		sort(nums, mid + 1, hi, temp);
		merge(nums, lo, mid, hi, temp);
	}

	void merge(vector<int>& nums, int lo, int mid, int hi, vector<int>& temp)
	{
		int i = lo;
		int j = mid + 1;
		for (int k = lo; k <= hi; k++)
		{
			temp[k] = nums[k];
		}
		for (int k = lo; k <= hi; k++)
		{
			if (i > mid) 
			{
				nums[k] = temp[j++];
			}
			else if (j>hi)
			{
				nums[k] = temp[i++];
			}
			else if (temp[j]<temp[i])
			{
				nums[k] = temp[j++];
			}
			else
			{
				nums[k] = temp[i++];
			}
		}
	}
};
```
> python3

``` python
def mergeSort(self, nums:List[int]):
    temp = [0]*len(nums)
    def sort(nums:List[int], lo:int, hi:int):
        if hi<=lo:
            return
        mid = lo + (hi-lo)//2
        sort(nums, lo, mid)
        sort(nums, mid+1, hi)
        merge(nums, lo, mid, hi)

    def merge(nums:List[int], lo:int, mid:int, hi:int):
        i,j = lo, mid+1
        for k in range(lo, hi+1):
            temp[k] = nums[k]

        for k in range(lo, hi+1):
            if i>mid :
                nums[k] = temp[j]
                j+=1
            elif j>hi:
                nums[k] = temp[i]
                i+=1
            elif temp[j]<temp[i]:
                nums[k] = temp[j]
                j+=1
            else:
                nums[k] = temp[i]
                i+=1
    sort(nums, 0, len(nums)-1)
```
> go

``` go
//自顶向下归并排序
func mergeSort(nums []int){
	temp :=make([]int, len(nums))

	var merge func(nums []int, lo int, mid int, hi int)
	merge = func(nums []int, lo int, mid int, hi int) {
		i:=lo
		j:=mid+1
		for k:=lo;k<=hi;k++ {
			temp[k] = nums[k]
		}
		for k:=lo;k<=hi;k++{
			if i>mid {
				nums[k] = temp[j]
				j++
			}else if j>hi {
				nums[k] = temp[i]
				i++
			}else if temp[j]<temp[i] {
				nums[k] = temp[j]
				j++
			}else {
				nums[k] = temp[i]
				i++
			}
		}
	}

	var sort func(nums []int, lo int, hi int)
	sort = func(nums []int, lo int, hi int) {
		if hi<=lo {
			return
		}
		mid := lo+(hi-lo)/2
		sort(nums, lo, mid)
		sort(nums, mid+1, hi)
		merge(nums, lo, mid, hi)
	}

	sort(nums, 0, len(nums)-1)
}
```