## Montgomery Reduction

在模运算计算中，蒙哥马利模乘，通常称为蒙哥马利乘法，是一种进行快速模乘的方法。它是由美国数学家Peter L. Montgomery在1985年提出的。该算法首先把整数表示成特殊的形式，称作Montgomery form。在Montgomery form下进行乘法运算，可以避免传统模乘中的开销最大的除法操作(变为高效的右移位运算)，对于计算机而言可以提升算法速度。

当然，把乘数转换成Montgomery form有相应的开销，因此在计算单次模乘的时候，蒙哥马利乘法要慢于传统方法和后面提到的`Barrett reduction`算法。但是在进行大量模乘计算的时候，比如模幂运算，转成Montgomery form和从Montgomery form转成普通形式两个过程的开销变得微不足道，蒙哥马利乘法要快于其它方法。RSA和DH协议这种需要大量大数模运算的算法使用该方法效果很好。

一个简单的例子：

- $1234\mod 97$一般很难一眼给出结果，传统计算分为两个步骤：
  1. 计算$q = \lfloor1234/97\rfloor$，这一步骤正是我们要解决掉的耗时操作
  2. 计算$r = 1234-q\times 97$
- $1234\mod 100$则可以很容易的给出结果34。因为此时计算除法不再是一个耗时操作，而变成了高效的移位操作

蒙哥马利乘法就是做到对于任意正整数$N$，可以选取$R>N$，且$\gcd(R,N) = 1$，通过蒙哥马利乘法把$\mod N$的运算变成$\mod R$的运算。选取恰当的$R$，模运算可以不做除法(变成做移位运算)，从而提高模运算的效率。对于2进制大脑的计算机而言，恰当的$R$其实就是2的方幂，例如$x\mod 251$$\Longrightarrow$$x\mod 256$，则计算机做模运算只需要简单的取$x$的后8位($256 = 2^8$)即可。

**整数的蒙哥马利形式：**当把$\mod N$转化为$\mod R$时，$a\mod N$首先需要转化为$aR\mod N$，$aR$称为整数$a$的蒙哥马利形式

- 加法：$aR+bR = (a+b)R\pmod{N}$

- 乘法：$aR\times bR = (abR)R\pmod{N}$，不符合蒙哥马利形式！需要除去一个R，可以通过计算其逆元得到(这要求$\gcd(R,N) = 1$)

  结果需要乘上$R$模$N$的逆元$R'$，即$aR\times bR \times R' = (ab)R\pmod N$

当然如果按照这种形式直接计算哪肯定比普通乘法慢很多，举个例子，要计算$7\times 15\mod 17$，选取$R=100$，则我们首先计算$7\times 100\pmod {17} = 3,~~15\times 100\pmod{17} = 4$，然后计算$3\times 4 = 12$，再用扩展欧几里得计算出$100$模17的逆元：$8⋅100 − 47⋅17 = 1$，故$R' = 8$，于是$12\times 8= 96$，再算$96\mod 17 = 11$，还不如直接计算。真正的计算方法用到下面的`REDC`算法，是整个蒙哥马利乘法的核心：

**蒙哥马利约化：**

<img src="https://github.com/cat538/images-auto/raw/main/img/image-20220124194117921.png" alt="image-20220124194117921" style="zoom: 80%;" />

即输入$(T,R,N,N')$，输出$TR^{-1}\mod N$

算法中$T+mN$一定可以被$R$整除，这一点容易验证。且有$m\in [0, R − 1]$，故$T+mN=(RN − 1) + (R − 1)N<2RN$，故$t<2N$，所以最后如果$t\geq N$， 则只需要做一次减法

注意算法中的$\mod R$都是移位运算，其他的都是乘法和加法减法

有了这个算法之后我们重复上面的例子，计算$7\times 15\mod 17$，首先仍然是计算$7\times 100\pmod {17} = 3,~~15\times 100\pmod{17} = 4$，然后计算$3\times 4 = 12$，此时不需要计算$R'$，而是调用`REDC(12,100,17,47)`($N'$用扩展欧几里得算)：

- The first step sets *m* to $12 ⋅ 47 \mod 100 = 64$
- The second step sets *t* to $(12 + 64 ⋅ 17) / 100$，注意这里12+64·17 = 1100，可以被100整除

得到结果为11，小于模数17，因此11就是最终结果

## Barrett reduction

[Barrett reduction - Wikipedia](https://en.wikipedia.org/wiki/Barrett_reduction)

### Single-word Barrett reduction

最原始的`reduce`：

``` go
func reduce(a uint) uint {
    q := a / n  // Division implicitly returns the floor of the result.
    return a - q * n
}
```

