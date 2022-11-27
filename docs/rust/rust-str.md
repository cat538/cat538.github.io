## Char
在rust中`char`类型表示单个字符。更具体地说，`char`实际上是“Unicode 标量值”(Unicode Scalar Value)。
> 这里涉及到Unicode 相关的知识，具体来说
> 
> - Unicode 是字符集；为每一个**字符**分配一个唯一的ID(code point)。code point 取值范围是0-10FFFF。
>   例如字符0的unicode number记作`U+0030`，字符1是`U+0031`，字符:heart:是`U+1F495`
> 
> - UTF-8 是编码规则的一种；编码规则是将一个code point 转换为字节序列的规则；有UTF-8，UTF-16等
> 其中UTF8是一套以 8 位为一个编码单位的可变长编码，规则如下：
> 
> U+ 0000 ~ U+  007F: 0XXXXXXX
> 
> U+ 0080 ~ U+  07FF: 110XXXXX 10XXXXXX
> 
> U+ 0800 ~ U+  FFFF: 1110XXXX 10XXXXXX 10XXXXXX
> 
> U+10000 ~ U+10FFFF: 11110XXX 10XXXXXX 10XXXXXX 10XXXXXX
>
> 按照上述规则可以知道，数字字符utf8编码后是1个字节，英文字母使用utf8编码后也是1个字节，而emoji编码后是4个字节

Unicode Scalar Value与Unicode Code Point类似，有所不同，但一般都用一个4字节整数来表示：
> `Unicode Scalar Value` is any `Unicode code point` except high-surrogate and low-surrogate code points. In other words, the ranges of integers *0* to *D7FF* and *E000* to *10FFFF* inclusive.


因此在rust中`char`可以直接由`u32`类型转换而来:

```rust
let c0 = char::from_u32(48).unwrap();
let c1 = char::from_u32(49).unwrap();
let ce;
unsafe { ce = char::from_u32_unchecked(0x1F495) }
println!("{c0}, {c1}, {ce}");// output: 0, 1, 💕
```
## String
`std::string::String`定义如下：

```rust
#[derive(PartialOrd, Eq, Ord)]
#[cfg_attr(not(test), rustc_diagnostic_item = "String")]
#[stable(feature = "rust1", since = "1.0.0")]
pub struct String {
    vec: Vec<u8>,
}
```
### 编码问题
不同于C++中的`std::string`不规定编码方式，rust中的`std::string::String`**总是使用utf-8编码**。也就是说，
`struct String`内部的字节vector存储的总是该字符串的utf-8编码序列(类似C++20中引入的`std::u8string`)。

使用 utf-8 encode后存储相比于直接存储对应的char数组(即unicode scalar value 数组 or说是utf32编码，)节省空间：
```rust
let ss = String::from("123abc");
let ss_utf8_len = ss.len();
let ss_chars_len = ss.chars().map(|x| mem::size_of_val(&x)).sum::<usize>();
println!("{}", ss_utf8_len);    // 6
println!("{}", ss_chars_len);   // 24
```
`String`**不允许使用下标索引**，因为一般来说，我们对一个字符串使用索引，
是期望获取到该字符串中的第n个"字符"。
但是因为`String`采用了utf8编码，没办法在常量时间内返回一个字符串的第n个字符（因为每个字符的长度可能是不一样的），只能通过：
```rust
ss.chars().nth(n); // 获取ss的第n个 char
```
理论上来说我们能在常量时间内索引到的是“该字符串的utf8编码序列中第n个字节”，但如果一个字符串的index返回一个u8字符，怎么说都有点怪。考虑上述原因，`std::string::String`不允许使用下标索引。


### method
- `String`实现了`Deref<Target = str>`，因此`String`拥有`str`类型的全部方法([参见Deref相关特性](https://doc.rust-lang.org/std/ops/trait.Deref.html#more-on-deref-coercion))
    ```rust
    impl ops::Deref for String {
        type Target = str;

        #[inline]
        fn deref(&self) -> &str {
            unsafe { str::from_utf8_unchecked(&self.vec) }
        }
    }
    ```
    而`str::from_utf8_unchecked` 实现如下：
    ```rust
    pub const unsafe fn from_utf8_unchecked(v: &[u8]) -> &str {
        // SAFETY: the caller must guarantee that the bytes `v` are valid UTF-8.
        // Also relies on `&str` and `&[u8]` having the same layout.
        unsafe { mem::transmute(v) }
    }
    ```
    其中`mem::transmute`将`&[u8]`解释为`&str`。因此对于形参为`&str`的函数，我们都可以传入`String`作为实参。
    
    而且从`String`到`&str`的转换几乎没有额外开销，因此除非有特殊设计，把接收字符串为参数的函数中参数类型设置为`&str`总是没错；同样也适用于`Vec<T>`和`&[T]`等。

- 标准库提供了`String`与其对应utf8编码序列的双向转换。
  
    通过`String`的`fn into_bytes(self) -> Vec<u8>`方法返回该字符串的utf8编码序列(这个方法会消费掉string本身，如果只需要一个引用 or )；而与之相对的逆操作是`fn from_utf8(vec: Vec<u8>) -> Result<String, FromUtf8Error>`，即验证一个`Vec<u8>`是否是合法的utf8编码序列，如果合法就映射为`String`。

- `String`还可以与其对应utf16编码序列的双向转换。
  
    通过`str`的`fn encode_utf16(&self) -> EncodeUtf16<'_>`方法返回该字符串的utf16编码序列；而与之相对的逆操作是`String`的method`fn from_utf16(v: &[u16]) -> Result<String, FromUtf16Error>`，即将一个`&[u16]`通过utf16编码方式映射为`String`。


- 而`String`的`len()`方法返回的也是字符串的utf8编码长度，即：
    ```rust
    pub fn len(&self) -> usize {
        self.vec.len()
    }
    // =============================================
    let s1 = String::from("我的Blog");
    println!("{}", s1.len()) //output: 10; <==10=3+3+1+1+1+1
    ```

### Trait
`std::string`模块还提供了一个trait`ToString`，但是该trait不应当由用户手动实现：
> This trait is automatically implemented for any type which implements the `Display` trait. As such, `ToString` shouldn’t be implemented directly: `Display` should be implemented instead, and you get the `ToString` implementation for free.

## OsString
> A type that can represent owned, mutable platform-native strings, but is cheaply inter-convertible with Rust strings.