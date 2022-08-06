---
title: FFT&NTT
date: 2021-07-29 13:36:54
tags: '数据结构&算法'
categories: '数据结构&算法'
description: "FFT算法以及其变种——NTT(数论变换)算法..."
# FFT在多项式的应用

两个多项式相乘传统算法复杂度$O(n^2)$；

FFT把多项式系数表示法$\Longrightarrow$点值表示法，这个过程复杂度为$O(n\log n)$；

两组点值相乘复杂度$O(n)$；

## 算法实现

FFT的递归过程：

- 递归展开：按照奇、偶幂次层层递归划分多项式，直至项数为1（此时$m$项多项式被分割为$m$个$1$次多项式，每一项对应一个点值）
  - 划分时：$P(x) = P_e(x^2) + x\cdot P_o(x^2)$
- 递归回收：对于$m$项的多项式，根据下一层已经计算出的，两个$m/2$项的多项式对应的点值，计算出$m$项多项式对应的$m$个点值
  - 恢复计算：$P(x) = P_e(x^2) + x\cdot P_o(x^2)~~、~~P(-x) = P_e(x^2) - x\cdot P_o(x^2)$

FFT利用了单位根的性质：

![image-20220112140049620](https://github.com/cat538/images-auto/raw/main/img/image-20220112140049620.png)

单位根：$z^n = 1$，方程的复数根$z$为**n次单位根**。单位的 n次根有n个：$e^{\frac{2\pi ki}{n}},k=0,1,2,3\cdots n-1$

单位的n次根以乘法构成n阶循环群。

一个例子作为说明：

<img src="https://github.com/cat538/images-auto/raw/main/img/fft.png" alt="fft" style="zoom: 67%;" />

如上图所示，使用FFT算法求7阶多项式$P(x)$在$\omega_1,\omega_2\cdots,\omega_8$8个点(8个8次单位根)的值：

- 求$P(x)$在$\omega^8_1,\omega^8_2\cdots,\omega^8_8$8个点的值$\Longrightarrow$只需要分别计算$P_e(x^2),P_o(x^2)$在$\omega^8_1,\omega^8_2\cdots,\omega^8_8$8个点的值，

  但**由于单位根的性质**，8个8次单位根$\omega^8_1,\omega^8_2\cdots,\omega^8_8$的平方恰好为4个4次单位根$\omega^4_1,\omega^4_2,\omega^4_3,\omega^4_4$；

  因此事实上计算$P_e(x^2),P_o(x^2)$在$\omega^8_1,\omega^8_2\cdots,\omega^8_8$8个点的值$\Longrightarrow$计算$P_e(x),P_o(x)$在$\omega^4_1,\omega^4_2,\omega^4_3,\omega^4_4$4个点(4个4次单位根)的值

  递归；对于7阶多项式，如上划分3次之后变为0阶多项式，即8个系数；因为每次都按奇偶划分，最后排列规律如下：

- 04261537的二进制数分别是000, 100, 010, 110, 001, 101, 011, 111，每个数倒置一下得到000, 001, 010, 011, 100, 101, 110, 111，正好就是01234567；对于其他2的幂也是成立的

- 然后就是从$P_e(x^2),P_o(x^2)$求$P(x)$的过程: 以最上层为例，要根据$P_e(x),P_o(x)$在$\omega^4_1,\omega^4_2,\omega^4_3,\omega^4_4$的值求出$P(x)$在$\omega^8_1,\omega^8_2\cdots,\omega^8_8$的值

  <img src="https://github.com/cat538/images-auto/raw/main/img/image-20220112140629546.png" alt="image-20220112140629546" style="zoom:70%;" />

  只需要根据$P(x) = P_e(x^2) + x\cdot P_o(x^2)~~、~~P(-x) = P_e(x^2) - x\cdot P_o(x^2)$，即只需前4个8次单位根$\omega^8_1,\omega^8_2,\omega^8_3,\omega^8_4$即可求出8个值
  
$$
\begin{align*}
P(\omega^8_1) &=P_e((\omega^8_1)^2) + \omega^8_1\cdot P_o((\omega^8_1)^2) = P_e(\omega^4_1) + \omega^8_1\cdot P_o(\omega^4_1)\\
P(\omega^8_5)=P(-\omega^8_1)  &= P_e((\omega^8_1)^2) - \omega^8_1\cdot P_o((\omega^8_1)^2) = P_e(\omega^4_1) - \omega^8_1\cdot P_o(\omega^4_1)\\
P(\omega^8_2) &=P_e((\omega^8_2)^2) + \omega^8_2\cdot P_o((\omega^8_2)^2) = P_e(\omega^4_2) + \omega^8_2\cdot P_o(\omega^4_2)\\
P(\omega^8_6)=P(-\omega^8_2) &= P_e((\omega^8_2)^2) - \omega^8_2\cdot P_o((\omega^8_2)^2) = P_e(\omega^4_2) - \omega^8_2\cdot P_o(\omega^4_2)\\
P(\omega^8_3) &=P_e((\omega^8_3)^2) + \omega^8_3\cdot P_o((\omega^8_3)^2) = P_e(\omega^4_3) + \omega^8_3\cdot P_o(\omega^4_3)\\
P(\omega^8_7)=P(-\omega^8_3)  &= P_e((\omega^8_3)^2) - \omega^8_3\cdot P_o((\omega^8_3)^2) = P_e(\omega^4_3) - \omega^8_3\cdot P_o(\omega^4_3)\\
P(\omega^8_4) &=P_e((\omega^8_4)^2) + \omega^8_4\cdot P_o((\omega^8_4)^2) = P_e(\omega^4_4) + \omega^8_4\cdot P_o(\omega^4_4)\\
P(\omega^8_8)=P(-\omega^8_4) &= P_e((\omega^8_4)^2) - \omega^8_4\cdot P_o((\omega^8_4)^2) = P_e(\omega^4_4) - \omega^8_4\cdot P_o(\omega^4_4)
\end{align*}
$$



以FFT的变种`ntt`代码为例：

```rust
pub fn ntt_memoized(poly: &mut [u64], poly_len: usize, modulus: u64, flag: bool) {
    ntt_prelude(poly, poly_len); // 迭代算法首先变换系数位置，按原来系数为止的2进制反转存储
    let mut x: u64;
    let mut y: u64;
    let mut layer_len = 1;
    // lookup_ntt 选取一系列单位根: 1个2次单位根 + 2个4次单位根 + 4个8次单位根+...
    // 对于 poly_len 长多项式，poly_len = 2^n; 共选取1+2+4+... = 2^n-1个单位根
    let ntt_cache = &lookup_ntt(modulus, poly_len); 
    let mut acc = 0; // index for wn
    let wn = if flag {
        &ntt_cache.roots;
    } else {
        &ntt_cache.roots_inv;
    };
    while layer_len < poly_len { // layer_len := 1,2,4...n/2
        for start in (0..poly_len).step_by(layer_len * 2) {
            for i in 0..layer_len { // layer_len = k ：由p_e(x)的k个点、值，p_o(x)的k个点、值计算p(x)的2k个点值
                // p_e(x) ==> poly[start..start + layer_len]
                // p_o(x) ==> poly[start + layer_len..start + 2 x layer_len]
                x = poly[start + i]; 
                y = wn[acc + i] * poly[start + i + layer_len] % modulus;
                poly[start + i] = (x + y) % modulus;
                poly[start + i + layer_len] = (modulus + x - y) % modulus;
            }
        }
        acc += layer_len;
        layer_len <<= 1;
    }
    if flag == false {
        let len_inv = qpow(poly_len as u64, modulus - 2, modulus); // 求len逆元
        for i in 0..poly_len {
            poly[i] = poly[i] * len_inv % modulus;
        }
    }
}
```

