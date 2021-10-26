---
layout: default
algorithm: true
modal-id: 10000013
date: 2021-10-22
img: pexels-vladislav-filippov-9001724.jpg
alt: 摩尔投票
project-date: October 2021
client: Tree
category: algorithm
subtitle: MajorityVote
description: 摩尔投票
---
### 摩尔投票
- - -
[摩尔投票论文地址](https://www.cs.ou.edu/~rlpage/dmtools/mjrty.pdf)
1. 抛出问题  
给定一个数组求出该数组中出现次数大于数组一半长度的数,要求时间复杂度为O(n),空间复杂度为O(1)
2. 解决方案  
- 若没有空间复杂度的要求,我们直接使用Map一次遍历记录没有数出现的次数,最后找到出现次数大于一半的即可
- 但是加上空间复杂度的要求上面的方法就行不通了,由此引申出摩尔投票
3. 定义  
摩尔投票的本质就是冲突抵消.因为要求的数在数组中出现超过数组一半长度,那么这个数肯定就只能有一个.
每次从数组中取出两个不相同的数进行抵消,最后剩下的一个数字或者几个数字可能就是我们要求的数.
这里为什么要说可能呢,因为有可能会出现以下情况:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/majorityVote/摩尔投票示意图.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="摩尔投票示意图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/majorityVote/摩尔投票示意图.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">摩尔投票示意图</div>
    </a>
</center>
&emsp;&emsp;正常情况如上图左侧,抵消之后的剩余的数就是要求的数;若是非异常情况,如上图右侧,抵消之后的数并不是要求的数.  
&emsp;&emsp;所以最后剩余的数不一定是我们要求的结果,还需要我们再进行一次遍历统计该数的出现次数是否满足要求.
- - -

### 力扣原题
- - -
[面试题 17.10. 主要元素](https://leetcode-cn.com/problems/find-majority-element-lcci/)  
&emsp;&emsp;首先我们先来看下简单题:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/majorityVote/主要元素.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="主要元素"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/majorityVote/主要元素.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">主要元素</div>
    </a>
</center>
根据摩尔投票的原理:
1. 定义候选值 candidate 和 出现次数 count
2. 遍历数组nums
3. 如果count为0 将num赋值给candidate
4. 如果当前num 等于 候选值 candidate 则将count++ 否则 count--
5. 最后剩下的candidate可能是结果
6. 还需要遍历nums数组求出candidate的出现次数是否符合要求

``` java
public static int majorityElement1(int[] nums){
    int candidate = -1;
    int count=0;
    for (int num:nums){
        if (count==0){
            candidate = num;
        }
        if (candidate == num){
            count++;
        }else {
            count--;
        }
    }

    count =0;
    for (int num:nums){
        if (num == candidate){
            count++;
        }
    }
    return count*2 >= nums.length? candidate:-1;
}
```
>时间复杂度:O(n)  
>空间复杂度:O(1)

[229. 求众数 II](https://leetcode-cn.com/problems/majority-element-ii/)  
&emsp;&emsp;下面加深点难度:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/majorityVote/求众数II.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="求众数II"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/majorityVote/求众数II.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">求众数II</div>
    </a>
</center>
&emsp;&emsp;因为题目要求出超过 ⌊n/3⌋ 次的元素,其实这里有个隐藏条件,即这样的数最多只有两个,因为如果有三个,每个都超过⌊n/3⌋,那么最终的次数肯定超过数组大小.  
&emsp;&emsp;本题我们同样应该使用摩尔投票,只不过稍微改造一下,使用三个数进行抵消,最后剩下来的两个数继续遍历统计个数是否大于⌊n/3⌋.
1. 定义element1、vote1、element2、vote2分别表示第一个元素的值及出现次数和第二个元素的值和出现次数
2. 多种情况讨论
- 若已有element1的投票即vote1大于0,且当前元素等于element1的话,则vote1++
- 若已有element2的投票即vote2大于0,且当前元素等于element2的话,则vote2++
- 若没有element1的投票即vote1等于0的话,将num赋值给element1且vote1++
- 若没有element2的投票即vote2等于0的话,将num赋值给element2且vote2++
- 最后若都不满足即三个数不相同,直接抵消,即vote1--,vote2--
3. 经过步骤2之后已经确定可能的结果element1、element2需要满足vote1和vote2大于0,再次遍历数组,统计两个元素的出现次数count1、count2
4. 最后判断count1、count2是否满足大于n/3,满足则加入结果列表返回

``` java
public  List<Integer> majorityElement(int[] nums) {
    int num1 = 0;
    int num2 = 0;
    int vote1=0;
    int vote2=0;
    for (int num:nums){
        if (vote1>0 && num==num1){
            vote1++;
        }else if (vote2>0 && num==num2){
            vote2++;
        }else if (vote1 == 0){
            num1 = num;
            vote1++;
        }else if (vote2 == 0){
            num2=num;
            vote2++;
        }else {
            vote1--;
            vote2--;
        }
    }

    int count1=0;
    int count2=0;
    for (int num:nums){
        if (vote1>0 && num1==num){
            count1++;
        }
        if (vote2>0 && num2==num){
            count2++;
        }
    }

    List<Integer> res = new ArrayList<>();
    if (vote1>0 && count1>nums.length/3){
        res.add(num1);
    }
    if (vote2>0 && count2>nums.length/3){
        res.add(num2);
    }
    return res;
}
```
>时间复杂度:O(n)  
>空间复杂度:O(1)

- - -
