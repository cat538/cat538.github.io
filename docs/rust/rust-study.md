---
title: Rust学习
date: 2021-7-5 19:50:30
tags: ['Rust']
categories: ['Rust']
description: "Rust basic syntax..."
cover: "https://github.com/cat538/images-auto/raw/main/img/rustlogo.jpg"
---


## 变量

### const

**定义常量使用const而不是let**

let定义的 Variable 虽然默认不可变，但是与const还是存在区别。

- constant 的生命周期是整个作用域

编译器可以推断 variable 的类型，但是声明 constant 必须指明其类型

constant必须使用 `constant expression` 来设置，如：

```rust
// 1. 常量表达式
const LEN: usize = 2*16;
// 2. const fn
impl A {
    const fn new () -> A {
        A{
            a: 0,
            b: 0,
            c: 0
        }
    }
}
const TST: A = A::new();
```



## 数据类型

rust 有两种数据类型： scalar and compound，即所谓简单类型和复合类型

> Keep in mind that Rust is a *statically typed language*, which means that it must know the types of all variables at compile time

**Scalar：**

integers, 

floating-point numbers, 

Booleans, and 

characters

关于 integer的溢出：

When you’re compiling in debug mode, Rust includes checks for integer overflow that cause your program to *panic* at runtime if this behavior occurs. Rust uses the term panicking when a program exits with an error; we’ll discuss panics in more depth in the "Unrecoverable Errors with panic!"section in Chapter 9 of the Rust Programming Language Book.

When you’re compiling in release mode with the --release flag, Rust does *not* include checks for integer overflow that cause panics. Instead, if overflow occurs, Rust performs *two’s complement wrapping*. In short, values greater than the maximum value the type can hold “wrap around” to the minimum of the values the type can hold. In the case of a u8, 256 becomes 0, 257 becomes 1, and so on. The program won’t panic, but the variable will have a value that probably isn’t what you were expecting it to have. Relying on integer overflow’s wrapping behavior is considered an error. If you want to wrap explicitly, you can use the standard library type `Wrapping`

```rust
let zero = Wrapping(0u8);
let tff = Wrapping(255u8);
println!("{}", zero + tff + tff);
// 254
```

Rust 中 char 是4字节，并且每个字符represents a Unicode Scalar Value,which means it can represent a lot more than just ASCII.

Unicode Scalar Values range from U+0000 to U+D7FF and U+E000 to U+10FFFF inclusive

```rust
let c = '端';
println!("{} is {} bytes", c, size_of_val(&c));
// 端 is 4 bytes
let mut b = [0; 4];
let result = '端'.encode_utf8(&mut b);
println!("result len: {} bytes",result.len());
// result len: 3 bytes
let result = 'd'.encode_utf8(&mut b);
println!("result len: {} bytes",result.len());
// result len: 1 bytes
```

可以看到编码成utf-8之后，一个中文占3个字节，英文占一个字节



### 复合数据类型(compound)

**tuple：**

```rust
let cat = ("Furry McFurson", 3.5);
let (name,age) = cat;
print!("{}", cat.0);
println!("{} is {} years old.", name, age);
```



## 函数

关于statement 和 expression的区别：

> Rust is an **expression-based language**, this is an important distinction to understand. Other languages don’t have the same distinctions, so let’s look at what statements and expressions are and how their differences affect the bodies of functions.
>
> We’ve actually already used statements and expressions. *Statements* are instructions that perform some action and **do not** return a value. *Expressions* evaluate to a resulting value. 



```rust
let x = (let y = 6);
```

The let y = 6 statement does not return a value, so there isn’t anything for x to bind to. This is different from what happens in other languages, such as **C and Ruby, where the assignment returns the value of the assignment**.



##  Ownership(所有权)

分配和访问 stack上的数据快于heap上的数据

> **Pushing to the stack is faster than allocating on the heap** because the allocator never has to search for a place to store new data; that location is always at the top of the stack. Comparatively, allocating space on the heap requires more work, because the allocator must first find a big enough space to hold the data and then perform bookkeeping to prepare for the next allocation.
>
> **Accessing data in the heap is slower than accessing data on the stack** because you have to follow a pointer to get there. Contemporary processors are faster if they jump around less in memory.  By the same token, a processor can do its job better if it works on data that’s close to other data (as it is on the stack) rather than farther away (as it can be on the heap). Allocating a large amount of space on the heap can also take time.

### what is borrowing？

> We call having references as function parameters *borrowing*.

关于引用的两个条件

- At any given time, you can have *either* one mutable reference *or* any number of immutable references.
- References must always be valid.

## Macro(宏)

**关于macro的声明位置：**

> In order to use a macro outside of its module, you need to do something special to the module to lift the macro out into its parent.
>
> The same trick also works on "extern crate" statements for crates that have exported macros, if you've seen any of those around.

例如：

```rust
#[macro_export]
mod macros {
    macro_rules! my_macro {
        () => {
            println!("Check out my macro!");
        };
    }
}

fn main() {
    my_macro!();
}
```

必须使用`#[macro_export]`，而不能`macros::my_macro!()`





## String

string不能使用 index，不妨探究一下String的存储方式：

```rust
#[derive(PartialOrd, Eq, Ord)]
#[cfg_attr(not(test), rustc_diagnostic_item = "string_type")]
#[stable(feature = "rust1", since = "1.0.0")]
pub struct String {
    vec: Vec<u8>,
}
```

在`string.rs`中可以很清楚的看到，String只是对`Vec<u8>`做了一个包装

因此 从 Rust 的角度来讲，事实上有三种相关方式可以理解字符串：字节、标量值和字形簇（最接近人们眼中 **字母** 的概念）。



## Type Conversion



## Errors

使用backtrace：

在powershell中：

```shell
$env:RUST_BACKTRACE=1 ; cargo run
```

在linux 终端中：

```shell
RUST_BACKTRACE=1 cargo run
```

### 错误处理指导原则



## Pattern match

下面介绍模式匹配的所有语法

### 1. 匹配命名变量

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(y) => println!("Matched, y = {:?}", y),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
}
```

​	让我们看看当 `match` 语句运行的时候发生了什么。第一个匹配分支的模式并不匹配 `x` 中定义的值，所以代码继续执行。

​	第二个匹配分支中的模式引入了一个新变量 `y`，它会匹配任何 `Some` 中的值。因为我们在 `match` 表达式的新作用域中，这是一个新变量，而不是开头声明为值 10 的那个 `y`。这个新的 `y` 绑定会匹配任何 `Some` 中的值，在这里是 `x` 中的值。因此这个 `y` 绑定了 `x` 中 `Some` 内部的值。这个值是 5，所以这个分支的表达式将会执行并打印出 `Matched, y = 5`。

 **`match` 会开始一个新作用域**



### 2. 多个模式&范围匹配

- 在 `match` 表达式中，可以使用 `|` 语法匹配多个模式，它代表 **或**（*or*）的意思。
- `..=` 语法允许你匹配一个闭区间范围内的值。（**注意只允许闭区间**）



### 3. 解构匹配

结构匹配元组、枚举、结构体等

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::ChangeColor(0, 160, 255);

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.")
        }
        Message::Move { x, y } => {//解构匹配结构体
            println!(
                "Move in the x direction {} and in the y direction {}",
                x,
                y
            );
        }
        Message::Write(text) => println!("Text message: {}", text),//普通的匹配
        Message::ChangeColor(r, g, b) => {//解构匹配元组
            println!(
                "Change the color to red {}, green {}, and blue {}",
                r,
                g,
                b
            )
        }
    }
}
```



**在解构匹配中，可以使用 `..` 忽略剩余值：**

对于有多个部分的值，可以使用 `..` 语法来只使用部分并忽略其它值，同时避免不得不每一个忽略值列出下划线。`..` 模式会忽略模式中剩余的任何没有显式匹配的值部分。在示例 18-23 中，有一个 `Point` 结构体存放了三维空间中的坐标。在 `match` 表达式中，我们希望只操作 `x` 坐标并忽略 `y` 和 `z` 字段的值：

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

let origin = Point { x: 0, y: 0, z: 0 };

match origin {
    Point { x, .. } => println!("x is {}", x),
}
```



### 4. @ 绑定

*at* 运算符（`@`）允许我们在创建一个存放值的变量的同时测试其值是否匹配模式。示例 18-29 展示了一个例子，这里我们希望测试 `Message::Hello` 的 `id` 字段是否位于 `3..=7` 范围内，同时也希望能将其值绑定到 `id_variable` 变量中以便此分支相关联的代码可以使用它。可以将 `id_variable` 命名为 `id`，与字段同名，不过出于示例的目的这里选择了不同的名称。



## Trait

### trait 作为参数

知道了如何定义 trait 和在类型上实现这些 trait 之后，我们可以探索一下如何使用 trait 来接受多种不同类型的参数。

示例中为 `NewsArticle` 和 `Tweet` 类型实现了 `Summary` trait。我们可以定义一个函数 `notify` 来调用其参数 `item` 上的 `summarize` 方法，该参数是实现了 `Summary` trait 的某种类型。为此可以使用 `impl Trait` 语法，像这样：

```rust
pub fn notify(item: impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

对于 `item` 参数，我们指定了 `impl` 关键字和 trait 名称，而不是具体的类型。该参数支持任何实现了指定 trait 的类型。在 `notify` 函数体中，可以调用任何来自 `Summary` trait 的方法，比如 `summarize`。我们可以传递任何 `NewsArticle` 或 `Tweet` 的实例来调用 `notify`。任何用其它如 `String` 或 `i32` 的类型调用该函数的代码都不能编译，因为它们没有实现 `Summary`。

**`impl Trait` 语法适用于直观的例子，它不过是一个较长形式的语法糖。这被称为 *trait bound*，这看起来像：**

```rust
pub fn notify<T: Summary>(item: T) {
    println!("Breaking news! {}", item.summarize());
}
```

这与之前的例子相同，不过稍微冗长了一些。trait bound 与泛型参数声明在一起，位于尖括号中的冒号后面。

### where

Rust 有另一个在函数签名之后的 `where` 从句中指定 trait bound 的语法。所以除了这么写：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: T, u: U) -> i32 {
```

还可以像这样使用 `where` 从句：

```rust
fn some_function<T, U>(t: T, u: U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

这个函数签名就显得不那么杂乱，函数名、参数列表和返回值类型都离得很近，看起来类似没有很多 trait bounds 的函数。



### 生命周期

每一个引用都有一个生命周期。

生命周期在函数中用来解决悬垂引用的问题：即函数返回一个引用，Rust要保证这个引用是有效的

例如：

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

将不能被编译通过，因为 x or y都有可能是一个悬垂引用，考虑以下情况：

```rust
fn main() {
    let string1 = String::from("xyz");
    let result;
    {
        let string2 = String::from("long string is long");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

(result为string2的引用，在离开作用域后失效)



当从函数返回一个引用，返回值的生命周期参数需要与一个参数的生命周期参数相匹配。

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

如图所示：返回的引用 **没有** 指向任何一个参数，那么唯一的可能就是它指向一个函数内部创建的值，它将会是一个悬垂引用，因为它将会在函数结束时离开作用域。

### 生命周期省略（Lifetime Elision）

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

上述代码可以编译通过。

> 这个函数没有生命周期注解却能编译是由于一些历史原因：在早期版本（pre-1.0）的 Rust 中，这的确是不能编译的。
>
> 在编写了很多 Rust 代码后，Rust 团队发现在特定情况下 Rust 程序员们总是重复地编写一模一样的生命周期注解。这些场景是可预测的并且遵循几个明确的模式。接着 Rust 团队就把这些模式编码进了 Rust 编译器中，如此借用检查器在这些情况下就能推断出生命周期而不再强制程序员显式的增加注解。

### 静态生命周期

这里有一种特殊的生命周期值得讨论：`'static`，其生命周期**能够**存活于整个程序期间。所有的字符串字面值都拥有 `'static` 生命周期，我们也可以选择像下面这样标注出来：

```rust
let s: &'static str = "I have a static lifetime.";
```

这个字符串的文本被直接储存在程序的二进制文件中而这个文件总是可用的。因此所有的字符串字面值都是 `'static` 的。



## 多线程

`spawn()`可以船舰一个子线程，返回一个`JoinHandle`句柄，可以使用`join()`方法进行阻塞

`move` 闭包，经常与 `thread::spawn` 一起使用，因为它允许我们在一个线程中使用另一个线程的数据。

> 在参数列表前使用 `move` 关键字**强制闭包获取其使用的环境值的所有权**。这个技巧在创建新线程将值的所有权从一个线程移动到另一个线程时最为实用。（**这一点与C++很不同，在某种程度上于编译期杜绝了资源竞争问题**）

### 消息传递

> 一个日益流行的确保安全并发的方式是 **消息传递**（*message passing*），这里线程或 actor 通过发送包含数据的消息来相互沟通。这个思想来源于 [Go 编程语言文档中](http://golang.org/doc/effective_go.html) 的口号：“不要通过共享内存来通讯；而是通过通讯来共享内存。”（“Do not communicate by sharing memory; instead, share memory by communicating.”）
>
> Rust 中一个实现消息传递并发的主要工具是 **通道**（*channel*），Rust 标准库提供了其实现的编程概念。你可以将其想象为一个水流的通道，比如河流或小溪。如果你将诸如橡皮鸭或小船之类的东西放入其中，它们会顺流而下到达下游。



# 死灵书学习

## 别名（alias）

```c
int compute(int* input, int* output) {
    if(*input > 10) {
        *output = 1;
    }
    if (*input > 5) {
        *output *= 2;
    }
}
```

```rust
fn comput(input:&i32, output:&mut i32) {
    if *input > 10 {
        *output = 1;
    }
    if *input > 5 {
        *output *= 2;
    }
}
```

rust编译器可以将代码优化为，在C中则不可能做出这种优化（**考虑input 和output指针指向同一地址**）：

```rust
fn comput(input:&i32, output:&mut i32) {
    let cached_input = *input;
    if cached_input > 10 {
        *output = 2; // x>10 则必然>5，直接加倍退出
    } else if cached_input > 5 {
        *output *= 2;
    }
}
```

![image-20210825005100468](https://github.com/cat538/images-auto/raw/main/img/image-20210825005100468.png)![image-20210825005110601](https://github.com/cat538/images-auto/raw/main/img/image-20210825005110601.png)

