---
layout: default
algorithm: true
modal-id: 16
date: 2021-08-23
img: pexels-alena-beliaeva-8697338.jpg
alt: shellSort
project-date: April 2021
client: Start Bootstrap
category: algorithm
description: 希尔排序
---
### 缘由
- - -

对于大规模乱序数组插入排序很慢,因为它只会交换相邻的元素,因此元素只能一点点的从数组的一端移动到另一端.  
例如,如果主键最小的元素正好在数组的尽头,要将它挪到正确的位置就需要N-1次移动.  
希尔排序为了加速简单地改进了插入排序,交换不相邻的元素以对数组的局部进行排序并最终用插入排序将局域有序的数组排序.

- - -

### 基本思想
- - -
使数组中任意间隔为h的元素都是有序的,这样的数组称为h有序数组.  
换句话说,一个h有序数组就是h个互相独立的有序数组编织在一起组成的一个数组,在进行排序时,如果h很大,我们就能将元素移动到很远的地方,为实现更小的h有序创造方便.  
用这种方法,对于任意以1结尾的h序列,我们都能够将数组排序.
- - -

### 思路
- - -
对于每个h,用插入排序将h个子数组独立的排序.但因为子数组是相互独立的,一个更简单的方法是在h-子数组中将每个元素交换到比它大的元素之前去（将比它大的元素向右移动一格）.只需要在插入排序的代码中将移动元素的距离由1改为h即可.这样,希尔排序的实现就转化为了一个类似于插入排序但使用不同增量的过程.  
希尔排序更高效的原因是它权衡了子数组的规模和有序性.排序之初,各个子数组都很短,排序之后子数组都是部分有序的,这两种情况都很适合插入排序.子数组部分有序的程度取决于递增序列的选择.
- - -


### 如何选择递增序列
- - -
算法的性能不仅取决于h,还取决于h之间的数学性质,比如他们的公因子等.这个需要视情况而定.
- - -

### 图解
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/shellSort/shellSort.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="希尔排序"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/shellSort/shellSort.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">ShellSort</div>
    </a>
</center>

#### 代码
>java

``` java
public static void shellSort(int[] nums){
    int len = nums.length;
    int factor = 2;
    int h=1;
    while (h<len/factor){
        h = h*factor+1;
    }
    while (h>=1){
        for (int i=h;i<len;i++){
            for (int j=i;j>=h && nums[j]<nums[j-h];j-=h){
                swap(nums, j, j-h);
            }
        }
        h/=factor;
    }
}

public static void swap(int[] nums, int i, int j){
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

> 时间复杂度:O(n^3/2)
>
> 空间复杂度:O(1)



### 其他语言
- - -
> C++

``` cpp
class Solution {
public:
	void shellSort(vector<int>& nums)
	{
		int len = nums.size();
		int factor = 2;
		int h = 1;
		while (h<len/factor)
		{
			h = h * factor + 1;
		}

		while (h>=1)
		{
			for (int i = h; i < len; i++)
			{
				for (int j = i; j >=h; j-=h)
				{
					if (nums[j]< nums[j-h])
					{
						swap(nums, j, j - h);
					}
				}
			}
			h /= factor;
		}
	}

	void swap(vector<int>& nums, int i,int j) 
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
    def shellSort(self, nums:List[int]):
        length = len(nums)
        factor = 2
        h=1
        while h<length//factor:
            h=h*factor+1

        while h>=1:
            for i in range(h, length):
                for j in range(i, h-1,-2):
                    if nums[j] < nums[j-h]:
                        self.swap3(nums, j, j-h)
            h//=factor


    def swap3(self,nums:List[int], i,j:int):
        temp = nums[i]
        nums[i] = nums[j]
        nums[j] = temp
```
> go

``` go
func shellSort(nums []int){
	len := len(nums)
	factor:=2
	h :=1
	for h<len/factor {
		h = h*factor+1
	}

	for h>=1 {
		for i:=h;i<len;i++ {
			for j:=i;j>=h && nums[j]<nums[j-h];j-=h {
				swap3(nums, j, j-h)
			}
		}
		h/=factor
	}
}

func swap3(nums []int, i,j int)  {
	temp := nums[i]
	nums[i] = nums[j]
	nums[j] = temp
}
```