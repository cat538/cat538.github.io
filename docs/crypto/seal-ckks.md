---
title: SEAL-CKKS
cover: 'https://github.com/cat538/images-auto/raw/main/img/fire.png'
date: 2022-03-16 09:50:23
tags:
categories:
mathjax: true
---

这篇文章是阅读SEAL库的记录，首先通过一个例子介绍SEAL库CKKS方案的调用，然后通过源码分析SEAL库中CKKS方案的实现。本篇内容基于`SEAL 4.0`

## example

首先通过一个例子感受一下SEAL库中的CKKS，下面的代码展示了两个浮点向量做点积运算如何使用CKKS方案。

```C++
#include "seal/seal.h"
using namespace std;
using namespace seal;

/* parms setting */
EncryptionParameters parms(scheme_type::ckks);
uint32_t poly_modulus_degree = 8192;
parms.set_poly_modulus_degree(poly_modulus_degree);
parms.set_coeff_modulus(CoeffModulus::Create(poly_modulus_degree, { 25, 21, 21, 21, 21, 21, 25 }));
double scale = pow(2.0, 21);
auto context = SEALContext(parms);
KeyGenerator keygen(context);
PublicKey public_key;
SecretKey secret_key;
RelinKeys relin_keys;
keygen.create_public_key(public_key);
secret_key = keygen.secret_key();
keygen.create_relin_keys(relin_keys);

Encryptor encryptor(context, public_key);
Evaluator evaluator(context);
Decryptor decryptor(context, secret_key);

CKKSEncoder encoder(context);
uint32_t slot_count = encoder.slot_count();
cout << "Number of slots: " << slot_count << endl;


```

## EncryptionParameters

上述示例代码的前几行：

```C++
/* parms setting */
EncryptionParameters parms(scheme_type::ckks);
uint32_t poly_modulus_degree = 8192;
parms.set_poly_modulus_degree(poly_modulus_degree);
parms.set_coeff_modulus(CoeffModulus::Create(poly_modulus_degree, { 25, 21, 21, 21, 21, 21, 25 }));
double scale = pow(2.0, 21);
auto context = SEALContext(parms);
```

从名字能推测出，这几行代码是在设置加密相关参数，涉及到的第一个结构就是：`EncryptionParameters`，接下来追踪一下这个结构，在文件`src/encryptionparams.h`中：

```C++
class EncryptionParameters {
    friend struct std::hash<EncryptionParameters>;
private:
    // 指向内存池的指针
	MemoryPoolHandle pool_ = MemoryManager::GetPool();
    // 同态方案类型,截至到SEAL4.0,有BFV,CKKS,BGV三种方案
    scheme_type scheme_;
    // 多项式degree
    std::size_t poly_modulus_degree_ = 0;
    // 多项式模数,由若干素数组成
    std::vector<Modulus> coeff_modulus_{};
    // 随机数生成器
    std::shared_ptr<UniformRandomGeneratorFactory> random_generator_{ nullptr };
    // 明文模数,在CKKS方案中没用
    Modulus plain_modulus_{};
    // 一个标签
    parms_id_type parms_id_ = parms_id_zero;
};
```

去除掉成员方法，只看数据类型，类型`EncryptionParameters`如上所示：

其中对于CKKS方案而言，需要用户传入参数设置的是多项式idegree和多项式模数。其中比较有意思的是`parms_id_type`这一类型，经过查看可知这个类型是`std::array<uint64_t,4>`，根据官方文档解释，

> EncryptionParameters类始终维护当前所设置的加密参数的256位hash值，称为parms_id。这个散列作为加密参数的唯一标识符，并被为这些加密参数创建的所有其他对象使用。parms_id不能由用户直接修改，而是在内部用于预计算以及输入有效性检查。在mod_switch中，用户可以使用parms_id来跟踪加密参数链。parms_id不暴露在EncryptionParameters的公共API中，但一旦SEALContext被创建(说明通过了参数合法性校验)，就可以通过SEALContext::ContextData类访问。

在该文件中的`void EncryptionParameters::compute_parms_id()`定义了这个散列值的计算方法，具体就是把这个类中的加密参数放到buffer里，调用一次SEAL中`HashFunction`模块中的`hash`方法(在文件`src/util/hash.h`中实现)，在SEAL4.0中为`blake2b`算法。

为了进一步了解`parms_id_type`，需要涉及到另外一个存储加密参数的类型`SEALContext`，在上面的例子中，该类型实例通过：

```C++
auto context = SEALContext(parms);
```

被创建。接下来在文件`src/context.h`中看一下`SEALContext`类型。根据官方文档所述，`SEALContext`类型存储了一系列预计算的结果





## poly运算

`SEAL`库与`INTEL_HEXL(Intel Homomorphic Encryption Acceleration Library)`深度结合，后者为SEAL中的整数运算提供了加速，在`SEAL`源码中有大量`SEAL_USE_INTEL_HEXL`的宏，暂时先不考虑这一加速措施。

多项式运算在源码`src/utils/polyarithsmallmod.h`中声明。

以多项式加法为例，看一下`add_poly_coeffmod`，即多项式加法是怎么实现的，因为这是第一个例子，牵涉很多新概念，都要一一追溯，所以这个例子要长一些，后面就没有这么长了。

在排除掉`SEAL_USE_INTEL_HEXL`和用来debug的`SEAL_DEBUG`宏后，`add_poly_coeffmod`代码如下所示：

```c++
void add_poly_coeffmod(
    ConstCoeffIter operand1, ConstCoeffIter operand2, std::size_t coeff_count, const Modulus &modulus,
    CoeffIter result) {
    const uint64_t modulus_value = modulus.value();
    SEAL_ITERATE(iter(operand1, operand2, result), coeff_count, [&](auto I) {
        std::uint64_t sum = get<0>(I) + get<1>(I);
        get<2>(I) = SEAL_COND_SELECT(sum >= modulus_value, sum - modulus_value, sum);
    });
}
```

- 参数中两个操作数`operand`类型`ConstCoeffIter`是`PtrIter<const std::uint64_t *>`的别名(其中`PtrIter`只是一个对裸指针的包装，方便与`SEAL`中定义的操作函数适配)，说白了头两个参数就是两个`uint64_t`类型的数组，存的是两个多项式的系数

- 参数`coeff_count`指明多项式长度

- 参数`modulus`类型是`const Modulus &`。追溯一下类型`Modulus`，声明在`src/modulus.h`中，存储一个至多**61比特**的unsigned整型，在CKKS方案中用于表示多项式系数的模数(具体来说是每一个single RNS polynomial component的模数，这个在下文提到)。

  这个class功能除了一些对接文件的读(`load`)写(`save`)操作以及封装了一些如`is_prime`等方便使用的方法之外，主要是用于执行取模(reduction)运算，通过成员方法`std::uint64_t reduce(std::uint64_t value) const;`执行，使用的是**Barrett reduction**算法，这个reduction算法需要保存一些预计算的量，关于这个算法本身，见[Barrett reduction](# Barrett reduction)

  ```c++
  class Modulus{
  private:
      std::uint64_t value_ = 0;
      // The first two components of the Barrett ratio are the floor of 2^128/value,
  	// and the third component is the remainder.
      std::array<std::uint64_t, 3> const_ratio_{ { 0, 0, 0 } };
      // the size (in 64-bit words) of the value of the current Modulus
      std::size_t uint64_count_ = 0;
      // the significant bit count of the value of the current Modulus
      int bit_count_ = 0;
      bool is_prime_ = false;
  };
  ```

  所有的数据成员如上所示，`Modulus`方法`reduce`是直接调用`src/utils/uintarithsmallmod.h/barrett_reduce_64()`

- 参数`result`存放结果多项式系数

接下来看函数体：

- 其中`SEAL_ITERATE`可以看成是`std::for_each_n`("C++14 does not have std::for_each_n so we use a custom implementation")
- 其中`SEAL_COND_SELECT`就是一个三元表达式，因为整个SEAL中模数的选择在63比特及以下，因此模加运算不会有溢出的风险，可以直接相加然后做一次条件判断，看是否要减去模数即可

不过到此为止并没有结束，SEAL的poly运算中的多项式加法`add_poly_coeffmod`除了上述定义之外还有另外2个重载：

```C++
void add_poly_coeffmod(ConstRNSIter operand1, ConstRNSIter operand2, std::size_t coeff_modulus_size, ConstModulusIter modulus,RNSIter result);
void add_poly_coeffmod(ConstPolyIter operand1, ConstPolyIter operand2, std::size_t size, ConstModulusIter modulus, PolyIter result);
```

什么是`RNSIter`和`PolyIter`？这要说到SEAL的CKKS方案属于Rns CKKS，是CKKS实现方式一种。泛泛地说，就是CKKS中有一个模数链地概念，多项式系数模数由若干个素数相乘得到，比方说4个40比特的素数(下面以此为例)，那么这么一个160比特的数，在运算的时候如何运算呢？是使用类似RSA那样的大数运算吗？并不是，Rns CKKS方案给出的解决方式是基于中国剩余定理(CRT)，把一个模数为比方说160比特的多项式利用CRT拆成4个模数在40比特的多项式(RNS polynomial component)，在计算多项式加法和乘法的时候，只需要各个signle RNS polynomial component 计算加法和乘法，最后需要结果的时候用CRT逆回去就可以了。

下面这张图说明了他们的关系，一个RNSIter其实就可以看作一个2级指针，dereference得到的是一个多项式系数数组(u64*)

<img src="https://github.com/cat538/images-auto/raw/main/img/image-20220324174818364.png" style="zoom:67%;" />

到此为止我们说完了多项式环上的加法运算，同样的，SEAL在`src/utils/polyarithsmallmod.h`中还实现了:

- `modulo_poly_coeffs`，对多项式系数调用`barrett_reduce_64()`
- `negate_poly_coeffmod`，即把多项式系数求`negative`
- `sub_poly_coeffmod`，减法
- `add_poly_scalar_coeffmod`、`sub_poly_scalar_coeffmod`为每个系数加上/减去一个scalar
- `multiply_poly_scalar_coeffmod`，即多项式乘一个标量，这里底层用的是`barrett_reduce_128()`优化reduction
- `dyadic_product_coeffmod`，==不知道啥意思==
- `poly_infty_norm_coeffmod`，把系数转到`[-modulus,modulus)`之间，并返回绝对值最大值
- `negacyclic_shift_poly_coeffmod`、`negacyclic_multiply_poly_mono_coeffmod`，==不知道啥意思==

以上这些基本运算都与加法类似，有3种重载:Coeff,RNS,Poly(有的有更多类型重载)



## 涉及的算法

### Barrett reduction

$$
c = a\mod n
$$

naive的算法需要使用除法，P.D. Barrett在1986年提出该算法，使取模算法中用乘法代替除法

#### General idea

令$s = \frac{1}{n}$，则有：$a\mod n = a-\lfloor as\rfloor n$

比方说a等于3.5个n，那么a模n相当于a减去3n

然而，除法操作在计算机中相比乘法昂贵很多，Barrett对上述经典的取模操作做了改进：用$m/2^k$去近似$1/n$，因为除以2的幂次在计算机中是位移操作，比传统的除法cheap

#### procedure

首先我们假设给定$2^k$，要计算一个合适的$m$去近似$1/n$，则有$\frac{m}{2^k} = \frac{1}{n}\Longleftrightarrow m=\frac{2^k}{n}$，广泛的做法是取$m=\lfloor 2^k/n\rfloor$，因为四舍五入可能会有underflow的风险

这样一来，reduction算法如下所示：

```go
func reduce(a uint) uint {
    q := (a * m) >> k // ">> k" denotes bitshift by k.
    return a - q * n
}
```

但是由于$m=\lfloor 2^k/n\rfloor$，因此实际上$\frac{m}{2^k}\leq\frac{1}{n}$，因此算法输出实际上并不能保证在$[0,n)$，而是只能保证在$[0,2n)$，这一点可以通过数学证明，这里略去。因此算法要做以下修正：

```go
func reduce(a uint) uint {
    q := (a * m) >> k
    a -= q * n
    if n <= a {
        a -= n
    }
  return a
}
```

下面给出算法描述：

> Precomputation:
>
> 1. Assume the modulus $n\in N$ is such that $n≥3$ and $n$ is not a power of $2$.(modulo-power-of-2 is trivial.)
> 2. Choose $k\in N$ such that $2k>n$. (The smallest choice is $k=⌈log_2 n⌉$)
> 3. Calculate $r = \lfloor \frac{2^{2k}}{n}\rfloor$(This is the precomputed factor.)
>
> Reduction:
>
> 1. We are given  $x\in N$, such that $0≤x<n^2$, as the number that needs to be reduced modulo $n$.
> 2. Calculate $t = x-\lfloor \frac{xr}{2^{2k}}\rfloor n$
> 3. return $t\geq n ~~?~~ t-n : t$

看SEAL中 `barrett reduction`的实现(在文件`src/utils/uintarithsmallmod.h`声明):

```C++
/*Returns input mod modulus. This is not standard Barrett reduction.
	Correctness: modulus must be at most 63-bit.*/
uint64_t barrett_reduce_64(uint64_t input, const Modulus &modulus){
    // Reduces input using base 2^64 Barrett reduction
    // floor(2^64 / mod) == floor( floor(2^128 / mod) )
    unsigned long long tmp[2];
    const uint64_t *const_ratio = modulus.const_ratio().data();
    // 这一步就看成 tmp[1] = input * const_ratio[1]; 即可, 即 q = a x m
    multiply_uint64_hw64(input, const_ratio[1], tmp + 1);
    // Barrett subtraction, 即 a = a- q*n
    tmp[0] = input - tmp[1] * modulus.value();
    // One more subtraction is enough, 即 a < n ? a: a-n
    return SEAL_COND_SELECT(tmp[0] >= modulus.value(), tmp[0] - modulus.value(), tmp[0]);
}
```

