---
layout: default
algorithm: true
modal-id: 10000021
date: 2023-11-18
img: pexels-muhammet-cengiov-19017576.jpg
alt: BitOperation
project-date: October 2023
client: Sort
category: algorithm
subtitle: BitOperation
description: 位运算
---

#### 常见运算符
- 按位与 &  
&nbsp;&nbsp;两个操作数相应的二进制位都为1，结果为1，反之为0。
- 按位或 |  
&nbsp;&nbsp;两个操作数相应的二进制位只要有一个为1，结果为1，反之为0。
- 按位异或 ^  
&nbsp;&nbsp;两个操作数相应的二进制位值只要相同，运算结果为0，反之为1。
- 按位取反 ~  
&nbsp;&nbsp;单目运算符，只有一个操作数，是将操作数相应的二进制位数取反。
- 左移 <<  
&nbsp;&nbsp;将一个数的各二进制位左移若干位，移动的位数由右操作数指定。  
- 右移 >>  
&nbsp;&nbsp;与左移相反
- 二进制数最低位1和后面所有0组成的 $$ 2^k $$ lowbit  
&nbsp;&nbsp;将x的二进制所有位全部取反，再加1，就可以得到 -x 的二进制编码。例如6的二进制编码是 110 ，全部取反后得到 001，加1得到010。  
&nbsp;&nbsp;设原先x的二进制编码是 (...)10...00，全部取反后得到[...]01...11，加1后得到[...]10...00，也就是-x的二进制编码。这里x表示二进制中第一个1是x最低位的1。  
&nbsp;&nbsp;(...)和[...]中省略号的每一位分别相反，所以 x&-x = (...)10...00 & [...]10...00 = 10...00，得到的结果就是 lowbit。



