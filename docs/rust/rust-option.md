# Rust-Option

## Rust 的 Option枚举

Option是一个枚举类型，定义如下，

```rust
enum Option<T> {
    Some(T),
    None,
}
```

### 初始化Option：

使用`let x = Some(T)`，编译器可以自动推断x对应的枚举类型，如`let some_string = Some("a string");`

但使用`None`则不可以如此：

因为编译器只通过 `None` 值无法推断出 `Some` 成员保存的值的类型。

需要事先声明`Option`的类型，如`let absent_number: Option<i32> = None;`



### 解开Option：

- 可以使用 `match` 语句来解开 `Option`
- 可以使用`unwrap()`来获得`Option`中的`Some`，但如果使用这种方式，获得`None`的时候程序将会panic

### Monad operation

实际上 `Option` 是一个Monad，使用`Option`的编程范式应该是函数式的，而不是总是先使用`match` 或者`unwrap*()`取出其中的值。下面是与 ChatGPT的对话：

> 🐋: Show me an example of monad in rust
> 
> 🦭: Here's an example of using monads in Rust:
>
> ```rust
> use std::option::Option;
> 
> fn add_two_numbers(x: i32, y: i32) -> Option<i32> {
>     Some(x + y)
> }
> 
> fn square_number(x: i32) -> Option<i32> {
>     Some(x * x)
> }
> 
> fn main() {
>     let result = add_two_numbers(2, 3)
>         .and_then(square_number)
>         .unwrap_or(-1);
> 
>     println!("Result: {}", result);
> }
> ```
> 
> In this example, we have two functions `add_two_numbers` and `square_number` that each return an `Option` monad. The `and_then` method is used to chain these functions together and perform a calculation on the result of the first function before passing it to the next. The `unwrap_or` method is used to provide a default value if either of the functions return None.
> 
> In this example, the calculation `add_two_numbers(2, 3)` results in `Some(5)`, which is then passed to `square_number` and returns `Some(25)`. Finally, the value is unwrapped to `25` and printed to the console.
> 
> 🐋: What's the difference between this programming paradigm and first checking whether the value exists, and then calculating
> 
> 🦭: The main difference between using monads and checking for values before performing a calculation **is the way that error handling and control flow are managed**.
> 
> 在执行计算之前检查值时，通常会使用 `if` 语句检查值是否存在且有效，如果存在则执行计算。 这种方法通常会导致嵌套的 `if` 语句，并可能导致代码难以阅读和维护。
> 
> 相反，当使用 monad 时，错误处理和控制流以 声明式 的方法处理。 您可以使用 monad 将操作链接在一起并自动处理错误情况，而不需要显式条件判断，不使用嵌套的 `if` 语句。 这使得代码更干净、更简洁、更易于阅读和维护。
> 
> For example, consider the following code that **checks for values before performing a calculation**:
>
> ```rust
> let x = get_input();
> let y = get_input();
> 
> if x.is_some() && y.is_some() {
>     let x = x.unwrap();
>     let y = y.unwrap();
>     let result = x + y;
>     println!("Result: {}", result);
> } else {
>     println!("Error: invalid input");
> }
> ```
> 此代码使用 `is_some` 方法检查 x 和 y 是否为 `Some` 值，如果是，则解包并执行计算。
> 
> Here's the same code written using monads:
>
> ```rust
> let result = get_input()
>     .and_then(|x| get_input().map(|y| x + y))
>     .unwrap_or_else(|| println!("Error: invalid input"));
> 
> println!("Result: {}", result);
> ```
>
> 在这段代码中， `and_then` 方法用于将两个 `get_input` 调用链接在一起并执行计算， `unwrap_or_else` 方法用于处理错误情况。 结果与前面的示例相同，但代码更干净、更简洁。

### 使用`Option<T>` 为什么比使用空值要好？

**Option解决了空值引发的一系列问题？？？**

空值的问题在于当你尝试像一个非空值那样使用一个空值，会出现某种形式的错误？？？

而因为 `Option<T>` 和 `T`（这里 `T` 可以是任何类型）是不同的类型，编译器不允许像使用一个肯定有效的值那样使用 `Option<T>`。例如，这段代码不能编译，因为它尝试将 `Option<i8>` 与 `i8` 相加：

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);
let sum = x + y;
```

> 当在 Rust 中拥有一个像 `i8` 这样类型的值时，编译器确保它总是有一个有效的值。我们可以自信使用而无需做空值检查。只有当使用 `Option<i8>`（或者任何用到的类型）的时候需要担心可能没有值，而编译器会确保我们在使用值之前处理了为空的情况。
>
> 在对 `Option<T>` 进行 `T` 的运算之前必须将其转换为 `T`。通常这能帮助我们捕获到空值最常见的问题之一：假设某值不为空但实际上为空的情况。
>
> 为了拥有一个可能为空的值，你必须要显式的将其放入对应类型的 `Option<T>` 中。接着，当使用这个值时，必须明确的处理值为空的情况。只要一个值不是 `Option<T>` 类型，你就 **可以** 安全的认定它的值不为空。**也就是从语法的层面上强制程序员事先对可能出现空值的地方进行处理。**
