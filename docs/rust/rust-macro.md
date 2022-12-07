# Rust-macro

使用 `macro_rules!` 的 声明（Declarative）宏，和三种 过程（Procedural）宏：

- 自定义`#[derive]`宏在结构体和枚举上指定通过 derive 属性添加的代码
- 类属性宏（Attribute-like macro）定义可用于任意项的自定义属性
- 类函数宏看起来像函数不过作用于作为参数传递的 token

> 宏和函数区别？
> - 宏是一种为写其他代码而写代码的方式，即所谓的 元编程（metaprogramming）。自定义`#[derive]`宏，例如生成各种`trait`的实现。
> 
> - 宏能够接受可变数量的参数(`println!`)。而且，宏可以在编译器翻译代码前展开，例如，宏可以在一个给定类型上实现 trait 。而函数则不行，因为函数是在运行时被调用，同时 trait 需要在编译时实现。
>
> 宏和函数的最后一个重要的区别是：在一个文件里调用宏**之前**必须定义它，或将其引入作用域，而函数则可以在任何地方定义和调用。


## 声明宏(Declarative Macro)

使用 `macro_rules!` 进行声明，是rust中最常用的宏。它允许我们编写一些类似 rust `match` 表达式的代码，接收一个表达式，与表达式的结果进行模式匹配，然后根据模式匹配执行相关代码。

下面是一个简化的`vec!`宏：

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

1. `#[macro_export]` 注解表明只要导入了定义这个宏的crate，该宏就应该是可用的。 如果没有该注解，这个宏不能被引入作用域。
2. 接着使用 macro_rules! 和宏名称开始宏定义，且所定义的宏并 不带 感叹号。名字后跟大括号表示宏定义体，在该例中宏名称是 vec 。
3. 


## 过程宏(Procedure Macro)
有三种类型的过程宏（自定义派生（derive），类属性和类函数），不过它们的工作方式都类似。

过程宏**接收Rust代码作为输入**，在这些代码上进行操作，然后**产生另一些代码作为输出**，而非像声明式宏那样匹配对应模式然后以另一部分代码替换当前代码。

### 自定义派生宏(Derive macro)

用于`struct`、`enum`、`union`类型，可为其实现函数或`trait`。



### 类属性宏(Attribute-like macro)

类属性宏与自定义派生宏相似，不同于为`derive`属性生成代码，它们允许你创建新的attribute。`derive` 只能用于结构体和枚举；属性还可以用于其它的，比如函数。

作为一个使用类属性宏的例子，可以创建一个名为`route`的属性用于标注web框架中路由的函数：
```rust
#[route(GET, "/")]
fn index() {
```
`#[route]`属性将由框架本身定义为一个过程宏。其宏定义的函数签名看起来像这样：

```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
```
### 类函数宏(Function-like macro)

用法与普通的规则宏类似，但功能更加强大，可实现任意语法树层面的转换功能。
