# Rust 入门记录

## 变量

### const

**定义常量使用const而不是let**

Rust 中`let`定义的 Variable 虽然默认不可变，但是与const还是存在区别：

- constant 的生命周期是整个作用域
- 编译器可以推断 variable 的类型，但是定义 constant 必须显式声明类型

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

> 记住 Rust is a **statically typed language**, 这意味着rust需要在编译期知道所有变量的类型（想想js或python?）

**Scalar：**

- integers, 

- floating-point numbers, 

- Booleans, and 

- characters

### 关于 integer的溢出：

- debug模式编译时：rust执行integer overflow chek。如果程序运行时发生整数溢出，会直接panic；

- release模式编译时(`cargo build --release`)：rust不执行integer overflow check。如果程序运行时发生整数溢出，将进行wrap（与C语言默认行为相同），即，大于该类型最大值的数值将会“wrap around”到该类型可以容纳的最小值。在 u8 的情况下，256 变为 0，257 变为 1，依此类推。
  
    如果要显式wrap，可以使用标准库类型 `Wrapping`（**但是，可能是我眼界窄，从未见人使用过**）：
```rust
let zero = Wrapping(0u8);
let tff = Wrapping(255u8);
println!("{}", zero + tff + tff);
// 254
```

### char类型
!!!note
    关于字符和字符串类型，更详细的解释见[rust-str](./rust-str.md)

与C语言中的简单使用8位有符号整型来代表`char`类型不同：

Rust 中 `char` 类型是4字节，并且每个字符表示一个 `Unicode Scalar Value`,which means it can represent a lot more than just ASCII.

> Unicode Scalar Values range from `U+0000` to `U+D7FF` and `U+E000` to `U+10FFFF` inclusive

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

可以看到编码成utf-8之后，一个中文占3个字节，英文占一个字节。


### 复合数据类型(compound)

**tuple：**

```rust
let cat = ("Furry McFurson", 3.5);
let (name,age) = cat;
print!("{}", cat.0);
println!("{} is {} years old.", name, age);
```



## 函数
> Function bodies are made up of a series of **statements** optionally ending in an **expression**. 


### 语句statement 和 表达式expression

> **TL;DR:** *Statements*  **do not** return a value. *Expressions* evaluate to a resulting value. 
> 
> Rust is an **expression-based language**, this is an important distinction to understand. Other languages don’t have the same distinctions, so let’s look at what statements and expressions are and how their differences affect the bodies of functions.
> 

```rust
let x = (let y = 6);
```

`let y = 6`语句不返回值，所以 x 没有任何东西可以绑定。这与其他语言（例如 C 和 Ruby）中发生的情况不同，C的赋值语句返回赋值的值，因此可以写`int x = y = 3`（这也是为什么C++中，为一个自定义class定义`operator=`需要返回`Object&`），而在rust是不可行的。

表达式可以是语句的一部分：`let y = 6`中的`6`是一个表达式，它计算出的值是`6`；函数调用是一个表达式；宏调用是一个表达式；我们用来创建新作用域的大括号（代码块） `{}` 也是一个表达式，例如：
```rust
fn main() {
    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {}", y);
}
```
##  Ownership(所有权)

- 分配和访问 stack上的数据快于heap上的数据

    > **Pushing to the stack is faster than allocating on the heap**。因为分配栈变量不必搜索存储新数据的位置；只需要移动栈顶指针（ESP）。相比之下，在堆上分配空间需要更多的工作，因为分配器必须首先找到足够大的空间来存放数据，然后进行标记，为下一次分配做准备。
    >
    > **Accessing data in the heap is slower than accessing data on the stack**。
    主要是cache的原因，在内存中跳动小，局部性好，速度就会更快。同样的道理，在堆上分配大量的空间也会花费时间。

所有权的规则：

- Rust 中的每一个值都有一个被称为其 所有者（owner）的变量。
- 值在任一时刻有且只有一个所有者。
- 当所有者（变量）离开作用域，这个值将被丢弃。

**考虑C++中的`unique_ptr`**

### 引用(reference)和借用(borrowing)
!!! note "TL;DR"
    关于引用的两个条件：

    - 在任意给定时间，要么 只能有一个可变引用，要么 只能有多个不可变引用。
    - 引用必须总是有效的(编译器检查并禁止Dangling References)。
> 我们将创建一个引用的行为称为 借用（borrowing）

多数情况下，我们更希望能访问数据，同时不取得其所有权。为实现这点，Rust 使用 了**借用**（borrowing）机制。对象可以通过引用（`&T`）来传递，从而取代通过 值（`T`）来传递。

**另外需要注意的一点是：**

一个数据可以有多个不可变借用，但是在使用数据的不可变借用的同时，不能使用数据的可变借用。

或者说，同一时间内只允许**一个**可变借用。仅当最后一次使用可变借用**之后**，原始数据才可以再次借用。如下例子：

```rust
struct Point { x: i32, y: i32, z: i32 }

fn main() {
    let mut point = Point { x: 0, y: 0, z: 0 };
    let borrowed_point = &point;
    let another_borrow = &point;
    // 数据可以通过引用或原始类型来访问
    println!("Point has coordinates: ({}, {}, {})",
                borrowed_point.x, another_borrow.y, point.z); 
    // 报错！`point` 不能以可变方式借用，因为当前还有不可变借用。
    // let mutable_borrow = &mut point;
    // TODO ^ 试一试去掉此行注释
    
    // 被借用的值在这里被重新使用
    println!("Point has coordinates: ({}, {}, {})",
                borrowed_point.x, another_borrow.y, point.z);
    // 不可变的引用不再用于其余的代码，因此可以使用可变的引用重新借用。
    let mutable_borrow = &mut point;
    // 通过可变引用来修改数据
    mutable_borrow.x = 5;
    mutable_borrow.y = 2;
    mutable_borrow.z = 1;
    // 报错！不能再以不可变方式来借用 `point`，因为它当前已经被可变借用。
    // let y = &point.y;
    // TODO ^ 试一试去掉此行注释

    // 报错！无法打印，因为 `println!` 用到了一个不可变引用。
    // println!("Point Z coordinate is {}", point.z);
    // TODO ^ 试一试去掉此行注释

    // 正常运行！可变引用能够以不可变类型传入 `println!`
    println!("Point has coordinates: ({}, {}, {})",
                mutable_borrow.x, mutable_borrow.y, mutable_borrow.z);

    // 可变引用不再用于其余的代码，因此可以重新借用
    let new_borrowed_point = &point;
    println!("Point now has coordinates: ({}, {}, {})",
             new_borrowed_point.x, new_borrowed_point.y, new_borrowed_point.z);
}
```

## Macro(宏)
!!! note
    这一部分详细信息见[rust-macro](./rust-macro.md)
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
!!!note
    关于字符和字符串类型，更详细的解释见[rust-str](./rust-str.md)

string不能使用 index。

rust中的String存储方式：

```rust
#[derive(PartialOrd, Eq, Ord)]
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

在当有可能会导致有害状态的情况下建议使用 `panic!` —— 有害状态是指当一些假设、保证、协议或不可变性被打破的状态，例如无效的值、自相矛盾的值或者被传递了不存在的值 —— 外加如下几种情况：

有害状态是非预期的行为，与偶尔会发生的行为相对，比如用户输入了错误格式的数据。
在此之后代码的运行依赖于不处于这种有害状态，而不是在每一步都检查是否有问题。
没有可行的手段来将有害状态信息编码进所使用的类型中的情况。我们会在第十七章 “将状态和行为编码为类型” 部分通过一个例子来说明我们的意思。
如果别人调用你的代码并传递了一个没有意义的值，最好的情况也许就是`panic!`并警告使用你的库的人他的代码中有 bug 以便他能在开发时就修复它。类似的，如果你正在调用不受你控制的外部代码，并且它返回了一个你无法修复的无效状态，那么 `panic!` 往往是合适的。

然而当错误预期会出现时，返回 `Result` 仍要比调用 `panic!` 更为合适。这样的例子包括解析器接收到格式错误的数据，或者 HTTP 请求返回了一个表明触发了限流的状态。在这些例子中，应该通过返回 `Result` 来表明失败预期是可能的，这样将有害状态向上传播，调用者就可以决定该如何处理这个问题。使用 `panic!` 来处理这些情况就不是最好的选择。

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



## Rustling-别名（alias）

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

## Rust monad

这里以 I/O 操作为例，下面是与 ChatGPT的对话：

> 🐋: what's the main difference between monad I/O style and C-like I/O style?
>
> 🦭: The main difference between Monad I/O style and C-like I/O style lies in the way they handle Input/Output (I/O) operations.
> 
> In the C-like I/O style, I/O is performed by calling functions and passing parameters, and **I/O functions are typically side-effecting and can change the state of the system**. For example, to read a line of text from standard input in C, you might use the fgets function, which reads a specified number of characters from an input stream into a buffer, and can change the position of the input stream's file pointer.
> 
> On the other hand, the Monad I/O style, which is commonly used in functional programming languages, **models I/O as a computation that produces a value and possibly has a side effect**. In this style, I/O operations are performed by constructing a computation, usually represented as a monad, that represents the I/O action. The monad can then **be executed by a runtime system, which will perform the actual I/O** and return the result to the caller.
> 
> One of the main advantages of the Monad I/O style is that it makes it easier to write code that is pure and composable, as it **separates the description of I/O operations from their execution**, which can make it easier to test and reason about the code. In contrast, C-like I/O style can make it harder to write code that is pure, as I/O functions can have side effects that change the state of the system and make it difficult to predict the outcome of a computation.
