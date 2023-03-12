# Alignment

## Alignment 概念
内存对齐指的是数据在内存中的布局需要保证一定的对齐规则。
保证字节对齐主要是为了保证CPU高效读取数据。
对于写代码的人来说，使用字节对齐的技巧在涉及到搭建底层数据结构，以及网络编程，数据结构序列化/反序列化等场景时非常有用。

关于C/C++ POT类型字节对齐规则网上有很多资料。可以参考
[Microsoft-learn-Alignment](https://learn.microsoft.com/en-us/cpp/cpp/alignment-cpp-declarations?view=msvc-170)

简单来说：
> 1. By default, the compiler aligns class and struct members on their size value: `bool` and `char` on 1-byte boundaries, `short` on 2-byte boundaries, `int`, `long`, and `float` on 4-byte boundaries, and `long long`, `double`, and `long double` on 8-byte boundaries.
> 2. 对于结构体来说，除了保证上述规则之外，还需要保证结构体的大小是其成员中最大的那一个的整数倍（例如，结构体中有uint64_t，那么结构体需要保证自己的size为8字节的倍数）

```C++
struct x_
{
   char a;     // 1 byte
   int b;      // 4 bytes
   short c;    // 2 bytes
   char d;     // 1 byte
} bar[3];

// Shows the actual memory layout
struct x_
{
   char a;            // 1 byte
   char _pad0[3];     // padding to put 'b' on 4-byte boundary
   int b;            // 4 bytes
   short c;          // 2 bytes
   char d;           // 1 byte
   char _pad1[1];    // padding to make sizeof(x_) multiple of 4
} bar[3];
```
- Both declarations return sizeof(struct x_) as 12 bytes.
- Both declarations return alignof(struct x_) as 4 bytes.

## Alignment 使用场景
这个概念落实到代码上主要表现为，有时我们需要保证自定义的`struct`,`class`等遵循一定的对齐规则。

比如在上面的例子中，可以看到`struct x_`在default情况存在一定的填充，而这个填充在不同的机器(or 不同编译器)上编译出的程序可能是不一样的。考虑一个将该结构体序列化，通过网络收发的场景，如果没有预先沟通好字节对齐规则的话，则可能出现解析错误。

再比如SIMD指令使用中，往往对于数据的对齐有一些要求：
```c++
/*将 256 位（由 4 个打包的 64 位整数组成）从内存加载到 dst。 mem_addr 必须在 32 字节边界上对齐，否则可能会生成一般保护异常。
*/
__m256i _mm256_load_epi64 (void const* mem_addr)

/*将 256 位（由 4 个打包的 64 位整数组成）从内存加载到 dst。 mem_addr 不需要在任何特定边界上对齐。
*/
__m256i _mm256_loadu_epi64 (void const* mem_addr)

```
可以看到`_mm256_load_epi64`就要求load的地址必须是32字节对齐的，这个约束同时也会带来效率上的提升。因此在使用SIMD时，应当尽量保证字节对齐要求，避免性能损失。
> [Stackoverflow-difference between loadu and load?](https://stackoverflow.com/questions/15964367/what-is-the-difference-between-loadu-and-load)

## 使用`alignof`,`alignas`,`aligned_new`
在C++11之前，通过使用不同编译器的不同built-in指令来控制内存对齐：
```c++
#if !defined(ALIGN16)
#if defined(__GNUC__)
#define ALIGN16 __attribute__((aligned(16)))
#define ALIGN32 __attribute__((aligned(32)))
#else
#define ALIGN16 __declspec(align(16))
#define ALIGN32 __declspec(align(32))
#endif
#endif

uint8_t unaligned_arr[42]{};
ALIGN16 uint8_t aligned16_arr[42]{};
ALIGN32 uint8_t aligned32_arr[42]{};
```
在C++11之后，标准库提供了一个关键字operator `alignof`，提供了`alignas specifier`来保证分配数据的内存对齐（即代替上面的`ALIGN16/32`）。
C++17 还提供了`aligned_new`来提供保证自定义对齐规则的内存分配，即`new(std::align_val_t(16)) Bar`。
```C++
// alignas_alignof.cpp
// compile with: cl /EHsc alignas_alignof.cpp
#include <iostream>

struct alignas(16) Bar {
    int i;       // 4 bytes
    int n;      // 4 bytes
    alignas(16) char arr[3];
    short s;          // 2 bytes
};

int main() {
    std::cout << alignof(Bar) << std::endl; // output: 16
    std::cout << sizeof(Bar) << std::endl; // output: 32
    std::unique_ptr<__m256i[]> arr{new(std::align_val_t{alignof(__m256i)}) __m256i[32]}; // aligned new
}
```
