---
layout: default
algorithm: true
modal-id: 10000014
date: 2021-11-01
img: pexels-regina-pivetta-9945338.jpg
alt: 快速幂
project-date: November 2021
client: Power
category: algorithm
subtitle: QuickPower
description: 快速幂
---
### 快速幂定义
- - -
&emsp;&emsp;快速幂就是快速算底数的n次幂.其时间复杂度为O(logN),与O(N)相比效率有了极大的提高.
- - -

### 原理
- - -
&emsp;&emsp;快速幂的核心思想就是每一步都把指数分成两半,而相应的底数做平方运算.
这样不仅能把非常大的指数给不断变小,所需要执行的循环次数也变小,而最后表示的结果却一直不会变.
&emsp;&emsp;举个简单的例子,比如我们要求3的10次方
1. 若我们使用通常的方法则是每次乘3直到乘了9次,即3 * 3 * 3 * 3 * 3 * 3 * 3 * 3 * 3 * 3
2. 若我们先算3的五次方,然后在算它的平方,只需要乘五次即可,即(3 * 3 * 3 * 3 * 3)^2
3. 若我们先算3 * 3,则3的五次方为9 * 9 *3,再算它的平方,共进行了4次乘法
模仿这样的过程,我们得到一个在O(logN)时间内计算出幂的算法,也就是快速幂.
- - -

### 递归快速幂
- - -
以上的递归方程如下:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/递归方程.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="递归方程"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/递归方程.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">递归方程</div>
    </a>
</center>
&emsp;&emsp;计算a的n次方,有三种情况:
- 如果n为偶数(不为0),那么就先计算a的n/2次方,然后平方
- 如果n是奇数,那么就先计算a的n-1次方
- 递归出口就是a的0次方,为1

&emsp;&emsp;递归快速幂的思路比较清晰:
``` java
//递归快速幂
int qpow(int a, int n){
    if(n==0){
        return 1;
    }else if(n%2==1){
        return qpow(a, n-1)*a;
    }else{
        int temp = qpow(a, n/2);
        return temp*temp;
    }
}
```
&emsp;&emsp;注意这个temp变量是必须的,如果不发a^(n/2)记录下来,直接写成qpow(a, n/2)*qpow(a, n/2),那会计算两次 a^(n/2),整个算法就退化为O(n).  
&emsp;&emsp;在实际问题中,题目常常会要求一个大素数取模,这是因为计算结果可能会非常巨大,但是在这考察高精度又没有必要.
这时我们的快速幂也应当进行取模,此时应当注意,原则是步步取模,如果MOD较大,还应当注意是否会超过int的大小,使用long.
``` java
//递归快速莫 (对大素数取模)
int MOD 1000000007
long qpow(long a, long n){
    if(n==0){
        return 1;
    }else if(n%2==1){
        return qpow(a, n-1)*a % MOD;
    }else{
        long temp = qpow(a, n/2) % MOD;
        return temp*temp%MOD;
    }
}
```
&emsp;&emsp;递归虽然简洁,但会产生额外的空间开销.可以把递归改写为循环,来避免对栈空间的大量占用,也就是非递归快速幂.

- - -

### 非递归快速幂
- - -
&emsp;&emsp;换一个角度来引入非递归的快速幂,例如7的10次方,把10写成二进制的形式(1010).  
&emsp;&emsp;要计算7^(1010),我们可以把它们拆分为 __7^(1000) * 7^(10)__ .实际上,对于任意的整数,我们都可以把它拆成若干个7^(100..)的形式相乘.
而这些7^(100..)恰好就是 __7^1、7^2、7^4...__ ,我们只需不断把底数平方即可算出它们.
``` java
//非递归快速幂
int qpow(int a, int n){
    int ans=1;
    while(n){
        if(n&1){    //如果n的当前末尾为1
            ans*=a; //ans乘上当前的a
        }
        a*=a;       //a自乘
        n>>=1;      //n往右移一位
    }
}
``` 
&emsp;&emsp;运算过程如下:  
&emsp;&emsp;最初ans为1,然后我们一位一位算:  
&emsp;&emsp;1010的最后一位是0,所以a^1这一位不要.然后1010变为101,a变为a^2  
&emsp;&emsp;101的最后一位是0,所以a^2这一位是需要的,乘入ans.101变为10,a再自乘变为a^4  
&emsp;&emsp;10的最后一位是0,跳过,右移10变为1,a再自乘变为a^8  
&emsp;&emsp;最后1的最后一位是1,ans再乘上a^8.循环结束,返回结果  
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/非递归快速幂.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="非递归快速幂"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/非递归快速幂.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">非递归快速幂</div>
    </a>
</center>
&emsp;&emsp;这里的位运算符 >> 是右移,表示把二进制数往右移动一位,相当于/2;&是按位与,&1可以理解为取出二进制数的最后一位,相当于%2==1.
- - -

### 矩阵快速幂
- - -
&emsp;&emsp;矩阵快速幂用于求解一般性问题:给定大小为n*n的矩阵M,求答案矩阵M^k,并对答案矩阵中的每位元素对P取模.
对于类似斐波那契此类的数列递推问题,我们可以使用矩阵快速幂来进行加速.使用矩阵快速幂,我们只需要O(logn)的复杂度.

- 泰波那契序列
泰波那契序列 Tn 定义如下:  
T0 = 0, T1 = 1, T2 = 1, 且在 n >= 0 的条件下 Tn+3 = Tn + Tn+1 + Tn+2  
给你整数n,请返回第n个泰波那契数Tn的值.  

根据题目的递推关系(i>=3):
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/递推关系.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="递推关系"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/递推关系.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">递推关系</div>
    </a>
</center>
我们发现求解 f(i) 其依赖的是 f(i-1)、f(i-2)和f(i-3).
我们可以将其存成一个列向量:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/列向量.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="列向量"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/列向量.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">列向量</div>
    </a>
</center>
利用题目给定的依赖关系,对目标矩阵元素进行展开:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/矩阵展开.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="矩阵展开"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/矩阵展开.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">矩阵展开</div>
    </a>
</center>
那么根据矩阵乘法,即有:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/公式.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="公式"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/公式.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">公式</div>
    </a>
</center>
我们令:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/因子.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="因子"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/因子.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">因子</div>
    </a>
</center>
然后发现利用Mat也能实现数列递推:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/数列递推.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="数列递推"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/数列递推.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">数列递推</div>
    </a>
</center>
再根据矩阵运算的结合律,最终有:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/最终公式.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="最终公式"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/最终公式.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">最终公式</div>
    </a>
</center>
从而将问题转化为求解Mat^(n-2),这时候可以套用矩阵快速幂解决.  
``` java
class Solution {
    int N = 3;
    int[][] mul(int[][] a, int[][] b) {
        int[][] c = new int[N][N];
        for (int i = 0; i < N; i++) {
            for (int j = 0; j < N; j++) {
                c[i][j] = a[i][0] * b[0][j] + a[i][1] * b[1][j] + a[i][2] * b[2][j];
            }
        }
        return c;
    }
    public int tribonacci(int n) {
        if (n == 0) return 0;
        if (n == 1 || n == 2) return 1;
        int[][] ans = new int[][]{
            {1,0,0},
            {0,1,0},
            {0,0,1}
        };
        int[][] mat = new int[][]{
            {1,1,1},
            {1,0,0},
            {0,1,0}
        };
        int k = n - 2;
        while (k != 0) {
            if ((k & 1) != 0) ans = mul(ans, mat);
            mat = mul(mat, mat);
            k >>= 1;
        }
        return ans[0][0] + ans[0][1];
    }
}
```
>时间复杂度:O(logN)  
>空间复杂度:O(1)

- 斐波那契数列  
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/斐波那契数列.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="斐波那契数列"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/斐波那契数列.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">斐波那契数列</div>
    </a>
</center>
推算方法如下:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/斐波那契数列推算.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="斐波那契数列推算"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/algorithm/fastPower/斐波那契数列推算.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">斐波那契数列推算</div>
    </a>
</center>
这样,我们把原来较为复杂的问题转化成了求某个矩阵的幂的问题,可以应用快速幂求解.
``` java
#include <cstdio>
#define MOD 1000000007
typedef long long ll;

struct matrix
{
    ll a1, a2, b1, b2;
    matrix(ll a1, ll a2, ll b1, ll b2) : a1(a1), a2(a2), b1(b1), b2(b2) {}
    matrix operator*(const matrix &y)
    {
        matrix ans((a1 * y.a1 + a2 * y.b1) % MOD,
                   (a1 * y.a2 + a2 * y.b2) % MOD,
                   (b1 * y.a1 + b2 * y.b1) % MOD,
                   (b1 * y.a2 + b2 * y.b2) % MOD);
        return ans;
    }
};

matrix qpow(matrix a, ll n)
{
    matrix ans(1, 0, 0, 1); //单位矩阵
    while (n)
    {
        if (n & 1)
            ans = ans * a;
        a = a * a;
        n >>= 1;
    }
    return ans;
}

int main()
{
    ll x;
    matrix M(0, 1, 1, 1);
    scanf("%lld", &x);
    matrix ans = qpow(M, x - 1);
    printf("%lld\n", (ans.a1 + ans.a2) % MOD);
    return 0;
}
```
- - -
