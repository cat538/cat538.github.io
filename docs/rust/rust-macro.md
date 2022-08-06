---
title: Rust-macro
cover: 'https://github.com/cat538/images-auto/raw/main/img/fire.png'
date: 2022-05-30 23:05:52
tags: ['rust']
categories:
mathjax: 
---



## 声明宏(Declarative Macro)

使用 `macro_rules!` 进行声明，是rust中最常用的宏。它允许我们编写一些类似 rust `match` 表达式的代码，接收一个表达式，与表达式的结果进行模式匹配，然后根据模式匹配执行相关代码。

下面是一个简化的`vec!`宏。标准库中实际定义的 `vec!` 包括预分配适当量的内存的代码。这部分为代码优化，为了让示例简化，此处并没有包含在内。

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

`#[macro_export]` 注解表明只要导入了定义这个宏的crate，该宏就应该是可用的。 如果没有该注解，这个宏不能被引入作用域。



## 过程宏(Procedure Macro)

### 派生宏(Derive macro)

用于`struct`、`enum`、`union`类型，可为其实现函数或`trait`



### 属性宏(Attribute macro)

用在结构体、字段、函数等地方，为其指定属性等功能。如标准库中的`#[inline]`、`#[derive(...)]`等都是属性宏

[rust - How do I use conditional compilation with `cfg` and Cargo? - Stack Overflow](https://stackoverflow.com/questions/27632660/how-do-i-use-conditional-compilation-with-cfg-and-cargo)



### 函数式宏(Function-like macro)

用法与普通的规则宏类似，但功能更加强大，可实现任意语法树层面的转换功能。



## 属性

rust中的条件编译通过**属性(Attributes)**来实现。属性是根据名称、约定、语言和编译器版本解释的通用、free-form的元数据。

- 内部属性(*Inner attributes*)：形式为`#![]`例如`#![cfg(not(test))]`
- 外部属性(*Outer attributes*)：形式为`#[]`例如`#[test]` 把一个函数标记为测试function

属性可以分为如下几类：

1. 内置属性(Built-in attributes)

2. 属性宏(Macro attributes)

3. 派生宏的辅助属性(Derive macro helper attributes)

   

4. 工具属性(Tool attributes)

   这种属性格式：第一个字段是外部工具的名称，后续的一个或多个字段由工具决定。例如：

   ```rust
   // Tells the rustfmt tool to not format the following element.
   #[rustfmt::skip]
   struct S {}
   // Controls the "cyclomatic complexity" threshold for the clippy tool.
   #[clippy::cyclomatic_complexity = "100"]
   pub fn f() {}
   ```

   其中`rustfmt`和`clippy`是两个官方提供的工具`rust-clippy`和`rust-fmt`，前者用于检查代码：

   > 在编程实践中，将有助于将代码写的容易维护，质量合乎一定规范的做法，称为Linting，在Rust中，相应的工具是`clippy`。

   后者用于格式化代码

