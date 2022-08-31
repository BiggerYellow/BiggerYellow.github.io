---
layout: default
algorithm: true
modal-id: 10000019
date: 2022-06-15
img: pexels-emmanuel-hernndez-12217674.jpg 
alt: ReservoirSampling
project-date: June 2022
client: algorithm
category: algorithm
subtitle: Reservoir Sampling
description: 蓄水池采样算法
---
### 定义
- - -
&emsp;&emsp;给定一串很长的数据流,对该数据流中数据只能访问一次,使得数据流中所有数据被选中的概率相等.  
&emsp;&emsp;解决这样的问题,就可以利用蓄水池算法(Reservoir Sampling).  
- - -

### 基本思路
- - -
&emsp;&emsp;假设需要采样的数量为 __k__ .
1. 首先构建一个 __k__ 个元素的数组,将序列前的 __k__ 个元素放入数组中.
2. 对于从第 __j__ 个元素 __(j>k)__ ,以 __$\frac{k}{j}$__ 的概率来决定该元素是否被替换到数组中,数组中的 __k__ 个元素被替换的概率是相同的.
3. 当遍历完所有元素之后,数组中剩下的元素即为采样样本.
- - -

### 证明过程
- - -
1. 对于第 __i__ 个元素 __(i<k)__ .在 __k__ 步之前,被选中的概率为1.
当走到第 __k+1__ 步时,被 __k+1__ 个元素替换的概率 = 第 __k+1__ 个元素被选中的概率 __$\times$__ __i__ 被选中替换的概率,即 __$\frac{k}{k+1} \times \frac{1}{k} = \frac{1}{k+1}$__ .  
那么,不被第 __k+1__ 个元素替换的概率为 __$1-\frac{1}{k+1} = \frac{k}{k+1}$__ .以此类推,不被 __k+2__ 个元素替换的概率为 __$1-\frac{k}{k+2} \times \frac{1}{k} = \frac{k+1}{k+2}$__ .  
当递推到第 __n__ 个元素时,第 __i__ 个元素被保留的概率 = 被选中的概率 __$\times$__ 不被替换的概率,即 __\\[1 \times \frac{k}{k+1} \times \frac{k+1}{k+2} \times \frac{k+2}{k+3} ... \times \frac{n-1}{n} = \frac{1}{n}\\]__
2. 对于第 __j__ 个元素 __(j>k)__ . 在第 __j__ 步被选中的概率为 __$\frac{k}{j}$__ ,不被 __j+1__ 个元素替换的概率为 __$1- \frac{k}{j+1} \times \frac{1}{k}$__ ,当递推到第 __n__ 个元素时,被保留的概率 = 被选中的概率 * 不被替换的概率，即 __\\[\frac{k}{j} \times \frac{j}{j+1} \times \frac{j+1}{j+2} ... \times \frac{n-1}{n} = \frac{k}{n}\\]__ 因此对于每个元素,被保留的概率都是 __$\frac{k}{n}$__
- - -


### 理解过程
- - -
&emsp;&emsp;遍历长度等于 __n__ 的数组.当第 __i__ 次遇到 __target__ 的元素时,随机选择区间 __[0, i)__ 内的一个整数,如果其等于 __0__ ,则将返回值置为该元素的下标,否则返回值不变.  
&emsp;&emsp;设nums中有 __k__ 个值为 __target__ 的元素,该算法会保证这 __k__ 个元素的下标最终返回值概率均为$\frac{1}{k}$,证明如下  
P(第i次遇到值为target的元素下标称为最终返回值) = P(第i次随机选择的值=0) __$\times$__ P(第i+1次随机选择的值!=0) __$\times...\times$__ P(第k次随机选择的值!=0)

$$\frac{1}{i} \times (1-\frac{1}{i+1}) \times...\times(1-\frac{1}{k}) = \frac{1}{i} \times \frac{i}{i+1} \times...\times \frac{k-1}{k} = \frac{1}{k} $$

- - -

### 例题
---
1.[随机数索引-随机选一个数](https://leetcode.cn/problems/random-pick-index/)
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/reservoirSampling/随机数索引.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="随机数索引"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/reservoirSampling/随机数索引.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">随机数索引</div>
    </a>
</center>

>java

``` java
class Solution {
    int[] num;
    Random random;
    public Solution(int[] nums) {
        num = nums;
        random = new Random();
    }
    
    public int pick(int target) {
        int res = 0;
        for(int i=0, count = 0;i<num.length;i++){
            if(num[i] == target){
                if(random.nextInt(++count) == 0){
                    res = i;
                }
            }
        }
        return res;
    }
}
```

> 时间复杂度:O(n)  
> 空间复杂度为:O(1)  

2. 随机选k个数
>java

``` java
public int[] selectK(int[] nums, int k){
    Random random = new Random();
    int[] res = new int[k];
    int len = nums.length;
    for(int i=0;i<k;i++){
        res[i] = nums[i];
    }
    int count = k;
    int i = k;
    while(i < len){
        int j = random.nextInt(++count);
        if(j<k){
            res[j] = nums[i];
        }
        i++;
    }
    return res;
}
```
> 时间复杂度:O(n)  
> 空间复杂度度:O(1)


### 其他语言
- - -
> C++

``` cpp
//随机选一个数
class Solution {
	vector<int>& nums;

public:
		Solution(vector<int>& nums) : nums(nums) {}

		int pick(int target) {
			int res;
			for (int i = 0, count = 0;i < nums.size(); i++) {
				if (nums[i] == target)
				{
					++count;
					if (rand() % count == 0)
					{
						res = i;
					}
				}
			}
			return res;
		}
};

//随机选K个数
class Solution {
	vector<int> selectK(vector<int>& nums, int k)
	{
		vector<int> res(k);
		for (int i = 0;i < k;i++)
		{
			res[i] = nums[i];
		}
		
		int count = k;
		int i = k;
		while (i < nums.size())
		{
			++count;
			int j = rand() % count;
			if (j < k)
			{
				res[j] = nums[i];
			}
		}
		return res;
	}
};
```
> python3

``` python
//随机选一个数
class Solution:
    def __init__(self, nums:List[int]):
        self.nums = nums

    def pick(self, target:int) -> int:
        ans = cnt = 0
        for i, num in enumerate(self.nums):
            if num == target:
                cnt += 1
                if randrange(cnt) == 0:
                    ans = i
        return ans
        
//随机选K个数        
class Solution:
    
    def selectK(self, nums:List[int], k:int) -> List[int]:
        res = list()
        for i in range(0, k):
            res[i] = nums[i]
        count = k
        i = k
        while i < len(nums):
            ++count
            j = randrange(count)
            if j < k:
                res[j] = nums[i]
        return res
```
> go

``` go
//随机选一个数
type Solution []int

func Constructor1(nums []int) Solution {
	return nums
}

func (nums Solution) Pick(target int) (res int) {
	count:=0
	for i, num := range nums{
		if num == target {
			count++
			if rand.Intn(count) == 0 {
				res = i
			}
		}
	}
	return
}

//随机选K个数
func SelectK(nums []int, k int) (res []int) {
	for i:=0;i<k;i++{
		res[i] = nums[i]
	}
	count:=k
	i:=k
	for i<len(nums) {
		count++
		j := rand.Intn(count)
		if j < k{
			res[j] = nums[i]
		}
	}
	return res
}
```