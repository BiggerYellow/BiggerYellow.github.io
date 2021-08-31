---
layout: default
algorithm: true
modal-id: 10000008
date: 2021-08-25
img: pexels-roman-odintsov-5327786.jpg
alt: radixSort
project-date: April 2021
client: Sort
category: algorithm
subtitle: RadixSort
description: 基数排序
---
### 前置概念
- - -

最高位优先(Most Significant Digit First)
- 简称MSD法,先按 __k1__ 排序分组,同一组中记录,关键码 __k1__ 相等,再对各组按 __k2__ 排序分成子组,之后,对后面的关键码继续这样的排序分组,直到按最次位关键码 __kd__ 对各子组排序后,再将各组连接起来,便得到一个有序序列.

最低位优先(Least Significant Digit First)
- 简称LSD法,先从 __kd__ 开始排序,再对 __kd-1__ 进行排序,依次重复,直到对 __k1__ 排序后便得到一个有序序列

- - -

### 基本思想
- - -
基数排序又称为桶排序(bucket sort).他是通过键值的部分信息,将要排序的元素分配到某些桶中,再从从小到大的桶中取出数据放回到原数组中,直到所有情况的桶都排序完成后,整个数组有序.  
- - -


### 思路
- - -
以最低位优先为例:
1. 首先将给定的nums根据个位数的数值,将他们分配值编号0到9的桶中,使用temp来记录第 __i__ 号桶内存放的nums元素,使用order来记录第 __i__ 号桶内有多少个元素
2. 接着将temp中的元素按照桶从0到9的顺序在放回数组nums中,并记得重置对应的order值
3. 最后将位数不断上升,重复以上步骤,最高位处理完即数组有序
- - -

### 图解
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/radixSort/基数排序.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="基数排序"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/radixSort/基数排序.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">RadixSort</div>
    </a>
</center>

### 代码
>java

``` java
public static void radixSort(int[] nums, int d){
    //每次排序使用的索引
    int k=0;
    //表明取的第几位数
    int n=1;
    //表明当前正在排序第几位数
    int m=1;
    int len = nums.length;

    //temp i存放0-9 j存放对应的值
    //例如 给定 nums[20] 那么 temp[0][1] = 20
    int[][] temp = new int[10][len];
    //存放0-9每个位置有多少个值 用于temp找到nums中的元素
    int[] order = new int[10];

    //m从1开始一直处理到d
    while (m<=d){
        //首轮遍历 填充temp和order
        //1. 首先先计算出最低位 lsd
        //2. 然后根据lsd的值将nums填充到temp指定的位置,并将order对应lsd位置加一
        for (int i=0;i<len;i++){
            int lsd = (nums[i]/n)%10;
            temp[lsd][order[lsd]++] = nums[i];
        }

        //经过第一轮遍历 我们已经按照lsd将数组赋值到 temp中, 接下来进行排序
        //因为数字最多就是9位,所以第二轮遍历就是从0-9
        //只要order[i]还存在元素,即order[i]!=0, 那么我们就要遍历order[i],从temp中取到对应值即temp[i][j],并赋值到原始数组nums中. 最后要把order[i]置为0即尾数为i的全都处理完毕
        for (int i=0;i<10;i++){
            if (order[i] != 0){
                for (int j=0;j<order[i];j++){
                    nums[k++] = temp[i][j];
                }
            }
            order[i]=0;
        }
        //最后重置 k为0,即下次排序继续从0开始
        //n=n*10 表明取下一位数 个位-十位-百位
        //m+=1 表明处理到了第几位
        k=0;
        n=n*10;
        m+=1;
    }
}
```

> 时间复杂度:O(NlogN)
>
> 空间复杂度:O(1)


### 其他语言
- - -
> C++

``` cpp
class Solution {
public:
	void radixSort(vector<int>& nums, int d)
	{
		int k = 0;
		int m = 1;
		int n = 1;
		int len = nums.size();
		vector<vector<int>> temp(10, vector<int>(len));
		vector<int> order(10);

		while (m<=d)
		{
			for (int i = 0; i < len; i++)
			{
				int lsd = (nums[i] / n) % 10;
				temp[lsd][order[lsd]++] = nums[i];
			}

			for (int i = 0; i < 10; i++)
			{
				if (order[i] !=0)
				{
					for (int j = 0; j < order[i]; j++) 
					{
						nums[k++] = temp[i][j];
					}
				}
				order[i] = 0;
			}
			k = 0;
			m++;
			n *= 10;
		}
	}
};
```
> python3

``` python
class Solution:
    def radixSort(self, nums:List[int], d:int):
        k=0
        n=1
        m=1
        length = len(nums)
        temp = [[0]*length for i in range (10)]
        order = [0]*10

        while m<=d:
            for i in range(0,length):
                lsd  = (nums[i]//n)%10
                temp[lsd][order[lsd]] = nums[i]
                order[lsd]+=1

            for i in range(0,10):
                if order[i] != 0:
                    for j in range(0, order[i]):
                        nums[k] = temp[i][j]
                        k+=1
                order[i]=0
            k=0
            m+=1
            n*=10
```
> go

``` go
func radixSort(nums []int, d int)  {
	k := 0
	n := 1
	m := 1
	len := len(nums)
	temp := make([][]int, 10)
	for i := range temp{
		temp[i] = make([]int, len)
	}
	
	order := make([]int, 10)

	for m<=d {
		for i:=0;i<len;i++ {
			lsd := (nums[i]/n)%10
			temp[lsd][order[lsd]] = nums[i]
			order[lsd]++
		}

		for i:=0;i<10;i++ {
			if order[i] != 0 {
				for j:=0;j<order[i];j++ {
					nums[k] = temp[i][j]
					k++
				}
			}
			order[i] = 0
		}
		m++
		k=0
		n=n*10
	}
}
```