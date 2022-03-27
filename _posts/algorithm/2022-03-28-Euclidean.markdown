---
layout: default
algorithm: true
modal-id: 10000018
date: 2022-03-28
img: pexels-alexey-demidov-11341064.jpg
alt: Euclidean
project-date: March 2022
client: Sort
category: algorithm
subtitle: Euclidean Algorithm
description: 欧几里得算法
---
### 定义
- - -
&emsp;&emsp;欧几里得算法又称辗转相除法,是指用于计算两个非负整数a、b的最大公约数.  
- - -

### 基本思路
- - -
定理:  
&emsp;&emsp;两个整数的最大公约数等于其中较小的那个数和两数相除余数的最大公约数.最大公约数(Greatest Common Divisor)缩写GCD.  
原理:  
&emsp;&emsp;gcd(a, b) = gcd(b, a mod b)  
条件:  
&emsp;&emsp;a>b 且 a mod b 不为0
- - -

### 示例
- - -
假如需要求1997和615两个正整数的最大公约数,用欧几里得算法进行:  
1. 1997/615 = 3 (余152)
2. 615/152 = 4 (余7)
3. 152/7 = 21 (余5)
4. 7/5 = 1 (余2)
5. 5/2 = 2 (余1)
6. 2/1 = 2 (余0)

至此,最大公约数为1.  
以除数和余数反复做除法运算,当余数为0时,取当前算式除数为最大公约数,所以1997和615的最大公约数为1.
- - -

### 代码
>java

``` java
 gcd(a, b) = gcd(b,a mod b)
```
