---
title: RSA解析
date: 2021-05-01 15:33:03
categories: '密码学'
description: "关于RSA算法的相关证明、应用场景，及公钥题目中一些常见套路..."
cover: "https://github.com/cat538/images-auto/raw/main/img/cipher-16421552297502.png"
mathjax: true
---

## RSA加密

1. 随机选择大素数$p、q$

2. 计算$n = p\cdot q\quad 、\quad \varphi(n) = (p-1)(q-1)$  其中$\varphi()$为欧拉函数，$\varphi(n) = \varphi(p)\varphi(q)$

3. 选择$e:gcd(e,(\varphi(n))) = 1$

4. 确定$d:ed\equiv 1 \pmod {\varphi(n)}$  其中 $\varphi(n) = (p-1)(q-1)$

5. 加密：$c = m^e\pmod{n}$

   解密：$m = c^d\pmod{n}$



- 其中$(n,e)$作为公钥公开，$(n,d)$作为私钥保密

- 敌手如果想要直接解密，则需要知道d，计算d则需要计算$\varphi(n)$，而由$n\rightarrow\varphi(n)$目前没有直接解法**【1】**，只能分解$n$求出$pq$，因此RSA的安全性依赖于因子分解的困难性



**【1】**：$\varphi(n) = (p-1)(q-1) = pq-(p+q)+1\Longrightarrow p+q = n-\varphi(n)+1$，如果知道$\varphi(n)$，则可以很轻易地分解n，换句话说，计算$\varphi(n)$并不比分解$n$容易



### RSA解密过程的正确性证明

$c = m^e\pmod{n}\quad m = c^d\pmod{n}\quad\Longrightarrow m^{ed}\equiv m\pmod{n}$

**证明：**因为$ed\equiv 1 \pmod {\varphi(n)}\Longrightarrow 即证m^{k\varphi(n)+1}\equiv m\pmod{n}$

这里不能直接写欧拉定理，因为欧拉定理$m^{\varphi(n)}\equiv1\pmod{n}$要求$m\in n^*$

分情况：

- $(m,n) = 1\Longrightarrow$ 直接欧拉​定理

- $(m,n)\neq1\Longrightarrow 至少有 p|m ~或~ q|m$

  以$p|m$为例：$m\equiv0\pmod{p}\Longrightarrow m^{k\varphi(n)+1}\equiv m\pmod{p}$

  无论$q|m~or~q\nmid m $，均有$m^{k\varphi(n)+1}\equiv m\pmod{p}$前者用欧拉，后者同上一行

  而$pq$互素，因此有$m^{k\varphi(n)+1}\equiv1\pmod{n}$



### RSA中几个时间复杂度

模$n$运算，其2进制长度为$k$：

| 运算          | 时间复杂度             | 算法       |
| ------------- | ---------------------- | ---------- |
| 加法          | $O(k)$                 |            |
| 减法          | $O(k)$                 |            |
| 乘法          | $O(k^2)$               |            |
| 求逆          | $O(k^3)$               |            |
| 模幂(指数为c) | $O(\log c \times k^2)$ | 平方乘算法 |

#### 解密算法的加速：

因为解密方知道p的分解m、n所以可以把$m = {c}^d\pmod n$换成两个方程：
$$
m_1 = c_1^{d_1}\pmod p\\
m_2 = c_2^{d_2}\pmod q\\
其中c_1 = c \pmod p,c_2 = c\pmod q\\
d_1 = d\pmod {p-1},d_2 = d\pmod {q-1}
$$
然后用CRT逆回去即可，能够提高4-8倍



### RSA安全性分析

#### 假设存在计算d的算法$\rightarrow$构造分解n的算法

**Las Vegas算法**

- 基本思想：利用1模n的非平凡平方根

  $x^2 \equiv1\pmod{n}$即$n|(x-1)(x+1)\Longrightarrow pq|(x-1)(x+1)$且有$n\nmid x-1 \quad,\quad n\nmid x+1$	

  > 因为如果$n\mid x-1\rightarrow x\equiv1\pmod{n}$，x是平凡平方根。$n\mid x+1$同理

  若要满足上述条件，则只有情况

  1. $p|x-1,q|x+1$
  2. $p|x+1,q|x-1$

  即有$\gcd(x+1,n) = p(或q)或者\gcd(x-1,n) = q(或p)$

  就成功分解了n，下面考虑如何求解非平凡平方根

- 求解非平凡平方根（需要$e，d$）：

  因为$ed\equiv1\pmod{\varphi(n)}$，所以$ed-1 = k\varphi(n)$

  表示成$ed-1 = 2^s\cdot r$，随机选择$w\in\{1,2,3\cdots n\}$如果$\gcd(w,n) \neq 1$，则分解成功···（概率很低）

  否则的话，$w\in n^*$，计算$w^{2^ir}$，如果$w^{2^ir} = 1\pmod{n}$，且$w^{2^{i-1}r}\neq{-1}\pmod{n}$，则 $w^{2^{i-1}r}$即为一个非平凡解

  其中$i\in\{0,1,2\cdots s\}$，如果遍历0-s均未找到，则另选一个w计算，这个过程成功的概率大于$\frac{1}{2}$

上述攻击告诉我们：**一旦解密指数泄露，必须要更换模n，不能仅更换d，否则敌手可以分解n**



#### 同模攻击

攻击场景：一组用户（A、B）共享模n，使用不同的加密指数e和解密指数d，C向A、B发送消息，敌手可以不知道解密指数获取明文

设$m$是消息，共同的模数是$n$，两个密文分别为

- $C_1 = m^{e_1}\pmod{n}$
- $C_2 = m^{e_2}\pmod{n}$

如果$e_1,e_2$互素，可以根据扩展欧几里得计算出$r,s满足re_1+se_2 = 1$，则$C_1^rC_2^s = m^{e_1r}m^{e_2s} = m$

即敌手不用获得解密指数，也可以恢复明文

上述攻击告诉我们：**不能在一组用户之间共享模n**



#### 选择密文攻击

利用了RSA加密同态的性质，即$Enc(m_1)Enc(m_2) = Enc(m_1m_2)$

设公钥为$(n,e)$对于敌手想要破译的消息$C^*$，其计算$C' =C^*\cdot x^e\pmod{n}$

询问获得$C'$对应的明文$m'$，可以看出来，$m' = m^*\cdot x\pmod{n}$

$m'/x$即为$m^*$



#### 克服同态攻击的RSA OAEP模式



#### 其它问题

- 安全隐患：不同的模共享素因子
- 参数设置：p、q安全素数，不能太接近（太接近可以直接开方穷搜，有太小的因子又会容易被分解），一般相差几位
- 加密解密指数不能设置太小



## 素性检测与因子分解

### 素性检测

#### Solovay-Strassen算法

- 错误概率至多为$1/2$

#### Miller-Rabin算法

- 错误概率至多为$1/4$
- 通过的素数可能是强伪素数



**因子分解算法时间复杂度：**

| 算法               | 时间复杂度                                             | 备注                                          |
| ------------------ | ------------------------------------------------------ | --------------------------------------------- |
| Pollard p-1算法    | $O(B~\log B(\log n)^2+(\log n)^3)$ ( B次模幂 + 1次GCD) | 告诉我们要选安全素数，p-1应该有足够大的素因子 |
| Pollard $\rho$算法 | $1.17\sqrt{p}$                                         |                                               |
| Dixon随机平方算法  | $O(e^{1+o(1)\sqrt{\ln n \ln \ln n}})$                  |                                               |



### Pollard p-1算法

假定$p$是$n$的一个素因子，且$p-1$的每一个素数幂次$q$有$q\leqslant B$，则有$(p-1)|B!$

令$a\equiv2^{B!}\pmod{n}$，因$(p-1)|B!$，故有$a\equiv 2^{p-1}\pmod{p}\Longrightarrow a\equiv1\pmod{p}$

故有$p|a-1$，求$\gcd(a-1,n)$即可得到素因子$p$

<img src="https://github.com/cat538/images-auto/raw/main/img/image-20210506172158768.png" alt="image-20210506172158768" style="zoom:67%;" />

#### Pollard $\rho$算法

寻找两个元素$x_1,x_2$满足$x_1\neq x_2\pmod{n},x_1=x_2\pmod{p}\Longrightarrow p|x_1-x_2$

求$\gcd(x_1-x_2,p)$即可分解

### Dixon随机平方算法

$x\neq \pm y\pmod{n}$

$x^2\equiv y^2\pmod{n}$

$x^2-y^2\equiv0\pmod{n}\Longrightarrow n|(x+y)(x-y)$

假设 $n$分解成$pq$，则因为$n\nmid x\pm y$，故只有$p|(x+y),q|(x-y)$或$p|(x-y),q|(x+y)$两种情况

求$\gcd(x+y,n)$即可得到一个因子



## EIGamal加密

在循环群$Z_p^*$上，循环群的阶为$p-1$

加密：

- 选择随机元素k
- 计算$c_1 = g^k\pmod p,c_2 = y^k\cdot m\pmod p$
- 输出$(c_1,c_2)$

$y^k$被视为一个均匀随机元素，使用了均匀随机元素隐藏明文m

解密：

- 因为$y^k = (g^x)^k = (c_1)^x$
- 所以$m = c_2/(c_1)^x \pmod p$

### 离散对数问题

对于$a^\alpha = \beta\pmod n$

已知$\alpha,\beta$求解$a$

**求解离散对数问题方法：**

| 方法           | 时间复杂度                                            | 备注                                         |
| -------------- | ----------------------------------------------------- | -------------------------------------------- |
| Shanks（BSGS） | $O(\sqrt{n})$                                         | 告诉我们循环群的阶应该足够大保证不容易被分解 |
| Pohlig-Hellman |                                                       | 告诉我们群的阶应该有足够大的素因子           |
| Pollard-$\rho$ | $O(\sqrt{n})$                                         | 告诉我们循环群的阶应该足够大保证不容易被分解 |
| 指数计算算法   | $O(e^{(1/2+o(1))\sqrt{\ln{p\ln \ln p}}})$（在线阶段） | 只针对$z_p^*$上的离散对数问题                |

